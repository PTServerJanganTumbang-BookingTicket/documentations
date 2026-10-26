# ADR-005: Keputusan Arsitektur Final — Server-Sent Events + ECS Fargate

**Status**: Accepted — menunggu keputusan pemilik tender (Bagian 17 #1–3)
**Tanggal**: 24 Oktober 2026
**Decider**: SJT Team
**Supersedes**: ADR-003, ADR-004
**Dokumen Referensi**: `requirement/requirements.md` v1.0 (Finalized), Tender Rusuh Booking Ticket.pdf, ADR-001, ADR-002

---

## Konteks

Rusuh Booking Tiket menjual tiket konser di bawah beban *flash crowd*: permintaan jauh
melampaui stok, dan tantangan utamanya bukan fitur melainkan menjaga pengalaman pengguna
tetap stabil dan adil ketika ratusan ribu orang tiba bersamaan.

ADR-001 menetapkan teknologi antrian (Redis counter model) dan ADR-002 menetapkan mekanisme
penentuan posisi (`INCR` + `now_serving`). ADR-003 memilih SSE sebagai transport dan ADR-004
memilih ECS Fargate sebagai compute.

Dokumen ini adalah **arsitektur of record** untuk build dan untuk jawaban tender. Ia memenuhi
seluruh FR, NFR dan BR dalam `requirements.md` v1.0; pemetaan lengkapnya ada di Bagian 14.

---

## 1. Ringkasan Keputusan

| Aspek | Keputusan |
|---|---|
| Transport real-time | **Server-Sent Events (SSE)** — web dan mobile |
| Compute SSE | **ECS Fargate** (`rusuh-sse-server`) |
| Compute admission + expiry + rekonsiliasi | **ECS Fargate** (`rusuh-admission-ctl`, 1 task, loop 500ms) |
| Compute REST API | **Lambda + API Gateway** |
| Autentikasi stream | Cookie `HttpOnly` (web) / header `Authorization` (mobile) |
| Verifikasi NIK | **Dukcapil API**, NIK saja. **Gagal apa pun sebabnya → tidak bisa bayar** |
| Dukcapil tidak tersedia | **Fail closed** + pause admission. Dukcapil adalah SPOF yang diterima |
| Pembayaran | Charge sinkron; efek samping **inline best-effort + outbox fallback** |
| Konfirmasi pembayaran | **`POST /payments/webhook`** dari gateway |
| Email | **Outbox → SQS → SES**, satu email per pemegang NIK, kegagalan tidak membatalkan transaksi |
| Sumber kebenaran deadline | **Aurora** (transaksional); ZSET hanya penjadwal |
| Anchor TTL | **Serial**: booking = `slot_claimed_at` +5 menit; pembayaran = `held_at` +10 menit (BR-03, BR-05) |
| Proteksi dependensi eksternal | Rate limiter + circuit breaker terdistribusi (Redis) |
| WAF | **Tidak dipakai di POC** — konsekuensinya dicatat di Bagian 16 |
| Push notification front-end | **Tidak dibangun** fase ini |
| Target kapasitas | **300.000 concurrent** (NFR-03 meminta 200.000+) |
| Dasar pertimbangan utama | **NFR-06** — p95 < 1 detik |

---

## 2. NFR-06 Sebagai Pertimbangan Utama

> **NFR-06**: Pembaruan posisi antrian dan halaman utama memiliki latensi rendah
> (p95 < 1 detik) di bawah beban tinggi — **semua posisi**.

Tim membaca ini sebagai klausa *freshness* dan memenuhinya untuk semua posisi. Versi
sebelumnya dari `requirements.md` membatasinya pada *"posisi ≤100"*, sebuah angka yang
berasal dari tier-1 desain polling dan tidak ada di tender asli; batas itu **sudah dihapus**
agar requirement kembali sesuai dengan tender. Arsitektur ini dirancang terhadap pembacaan
yang lebih ketat sejak awal, sehingga koreksi tersebut tidak mengubah apa pun di sini.

| Opsi | NFR-06 | Alasan tidak dipilih |
|---|---|---|
| A — WebSocket | ✅ Pass | **Kompleksitas.** 8 komponen divergen vs 2: connection registry, fan-out kustom, cleanup 410, keepalive, reconnect manual, REST fallback. Bidirectional tidak terpakai. Lebih sering diblokir proxy dan carrier NAT |
| B — Short Polling | ✅ Pass | **Biaya perangkat klien.** Sub-detik untuk seluruh crowd menuntut 1 request/detik selama 30–60 menit. Pada konfigurasi hemat baterai freshness tidak terpenuhi |
| C — AppSync | ✅ Pass | **Biaya dan protokol.** ~2× biaya per pesan; menambah GraphQL; tetap WebSocket di bawahnya |
| **D — SSE + ECS** | ✅ **Pass** | **Dipilih** |

**Mengapa D.** Keempat opsi memenuhi p95 < 1 detik; yang membedakan adalah ongkosnya. Dari
keempatnya, **D satu-satunya yang memenuhi freshness tanpa membebani perangkat klien**
(berbeda dari B, yang menuntut satu request per detik selama puluhan menit), **tanpa
connection registry dan fan-out kustom** (berbeda dari A), dan **tanpa menambah protokol
baru** (berbeda dari C). HTTP `GET` biasa juga lolos proxy korporat, carrier NAT, dan
WebView — tiga tempat WebSocket paling sering gagal.

Konsekuensinya compute tidak lagi serverless. Itu diterima sebagai harga keputusan ini dan
dikelola lewat Bagian 11.

---

## 3. Mengapa ECS, Bukan Lambda / App Runner

| Kandidat | Constraint | Dampak |
|---|---|---|
| API Gateway HTTP | Timeout 29 detik (hard) | SSE putus tiap 29 detik |
| Lambda | Max 15 menit, billing per-ms aktif | Tidak cukup untuk antrian 30–60 menit |
| App Runner | `request_timeout` maks 120 detik | **Didiskualifikasi** |
| **EventBridge** | **Minimum 1 menit** | Menyentuh tiga pekerjaan sub-detik: admission, expiry sweep, rekonsiliasi → satu loop ECS |

---

## 4. Arsitektur Final

```
  ┌──────────────────────────────┐              ┌──────────────────────────────┐
  │         Web Browser          │              │         Mobile App            │
  │  EventSource(url,{withCreds})│              │  react-native-sse + header   │
  │  Cookie HttpOnly             │              │  Token TIDAK di URL          │
  │                              │              │                              │
  │  HANDLER KLIEN               │              │  idem                        │
  │   on data    → posisi =      │              │                              │
  │     ticket_no − nowServing;  │              │  on resume: reconnect + EID  │
  │     banner pause bila paused │              │                              │
  │     serverTime → sinkron jam │              │  push notif: TIDAK DIBANGUN  │
  │   on grant   → GET /{ev}/me  │              │                              │
  │   on status  → {stale} →     │              │                              │
  │     indikator "menyambung…"  │              │                              │
  │   on sold-out →              │              │                              │
  │     tampilkan layar terminal │              │                              │
  │     **es.close()**  ← WAJIB  │              │                              │
  │     tanpa ini EventSource    │              │                              │
  │     reconnect otomatis       │              │                              │
  │   on error   → biarkan retry │              │                              │
  │     bawaan (kecuali 410)     │              │                              │
  │  Payment page: poll          │              │                              │
  │    GET /bookings/{id} 2s      │              │                             │
  │  i18n: Accept-Language       │              │  idem                   FR-32 │
  └──────────────┬───────────────┘              └──────────────┬───────────────┘
                 └─────────────────────────┬────────────────────┘
               ┌─────────────────────────┴────────────────────────────┐
               ▼                                                       ▼
  ┌──────────────────────────────────┐      ┌──────────────────────────────────┐
  │  ALB — events.rusuhtiket.com     │      │  API Gateway — api.rusuhtiket.com│
  │  HTTP/2 · idle 3600s             │      │ GET  /events                FR-01│
  │  deregistration delay 60s        │      │ GET  /events/{ev}           FR-11│
  │  sticky: OFF · health /health    │      │ GET  /events/{ev}/tickets   FR-12│
  │  slow start 30s                  │      │ ── /events/{ev}/queue ──         │
  │  CORS Allow-Credentials: true    │      │   POST …/join               FR-04│
  │  ⚠ ramp ~5.000 conn/s di T=0     │      │   GET  …/stream             FR-07│
  │                                  │      │   GET  …/me                 BR-09│
  │  RATE LIMIT (pengganti WAF):     │      │   GET  …/serving         fallback│
  │   token bucket Redis per IP      │      │ POST /nik/validate         FR-15a│
  │   rl:join:{ip}  20 / 5 menit     │      │ POST /bookings              FR-16│
  │   rl:sse:{ip}   5 conn / detik   │      │ GET  /bookings/{id}         FR-17│
  │                                  │      │ GET  /bookings/{id}/tickets FR-27│
  │                                  │      │ GET  /payments/{id}/methods FR-20│
  │                                  │      │ POST /payments/{id}         FR-20│
  │                                  │      │ POST /payments/{id}/retry        │
  │                                  │      │ GET  /payments/{id}/status  FR-21│
  │                                  │      │ POST /payments/webhook      FR-25│
  │                                  │      │                                  │
  │                                  │      │                                  │
  │                                  │      │                                  │
  │                                  │      │ /payments/webhook:               │
  │                                  │      │  RESOURCE POLICY allowlist IP    │
  │                                  │      │  gateway + verifikasi HMAC       │
  │                                  │      │  (dulu rule WAF)                 │
  └───────────────┬──────────────────┘      └──────────────────────────────────┘
                  │
  ┌───────────────▼──────────────────────────────────────────────────────────┐
  │  ECS Fargate: rusuh-sse-server    300.000 koneksi · max_tasks 60         │
  │  2 vCPU / 4 GB per task · N = 300.000 ÷ kapasitas TERUKUR (Bagian 11)    │
  │                                                                           │
  │   connections: Map<eventId, Set<Response>> · byToken: Map<token, Response>│
  │                                                                           │
  │   GET /events/{ev}/queue/stream                                          │
  │     0. GET queue:{ev}:state                                              │
  │        bila closed_sold_out → HTTP 410 Gone, JANGAN buka stream          │
  │        ← status non-200 membuat EventSource berhenti permanen,           │
  │          mencegah badai reconnect dari 300.000 klien setelah sold out    │
  │     1. token dari cookie/header → HGET token:{uuid}                      │
  │     2. headers: text/event-stream · no-cache · X-Accel-Buffering: no     │
  │     3. SNAPSHOT AWAL: { nowServing, state, ratePerMin, paused,           │
  │        serverTime } + id    ← FR-10: posisi utuh segera setelah refresh  │
  │     4. `: keepalive` 15 detik · close → hapus dari KEDUA map             │
  │                                                                           │
  │   HANDLER PER CHANNEL                                                    │
  │     sse-broadcast → fan-out ter-chunk; payload membawa nowServing,       │
  │                     state, ratePerMin, paused, serverTime                │
  │     sse-grant     → byToken.get(token)?.write("event: grant\ndata:{}")   │
  │     sse-sold-out  → fan-out "event: sold-out", lalu res.end().           │
  │                     Klien WAJIB es.close(); klien lama yang tetap        │
  │                     reconnect menerima 410 pada langkah 0 dan berhenti.  │
  │     Redis putus   → "event: status" {"stale":true}                       │
  │     Redis kembali → GET now_serving → re-broadcast + {"stale":false}     │
  │                                                                           │
  │   SIGTERM → retry berjitter 1–15 detik + event reconnect (Bagian 11)     │
  │             ← di sini reconnect otomatis justru DIINGINKAN               │
  └────────────────────────────────┬─────────────────────────────────────────┘
  ┌────────────────────────────────▼─────────────────────────────────────────┐
  │  ECS Fargate: rusuh-admission-ctl   desired_count 1 · 0.5 vCPU · 500ms   │
  │   0. paused := (breaker payment gateway == OPEN)                        │
  │                OR (breaker Dukcapil == OPEN)        ← BARU, Bagian 8     │
  │      PUBLISH sse-broadcast bila nowServing berubah, ATAU paused berubah,│
  │      ATAU 10 detik berlalu (heartbeat) ← resume otomatis                │
  │   1. SWEEP   ZRANGEBYSCORE expiries:{ev} 0 now  → VERIFIKASI ke Aurora  │
  │              sebelum meng-expire (Bagian 7)                             │
  │   2. ADVANCE bila paused → LEWATI SELURUH LANGKAH INI                   │
  │              (tanpa ini, banner pause tampil sementara slot             │
  │               terus dibagikan — pelanggaran NFR-02)                     │
  │              EVALSHA advance_now_serving.lua                            │
  │              → batch INSERT bookings(status='GRANTED', granted_at=now())│
  │              → ZADD expiries:{ev} (granted_at + GRACE) booking_id       │
  │              → gagal INSERT setelah retry: DECRBY slots:open,           │
  │                SADD readmit, JANGAN publish grant (Bagian 6)            │
  │   3. PUBLISH sse-broadcast / sse-grant / sse-sold-out                   │
  │   4. REKONSILIASI tiap tick ke-20 (≈10 detik) — Bagian 6                │
  └────────────────────────────────┬─────────────────────────────────────────┘
   ┌───────────────────────────────▼──────────────────────────────────────────┐
   │  REDIS — FAST PATH · PUB/SUB · RATE LIMIT · BREAKER · LEASE              │
   │  QUEUE    queue:{ev}:issued · now_serving · state · slots:open:{ev}      │
   │           token:{uuid} (ticket_no, event_id, lapses, account_id?)        │
   │           by_ticket (ZSET score=ticket_no) · readmit:{ev}                │
   │           grant:{token} EX GRANT_TOKEN_TTL_SECONDS (1200)                │
   │  BOOKING  stock:{ev}:{cat} · nik:used:{ev}:{hmac} NX                     │
   │           nik:valid:{hmac} EX 86400   -- cache verifikasi Dukcapil       │
   │  QUEUECAP queue:{ev}:capacity · depth = issued − now_serving             │
   │           expiries:{ev} (ZSET) — PENJADWAL, bukan sumber kebenaran       │
   │  DEPEND   rl:{service} · breaker:{service}:state/failures                │
   │  INGRESS  rl:join:{ip} · rl:sse:{ip}        ← pengganti WAF              │
   │  OUTBOX   processed:{outbox_id}  LEASE 30s → "done" EX 86400             │
   │  PUB/SUB  sse-broadcast · sse-grant · sse-sold-out                       │
   │  ElastiCache cluster mode · Multi-AZ · r6g.large × 3 · working set ≈80 MB│
   └────────────────────────────────┬─────────────────────────────────────────┘
   ┌────────────────────────────────▼─────────────────────────────────────────┐
   │  AURORA POSTGRESQL — SYSTEM OF RECORD · writer + 2 readers · PITR        │
   │                                                                          │
   │   bookings(                                                              │
   │     id, event_id, queue_token, account_id NULL, contact_email NULL,      │
   │     status, granted_at, slot_claimed_at, held_at,                        │
   │     booking_expires_at, payment_expires_at,   ← OTORITATIF; masing-      │
   │     payment_ref, payment_attempts, payment_method,                       │
   │                                                  masing ditulis saat     │
   │     terms_version, terms_accepted_at)            CLAIM dan HELD          │
   │                                                                          │
   │   -- contact_email nullable di DDL karena baris GRANTED belum punya      │
   │   -- email. Penegakannya bersyarat status:                               │
   │   ALTER TABLE bookings ADD CONSTRAINT contact_email_required             │
   │     CHECK (status IN ('GRANTED','LAPSED') OR contact_email IS NOT NULL); │
   │   ALTER TABLE bookings ADD CONSTRAINT terms_required                     │
   │     CHECK (status IN ('GRANTED','SELECTING','LAPSED')                    │
   │            OR terms_accepted_at IS NOT NULL);                            │
   │                                                                          │
   │   -- status: GRANTED SELECTING HELD AWAITING_PAYMENT                     │
   │   --         PENDING_CANCELLATION CONFIRMED EXPIRED LAPSED ABANDONED     │
   │                                                                          │
   │   -- SATU BARIS PER KURSI (BR-02: 1 NIK = 1 seat)                        │
   │   -- Nomor bangku tidak diimplementasikan di POC                         │
   │   booking_items(id, booking_id, category, unit_price,                    │
   │                 nik_hmac, holder_email)                         FR-26    │
   │   UNIQUE (booking_id, nik_hmac)   -- cegah NIK ganda DALAM satu booking  │
   │                                                                          │
   │   -- PENEGAK BR-02 LINTAS BOOKING (1 NIK = 1 kursi per event):           │
   │   nik_claims(event_id, nik_hmac, booking_id, claimed_at)                 │
   │   UNIQUE (event_id, nik_hmac)     -- invariant basis data, bukan Redis   │
   │                                                                          │
   │   payment_events(id, booking_id, gateway_event_id UNIQUE, type,          │
   │                  raw, rcvd_at)   ← kunci milik GATEWAY, bukan kita       │
   │   tickets(id, booking_item_id, seat_category,                            │
   │           nik_hmac, qr_payload, issued_at)                        FR-27  │
   │   UNIQUE (booking_item_id)   ← cegah QR ganda saat outbox retry          │
   │   outbox(id, aggregate_id, type, payload jsonb, published_at NULL)       │
   │   -- AKUN, NIK TERDAFTAR, DAN NIK TERSIMPAN                              │
   │   accounts(id, email, email_verified_at, verify_token,                   │
   │            nik_hmac, nik_verified_at, dukcapil_ref)         FR-31        │
   │   UNIQUE (nik_hmac)   -- satu NIK = satu akun, GLOBAL                    │
   │   -- nik_verified_at IS NULL = akun BELUM boleh membeli. Tidak ada       │
   │   -- kolom status terpisah: satu sumber kebenaran, tanpa enum yang       │
   │   -- bisa menyimpang dari kenyataan.                                     │
   │                                                                          │
   │   saved_niks(id, account_id, nik_hmac, label,                            │
   │              verified_at, dukcapil_ref)   -- Daftar_NIK_Teman            │
   │   UNIQUE (account_id, nik_hmac)                                          │
   │   CHECK  (verified_at IS NOT NULL)                                       │
   │   -- BARIS INI TIDAK DAPAT ADA TANPA BUKTI VERIFIKASI. Tidak ada         │
   │   -- status PENDING: entri yang belum terverifikasi akan dipakai         │
   │   -- checkout untuk MELEWATI Dukcapil, jadi ia tidak boleh tersimpan.    │
   │                                                                          │
   │   consent_records(id, account_id, policy_version, consented_at)          │
   │   -- APPEND-ONLY: tanpa UPDATE, tanpa DELETE (UU PDP)                    │
   └───────────────────────────────┬──────────────────────────────────────────┘
     ┌─────────────┬───────────────┼───────────────┬────────────────────────┐
     ▼             ▼               ▼               ▼                        ▼
┌──────────┐ ┌───────────┐ ┌────────────┐ ┌──────────────┐ ┌──────────────────┐
│ queue-   │ │ booking-  │ │ nik-verify │ │ payment-api  │ │ outbox-publisher │
│ service  │ │ service   │ │            │ │              │ │                  │
│ /events  │ │ /booking/ │ │ validasi   │ │ /payments/…  │ │ tiap 1 detik     │
│ /join    │ │  create   │ │ rate limit │ │  rate limit  │ │ FOR UPDATE SKIP  │
│ /me      │ │ GUARD:    │ │ + breaker  │ │  + breaker   │ │  LOCKED          │
│ /serving │ │  qty<=3   │ │ FAIL CLOSED│ │  idempotency │ │ lease guard      │
│          │ │  email +  │ │ INVALID →  │ │  TANPA retry │ │ FALLBACK saja    │
│ Set-     │ │  T&C +    │ │  BLOKIR    │ │  buta        │ │ ticket.issue →SQS│
│ Cookie   │ │  NIK ter- │ │ DOWN →     │ │ TIDAK menulis│ │  (per pemegang)  │
│ bypass   │ │  verifi-  │ │  BLOKIR    │ │  deadline    │ │ booking.expired  │
│ antrian  │ │  kasi     │ │ → SETEX    │ │ webhook      │ │ account.verify   │
│ kosong   │ │ EVALSHA   │ │  nik:valid │ │  verify sig  │ │                  │
│  FR-05   │ │ hold_     │ │            │ │  → TXN       │ │                  │
│          │ │ booking   │ │            │ │  → INLINE    │ │                  │
└──────────┘ └───────────┘ └─────┬──────┘ └──────┬───────┘ └────────┬─────────┘
                                 ▼               ▼                  ▼
                        ┌────────────────┐ ┌──────────────┐ ┌──────────────┐
                        │ Dukcapil API   │ │ Payment      │ │ SQS → SES    │
                        │ POC: mock      │ │ Gateway      │ │ → DLQ        │
                        │ SPOF — Bag. 16 │ │ PCI-DSS      │ │              │
                        └────────────────┘ └──────────────┘ └──────────────┘

  OBSERVABILITY — EventBridge 1 min → metrics-scraper → EMF RusuhBooking/Queue
    QueueDepth · ActiveSSEConnections · SlotReleaseToAdmit p50/p95/p99
    ConversionRate (FR-34) · ExpiryRate per tahap (FR-35) · StockLevel · OpenSlots
    SlotsOpenDrift · StockDrift · HoldCompensated · GrantInsertFailures · ExpiryRescored
    SoldOutStreamRejections (410) · JoinRateLimited · SSERateLimited
    ReadmitSetSize · SSEWriteBackpressure · OutboxLag · OutboxInlineHitRate
    DukcapilLatency p95 · DukcapilCircuitState · DukcapilThrottled
    NikRejectedRate · DukcapilBlockedSales · PaymentGatewayLatency p95
    PGCircuitState · PGThrottled · AdmissionPausedSeconds · WebhookLag · EmailDLQDepth
```

---

## 5. Halaman Utama dan Jalur Masuk

| Requirement | Implementasi |
|---|---|
| FR-01 daftar event | `GET /events` — Lambda + cache CloudFront 30 detik. Halaman utama dapat di-cache; ia tidak punya konsep freshness per-customer |
| FR-02 event aktif | `status = 'available' AND stock > 0` → `clickable: true` |
| FR-03 event sold out | `GET queue:{ev}:state = closed_sold_out` ATAU `stock = 0` → `clickable: false`, label status |
| FR-04 ada antrian → Waiting Room | `POST /events/{ev}/queue/join` mengembalikan `{ticket_no, nowServing, streamUrl}` |
| **FR-05 tidak ada antrian → langsung Detail** | `POST /events/{ev}/queue/join` mengecek `issued − now_serving == 0 AND slots:open < MAX AND stock > 0`. Bila benar, Lua memberi grant pada permintaan yang sama dan respons berisi `{granted: true, accessToken}` — klien langsung ke Detail Ticket Page tanpa pernah membuka stream |

Jalur bypass memakai Lua yang sama (`advance_now_serving.lua` dengan `deficit = 1`), sehingga
tidak ada jalur admission kedua yang bisa menyimpang dari yang utama.

---

## 6. Satu Jalur Admission, dan Integritas `slots:open`

Saat `POST /events/{ev}/queue/join`: `HSET token:{uuid}` **dan** `ZADD queue:{ev}:by_ticket {ticket_no} {uuid}`.

```lua
-- advance_now_serving.lua   (SCRIPT LOAD sekali, dipanggil via EVALSHA)
-- KEYS[1]=now_serving KEYS[2]=slots:open KEYS[3]=state KEYS[4]=readmit KEYS[5]=by_ticket
-- ARGV[1]=MAX_CONCURRENT_SLOTS  ARGV[2]=remaining_stock
-- ARGV[2] adalah SISA STOK TOTAL seluruh kategori, bukan per kategori: admission-ctl
-- tidak tahu kategori apa yang akan dipilih customer yang belum masuk.

local stock = tonumber(ARGV[2])
if stock <= 0 then
  redis.call('SET', KEYS[3], 'closed_sold_out')
  return {0, 0, 'SOLD_OUT', {}}
end

local open    = tonumber(redis.call('GET', KEYS[2]) or 0)
local target  = math.min(tonumber(ARGV[1]), stock)
local deficit = target - open
if deficit <= 0 then return {0, 0, 'NOOP', {}} end

local tokens = {}
local readmitted = redis.call('SPOP', KEYS[4], deficit)
for i = 1, #readmitted do tokens[#tokens + 1] = readmitted[i] end
if #readmitted > 0 then
  redis.call('INCRBY', KEYS[2], #readmitted)
  deficit = deficit - #readmitted
end

local from, to = 0, 0
if deficit > 0 then
  to   = redis.call('INCRBY', KEYS[1], deficit)
  from = to - deficit + 1
  local fresh = redis.call('ZRANGEBYSCORE', KEYS[5], from, to)
  redis.call('ZREMRANGEBYSCORE', KEYS[5], from, to)
  for i = 1, #fresh do tokens[#tokens + 1] = fresh[i] end
  -- slots:open naik sebanyak token yang BENAR-BENAR di-grant, bukan sebanyak deficit
  if #fresh > 0 then redis.call('INCRBY', KEYS[2], #fresh) end
end

return {from, to, 'ADMITTED', tokens}
```

Formula ini setara dengan `canEnter = (freeSlots > 0) AND (availableTickets > 0)` di
`requirements.md` Bagian 6, dinyatakan sebagai *deficit* agar satu tick dapat memberi banyak slot
sekaligus secara atomik.

### Baris `bookings` dibuat saat GRANTED

Prasyarat rekonsiliasi (tanpa baris ini `COUNT(*)` tidak melihat slot granted-unclaimed,
sehingga rekonsiliasi justru menyebabkan over-admission) dan penutup celah rebuild.

**Kompensasi bila batch INSERT gagal:** retry hingga `GRANT_INSERT_MAX_RETRIES`; bila tetap
gagal → `DECRBY slots:open`, `SADD readmit`, **jangan publish** `sse-grant`, emit
`GrantInsertFailures`. Urutan wajib **INSERT dulu, PUBLISH kemudian** — mem-publish lebih
dahulu berarti memberi tahu customer gilirannya tiba untuk slot yang kemudian dikompensasi,
pelanggaran NFR-02 "Jangan PHP" paling langsung.

### Kepemilikan `slots:open`

> **Berkurang HANYA pada transisi TERMINAL.** Lua hanya `INCR`. `DECR` oleh pemanggil,
> **sebelum** `EVALSHA`, hanya bila `UPDATE` Aurora mengenai tepat 1 baris.

| Transisi | Komponen | DECR? |
|---|---|---|
| — → GRANTED | *(Lua)* + INSERT baris | **INCR +1** |
| GRANTED → SELECTING (claim) | `booking-service` | ❌ Slot yang sama |
| SELECTING → HELD (tekan Pay) | `booking-service` | ❌ Slot yang sama |
| HELD → AWAITING_PAYMENT | `payment-api` | ❌ Slot yang sama |
| AWAITING_PAYMENT → PENDING_CANCELLATION | `rusuh-admission-ctl` | ❌ Stok masih di-hold selama grace |
| PENDING_CANCELLATION → CONFIRMED | webhook | ✅ Terminal |
| PENDING_CANCELLATION → EXPIRED | `rusuh-admission-ctl` | ✅ Terminal + lepas stok + `DEL nik:used` |
| AWAITING_PAYMENT → CONFIRMED | **webhook inline**, fallback outbox (Bagian 9) | ✅ Terminal |
| apa pun → EXPIRED | `rusuh-admission-ctl` | ✅ Terminal + `DEL nik:used` |
| GRANTED → LAPSED | `rusuh-admission-ctl` | ✅ Terminal; token → `readmit` |
| apa pun → ABANDONED | `booking-service` | ✅ Terminal + `DEL nik:used` |

Double-DECR → `slots:open` negatif → `deficit = target − (−n)` → over-admission.

### Rekonsiliasi

```
Tiap tick ke-20 (≈10 detik):
  C = SELECT COUNT(*) FROM bookings WHERE event_id = $1
       AND status IN ('GRANTED','SELECTING','HELD','AWAITING_PAYMENT')
      -- WRITER endpoint. Lag replika terbaca sebagai drift semu.
  R = GET slots:open:{ev}
  current_drift = R - C ;  emit SlotsOpenDrift

  if |current_drift| >= DRIFT_THRESHOLD and |last_drift| >= DRIFT_THRESHOLD:
      INCRBY slots:open:{ev} (-current_drift)
      -- current_drift, BUKAN last_drift (berumur ~10 detik, sudah basi;
      --   last_drift hanya konfirmasi bahwa drift persisten)
      -- INCRBY, BUKAN SET (SET menimpa grant antara pembacaan dan koreksi)
  last_drift = current_drift
```

### Rekonsiliasi stok

`slots:open` bukan satu-satunya hitungan Redis yang dapat menyimpang dari Aurora. `EVALSHA
hold_booking.lua` berjalan **di dalam** transaksi Aurora tetapi Redis tidak ikut transaksi itu:
bila Lua berhasil lalu `UPDATE`/`INSERT` gagal dan transaksi di-rollback, stok sudah berkurang
dan `nik:used` sudah terkunci sementara Aurora tetap `SELECTING`. Sweep kemudian meng-expire
baris itu lewat jalur `SELECTING → EXPIRED`, yang **tidak** mengembalikan stok — karena pada
jalur normal memang tidak ada stok yang diambil. Akibatnya tiket hilang dari peredaran dan NIK
terkunci selamanya.

Kompensasi langsung menangani kasus umum (Bagian 8); rekonsiliasi ini adalah jaring pengaman
bila kompensasi itu sendiri gagal:

```
Tiap tick ke-120 (≈60 detik), per kategori:
  E = SELECT capacity FROM event_categories WHERE id = $cat          -- kapasitas awal
    − SELECT COUNT(*) FROM booking_items bi JOIN bookings b ON b.id = bi.booking_id
        WHERE bi.category = $cat
          AND b.status IN ('HELD','AWAITING_PAYMENT','CONFIRMED')     -- WRITER
  R = GET stock:{ev}:{cat}
  current_drift = R - E ;  emit StockDrift

  if |current_drift| >= STOCK_DRIFT_THRESHOLD and |last_drift| >= STOCK_DRIFT_THRESHOLD:
      INCRBY stock:{ev}:{cat} (-current_drift)
      -- aturan yang sama dengan slots:open — INCRBY bukan SET, current_drift bukan
      -- last_drift, dan dua konfirmasi berturut-turut sebelum mengoreksi
  last_drift = current_drift
```

`nik:used` direkonsiliasi pada tick yang sama, terhadap `nik_claims`:

```
  A = SELECT nik_hmac FROM nik_claims WHERE event_id = $ev        -- WRITER
  untuk hmac dalam A yang TIDAK ada di Redis:
      SET nik:used:{ev}:{hmac} 1 EX NIK_USED_TTL_SECONDS          -- pulihkan
      emit NikClaimRestored
  -- Arah sebaliknya (ada di Redis, tidak di nik_claims) TIDAK dihapus otomatis:
  -- itu bisa jadi hold yang sedang berjalan dan belum COMMIT. Kunci itu dibatasi
  -- TTL-nya sendiri, dan `NikOrphanRate` dialarmkan bila jumlahnya tidak wajar.
```

Inilah alasan `nik_claims` ada. Tanpanya `nik:used` tidak punya pembanding dan kebocoran NIK
tidak terlihat di mana pun. `NIK_USED_TTL_SECONDS` tetap harus **melampaui akhir sale**:
Redis adalah jalur cepat yang menolak duplikat sebelum transaksi dimulai, dan bila ia
kedaluwarsa lebih awal setiap permintaan jatuh ke unique index Aurora — tetap benar, tetapi
customer baru tahu NIK-nya terpakai setelah menekan Pay, bukan saat mengetik.

**Interval 60 detik, bukan 10.** Query-nya lebih berat (join ke `booking_items` per kategori)
dan drift stok tidak menyebabkan over-admission seperti drift `slots:open` — ia hanya menahan
tiket dari peredaran, sehingga koreksi yang lebih lambat dapat diterima.

### Kapasitas antrian dan state yang terlihat customer

Antrian dibatasi `QUEUE_CAPACITY` (300.000). `POST /events/{ev}/queue/join` menolak dengan
`QUEUE_FULL` ketika `issued − now_serving >= QUEUE_CAPACITY`, **fail fast** — tanpa buffering
dan tanpa memberi nomor antrian yang tidak akan pernah terlayani.

State antrian per event tersedia di `GET /events` dan dipakai halaman utama:

| `queue_state` | Arti | Tombol |
|---|---|---|
| `OPEN` | Dapat bergabung | "Beli Tiket" aktif |
| `FULL` | `depth >= QUEUE_CAPACITY` | Nonaktif, "Antrian penuh" |
| `CLOSED` | `state = closed_sold_out` | Nonaktif, "Sold Out" |

Menampilkannya di halaman utama berarti customer melihat antrian penuh **sebelum** mengklik,
bukan sesudah. `GET /events` di-cache CDN 30 detik, sehingga `depth` yang ditampilkan adalah
perkiraan; penegaknya tetap pemeriksaan atomik saat join.

**Tidak ada `Session_TTL_Antrian`.** Yang melepas posisi hanyalah grace period setelah slot
diberikan: customer yang tidak masuk ke Detail Ticket Page dalam `GRACE_SECONDS` kehilangan
slotnya ke antrian berikutnya (maksimum dua kali re-admit), dan menutup tab tidak melepas
posisi. Memberi antrian batas waktu absolut tidak menambah proteksi di atas itu — ia hanya
menghukum customer yang menunggu lama karena antrian memang panjang.

### `readmit:{ev}` dan pause/resume

```
grace lapse:   n = HINCRBY token:{uuid} lapses 1
               if n <= 2 then SADD readmit:{ev} {uuid} else ZREM by_ticket end
sale ditutup:  EXPIRE readmit:{ev} 86400 ; EXPIRE by_ticket 86400
post-sale job: DEL readmit:{ev} by_ticket
```

> **Slot yang terisi dari `readmit` tidak memajukan `now_serving`.** Lua menguras `readmit`
> lebih dulu dan mengurangi `deficit` sebesar jumlah itu, sehingga `INCRBY now_serving` hanya
> berjalan untuk sisanya. Ini benar — token yang di-readmit sudah pernah mendapat gilirannya,
> ia tidak menyerobot — tetapi berarti customer yang menunggu kadang melihat slot terbebas
> tanpa posisinya bergerak. Perlu diketahui tim dukungan sebelum hari-H.

### Apa yang dihentikan `paused`, dan apa yang tetap jalan

`paused` bukan sekadar field untuk banner. Ia **melewati seluruh langkah ADVANCE** pada tick
admission-ctl (Bagian 4 langkah 2):

| Saat `paused = true` | Jalan? | Alasan |
|---|---|---|
| ADVANCE — `INCRBY now_serving` + grant baru | ❌ | tidak ada yang dapat menyelesaikan pembelian |
| Pengurasan `readmit` | ❌ | ia juga memasukkan orang ke funnel yang sama |
| SWEEP — expiry booking dan pembayaran | ✅ | stok yang tertahan harus tetap dilepas |
| Broadcast SSE + heartbeat | ✅ | customer harus melihat bannernya dan posisinya |
| REKONSILIASI `slots:open` | ✅ | drift tidak berhenti hanya karena admission berhenti |

Karena pengurasan `readmit` ada **di dalam** `advance_now_serving.lua`, gerbangnya diletakkan
di admission-ctl — tick melewatkan `EVALSHA` sepenuhnya — bukan sebagai `ARGV` baru.
Keputusan pause tidak sensitif terhadap balapan: sebuah tick menjalankan advance atau tidak.

> **SWEEP sengaja TIDAK di-pause.** `booking_expires_at` adalah timestamp absolut yang
> tersimpan; mem-pause sweep hanya menghentikan **penagihannya**, bukan jamnya. Saat pause
> berakhir, tenggat itu sudah lewat dan sweep tetap meng-expire mereka — lebih lambat, pada
> saat yang tidak dapat diduga customer, dan dalam satu gelombang besar. Selama itu
> `slots:open` tetap terisi booking mati sehingga admission tidak dapat dimulai bersih
> meskipun breaker sudah CLOSED.
>
> Perbaikan yang benar adalah **menggeser tenggatnya**, bukan berhenti menagih — masih
> menunggu keputusan pemilik tender, Bagian 17 #4.

Pause adalah **field pada payload broadcast**, bukan event tersendiri:

```json
{ "nowServing": 47600, "state": "OPEN", "ratePerMin": 318,
  "paused": false, "serverTime": "2026-10-24T04:22:31.507Z", "id": 47600 }
```

Publish bila `nowServing` berubah, **atau `paused` berubah**, atau 10 detik berlalu.
Resume otomatis tanpa kode deteksi transisi di klien; snapshot awal juga membawa `paused`.
`serverTime` di setiap payload memenuhi BR-10 dan sinkronisasi 30 detik di `requirements.md`
Bagian 5 **tanpa endpoint terpisah**. Tidak ada `GET /time`: setiap respons REST juga membawa
`server_time`, sehingga klien yang belum membuka stream sudah menerima offset jamnya dari
panggilan yang memang harus ia lakukan. Klien menghitung
`offset = server_time − waktu lokal` sekali, lalu merender semua countdown terhadap offset itu
— jam perangkat tidak pernah dipercaya langsung.

---

## 7. Expiry: Aurora Otoritatif, ZSET Penjadwal

### Dua TTL yang berurutan, bukan tumpang tindih

BR-03 memberi 5 menit di Detail Ticket Page. BR-05 memberi **10 menit sejak booking dibuat** —
dan "booking dibuat" adalah saat customer menekan tombol bayar: kursi dipilih, NIK diisi, stok
di-hold (FR-16). Sebelum itu tidak ada booking, hanya sesi pemilihan. Karena itu keduanya
**berurutan**, masing-masing dengan anchor sendiri:

```
booking_expires_at = slot_claimed_at + BOOKING_TTL_SECONDS   (300)   ← masuk Detail Ticket
payment_expires_at = held_at         + PAYMENT_TTL_SECONDS   (600)   ← tekan tombol bayar
```

Setiap customer karena itu mendapat **10 menit penuh** di halaman pembayaran, berapa pun lama
ia memilih kursi — sesuai wireframe tender.

**Perjalanan maksimum karena itu 17 menit, bukan 15**, dan umur `accessToken` harus
diturunkan darinya, bukan ditulis tetap:

```
grace antrian   120 detik   GRANT → claim
booking TTL     300 detik   claim → tekan Pay
payment TTL     600 detik   tekan Pay → batas bayar
payment grace    30 detik   batas bayar → pembatalan final
                            ──────────
                            1050 detik = 17,5 menit

GRANT_TOKEN_TTL_SECONDS = GRACE_SECONDS + BOOKING_TTL_SECONDS
                        + PAYMENT_TTL_SECONDS + PAYMENT_GRACE_SECONDS + margin = 1200
```

> Angka tetap `EX 900` berlaku pada model anchor-tunggal sebelumnya dan **tidak lagi**:
> customer yang memakai hampir seluruh grace dan seluruh TTL booking akan kehilangan
> token-nya dengan sisa dua menit jendela pembayaran yang masih sah — 401 saat polling
> status, mungkin sambil berdiri di ATM dengan transfer sudah terkirim. Menurunkannya dari
> ketiga konstanta membuat perubahan `PAYMENT_TTL_SECONDS` kelak tidak diam-diam
> memendekkannya lagi.

| Transisi | Kolom Aurora (di dalam transaksi) | ZSET (penjadwal) |
|---|---|---|
| GRANT | `granted_at = now()` | `ZADD expiries (granted_at + GRACE) booking_id` |
| CLAIM → SELECTING | `slot_claimed_at = now()`, `booking_expires_at = slot_claimed_at + 300` | `ZADD expiries (booking_expires_at)` |
| **TEKAN PAY → HELD** | `held_at = now()`, **`payment_expires_at = held_at + 600`** | `ZADD expiries (payment_expires_at)` |
| HELD → AWAITING_PAYMENT | *(tidak menulis deadline baru)* | *(skor tidak berubah)* |
| Terminal apa pun | `status = <terminal>` | `ZREM expiries booking_id` |

**TTL booking berhenti mengatur begitu status menjadi `HELD`.** Keduanya tersimpan di baris
yang sama, tetapi sweep memilih berdasarkan status — tidak pernah keduanya — sehingga tidak
ada expiry ganda dan tidak ada tumpang tindih. Deadline pembayaran ditulis **dalam transaksi
hold yang sama**, jadi tidak ada jendela di mana stok sudah terkunci tetapi deadline-nya belum
ada.

### Grace period pembayaran: 30 detik untuk callback, bukan untuk bayar

`payment_expires_at` tidak lagi langsung terminal. Booking melewati
**`PENDING_CANCELLATION`** selama `PAYMENT_GRACE_SECONDS` (30 detik), dan callback sukses
yang tiba di jendela itu **tetap mengonfirmasi** booking.

> **Grace menutup latensi callback, bukan pembayaran terlambat.** Customer yang mentransfer
> pada menit 9:59 menghasilkan callback yang mungkin baru tiba 10:00:15 — pembayaran itu
> sah dan seharusnya tidak hangus karena jaringan Midtrans. Customer yang baru mentransfer
> pada 10:00:20 **tidak** tercakup: `custom_expiry` yang dikirim ke Midtrans tetap
> `PAYMENT_TTL_SECONDS`, bukan TTL + grace, sehingga VA sudah tidak dapat dibayar saat
> grace berjalan.

Stok **tetap di-hold** selama grace: melepasnya lalu menerima callback sukses 10 detik
kemudian berarti menjual ulang kursi yang sudah dibayar. Konsekuensinya perjalanan maksimum
menjadi 17,5 menit, dan `GRANT_TOKEN_TTL_SECONDS` harus mencakupnya.

### Sweep memverifikasi sebelum meng-expire

ZSET berada di luar transaksi Aurora, sehingga selalu ada jendela crash di antaranya. Jendela
itu tidak menghasilkan entri yang *hilang*, melainkan entri dengan **deadline lama**: booking
yang sudah `AWAITING_PAYMENT` masih membawa skor TTL booking, lalu di-expire beberapa puluh
detik setelah customer menerima nomor VA — tepat ketika ia sedang membayar di ATM. Alih-alih
menambal tiap jendela satu per satu, sweep memperlakukan ZSET sebagai **petunjuk** dan Aurora
sebagai **kebenaran**:

```
untuk booking_id dalam ZRANGEBYSCORE expiries:{ev} 0 now:

  row = SELECT status, granted_at, booking_expires_at, payment_expires_at
          FROM bookings WHERE id = booking_id          -- WRITER

  deadline = CASE row.status
               WHEN 'GRANTED'          THEN row.granted_at + GRACE_SECONDS
               WHEN 'SELECTING'        THEN row.booking_expires_at
               WHEN 'HELD'             THEN row.payment_expires_at
               WHEN 'AWAITING_PAYMENT' THEN row.payment_expires_at
               WHEN 'PENDING_CANCELLATION'
                                       THEN row.payment_expires_at + PAYMENT_GRACE_SECONDS
               ELSE NULL                                -- sudah terminal
             END

  bila deadline IS NULL:
      ZREM expiries:{ev} booking_id ; lanjut            -- sudah terminal

  bila deadline > now():
      ZADD expiries:{ev} deadline booking_id            -- ZSET basi → RESCORE
      emit ExpiryRescored ; lanjut                      -- JANGAN expire

  -- deadline benar-benar terlewat:
  UPDATE bookings SET status = <terminal>
    WHERE id = booking_id AND status = row.status       -- guarded, CAS
  bila 0 baris → ZREM ; lanjut                          -- pihak lain mendahului

  switch row.status:
    GRANTED          → LAPSED  : DECR slots:open ; lapses+1 ; readmit (maks 2×)
    SELECTING        → EXPIRED : DECR slots:open        -- TTL booking (5 menit)
    HELD             → EXPIRED : DECR slots:open ; INCR stock
                                 DELETE FROM nik_claims WHERE booking_id = $1
                                 DEL nik:used per item
                                                        -- TTL pembayaran (10 menit)
    AWAITING_PAYMENT → PENDING_CANCELLATION :
                                 -- TIDAK melepas stok. Grace 30 detik untuk
                                 -- callback yang sedang di jalan.
                                 ZADD expiries (payment_expires_at + PAYMENT_GRACE)
                                 TIDAK DECR slots:open

    PENDING_CANCELLATION → EXPIRED :
                                 idem HELD ; INSERT outbox 'booking.expired'   FR-28
                                 POST /v2/{order_id}/expire  → Midtrans
                                 -- VA/QR berhenti dapat dibayar. custom_expiry saat
                                 -- charge sudah menutupnya; ini menutup selisih jam
                                 -- antara Midtrans dan kita. Gagal → best-effort,
                                 -- tidak membatalkan expiry; emit MidtransExpireFailed.
                                 callback terlambat → auto-refund (Bagian 9)

  ZREM expiries:{ev} booking_id
  EVALSHA advance_now_serving.lua ; PUBLISH
```

Konsekuensinya: **tidak ada booking yang di-expire lebih awal karena `ZADD` yang hilang.**
Entri basi dikoreksi sendiri pada sweep berikutnya, dan `ExpiryRescored` yang tidak nol
menandakan ada jendela crash yang sering terjadi — sinyal untuk diinvestigasi, bukan
kerusakan senyap.

**Biaya**: satu `SELECT` per entri yang jatuh tempo. Pada laju expiry puncak (~500/menit) itu
~8 query/detik terhadap writer — dapat diabaikan, dan hanya untuk entri yang benar-benar
jatuh tempo, bukan seluruh ZSET.

---

## 8. Booking, Stok, NIK, dan Dependensi Eksternal

**Stok di-hold saat tombol Pay ditekan** (FR-16).

> **Gagal hold TIDAK menghilangkan slot.** `NO_STOCK`, `NIK_TAKEN` atau `QTY_EXCEEDED` →
> customer tetap `SELECTING` dengan sisa TTL; UI menampilkan pesan spesifik; pemilihan ulang
> kategori atau perbaikan NIK diizinkan. (NFR-02)

```lua
-- hold_booking.lua
-- KEYS[1]=stock:{ev}:{category}  KEYS[2..n+1]=nik:used:{ev}:{hmac}
-- ARGV[1]=qty  ARGV[2]=MAX_TICKETS_PER_TRANSACTION  ARGV[3]=NIK_USED_TTL_SECONDS
local qty = tonumber(ARGV[1])
if qty < 1 or qty > tonumber(ARGV[2]) then return {0, 'QTY_EXCEEDED', -1} end  -- FR-13
if #KEYS - 1 ~= qty then return {0, 'NIK_COUNT_MISMATCH', -1} end              -- BR-02
local stock = tonumber(redis.call('GET', KEYS[1]) or 0)
if stock < qty then return {0, 'NO_STOCK', -1} end                             -- FR-14
local claimed = {}
for i = 2, #KEYS do
  if redis.call('SET', KEYS[i], 1, 'NX', 'EX', tonumber(ARGV[3])) then
    -- EX membatasi kebocoran bila transaksi Aurora rollback setelah titik ini:
    -- NIK terkunci paling lama selama sale, bukan selamanya (Bagian 6)
    claimed[#claimed + 1] = KEYS[i]
  else
    for j = 1, #claimed do redis.call('DEL', claimed[j]) end
    return {0, 'NIK_TAKEN', i - 2}        -- indeks NIK yang bentrok
  end
end
redis.call('DECRBY', KEYS[1], qty)
return {1, 'HELD', -1}
```

### Verifikasi NIK — fail closed

Verifikasi berjalan progresif saat customer mengetik (`on blur` per field), bukan saat
menekan Pay — latensi API pemerintah keluar dari jalur kritis, dan cache 24 jam menyerap
percobaan ulang. Dukcapil memvalidasi **NIK saja**; nama dan tanggal lahir tidak dikirim.

| Hasil Dukcapil | Perlakuan |
|---|---|
| **Valid** | `SETEX nik:valid:{hmac} 86400`; field ditandai ✓ |
| **Tidak valid** | **BLOKIR.** Tombol Pay tidak aktif; customer memperbaiki NIK di tempat. Emit `NikRejectedRate` |
| **Tidak dapat dihubungi** (timeout / 5xx / breaker OPEN) | **BLOKIR.** Tidak ada pembayaran yang dapat dilakukan. Emit `DukcapilBlockedSales` |

`booking-service` menolak `POST /bookings` kecuali **setiap** NIK dalam permintaan punya
**bukti verifikasi**, yaitu salah satu dari tiga sumber berikut:

| # | Sumber bukti | Dibuat saat |
|---|---|---|
| 1 | `accounts.nik_verified_at` milik pemegang token | registrasi |
| 2 | `saved_niks.verified_at` milik pemegang token | penambahan Daftar NIK Teman |
| 3 | `nik:valid:{hmac}` masih hidup (cache 24 jam) | verifikasi Dukcapil terakhir |

Ketiganya setara: masing-masing adalah jawaban "valid" dari Dukcapil yang pernah benar-benar
terjadi. Yang membedakan hanya di mana jawaban itu disimpan dan berapa lama ia bertahan.

> **Sumber 1 dan 2 tidak memerlukan Redis.** Keduanya baris Aurora, dan gate memeriksanya
> dalam query yang sama dengan pemeriksaan kepemilikan booking — bukan panggilan tambahan.
> Menuntut `nik:valid:{hmac}` untuk NIK yang sudah terdaftar atau tersimpan akan membatalkan
> seluruh manfaat fitur itu: NIK yang sah akan ditolak hanya karena cache 24 jam sudah
> kedaluwarsa.

Tidak ada jalur yang menerbitkan tiket dengan NIK yang belum pernah diverifikasi, sehingga
kolom `nik_unverified` dan verifikasi ulang asinkron dihapus.

**Konsekuensi yang diterima secara sadar: Dukcapil menjadi single point of failure.** Bila
Dukcapil tidak dapat dihubungi, penjualan berhenti — bukan melambat. Karena itu breaker
Dukcapil yang OPEN **mem-pause admission** (Bagian 4 langkah 0): membiarkan orang masuk ke Detail
Ticket Page yang tidak mungkin mereka selesaikan melanggar NFR-02. Customer yang sudah berada
di dalam melihat banner pause yang sama dengan gangguan payment gateway, dan TTL mereka tetap
berjalan — lihat Bagian 16 untuk risiko yang belum tertutup di sini.

> **Istilah: "breaker OPEN" berarti dependensinya SEDANG TIDAK DAPAT DIPAKAI**, bukan
> sebaliknya. Circuit breaker yang terbuka memutus aliran panggilan — sama seperti sakelar
> listrik yang terbuka memutus arus. Jadi "breaker Dukcapil OPEN" = Dukcapil gagal/timeout
> melewati ambang → admission di-pause. Breaker CLOSED = normal, panggilan diteruskan.
> Jangan membaca OPEN sebagai "API terbuka / tersedia".

**Kedua dependensi diperlakukan sama: pause penuh.** Antrian berhenti maju untuk semua
orang tanpa pengecualian, dan tidak ada seorang pun yang kehilangan posisinya.

### Menyimpan NIK: Dukcapil dulu, baris kemudian

`saved_niks` dan `accounts.nik_hmac` adalah **bukti verifikasi**, dan checkout memperlakukan
keduanya setara dengan cache Dukcapil yang masih hidup. Karena itu keduanya hanya boleh
lahir dari jawaban Dukcapil yang berhasil:

```
POST /accounts/me/saved-niks
  1. Validasi format 16 digit + normalisasi (Bagian 8, normalisasi dan pepper)
  2. Akun sudah punya consent_records?        tidak → 422 CONSENT_REQUIRED
  3. BEGIN
       SELECT id FROM accounts WHERE id = $me FOR UPDATE   -- serialisasi per akun
       SELECT count(*) FROM saved_niks WHERE account_id = $me
         >= MAX_SAVED_NIKS → ROLLBACK ; 409 MAX_SAVED_NIKS_REACHED

  4.   Panggil Dukcapil  (rate limit → breaker, sama seperti checkout)
         tidak valid        → ROLLBACK ; 422 — tidak ada baris yang dibuat
         tidak dapat dihubungi → ROLLBACK ; 503 — tidak ada baris yang dibuat

  5.   INSERT INTO saved_niks (account_id, nik_hmac, label,
                               verified_at, dukcapil_ref)
            VALUES ($me, $hmac, $label, now(), $ref)
     COMMIT
```

Hal yang sama berlaku pada registrasi: `accounts.nik_verified_at` hanya diisi setelah
Dukcapil menjawab valid, dan akun dengan `nik_verified_at IS NULL` **tidak dapat membeli**.

> **Tidak ada status PENDING di sini, berbeda dari usulan deferred validation.** Entri yang
> belum terverifikasi akan dipakai checkout untuk *melewati* Dukcapil — artinya satu NIK
> yang belum pernah diverifikasi akan menghasilkan tiket. Itu persis lubang yang fail-closed
> dirancang untuk menutup, hanya dipindahkan satu langkah ke belakang.
>
> **Menolak di sini tidak merugikan customer**, dan itulah yang membedakannya dari checkout.
> Menyimpan NIK tidak berada di bawah tekanan TTL: tidak ada stok yang di-hold, tidak ada
> countdown berjalan, dan customer dapat mencoba lagi beberapa menit kemudian tanpa
> kehilangan apa pun. Menolak di checkout menghentikan penjualan; menolak di sini hanya
> menunda satu entri di halaman profil.

`CHECK (verified_at IS NOT NULL)` menjadikannya invariant basis data, bukan janji kode:
tidak ada jalur penulisan — termasuk migrasi, skrip perbaikan, atau backfill — yang dapat
menciptakan entri tanpa bukti.

`dukcapil_ref` menyimpan rujukan jawaban Dukcapil yang memverifikasinya, sehingga sengketa
di kemudian hari dapat ditelusuri ke panggilan yang sebenarnya.

### NIK tersimpan: memotong panggilan Dukcapil, bukan memotong verifikasi

Dukcapil berbayar per panggilan, dan itulah alasan utama `NIK_Terdaftar` dan
`Daftar_NIK_Teman` ada. Sebuah NIK diverifikasi **sekali**, saat didaftarkan, lalu dipakai
berkali-kali tanpa panggilan baru.

| Asal NIK saat checkout | Panggilan Dukcapil |
|---|---|
| `accounts.nik_hmac` (NIK_Terdaftar pembeli) | **tidak ada** — diverifikasi saat registrasi |
| `saved_niks` (Daftar_NIK_Teman, maks 5) | **tidak ada** — diverifikasi saat ditambahkan |
| `nik:valid:{hmac}` masih hidup (cache 24 jam) | **tidak ada** |
| Diketik manual, tidak tersimpan, tidak di cache | **satu panggilan** |

**Customer tamu membayar penuh.** Tanpa akun, dua baris pertama tidak dapat berlaku, sehingga
setiap NIK yang ia ketik — hingga tiga per transaksi, termasuk NIK-nya sendiri — memanggil
Dukcapil.

Satu pengecualian: `nik:valid:{hmac}` dikunci oleh **hash NIK saja, bukan oleh akun**.
Cache karena itu bersifat global: NIK yang sudah diverifikasi siapa pun dalam 24 jam terakhir
menghasilkan cache hit, bahkan untuk customer tamu yang baru pertama mengetiknya. Jarang
terjadi untuk NIK acak, tetapi nyata — customer yang mengulang setelah gagal, atau dua orang
yang membelikan tiket untuk teman yang sama.

> Konsekuensi privasi kecil tapi nyata: membedakan cache hit dari panggilan baru pada respons
> API akan memberi tahu pemanggil bahwa NIK itu baru saja diverifikasi **oleh orang lain**.
> Karena itu `POST /nik/validate` tidak mengekspos keduanya secara terpisah — lihat
> `api-contracts.md` §4.2.

Verifikasi tidak dilewati, hanya tidak diulang: `saved_niks.verified_at` dan
`accounts.nik_verified_at` adalah bukti verifikasi yang sudah terjadi, dan
`booking-service` memperlakukannya setara dengan `nik:valid:{hmac}` saat memutuskan boleh
tidaknya hold dijalankan.

> **Ini tidak melonggarkan fail-closed.** NIK tanpa salah satu dari empat sumber di atas
> tetap wajib melewati Dukcapil, dan Dukcapil yang tidak dapat dihubungi tetap memblokir.
> Yang berubah hanya **berapa kali** kita membayar untuk jawaban yang sama.

> **Asumsi yang dicatat: bukti verifikasi berlaku permanen, tanpa revalidasi berkala.**
> Hasil verifikasi NIK dianggap tidak berubah setelah Dukcapil menyatakannya valid, sehingga
> tidak ada job yang memverifikasi ulang entri lama. Keputusan ini disengaja — revalidasi
> berkala justru membayar ulang panggilan yang fitur ini ada untuk menghindarinya.
>
> Yang membuatnya aman di POC adalah lingkupnya: **sistem gate check-in berada di luar
> lingkup** (`requirements.md` §8), sehingga tidak ada satu pun komponen hilir yang memeriksa
> ulang NIK sebuah tiket. NIK yang statusnya berubah di Dukcapil setelah tersimpan tidak
> menghasilkan konsekuensi apa pun di sistem ini.
>
> **Batas asumsinya**: begitu gate check-in masuk lingkup — atau begitu NIK dipakai untuk
> sesuatu selain pembatasan kuota — revalidasi menjadi perlu, karena saat itu barulah ada
> pihak yang menolak tiket atas dasar NIK.

**Volume**: ~15.000 verifikasi per event pada sale pertama → **~12/detik rata-rata, burst
~40/s**. Pada sale berikutnya angka ini turun seiring bertambahnya NIK tersimpan — yang
merupakan tujuan fitur ini, dan sekaligus berarti **estimasi biaya Dukcapil tidak boleh
diturunkan dari sale pertama**. Bila
`DUKCAPIL_RATE_LIMIT_RPS` di bawah itu, **Dukcapil menjadi plafon throughput penjualan**.

### Normalisasi dan stabilitas pepper

Uniqueness hanya bekerja bila NIK yang sama selalu menghasilkan `nik_hmac` yang sama. Dua
syarat yang mudah terlewat:

- **Normalisasi sebelum HMAC**: buang spasi di awal/akhir dan pemisah apa pun, tolak bila
  hasilnya bukan tepat 16 digit ASCII. `"3171 1234 5678 9012"` dan `"3171123456789012"`
  harus menghasilkan hmac yang identik, bukan dua klaim berbeda.
- **Pepper KMS tidak boleh dirotasi selama sale berjalan.** Rotasi mengubah seluruh hmac,
  sehingga `nik_claims` dan `nik:used` yang sudah ada menjadi tidak dapat dicocokkan dan
  BR-02 berhenti ditegakkan secara senyap. Rotasi dijadwalkan di luar jendela penjualan, dan
  bila pernah dilakukan, klaim lama harus di-backfill dengan hmac baru.

### Persetujuan pemrosesan data pribadi

NIK milik **setiap penonton**, bukan hanya pembeli, dan dikirim ke API pemerintah eksternal —
transfer data pribadi pihak ketiga di bawah UU PDP 27/2022. Dua persetujuan terpisah dicatat
di checkout, dan keduanya wajib sebelum stok di-hold:

1. Pembeli menyatakan memiliki izin tiap pemegang NIK untuk menyerahkan datanya
2. Pembeli menerima syarat dan ketentuan pembelian

`terms_version` dan `terms_accepted_at` merekam penerimaan, sehingga sengketa dapat dirujuk
ke teks yang benar-benar berlaku saat itu. Klausa memerlukan **sign-off legal** sebelum rilis.

### Siklus hidup NIK

```
Terdaftar / tersimpan →  TIDAK memanggil Dukcapil (accounts.nik_verified_at
                         atau saved_niks.verified_at sebagai bukti)
Dukcapil valid      →  SETEX nik:valid:{hmac} 86400
Dukcapil invalid    →  BLOKIR; tidak ada key yang ditulis
Dukcapil tak dapat
  dihubungi         →  BLOKIR; admission di-pause
Tekan Pay           →  SET    nik:used:{ev}:{hmac} NX EX   (Redis, jalur cepat)
                    →  INSERT nik_claims (event_id, nik_hmac) (Aurora, OTORITATIF)
Booking expired     →  DEL    nik:used + DELETE nik_claims  ← cegah lockout permanen
Payment sukses      →  keduanya tetap (kursi final, NIK terpakai untuk event ini)
```

### Rate limiter terdistribusi dan circuit breaker

| | Dukcapil | Payment Gateway |
|---|---|---|
| Sifat | Read / verify | **Write — memindahkan uang** |
| Aman di-retry? | ✅ Idempoten | ❌ Retry buta bisa double-charge |
| Timeout = gagal? | ✅ Ya | ❌ Charge mungkin sudah berhasil |
| Probe half-open | Verify biasa | **Read-only status query** |
| Degradasi | **Fail closed** + pause admission | **Fail closed** + pause admission |

Keduanya kini fail closed; yang tetap berbeda adalah cara memulihkan. Dukcapil boleh
di-retry karena idempoten, payment gateway tidak.

Token bucket dan state breaker **wajib di Redis** — Lambda menskala horizontal, sehingga
bucket in-process memberi 100 × 10 rps = 1.000 rps, bukan 10. Rate limit ingress
(`rl:join:{ip}`, `rl:sse:{ip}`) memakai `rate_limit.lua` yang sama.

```
DUKCAPIL_BASE_URL  DUKCAPIL_RATE_LIMIT_RPS  DUKCAPIL_BURST_CAPACITY
DUKCAPIL_TIMEOUT_MS  DUKCAPIL_BREAKER_FAILURE_THRESHOLD
DUKCAPIL_BREAKER_OPEN_SECONDS

PG_BASE_URL  PG_RATE_LIMIT_RPS  PG_BURST_CAPACITY  PG_TIMEOUT_MS
PG_BREAKER_FAILURE_THRESHOLD  PG_BREAKER_OPEN_SECONDS
MIDTRANS_SERVER_KEY        MIDTRANS_CLIENT_KEY      MIDTRANS_WEBHOOK_IP_ALLOWLIST
  -- Server key dipakai untuk Basic Auth charge DAN verifikasi signature_key webhook.
  -- Client key dikirim ke browser untuk tokenisasi kartu; server key TIDAK PERNAH.

JOIN_RATE_LIMIT  JOIN_RATE_WINDOW_SECONDS  SSE_CONN_RATE_LIMIT

OUTBOX_LEASE_SECONDS=30     DRIFT_THRESHOLD=3        RECONCILE_EVERY_TICKS=20
STOCK_DRIFT_THRESHOLD=2     STOCK_RECONCILE_EVERY_TICKS=120
NIK_USED_TTL_SECONDS        -- durasi sale + margin; membatasi kebocoran nik:used
GRACE_SECONDS=120           BOOKING_TTL_SECONDS=300  PAYMENT_TTL_SECONDS=600
PAYMENT_GRACE_SECONDS=30    QUEUE_CAPACITY=300000    MAX_SAVED_NIKS=5
ACCOUNT_RATE_LIMIT_PER_IP_HOUR=5                     POLICY_VERSION
  -- booking dari slot_claimed_at; pembayaran dari held_at (berurutan, maks 15 menit)
GRANT_INSERT_MAX_RETRIES=3  BROADCAST_HEARTBEAT_SECONDS=10
MAX_TICKETS_PER_TRANSACTION=3                        TERMS_VERSION
MAX_PAYMENT_ATTEMPTS=5      -- tiap percobaan menerbitkan VA/QR baru di Midtrans
MIN_RETRY_REMAINING_SECONDS=60  -- di bawah ini, ganti metode ditolak
GRANT_TOKEN_TTL_SECONDS=1200    -- = GRACE + BOOKING_TTL + PAYMENT_TTL + margin.
  -- DITURUNKAN, bukan konstanta lepas: perjalanan maksimum 120+300+600 = 1020 detik.
```

**Payment Gateway**: Midtrans **Core API**. Probe half-open read-only; breaker terbuka →
**pause admission**. Tiga metode dibuka — `credit_card`, `bank_transfer` (BCA/BNI/BRI/CIMB
saja), dan `gopay`; rinciannya di `payment-method/`.

> **Permata, Mandiri Bill dan Indomaret sengaja tidak dibuka.** Midtrans mendokumentasikan
> `settlement → deny` (reversal, 1–5 menit, dana kembali ke customer) hanya untuk ketiganya.
> Menutup mereka berarti kelas kegagalan itu tidak dapat terjadi, dan tidak perlu handler
> yang menarik kembali tiket yang sudah terbit dan e-tiket yang sudah terkirim.

**PCI-DSS (NFR-07) adalah batasan arsitektural, bukan asumsi.** Nomor kartu bergerak
langsung dari browser ke Midtrans lewat Midtrans.js, dan backend hanya menerima `token_id`.
Klaim "tidak ada data kartu yang melewati sistem ini" **hanya benar selama itu dipertahankan**:
satu form yang mengirim nomor kartu ke backend kita akan meruntuhkannya dan memperluas
cakupan PCI-DSS secara drastis. Tidak ada endpoint yang boleh menerima field nomor kartu.

**`order_id` = `{booking_id}:{payment_attempts}`** — diturunkan dari state Aurora, **tidak
pernah dikirim klien**. Midtrans Core API membawa idempotency pada field `order_id`, bukan
pada header tersendiri, dan **menolak `order_id` yang sudah dipakai**. Karena itu percobaan
ulang wajib memakai nilai baru: `booking_id` telanjang akan ditolak mentah pada percobaan
kedua, dan customer terkunci dari membayar booking-nya sendiri.

`payment_attempts` hanya naik pada retry eksplisit setelah kegagalan, bukan pada duplikat
bersamaan — sehingga dua klik Pay selalu menurunkan `order_id` yang identik dan Midtrans
men-dedup-nya, sementara retry sesudah penolakan selalu menghasilkan yang baru.

**Charge reconciler** (`rusuh-admission-ctl`, tiap 5 menit): untuk tiap booking berstatus
`HELD` atau `AWAITING_PAYMENT` yang `payment_ref`-nya NULL sementara `payment_attempts > 0` —
yaitu charge yang timeout dan statusnya tidak pernah diketahui — lakukan **query status
read-only** ke gateway memakai kunci yang sama. Hasil `settled` → jalur webhook; `failed` atau
`not_found` → `payment_ref` tetap NULL dan customer boleh mencoba lagi. Reconciler **tidak
pernah mengirim charge baru**.

### Idempotency pada `/bookings`

Masalah yang sama dengan bentuk berbeda: submit yang diulang menjalankan `hold_booking.lua`
untuk kedua kalinya, `SET nik:used NX` gagal terhadap baris yang ditulis panggilan **pertama**,
dan customer menerima `NIK_TAKEN` atas NIK-nya sendiri. Guard-nya memakai objek durabel yang
sudah ada, bukan tabel kunci baru:

```
POST /bookings
  BEGIN
    SELECT id, status FROM bookings WHERE queue_token = $accessToken FOR UPDATE
    status sudah 'HELD' atau lebih jauh → COMMIT ; kembalikan booking yang ADA, 200
    status ≠ 'SELECTING'                → COMMIT ; 409

    EVALSHA hold_booking.lua            -- stok + NIK dikunci di Redis
    gagal → COMMIT (tanpa perubahan) ; kembalikan NO_STOCK / NIK_TAKEN / QTY_EXCEEDED
            -- slot TIDAK hilang; customer tetap SELECTING dengan sisa TTL

    UPDATE bookings
       SET status             = 'HELD',
           held_at            = now(),
           payment_expires_at = now() + PAYMENT_TTL_SECONDS,   -- ANCHOR BR-05
           contact_email      = $contact_email,
           terms_version      = $terms_version,
           terms_accepted_at  = now()
     WHERE id = $1 AND status = 'SELECTING'
    INSERT INTO booking_items (...)     -- satu baris per kursi
    INSERT INTO nik_claims (event_id, nik_hmac, booking_id, claimed_at)
      VALUES (...)                      -- satu baris per NIK
      -- UNIQUE (event_id, nik_hmac) → BR-02 ditegakkan basis data.
      -- Konflik di sini berarti Redis melewatkan sesuatu (flush / failover /
      -- TTL habis). Transaksi batal, kompensasi jalan, customer menerima
      -- NIK_TAKEN. Pelanggaran BR-02 mustahil tersimpan.
  COMMIT
  ZADD expiries:{ev} (payment_expires_at) booking_id   -- rescore dari TTL booking

  ON ROLLBACK setelah Lua berhasil:                    -- KOMPENSASI WAJIB
    INCRBY stock:{ev}:{cat} qty
    DEL    nik:used:{ev}:{hmac}  (setiap NIK dalam permintaan)
    emit   HoldCompensated
    -- nik_claims ikut di-rollback oleh Aurora; hanya Redis yang perlu kompensasi
    -- Redis tidak ikut transaksi Aurora. Tanpa langkah ini stok dan NIK bocor:
    -- Aurora tetap SELECTING, sweep meng-expire-nya lewat jalur yang TIDAK
    -- mengembalikan stok, dan tiket hilang dari peredaran selamanya.
    -- Bila kompensasi sendiri gagal → rekonsiliasi stok (Bagian 6) menangkapnya.
```

`held_at` dan `payment_expires_at` ditulis **di dalam transaksi hold yang sama**. Tidak ada
keadaan di mana stok sudah terkunci tetapi deadline pembayarannya belum ada, dan crash
sebelum `ZADD` tidak meng-expire lebih awal karena sweep membaca deadline dari Aurora
(Bagian 7).

---

## 9. Alur Pembayaran, Webhook, dan Outbox

### Langkah 1 — Charge (sinkron)

Baris `bookings` **adalah** catatan idempotency-nya. Tidak ada tabel kunci terpisah dan tidak
ada kunci yang dibuat klien: percobaan yang sedang berjalan dikenali dari state yang sudah
tersimpan, dan responsnya diputar ulang apa adanya.

```
POST /payments/{id}
  1. Validasi accessToken
  2. BEGIN
       SELECT status, payment_ref, payment_attempts
         FROM bookings
        WHERE id = $1 AND queue_token = $accessToken   -- KEPEMILIKAN, bukan hanya
          FOR UPDATE                                   --   token yang valid
       0 baris → COMMIT ; 404 (BUKAN 403 — jangan konfirmasi booking itu ada)

       status = 'AWAITING_PAYMENT' → COMMIT ; return 200 { payment_ref, payment_expires_at }
                                     ↑ PEMUTARAN ULANG. Tidak memanggil gateway,
                                       tidak menulis apa pun, respons identik.
       status = 'CONFIRMED'        → COMMIT ; return 200 { status: 'CONFIRMED' }
       status ≠ 'HELD'             → COMMIT ; return 409 { reason }
     -- transaksi tetap terbuka bila status = 'HELD'

  3. key = booking_id || ':' || payment_attempts     -- DITURUNKAN, bukan dari klien
  4. rate limit → breaker → POST /v2/charge dengan transaction_details.order_id = key
       + custom_expiry { order_time: held_at (WIB), expiry_duration: 10, unit: minute }
       -- WAJIB: expiry VA bawaan Midtrans 24 JAM vs TTL kita 10 menit. Tanpa ini
       -- VA tetap dapat dibayar sepanjang hari setelah stok dijual ulang.
       -- Tidak berlaku untuk kartu; jendelanya ditentukan batas waktu 3DS.
  5.   UPDATE bookings
          SET status = 'AWAITING_PAYMENT', payment_ref = $ref,
              payment_method = $method        -- credit_card | bank_transfer | gopay
        WHERE id = $1 AND status = 'HELD'
       -- payment_expires_at TIDAK ditulis di sini; sudah ada sejak HELD (Bagian 7)
       INSERT INTO outbox (type='payment.initiated', aggregate_id=$1, payload=…)
     COMMIT
  6. -- Tidak perlu ZADD: skor penjadwal sudah payment_expires_at sejak HELD.
  7. Kembalikan VA / QR / deeplink

  PENOLAKAN SINKRON (kartu, charge menjawab deny langsung):
     UPDATE bookings
        SET status = 'HELD', payment_ref = NULL, payment_method = NULL,
            payment_attempts = payment_attempts + 1     -- WAJIB
      WHERE id = $1 AND status = 'HELD'
     -- Kenaikan attempt di sini yang membuat charge berikutnya memakai order_id
     -- baru. Tanpanya Midtrans menolak percobaan kedua sebagai duplikat dan
     -- customer terkunci dari membayar booking-nya sendiri.
```

Dua klik Pay: keduanya membaca `payment_attempts` yang sama, menurunkan kunci yang sama,
gateway men-dedup charge-nya, lalu yang kalah pada `FOR UPDATE` masuk ke cabang langkah 2 dan
mengembalikan `payment_ref` yang sama. Tombol yang terklik dua kali tidak boleh terlihat
seperti kegagalan.

### Dua jalur berbeda untuk "ganti metode"

**Setelah penolakan — tidak ada endpoint khusus.** Jalur deny (sinkron maupun webhook) sudah
mengembalikan booking ke `HELD` **dan** menaikkan `payment_attempts`. Tidak ada transaksi
Midtrans yang hidup. Customer cukup memilih metode lain dan memanggil `POST /payments/{id}`
biasa; `order_id` yang diturunkan sudah baru.

**Saat pembayaran masih `pending` — `POST /payments/{id}/retry`.** Customer sudah menerima VA
BCA lalu berubah pikiran ingin GoPay. Tidak ada yang gagal; transaksi lama **masih dapat
dibayar**. Memanggil charge lagi begitu saja akan membuat dua instrumen hidup sekaligus untuk
satu booking.

```
POST /payments/{booking_id}/retry

  KELAYAKAN — diperiksa berurutan; yang paling murah dan paling aman lebih dulu
  BEGIN
    SELECT status, payment_ref, payment_attempts, payment_expires_at
      FROM bookings
     WHERE id = $1 AND queue_token = $accessToken      -- 1. KEPEMILIKAN
       FOR UPDATE                                      --    + serialisasi
    0 baris                                  → ROLLBACK ; 404
      -- 404, BUKAN 403: 403 memberi tahu penyerang bahwa booking itu ada.
      -- Pemeriksaan ini mendahului semua yang lain agar status booking orang
      -- lain tidak pernah bocor lewat kode respons.

    status != 'AWAITING_PAYMENT'             → ROLLBACK ; 409   -- 2. tidak ada
      -- Satu-satunya status dengan transaksi Midtrans hidup. HELD setelah
      -- penolakan TIDAK memenuhi syarat: tidak ada yang perlu dibatalkan,
      -- customer cukup memanggil POST /payments/{id}.

    payment_expires_at <= now()              → ROLLBACK ; 409   -- 3. TTL habis
      -- Sweep belum sempat menjalankannya. Menerbitkan VA baru di sini
      -- memberi customer instrumen yang mati sebelum ia sampai ke ATM.

    (payment_expires_at - now()) < MIN_RETRY_REMAINING_SECONDS
                                             → ROLLBACK ; 409   -- 4. terlalu mepet
      -- Sisa 15 detik tidak cukup untuk metode apa pun. Lebih jujur menolak
      -- daripada menyerahkan VA yang pasti kedaluwarsa.

    payment_attempts >= MAX_PAYMENT_ATTEMPTS → ROLLBACK ; 429   -- 5. batas
    breaker payment gateway == OPEN          → ROLLBACK ; 503   -- 6. percuma
      -- Tanpa Midtrans kita tidak dapat membatalkan transaksi lama, dan
      -- menaikkan attempt tanpa membatalkannya meninggalkan dua instrumen hidup.

    old_order_id = booking_id || ':' || payment_attempts
    POST /v2/{old_order_id}/cancel  → Midtrans        -- masih memegang row lock
      sudah settlement → ROLLBACK ; 409 — webhook akan mengonfirmasi secara normal
      gagal lain       → ROLLBACK ; 503 — JANGAN naikkan attempt

    UPDATE bookings
       SET status='HELD', payment_ref=NULL, payment_method=NULL,
           payment_attempts = payment_attempts + 1
  COMMIT
```

**Cancel sebelum COMMIT, bukan sesudah.** Kedua urutan bisa gagal, tetapi tidak simetris:
commit berhasil lalu cancel gagal meninggalkan attempt yang sudah naik **dan** VA lama yang
masih hidup — dua instrumen, dan customer dapat membayar keduanya. Cancel berhasil lalu
commit gagal hanya meninggalkan VA yang tidak lagi berfungsi, yang terlihat pada polling
status dan dibersihkan sweep pada TTL. Yang kedua dapat dipulihkan; yang pertama mengambil
uang dua kali.

Row lock ditahan melewati panggilan HTTP. Itu lazimnya dihindari, tetapi di sini kuncinya
adalah baris booking milik satu customer — ia hanya memblokir permintaan bersamaan customer
itu sendiri, yang memang harus diblokir. Timeout ke Midtrans dibuat pendek.

**`cancel`, bukan `expire`.** Tidak ada yang kedaluwarsa saat customer berganti metode;
merchant membatalkannya. Dashboard Midtrans melaporkan keduanya terpisah, sehingga sale yang
penuh pergantian metode tidak terbaca sebagai sale yang penuh timeout.

**Untuk kartu, klien wajib melakukan tokenisasi ulang** pada percobaan berikutnya: `token_id`
Midtrans hanya berlaku untuk satu transaksi.

**TTL tidak ikut mundur.** `custom_expiry.order_time` diisi `held_at`, bukan waktu request,
sehingga VA baru mewarisi sisa jendela yang benar: anchor 10:15 + 10 menit tetap berakhir
10:25 walau charge kedua terjadi pukul 10:23. Seandainya `order_time` diisi `now()`, setiap
pergantian metode akan memberi 10 menit baru dan TTL pembayaran tidak pernah habis.

Bila gateway timeout: **jangan kirim ulang charge** — biarkan `payment_ref` NULL dan serahkan
ke charge reconciler (Bagian 8).

### Langkah 2 — Webhook: commit, lalu terbitkan inline

```
POST /payments/webhook
  ← Resource policy API Gateway: allowlist IP gateway (dulu rule WAF)
  1. Verifikasi tanda tangan + IP allowlist
     signature_key == SHA512(order_id + status_code + gross_amount + ServerKey)
     -- konkatenasi polos, tanpa pemisah. Tidak cocok → ABAIKAN, jangan proses.
     -- Dokumentasi Midtrans memuat dua skema; SHA256withRSA pada halaman Security
     -- Specification berlaku untuk lini API lain, BUKAN webhook Core API.

  2. BEGIN
       INSERT INTO payment_events (booking_id, gateway_event_id, type, raw)
         -- gateway_event_id UNIQUE = ID EVENT DARI GATEWAY, bukan booking_id.   FR-25
         --   Memakai booking_id di sini akan menelan setiap event SAH berikutnya
         --   untuk booking yang sama (pending → settled → refund).
         -- Duplikat memicu unique violation → seluruh transaksi batal → 200.

       switch event.type:

         settlement, ATAU capture DENGAN fraud_status='accept':
           -- capture + fraud_status != accept BUKAN lunas: dana belum diterima.
           -- payment_type webhook boleh berbeda dari yang dikirim: pembayaran QR
           -- GoPay lewat aplikasi QRIS lain datang sebagai 'qris'. Jangan cocokkan.
           UPDATE bookings SET status='CONFIRMED'
            WHERE id=$1 AND status IN ('AWAITING_PAYMENT','PENDING_CANCELLATION')
           -- PENDING_CANCELLATION ikut diterima: itu seluruh gunanya grace.
           bila 0 baris DAN status saat ini = 'EXPIRED':
             -- UANG DITERIMA SETELAH BOOKING KEDALUWARSA. Stok sudah dijual ulang.
             UPDATE bookings SET payment_disputed = true WHERE id = $1
             INSERT INTO outbox (type='payment.failed_notice', aggregate_id=$1, …)
             emit LatePaymentRefund   ← dialarmkan, bukan ditelan diam-diam
             -- REFUND OTOMATIS TIDAK DIIMPLEMENTASIKAN DI POC. Booking ditandai
             -- disputed dan direkonsiliasi lewat GET /admin/reconciliation/disputed
             -- + refund manual lewat dashboard Midtrans. Alasannya di Bagian 9.
           bila 1 baris:
             INSERT INTO outbox (type='payment.confirmed', …)
             INSERT INTO outbox (type='ticket.issue', …)   -- satu per booking_item  FR-26

         deny / cancel / failure, ATAU capture dengan fraud_status != 'accept':
           UPDATE bookings
              SET status='HELD', payment_ref=NULL, payment_method=NULL,
                  payment_attempts = payment_attempts + 1     -- WAJIB
            WHERE id=$1 AND status='AWAITING_PAYMENT'
           -- kembali ke HELD, BUKAN EXPIRED: stok tetap milik customer sampai TTL.
           -- Kenaikan attempt di sini membuat customer cukup memilih metode lain
           -- lalu memanggil POST /payments/{id} biasa — order_id-nya sudah baru.
           -- TIDAK perlu /retry: tidak ada transaksi Midtrans yang hidup.

         expire (dari sisi gateway):
           UPDATE bookings SET status='EXPIRED' WHERE id=$1 AND status='AWAITING_PAYMENT'
           bila 1 baris: lepas stok, DEL nik:used, INSERT outbox 'booking.expired'

         lainnya: catat saja; tidak ada transisi status
     COMMIT

  3. INLINE BEST-EFFORT — di luar transaksi, setelah COMMIT:
       untuk baris outbox 'payment.confirmed':
         if claim_lease(outbox_id):
             ZREM expiries:{ev} booking_id
             DECR slots:open
             EVALSHA advance_now_serving.lua
             PUBLISH sse-broadcast / sse-grant
             UPDATE outbox SET published_at = now()   -- (i)
             complete_lease(outbox_id)                -- (ii)
             -- (i) SEBELUM (ii) WAJIB:
             --   crash antara (i)-(ii) → lease kedaluwarsa 30 detik dan
             --     SELECT … WHERE published_at IS NULL tidak mengambilnya. Aman.
             --   dibalik: crash antara (ii)-(i) → lease "done" 86400 detik
             --     sementara published_at tetap NULL → publisher mengambil baris
             --     tetapi claim_lease selalu gagal → tidak pernah tuntas dan
             --     OutboxLag naik palsu selamanya.
       Kegagalan di langkah ini TIDAK menggagalkan request.

  4. return 200   ← SELALU, termasuk saat unique violation (sudah diproses)
```

**Mengapa inline**: menunggu publisher memberi p95 ≈ 1107ms — melewati NFR-06. Inline memberi
**~157ms**, dan baris outbox yang sudah ter-commit tetap menjamin event tidak pernah hilang.
`ticket.issue` tidak diterbitkan inline — email tidak boleh menambah latensi webhook.

### Langkah 3 — Guard berbasis lease

```
claim_lease(id)    : SET processed:{id} <worker_id> NX EX 30
complete_lease(id) : SET processed:{id} "done" EX 86400
```

Flag permanen tanpa lease **bocor**: handler yang mengambil flag lalu crash sebelum `DECR`
membuat konsumer berikutnya melewatinya dan slot hilang selamanya. Lease 30 detik berbanding
pekerjaan ~60ms memberi margin ~500×.

> **Lease memberi at-least-once, bukan exactly-once.** Ia mencegah dua pekerja memproses satu
> baris outbox bersamaan; ia tidak mencegah pengulangan setelah crash di tengah pekerjaan.
> Bila inline crash setelah `DECR slots:open` tetapi sebelum `published_at`, lease kedaluwarsa
> dan publisher mengulang seluruh blok — termasuk `DECR`, satu-satunya operasi non-idempoten
> di dalamnya (`ZREM`, `EVALSHA` dan `PUBLISH` semuanya aman diulang). Yang membuat ini aman
> bukan lease, melainkan **rekonsiliasi `slots:open`** (Bagian 6), yang menangkap drift dalam
> ~10 detik. Jendela itu diterima; menyembunyikannya tidak.

### Langkah 4 — Konsumer outbox (fallback + email)

```
outbox-publisher, tiap 1 detik:
  SELECT * FROM outbox WHERE published_at IS NULL
    ORDER BY id FOR UPDATE SKIP LOCKED LIMIT 500
  untuk tiap baris:
    if not claim_lease(row.id): continue        -- sudah diterbitkan inline
    switch row.type:
      payment.confirmed → ZREM expiries → DECR slots:open → EVALSHA → PUBLISH
      ticket.issue      → terbitkan tickets(seat_category, qr_payload, nik_hmac)
                          → SQS → SES ke booking_items.holder_email   FR-26, FR-27
      booking.expired   → SQS → SES ke contact_email + tiap holder_email      FR-28
      payment.failed_notice → SQS → SES ke contact_email. Dikirim pada TIGA kondisi:
                          (a) callback gateway menyatakan transaksi gagal
                          (b) TTL + grace berlalu tanpa callback apa pun
                          (c) callback sukses tiba setelah pembatalan final (disputed)
                          Isi (c) menyatakan dana diterima, booking sudah batal, dan
                          refund sedang diproses — WAJIB email, bukan hanya log.
      account.verify    → SQS → SES link verifikasi email                     FR-31
      payment.refund    → POST /v2/{order_id}/refund ke Midtrans
                          → SQS → SES pemberitahuan + alarm ops
    UPDATE outbox SET published_at = now() WHERE id = row.id   -- urutan sama
    complete_lease(row.id)
```

### Refund disputed: dicatat di POC, tidak dieksekusi otomatis

Callback sukses yang tiba setelah pembatalan final menandai `bookings.payment_disputed` dan
menerbitkan `payment.failed_notice` ke customer. **Refund otomatis tidak dibangun di POC.**

Alasannya bukan kompleksitas melainkan arah operasi: refund memindahkan uang keluar, dan
jalur yang mengeksekusinya otomatis berdasarkan callback adalah jalur yang paling mahal bila
salah. Pada volume POC jumlahnya kecil dan dapat ditangani manusia dengan bukti lengkap.

```
GET /admin/reconciliation/disputed?event_id=…
  → daftar booking dengan payment_disputed = true:
    booking_id, order_id, gross_amount, paid_at, cancelled_at,
    payment_type, gateway_event_id, contact_email
```

Operator mencocokkannya dengan dashboard Midtrans dan menjalankan refund di sana. Setiap
baris sudah memuat `order_id` dan `gateway_event_id`, sehingga pencocokan tidak memerlukan
pencarian manual.

> **Sebelum produksi**: otomatiskan lewat `POST /v2/{order_id}/refund` pada konsumer outbox,
> dengan alarm pada tiap eksekusi. Jalur `payment.refund` sudah ada bentuknya di Bagian 9;
> yang ditunda adalah pemanggilannya, bukan desainnya.

### Langkah 5 — Email tidak boleh menggagalkan transaksi

> **Booking `CONFIRMED` sejak pembayaran tercatat di Aurora.** Kegagalan SES tidak mengubah
> status booking — sesuai NFR-10.

```
SES gagal → SQS retry backoff → N percobaan → DLQ → alarm ops
GET /bookings/{id}/tickets    ← e-tiket SELALU dapat diambil dari tabel tickets
```

Email adalah kenyamanan; endpoint ini adalah sumbernya. Tanpa endpoint ini, "kegagalan email
tidak menggagalkan proses" benar secara teknis tetapi salah bagi customer.

---

## 10. Rantai Latensi NFR-06

```
t1 EVALSHA advance  ~6ms   t4 Fan-out ter-chunk  ~50ms
t2 Redis PUBLISH    ~2ms   t5 Jaringan 4G p95    ~80ms
t3 Pub/Sub → task   ~2ms   t6 Hitung + render    ~17ms
```

| Jalur | p95 | Status |
|---|---|---|
| Pembayaran sukses — **inline** (Bagian 9) | **≈ 157ms** | ✅ |
| Expiry / grace lapse — tick 500ms | **≈ 632ms** | ✅ (margin ~368ms) |
| Pembayaran sukses — fallback outbox 1 detik | ≈ 1107ms | ⚠️ hanya saat inline gagal |
| *(pembanding)* tick admission 1 detik | ≈ 1107ms | ❌ ditolak — tanpa margin terhadap batas 1 detik |

**Fan-out per-task** dan jumlah task tumbuh seiring koneksi — angka ini **tidak berubah pada
300.000**. Jalur fallback berada di luar anggaran NFR-06; itu degradasi terukur lewat
`OutboxInlineHitRate`, bukan perilaku normal. Metrik yang dilaporkan: `SlotReleaseToAdmit`
(transisi status Aurora → `PUBLISH`) digabung RUM klien.

---

## 11. ECS Operations @ 300.000

| Kapasitas terukur per task | Task | vCPU Fargate |
|---|---|---|
| 7.000 (Node, konservatif) | 43 | 86 |
| 15.000 (Node, tuned) | 20 | 40 |
| 25.000 (Go) | 12 | 24 |

Rentang 3,5× pada permintaan kuota berlead-time seminggu → pengukuran **tidak dapat ditunda**.

```
Scale out : > 70% kapasitas TERUKUR per task
Scale in  : DINONAKTIFKAN selama sale aktif (selesai → desired_count = 0)
Min / Max : 0 → N / 60 · Cooldown out 60 detik
```

> `desired_count = 0` setelah sale bukan optimasi opsional: 20 task yang tertinggal menyala
> berbiaya ~$425/bulan tanpa ada yang menyadarinya. Alarm `IdleTaskCount > 0` di luar jendela
> sale.

### Pre-Sale Checklist

| Waktu | Aksi |
|---|---|
| **T-14 hari** | Load test single-task → kapasitas koneksi per task |
| **T-7 hari** | Service Quota: Lambda concurrent executions → 6.000 (lead time 3–5 hari kerja) |
| **T-7 hari** | Service Quota: Fargate On-Demand vCPU → 120 |
| **T-7 hari** | Cek burst concurrency region; kuota API Gateway (default 10.000 RPS) |
| **T-7 hari** | Konfirmasi `DUKCAPIL_RATE_LIMIT_RPS` riil vs kebutuhan ~40/s burst **dan SLA ketersediaannya** — kini ia SPOF |
| **T-7 hari** | Daftarkan URL `/payments/webhook`; uji tanda tangan dan resource policy IP |
| **T-7 hari** | Klausa T&C + persetujuan PDP final; sign-off legal; `TERMS_VERSION` dipasang |
| T-1 hari | Konfirmasi semua kuota naik di AWS Console |
| T-30 menit | ECS `desired_count` = N; Provisioned Concurrency `queue-service`, `nik-verify`, `payment-api` |
| T-10 menit | `rusuh-admission-ctl` healthy; subscriber Redis aktif; kedua breaker tertutup; `SlotsOpenDrift` = 0 |
| T-5 menit | Semua task healthy; `OutboxLag` = 0 |
| T-2 menit | Smoke test: 100 koneksi SSE + admission + grace lapse + NIK valid + NIK ditolak + **Dukcapil down → pause** + charge sandbox + **charge ganda bersamaan → satu charge, dua respons identik** + **webhook diputar ulang → 200 tanpa efek** + **webhook settled setelah EXPIRED → payment.refund** + rekonsiliasi `slots:open` + **rollback Aurora setelah Lua berhasil → stok dan NIK kembali** + **NIK yang sama di dua booking → ditolak** + **flush `nik:used` → rekonsiliasi memulihkan dari `nik_claims`** + resume + sold-out 410 + bypass antrian kosong |
| T-0 | Sale dibuka |

**Join storm**: 300.000 dalam 60 detik = 5.000 rps ≈ 5.000 eksekusi Lambda bersamaan; default
account concurrency 1.000 → sisanya `429` pada momen paling genting (NFR-01).
**ALB ramp**: ~5.000 koneksi baru/detik di T=0 — uji pada ramp, bukan steady state.

### Graceful shutdown, disconnect, fan-out

```javascript
process.on('SIGTERM', () => {
  for (const conns of allConnections.values())
    for (const res of conns) {
      // Di sini reconnect otomatis EventSource JUSTRU diinginkan —
      // jitter hanya menyebarkan waktunya. Bandingkan dengan sold-out (Bagian 4),
      // yang harus menghentikan reconnect sama sekali.
      res.write(`retry: ${1000 + Math.floor(Math.random() * 14000)}\n`);
      res.write('event: reconnect\ndata: {}\n\n');
      res.end();
    }
  setTimeout(() => process.exit(0), 5000);
});

res.on('close', () => {                 // WAJIB kedua map
  connections.get(eventId)?.delete(res);
  byToken.delete(token);
});

function broadcast(eventId, payload) {  // chunk + backpressure
  const conns = [...(connections.get(eventId) ?? [])];
  const frame = `id: ${payload.nowServing}\ndata: ${JSON.stringify(payload)}\n\n`;
  let i = 0;
  (function chunk() {
    const end = Math.min(i + 500, conns.length);
    for (; i < end; i++) {
      const res = conns[i];
      if (!res.write(frame)) {
        metrics.increment('SSEWriteBackpressure');
        if (++res.slowCount > 3) res.end();
      } else { res.slowCount = 0; }
    }
    if (i < conns.length) setImmediate(chunk);
  })();
}
```

### Handler klien — SSE (web dan mobile)

```javascript
const es = new EventSource(url, { withCredentials: true });

es.addEventListener('message', e => {
  const { nowServing, ratePerMin, paused, serverTime } = JSON.parse(e.data);
  syncClock(serverTime);                                 // BR-10
  render(myTicketNo - nowServing, ratePerMin, paused);   // paused → banner
});

es.addEventListener('grant', async () => {
  const { accessToken } = await fetch(`/events/${ev}/queue/me`).then(r => r.json());
  goToDetailTicketPage(accessToken);
});

es.addEventListener('status', e => setStaleIndicator(JSON.parse(e.data).stale));

es.addEventListener('sold-out', e => {
  showSoldOutScreen(JSON.parse(e.data));
  es.close();      // WAJIB — tanpa ini EventSource reconnect otomatis
});                //  dan 300.000 klien membanjiri ALB setelah sold out

es.onerror = () => { /* biarkan retry bawaan; 410 menghentikannya sendiri */ };
```

---

## 12. Autentikasi, Identitas, Kontak, dan Re-entrancy

Stream-ticket sekali pakai **merusak reconnect bawaan** `EventSource`: percobaan ulang ke URL
yang sama menemui ticket yang sudah dipakai → `401` → menurut HTML Living Standard status
non-200 membuat user agent **mengakhiri koneksi permanen**. Itulah mengapa autentikasi stream
memakai cookie/header — dan mengapa perilaku yang sama dipakai **dengan sengaja** sebagai
`410 Gone` pada sold-out (Bagian 4).

| Platform | Mekanisme | Token di URL? |
|---|---|---|
| Web | Cookie `HttpOnly; Secure; SameSite=Lax; Domain=.rusuhtiket.com`; `withCredentials: true` | ❌ |
| Mobile | `react-native-sse` custom header → `Authorization: Bearer` | ❌ |

| Kelas | Pengikatan `accessToken` | Requirement |
|---|---|---|
| **Login** | `account_id` | FR-29 opsional |
| **Anonim** | `queue_token` + device fingerprint | FR-29, FR-30 |

### Kepemilikan: token yang valid ≠ token untuk booking ini

`accessToken` yang sah membuktikan pemegangnya punya slot, **bukan** bahwa slot itu milik
`booking_id` di URL. Keduanya harus diikat, pada **setiap** endpoint yang menerima
`booking_id` atau `ticket_id` dari path:

```sql
WHERE id = $1 AND queue_token = $accessToken
```

Tanpa ikatan ini siapa pun yang memegang satu token sah dapat membaca booking orang lain,
membatalkan pembayaran orang lain lewat `/retry`, atau melakukan charge atas booking orang
lain. `booking_id` berbentuk `RB-00912` — berurutan dan mudah ditebak, dan pada 300.000
customer bersamaan enumerasinya sepele.

| Endpoint | Terikat |
|---|---|
| `GET /bookings/{id}` | ✅ |
| `GET /bookings/{id}/tickets` | ✅ |
| `GET /payments/{id}/methods` | ✅ |
| `POST /payments/{id}` | ✅ |
| `POST /payments/{id}/retry` | ✅ |
| `GET /payments/{id}/status` | ✅ |

**Tidak cocok → `404`, bukan `403`.** `403` mengonfirmasi bahwa booking itu ada, yang sudah
cukup untuk memetakan rentang id yang aktif. Pemeriksaan ini dijalankan **sebelum** validasi
status apa pun, agar kode respons tidak pernah membocorkan keadaan booking orang lain.

### Re-entrancy (BR-09)

`GET /events/{ev}/queue/me` adalah satu-satunya endpoint yang dipanggil klien saat kembali, apa pun
keadaannya, dan ia mengembalikan **token yang sama** selama TTL belum habis:

| Status booking | Respons | Halaman tujuan |
|---|---|---|
| tidak ada | `{queued, ticketNo, nowServing}` | Waiting Room |
| GRANTED | `{granted, accessToken}` | Detail Ticket |
| SELECTING / HELD | `{accessToken, bookingId, bookingExpiresAt, serverTime}` | Detail Ticket, countdown dilanjutkan |
| AWAITING_PAYMENT | `{accessToken, bookingId, paymentExpiresAt, paymentRef, serverTime}` | Payment, countdown dilanjutkan |
| CONFIRMED | `{bookingId}` | Halaman tiket |
| EXPIRED / LAPSED | `{expired: true}` | Layar expired |

`accessToken` disimpan di `grant:{token} EX GRANT_TOKEN_TTL_SECONDS` sehingga refresh, ganti tab, atau kembali
dari aplikasi bank tidak pernah menerbitkan token baru maupun memindahkan customer.

### `contact_email`, `holder_email`, dan penerimaan T&C

Baris `bookings` dibuat saat **GRANTED**, ketika customer belum mengisi apa pun. Kolom-kolom
ini karena itu **tidak bisa** dideklarasikan `NOT NULL`; penegakannya bersyarat status (Bagian 4).

| Kolom | Wajib sejak | Boleh NULL pada |
|---|---|---|
| `contact_email` (pembeli) | `SELECTING` | `GRANTED`, `LAPSED` |
| `terms_accepted_at` + `terms_version` | `HELD` | `GRANTED`, `SELECTING`, `LAPSED` |
| `booking_items.holder_email` | saat baris item dibuat (`HELD`) | — |

Constraint basis data adalah penegak sebenarnya, karena berlaku untuk **semua** penulis tanpa
bergantung pada disiplin tiap service.

| Kelas | Pengisian email |
|---|---|
| **Login** | Pre-fill dari akun, dapat diubah; **disalin ke `bookings`**, bukan di-join |
| **Anonim** | Wajib manual, validasi format, konfirmasi ulang (ketik dua kali) |

Disalin ke baris booking karena e-tiket harus menuju alamat yang diberikan **saat pembelian**,
sekalipun email akun berubah kemudian. `holder_email` default mengikuti `contact_email` dan
dapat diubah per kursi — FR-26 menuntut tiket sampai ke pemegang NIK, bukan hanya pembeli.

### Akun opsional (FR-31)

Pembuatan akun tidak pernah menghalangi pembelian. Bila customer memilih membuat akun,
`accounts(email, verify_token)` dibuat dan `account.verify` masuk outbox; tautan verifikasi
(bukan OTP) mengisi `email_verified_at`. Akun yang belum terverifikasi tetap dapat membeli —
verifikasi hanya membuka riwayat pesanan.

---

## 13. Mobile, Polling Fallback, Dashboard, i18n

**Mobile** memakai `react-native-sse` pada endpoint yang sama; auth via header; keepalive 15
detik; reconnect otomatis dengan `Last-Event-ID`; **handler `sold-out` wajib memanggil
`close()` sama seperti web**. API identik untuk web dan mobile (FR-24, NFR-08).

**Polling fallback.** `GET /events/{ev}/queue/serving` mengembalikan `{nowServing, ratePerMin,
paused, serverTime, pollAfterMs}` dan tetap didukung penuh untuk tiga keadaan: hidrasi awal
sebelum stream terbuka, jaringan yang memblokir `text/event-stream`, dan degraded mode saat
tier SSE tidak sehat. `pollAfterMs` mengikuti profil `requirements.md` Bagian 5
(10s >1000 → 5s 101–1000 → 2s 11–100 → 1s 1–10), sehingga klien tidak menanam interval sendiri.
Halaman pembayaran memang memakai polling `GET /bookings/{id}` 2 detik — status pembayaran
tiba lewat webhook, bukan lewat antrian.

| Skenario | Dampak | Posisi |
|---|---|---|
| Web tab terbuka / mobile foreground | ✅ SSE aktif | Terpenuhi |
| Web tab tertutup | Buka ulang dan reconnect | Acceptable |
| **Mobile background** | ⚠️ OS menangguhkan jaringan; grant tidak diketahui | **Diketahui, diterima** |

Grace 120 detik + `readmit` (maks 2×) → konsekuensinya penundaan, bukan kehilangan giliran.

**Dashboard operasional (FR-33, FR-34, FR-35).** Tidak ada endpoint aplikasi untuk ini.
`metrics-scraper` (EventBridge, tiap 1 menit) membaca Redis dan Aurora dan menerbitkan
**CloudWatch EMF** pada namespace `RusuhBooking/Queue`; dashboard dibangun di CloudWatch.

| Metrik EMF | Sumber | Requirement |
|---|---|---|
| `QueueDepth` | `issued − now_serving` | FR-33 |
| `ConversionRate` | `COUNT(CONFIRMED) / COUNT(DISTINCT queue_token)` | FR-34 |
| `ExpiryRate` per tahap | `COUNT(EXPIRED / LAPSED)` per status asal | FR-35 |
| `StockLevel`, `OpenSlots`, `DukcapilCircuitState`, `PGCircuitState` | Redis | NFR-09 |

**Dua keterbatasan yang diterima.** Granularitas EMF standar adalah **1 menit** ditambah jeda
ingestion log, sehingga "real-time" pada FR-33 dan NFR-09 berarti *dalam satu menit*, bukan
sub-detik. Dan dashboard CloudWatch menuntut akses IAM AWS: ia melayani tim SJT, **bukan**
pemilik acara atau stakeholder bisnis. Bila kelak metrik ini perlu dilihat di luar AWS,
endpoint aplikasi harus ditambahkan — `metrics-scraper` sudah memegang query-nya.

**i18n (FR-32).** Seluruh pesan API dikembalikan sebagai kode (`NIK_TAKEN`, `NO_STOCK`)
dengan katalog Indonesia dan Inggris di klien; `Accept-Language` menentukan bahasa email
yang diterbitkan `outbox-publisher`.

---

## 14. Pemenuhan Requirement

| ID | Requirement | Dipenuhi oleh |
|---|---|---|
| FR-01..03 | Daftar event, aktif, sold out | Bagian 5 `GET /events` |
| FR-04 | Ada antrian → Waiting Room | Bagian 5 `POST /events/{ev}/queue/join` |
| FR-05 | Tidak ada antrian → langsung Detail | Bagian 5 jalur bypass |
| FR-06, FR-07 | Posisi dan pembaruan otomatis | Bagian 4 SSE broadcast; posisi dihitung klien |
| FR-08 | Pindah otomatis saat giliran tiba | Bagian 4 `event: grant` |
| FR-09 | Estimasi waktu tunggu | `ratePerMin` pada payload |
| FR-10 | Konsisten saat refresh | Bagian 4 snapshot awal; Bagian 12 re-entrancy |
| FR-11, FR-12 | Detail event, kategori, stok, harga | `GET /events/{id}` |
| FR-13, FR-14 | Maks 3; tolak melebihi stok | Bagian 8 `hold_booking.lua` `QTY_EXCEEDED`, `NO_STOCK` |
| FR-15 | Format NIK 16 digit per tiket | Validasi klien + server sebelum Dukcapil |
| FR-15a | Validasi Dukcapil | Bagian 8 — **NIK saja**; lihat catatan di bawah |
| FR-16 | Hold stok saat lanjut bayar | Bagian 8 `hold_booking.lua` |
| FR-17, FR-19 | Countdown TTL booking, server-authoritative | Bagian 7 `booking_expires_at` + `serverTime` |
| FR-18 | TTL habis → batal, hold lepas, slot berikutnya | Bagian 7 sweep |
| FR-20 | Pilih metode pembayaran | Bagian 9 `POST /payments/{id}` |
| FR-21 | Countdown TTL pembayaran | Bagian 7 `payment_expires_at` — 10 menit penuh bagi setiap customer |
| FR-22 | Email e-tiket | Bagian 9 `ticket.issue` |
| FR-23 | 10 menit tidak selesai → batal, stok lepas | Bagian 7 sweep |
| FR-24 | API dipakai web dan mobile | Bagian 13 |
| FR-25 | Callback idempoten | Bagian 9 `payment_events.gateway_event_id UNIQUE`; charge diputar ulang dari state `bookings` |
| FR-26 | Email ke tiap pemegang NIK | Bagian 4 `booking_items.holder_email`; Bagian 9 |
| FR-27 | QR code, kategori tiket, NIK pemegang (nomor bangku tidak diimplementasikan di POC) | Bagian 4 `tickets(seat_category, qr_payload, nik_hmac)` |
| FR-28 | Email expired | Bagian 7, Bagian 9 `booking.expired` |
| FR-29, FR-30 | Login opsional; cukup NIK + email | Bagian 12 |
| FR-31 | Verifikasi email via link | Bagian 12 `accounts` + `account.verify`; link 24 jam, maks 3 kirim ulang/jam |
| **NIK_Terdaftar** | Satu NIK per akun, unik global | Bagian 4 `UNIQUE (accounts.nik_hmac)` |
| **Daftar_NIK_Teman** | Maks 5 NIK tersimpan, sudah terverifikasi | Bagian 4 `saved_niks`; Bagian 8 memotong panggilan Dukcapil |
| **Consent (UU PDP)** | Persetujuan eksplisit saat registrasi, rekam immutable | Bagian 4 `consent_records` append-only |
| **Grace pembayaran** | 30 detik setelah TTL, callback masih diterima | Bagian 7 `PENDING_CANCELLATION` |
| **Kapasitas antrian** | 300.000, `QUEUE_FULL` fail fast, state di halaman utama | Bagian 6 |
| FR-32 | Dua bahasa | Bagian 13 |
| FR-33..35 | Dashboard real-time | Bagian 13 — CloudWatch EMF via `metrics-scraper`; lihat keterbatasan di sana |
| NFR-01 | UX terjaga; waiting room melindungi | Bagian 4, Bagian 6 |
| NFR-02 | Jangan PHP | Bagian 6 INSERT-sebelum-PUBLISH; Bagian 8 gagal-hold tidak menghilangkan slot; Bagian 8 pause saat dependensi mati |
| NFR-03 | Flash crowd 200.000+ | Bagian 11 — disiapkan 300.000 |
| NFR-04 | Stok atomic, tidak oversell | Bagian 8 Lua `DECRBY` dalam satu skrip |
| NFR-05 | FCFS, tidak dapat dilangkahi | Bagian 6 `INCR` + `by_ticket` — lihat batasan WAF Bagian 16 |
| NFR-06 | p95 < 1 detik | Bagian 10 |
| NFR-07 | Token proteksi; PCI-DSS via gateway | Bagian 12; Bagian 8 — tokenisasi Midtrans.js, backend hanya menerima `token_id` |
| NFR-08 | REST/JSON platform-agnostic | Bagian 13 |
| NFR-09 | Observability real-time | Bagian 4 EMF; Bagian 13 dashboard |
| NFR-10 | Kegagalan email tidak membatalkan | Bagian 9 langkah 5 |
| BR-01 | Maks 3 per transaksi | Bagian 8 `MAX_TICKETS_PER_TRANSACTION` |
| BR-02 | 1 NIK = 1 seat per event | Bagian 8 — `UNIQUE (event_id, nik_hmac)` pada `nik_claims` sebagai penegak; `SET nik:used NX` sebagai jalur cepat; `UNIQUE (booking_id, nik_hmac)` mencegah duplikat dalam satu booking |
| BR-03, BR-04 | TTL booking 5 menit | Bagian 7 |
| BR-05, BR-06 | TTL pembayaran 10 menit sejak booking dibuat | Bagian 7 — anchor `held_at`, yaitu saat tombol bayar ditekan dan stok di-hold |
| BR-08 | Stok dibayar final; hold tidak bisa dibeli orang lain | Bagian 8 |
| BR-09 | Re-entrant kembali ke halaman sama, token sama | Bagian 12 |
| BR-10 | Server time source of truth | Bagian 6 — `serverTime` pada tiap payload SSE dan `server_time` pada tiap respons REST |

**Catatan terhadap `requirements.md`:**

1. **FR-15a** sebelumnya menyebut Dukcapil memvalidasi *"nama, status aktif"*. **Telah diputuskan: implementasi mengirim NIK saja** — form tidak mengumpulkan nama. `requirements.md` FR-15a dan Bagian 12 Asumsi telah diperbarui sesuai.
2. **Bagian 5 dan Bagian 11** sebelumnya mencatat transport sebagai adaptive polling. Telah diperbarui ke SSE.

BR-07 dihapus dan tidak diimplementasikan; konsekuensinya ada di Bagian 16.

---

## 15. Estimasi Biaya @ 300.000

| Komponen | Per sale event |
|---|---|
| ECS Fargate SSE (43 task × 2,5 jam @7.000 conn/task) | ~$10.61 |
| ECS Fargate admission-ctl | ~$0.06 |
| Egress SSE (payload 90 B, ~1 update/detik) | ~$17.50 |
| ALB (LCU aktif + burst koneksi) | ~$3.00 |
| API Gateway HTTP (~2,5 juta request) | ~$2.50 |
| Lambda (termasuk `nik-verify`, `outbox-publisher`) | ~$0.80 |
| Aurora (scale-up + baris GRANTED + rekonsiliasi + verifikasi sweep) | ~$1.05 |
| SQS + SES | ~$0.09 |
| Dukcapil API | **per kontrak — belum diketahui.** Volumenya bergantung pada rasio login; lihat catatan di bawah |
| **Total variabel per event** | **~$36 + biaya Dukcapil** |

Pada 15.000 conn/task, ECS turun ke ~$4.94 dan total ≈ **$30**.

**Biaya Dukcapil adalah variabel produk, bukan variabel teknis.** Penghematan dari
`NIK_Terdaftar` dan `Daftar_NIK_Teman` sepenuhnya bergantung pada berapa banyak pembeli yang
login dan sudah menyimpan NIK:

| Komposisi pembeli | Panggilan Dukcapil per 10.000 tiket |
|---|---|
| Seluruhnya tamu (_guest_) | ~10.000 — satu per NIK attendee |
| Seluruhnya login dengan NIK tersimpan | mendekati nol |

> **Jangan menurunkan angka tender dari asumsi adopsi login.** Rentangnya sekitar 5× antara
> kedua ujung, dan adopsi login adalah keputusan produk yang belum terjadi. Angka yang aman
> untuk dikutip adalah skenario tamu-berat; penghematan dari NIK tersimpan disajikan sebagai
> potensi, bukan sebagai dasar perhitungan. Egress adalah komponen
terbesar dan berbanding lurus dengan frekuensi update: publish hanya saat `now_serving`
berubah, bukan tiap tick, memotongnya secara proporsional.

**Fixed bulanan**: Redis $477 · Aurora Serverless v2 $44 · RDS Proxy $44 · KMS $3 ·
ALB idle $16 · Lambda terjadwal $21 · ECS scale-to-0 $0 → **$605/bulan**.
Total ≈ **$677/bulan** pada 2 event/bulan.

Penghapusan WAF memangkas ~$2/event dan ~$18/bulan. Itu bukan alasan menghapusnya; lihat Bagian 16.

Working set Redis pada 300.000 ≈ 80 MB terhadap 13 GB per node: sizing didasarkan
**throughput operasi dan Multi-AZ, bukan kapasitas memori**.

---

## 16. Konsekuensi

**Positif**

- NFR-06 terpenuhi pada semua jalur normal: ~157ms (pembayaran inline), ~632ms (expiry)
- Pembayaran tidak dapat kehilangan event; inline memberi kecepatan, outbox memberi jaminan
- **Tidak ada booking yang di-expire lebih awal**: Aurora memegang deadline, sweep
  memverifikasi dan me-rescore entri ZSET yang basi (Bagian 7)
- **Kedua deadline ditulis di satu transaksi saat claim**, menghapus jendela crash antara
  charge dan penjadwalan
- **Tidak ada tiket terbit dengan NIK yang belum terverifikasi** — tidak ada kelas sengketa
  "tiket sah tapi NIK palsu"
- **Tidak ada badai reconnect setelah sold out**: `es.close()` + `410 Gone`
- BR-02 adalah invariant basis data, bukan janji Redis: `UNIQUE (event_id, nik_hmac)`
- `slots:open` dan `stock` sama-sama sembuh sendiri lewat rekonsiliasi; phantom slot dan tiket
  yang hilang dari peredaran mustahil menumpuk
- Seluruh FR, NFR dan BR `requirements.md` v1.0 terpetakan (Bagian 14)

**Negatif / Risiko**

| Risiko | Mitigasi |
|---|---|
| **Dukcapil adalah SPOF.** Gangguan menghentikan penjualan, bukan memperlambatnya | Diterima secara sadar. Breaker OPEN mem-pause admission agar tidak ada yang masuk funnel buntu; `DukcapilBlockedSales` dialarmkan; SLA Dukcapil dikonfirmasi T-7 |
| **Customer yang sedang di dalam saat Dukcapil mati** tetap kehilangan TTL-nya | **BELUM TERTUTUP** — Bagian 17 #4. Opsi: bekukan TTL selama pause, atau perpanjang otomatis sebesar durasi pause |
| **Tanpa WAF, tidak ada proteksi bot di `/events/{ev}/queue/join`** | Rate limit Redis per IP (20 / 5 menit) menahan percobaan naif, **bukan** botnet terdistribusi. Dengan BR-07 dihapus, satu pihak dapat memborong slot antrian dan menggeser posisi orang lain — **degradasi NFR-05 yang diterima untuk POC**, harus dipulihkan sebelum produksi |
| Tanpa WAF, tidak ada CAPTCHA dan tidak ada managed rule | Idem. Diterima untuk POC |
| Okupansi slot naik ke maks 15 menit (5 + 10) | Hanya ekor yang terpengaruh: median selesai 2–4 menit. Slot yang ditinggalkan terkunci 15 menit, bukan 10 — dipantau lewat `ExpiryRate` per tahap |
| Crash antara COMMIT dan `ZADD` | Deadline otoritatif di Aurora; sweep me-rescore; `ExpiryRescored` dipantau (Bagian 7) |
| Klien lama tanpa `es.close()` pada sold-out | Server menjawab `410 Gone`; `SoldOutStreamRejections` dipantau |
| Jalur fallback outbox p95 ≈ 1107ms | `OutboxInlineHitRate`; turunkan interval publisher bila sering |
| Batch INSERT GRANTED gagal | Retry → `DECRBY` + `readmit` + tidak publish (Bagian 6) |
| Kuota Dukcapil menjadi plafon throughput | Konfirmasi T-7 hari; uji di rehearsal |
| Payment gateway tidak tersedia | Breaker fail closed + pause admission + resume otomatis. Pause melewatkan ADVANCE, termasuk pengurasan `readmit` (Bagian 6) |
| Double-charge saat timeout gateway | Kunci per-percobaan `{booking_id}:{attempt}` + charge reconciler read-only (Bagian 8); charge tidak pernah dikirim ulang |
| Outbox at-least-once merusak `slots:open` | Lease guard (Bagian 9) + rekonsiliasi (Bagian 6) |
| **Rollback Aurora setelah `hold_booking.lua` berhasil** | Redis di luar transaksi: stok dan NIK bocor, dan jalur expiry `SELECTING` tidak mengembalikannya. Kompensasi eksplisit (Bagian 8) + rekonsiliasi stok tiap 60 detik (Bagian 6) + `HoldCompensated` dialarmkan |
| `nik:used` bocor di Redis | Direkonsiliasi terhadap `nik_claims` tiap 60 detik + `EX` sepanjang sale (Bagian 6) |
| **Redis flush / failover menghapus `nik:used`** | BR-02 tetap ditegakkan `UNIQUE (event_id, nik_hmac)` di Aurora; rekonsiliasi memulihkan kunci Redis dan `NikClaimRestored` dialarmkan |
| Pepper KMS dirotasi saat sale berjalan | Seluruh hmac berubah dan BR-02 gagal senyap — rotasi dijadwalkan di luar jendela penjualan (Bagian 8) |
| Rekonsiliasi memakai drift basi | `current_drift`, bukan `last_drift` (Bagian 6) |
| Webhook duplikat dari gateway | `payment_events.gateway_event_id UNIQUE`; selalu 200 |
| **Pembayaran tiba setelah booking EXPIRED** | Stok sudah dijual ulang. Webhook `settled` terhadap baris EXPIRED memicu `payment.refund` + alarm `LatePaymentRefund` (Bagian 9) |
| Customer terkunci setelah charge ditolak | Jalur deny menaikkan `payment_attempts` sendiri → `order_id` berikutnya sudah baru (Bagian 9) |
| **`booking_id` dapat ditebak; token sah dipakai atas booking orang lain** | Setiap endpoint ber-`booking_id` mengikat `queue_token`; tidak cocok → `404` sebelum pemeriksaan status apa pun (Bagian 12) |
| **Entri NIK tersimpan tanpa bukti verifikasi** | Mustahil: `CHECK (verified_at IS NOT NULL)` + Dukcapil dipanggil sebelum INSERT, bukan sesudah (Bagian 8) |
| Status NIK berubah di Dukcapil setelah tersimpan | **Diterima.** Bukti verifikasi permanen, tanpa revalidasi berkala. Tidak berkonsekuensi selama gate check-in di luar lingkup (Bagian 8) |
| Stok dilepas lalu callback sukses tiba | Stok **tetap** di-hold selama `PENDING_CANCELLATION`; pelepasan hanya setelah grace berakhir (Bagian 7) |
| Refund disputed tidak otomatis di POC | Ditandai `payment_disputed`, customer diberi tahu lewat email, operator merekonsiliasi lewat `GET /admin/reconciliation/disputed` (Bagian 9) |
| Antrian menerima lebih banyak orang daripada yang bisa dilayani | `QUEUE_CAPACITY` + `QUEUE_FULL` fail fast; state antrian terlihat di halaman utama sebelum customer mengklik (Bagian 6) |
| **`accessToken` kedaluwarsa sebelum jendela pembayaran habis** | Umur token diturunkan dari GRACE + BOOKING_TTL + PAYMENT_TTL, bukan konstanta lepas (Bagian 7) |
| Ganti metode meninggalkan dua instrumen hidup | `/retry` membatalkan transaksi lama di Midtrans **sebelum** COMMIT; gagal cancel → attempt tidak naik (Bagian 9) |
| Customer menerbitkan puluhan VA dengan ganti metode berulang | `MAX_PAYMENT_ATTEMPTS` → 429 (Bagian 8) |
| **VA tetap dapat dibayar setelah stok dijual ulang** | `custom_expiry` menyamai TTL saat charge + `POST /v2/{order_id}/expire` dari sweep (Bagian 7, Bagian 9) |
| Form kartu kelak mengirim nomor kartu ke backend | Runtuhkan NFR-07 dan perluas cakupan PCI-DSS. Tidak ada endpoint menerima field kartu (Bagian 8) |
| Outbox retry menerbitkan QR ganda | `UNIQUE (booking_item_id)` pada `tickets` (Bagian 4) |
| Submit ulang `/bookings` menolak NIK sendiri | Guard `FOR UPDATE` mengembalikan booking yang ada (Bagian 8) |
| Task ECS tertinggal menyala setelah sale | Alarm `IdleTaskCount > 0` di luar jendela sale (Bagian 11) |
| Kuota Fargate vCPU / Lambda concurrency | Service Quota Request T-7 hari |
| ALB ramp 5.000 conn/s di T=0 | Uji pada ramp |
| Mobile background tidak menerima grant | Grace 120s + `readmit` maks 2× |
| Redis Pub/Sub kehilangan pesan saat failover | Resync + re-broadcast; `status {stale:true}`; heartbeat 10 detik |
| `readmit` livelock | Batas 2× per token; `EXPIRE` + job post-sale |
| Kontensi kategori saat tekan Pay | Aturan gagal-hold: slot tidak hilang (Bagian 8) |
| NIK terkunci oleh booking kedaluwarsa | `DEL nik:used` pada jalur expiry (Bagian 8) |
| PII pihak ketiga ke API eksternal | Persetujuan dicatat terpisah dari T&C; HMAC + KMS; retensi (Bagian 17) |

---

## 17. Yang Masih Terbuka

| # | Item | Pemilik |
|---|---|---|
| 1 | `DUKCAPIL_RATE_LIMIT_RPS` dan **SLA ketersediaan** Dukcapil | **Pemilik tender / Dukcapil** |
| 2 | Biaya per panggilan Dukcapil | **Pemilik tender** |
| 3 | Cap antrian / pesan jujur saat 300.000 : 10.000 tiket | **Pemilik tender** |
| 4 | Perlakuan TTL customer yang sedang aktif saat admission di-pause — biarkan berjalan, bekukan selama pause, atau geser tenggat sebesar durasi pause (dengan batas atas, karena `grant:{token}` hanya hidup 1200 detik) | **Pemilik tender** |
| 5 | Kapasitas koneksi per ECS task | Tim — **T-14 hari** |
| 6 | `MAX_CONCURRENT_SLOTS` + maksimum okupansi slot | Tim |
| 7 | Kebijakan retensi PII (NIK, `contact_email`, `holder_email`) | Tim |
| 8 | RPO / RTO | Tim |
| 9 | Daftar skenario load test + threshold | Tim + QA |
| 10 | Aksesibilitas (`aria-live` pada angka antrian dan banner pause) | Tim FE |

**Diputuskan dan ditutup**: transport SSE; perlakuan Dukcapil fail-closed; penghapusan WAF
untuk POC; TTL berurutan dengan anchor `slot_claimed_at` dan `held_at`; target kapasitas
300.000; Dukcapil memvalidasi NIK saja; penomoran kursi tidak diimplementasikan di POC —
tiket berisi QR code, kategori, dan NIK pemegang; push notification TIDAK dibangun di POC;
Session_TTL_Antrian TIDAK dipakai — grace period setelah alokasi slot sudah menutupinya;
refund disputed dicatat dan direkonsiliasi manual, tidak otomatis;
bukti verifikasi NIK berlaku permanen tanpa revalidasi berkala; kontrak API per endpoint — `api-contracts.md`
v2.0 berstatus Accepted dan mencakup seluruh endpoint.

---

## Referensi

- `requirement/requirements.md` v1.0 (Finalized) — FR-01..35, NFR-01..10, BR-01..10
- `Tender Rusuh Booking Ticket.pdf` — baseline
- `requirement/update-requirement-rusuh-booking-tiket.md` — login opsional, BR-01/BR-02, BR-07
- `payment-method/` — tiga metode Midtrans Core API, diagram alur dan tautan sumber
- [Midtrans Core API](https://docs.midtrans.com/docs/custom-interface-core-api) ·
  [Transaction Status Cycle](https://docs.midtrans.com/docs/transaction-status-cycle) ·
  [Signature notifikasi](https://docs.midtrans.com/reference/receiving-notifications) ·
  [custom_expiry](https://docs.midtrans.com/docs/coreapi-advanced-features)
- ADR-001 — Teknologi Antrian untuk Virtual Waiting Room
- ADR-002 — Mekanisme Penentuan Posisi Antrian per Customer
- ADR-003 — Mekanisme Pengiriman Update Posisi Antrian ke Client *(superseded by ADR-005)*
- ADR-004 — Pemilihan Compute Service *(superseded by ADR-005)*
- [HTML Living Standard: Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
  — `retry:`, `Last-Event-ID`, dan perilaku *fail the connection* pada status non-200
