# ADR-005: Keputusan Arsitektur Final — Server-Sent Events + ECS Fargate

**Status**: Accepted — menunggu keputusan pemilik tender (Bagian 17 #1–3)
**Tanggal**: 24 Oktober 2026
**Decider**: SJT Team
**Supersedes**: ADR-003, ADR-004
**Amended by**: ADR-006 (antrian: pending grant, formula admission, hitungan dari Aurora, satu penulis grant, broadcast saja, **pause admission dihapus**, estimasi waktu tunggu di frontend). Semua penyebutan `paused`, "pause admission", BR-13a, BR-22 dan `PAUSE_TTL_GRACE_SECONDS` di dokumen ini **tidak berlaku lagi** (ADR-006 §4.3)
> ⚠️ **Diubah oleh ADR-006 §8.10** — di seluruh dokumen ini, status antrian `SOLD_OUT` dibaca **`FULL_BOOKED`**
> (`sold_out_at` → `full_booked_at`, `SOLD_OUT_HOLD_SECONDS` → `FULL_BOOKED_HOLD_SECONDS`, error join `SOLD_OUT` → `FULL_BOOKED`).
> `CLOSED`, event SSE `sold-out` dan `SALE_CLOSED` tidak berubah. Teks asli di bawah dibiarkan apa adanya.

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
| Autentikasi stream | Cookie `HttpOnly` `queue_token` (web) / header `X-Queue-Token` (mobile) |
| Verifikasi NIK | **Dukcapil API**, NIK saja. **Gagal apa pun sebabnya → tidak bisa bayar** |
| Dukcapil tidak tersedia | **Fail closed** + pause admission. Dukcapil adalah SPOF yang diterima |
| Pembayaran | Charge sinkron; efek samping **inline best-effort + outbox fallback** |
| Konfirmasi pembayaran | **`POST /payments/webhook`** dari gateway |
| Email | **Outbox → SQS → SES**, satu email per pemegang NIK, kegagalan tidak membatalkan transaksi |
| Sumber kebenaran deadline | **Aurora** (transaksional); ZSET hanya penjadwal |
| Anchor TTL | **Serial**: booking = `granted_at` +5 menit; pembayaran = `held_at` +10 menit (BR-03, BR-05). Tidak ada grace antrian — jam booking mulai saat slot diberikan |
| Proteksi dependensi eksternal | Rate limiter + circuit breaker terdistribusi (Redis) |
| WAF | **Tidak dipakai di POC** — digantikan rate limit Redis + CAPTCHA tak terlihat di `/join`; sisa celahnya di Bagian 16 |
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

> ⚠️ **Diubah oleh ADR-006 §8.12** — `queue:{ev}:state`, `:issued`, `:full_booked_at` kini field hash `queue:{ev}`; `queue:{ev}:resume` dan `expiries:{ev}` dihapus (sweep tenggat membaca Aurora). Berlaku untuk seluruh dokumen ini.
>
> ⚠️ **Diubah oleh ADR-006** — diagram di bagian ini masih menampilkan `regrant`, `slots:open` dan `sse-grant`; ketiganya dihapus (ADR-006 §2, §3, §4).

```
  ┌──────────────────────────────┐              ┌──────────────────────────────┐
  │         Web Browser          │              │         Mobile App           │
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
  │  i18n: Accept-Language       │              │  idem                   FR-32│
  └──────────────┬───────────────┘              └──────────────┬───────────────┘
                 └─────────────────────────┬────────────────────┘
               ┌─────────────────────────┴────────────────────────────┐
               ▼                                                       ▼
  ┌──────────────────────────────────┐      ┌──────────────────────────────────┐
  │  ALB — events.my-rusuh.my.id     │      │  API Gateway — api.my-rusuh.my.id│
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
  │                                  │      │ GET  /payments/{id}/methods FR-20│
  │                                  │      │ POST /payments/{id}         FR-20│
  │                                  │      │ POST /payments/{id}/retry        │
  │                                  │      │ GET  /payments/{id}/status  FR-21│
  │                                  │      │ POST /payments/webhook      FR-25│
  │  CAPTCHA tak terlihat (/join):   │      │                                  │
  │   token diterbitkan PRA-gerbang  │      │                                  │
  │   rate limit dulu, lalu verify   │      │                                  │
  │   provider mati → FAIL-OPEN      │      │                                  │
  │                                  │      │ /payments/webhook:               │
  │                                  │      │  RESOURCE POLICY allowlist IP    │
  │                                  │      │  gateway + verifikasi HMAC       │
  │                                  │      │  (dulu rule WAF)                 │
  └───────────────┬──────────────────┘      └──────────────────────────────────┘
                  │
  ┌───────────────▼──────────────────────────────────────────────────────────┐
  │  ECS Fargate: rusuh-sse-server    300.000 koneksi · max_tasks 60         │
  │  2 vCPU / 4 GB per task · N = 300.000 ÷ kapasitas TERUKUR (Bagian 11)    │
  │                                                                          │
  │  connections: Map<eventId, Set<Response>> · byToken: Map<token, Response>│
  │                                                                          │
  │   GET /events/{ev}/queue/stream                                          │
  │     0. GET queue:{ev}:state                                              │
  │        ∈ SCHEDULED | OPEN | SOLD_OUT | CLOSED                            │
  │        bila CLOSED    → HTTP 410 Gone, JANGAN buka stream                │
  │        bila SCHEDULED → HTTP 409, sale belum dibuka                      │
  │        bila SOLD_OUT  → BUKA stream seperti biasa — stok dapat           │
  │                         kembali dari booking yang kedaluwarsa            │
  │        ← status non-200 membuat EventSource berhenti permanen,           │
  │          mencegah badai reconnect setelah sale benar-benar habis         │
  │     1. token dari cookie/header → HGET token:{uuid}                      │
  │     2. headers: text/event-stream · no-cache · X-Accel-Buffering: no     │
  │     3. SNAPSHOT AWAL: { nowServing, state, ratePerMin, paused,           │
  │        serverTime } + id    ← FR-10: posisi utuh segera setelah refresh  │
  │     4. `: keepalive` 15 detik · close → hapus dari KEDUA map             │
  │                                                                          │
  │   HANDLER PER CHANNEL                                                    │
  │     sse-broadcast → fan-out ter-chunk; payload membawa nowServing,       │
  │                     state, ratePerMin, paused, serverTime                │
  │     sse-grant     → byToken.get(token)?.write("event: grant\ndata:{}")   │
  │     sse-sold-out  → fan-out "event: sold-out", lalu res.end().           │
  │                     Klien WAJIB es.close(); klien lama yang tetap        │
  │                     reconnect menerima 410 pada langkah 0 dan berhenti.  │
  │     Redis putus   → "event: status" {"stale":true}                       │
  │     Redis kembali → GET now_serving → re-broadcast + {"stale":false}     │
  │     TIDAK ada sse-broadcast > SSE_STALE_AFTER_SECONDS, sementara         │
  │       state ∈ {OPEN, SOLD_OUT} → "event: status"                         │
  │       {"stale":true,"reason":"no_admission_updates"}                     │
  │       ← admission-ctl diam; Redis dan sse-server sendiri sehat,          │
  │         jadi tidak ada jalur lain yang menyalakannya (Bagian 11)         │
  │                                                                          │
  │   SIGTERM → retry berjitter 1–15 detik + event reconnect (Bagian 11)     │
  │             ← di sini reconnect otomatis justru DIINGINKAN               │
  └────────────────────────────────┬─────────────────────────────────────────┘
  ┌────────────────────────────────▼─────────────────────────────────────────┐
  │  ECS Fargate: rusuh-admission-ctl   desired_count 2 · 0.5 vCPU · 500ms   │
  │   1 leader + 1 standby · leader lease di Redis (Bagian 11)               │
  │  -1. SET ctl:leader {task_id} NX EX 5   (atau perpanjang milik kita)     │
  │      BUKAN leader → lewati seluruh tick; standby diam tetapi hangat      │
  │   0. paused := (breaker payment gateway == OPEN)                         │
  │                OR (breaker Dukcapil == OPEN)        ← BARU, Bagian 8     │
  │      PUBLISH sse-broadcast bila nowServing berubah, ATAU paused berubah, │
  │      ATAU 10 detik berlalu (heartbeat) ← resume otomatis                 │
  │      Heartbeat TIDAK bersyarat: terbit juga saat paused dan saat         │
  │      SOLD_OUT. Diam berarti loop mati, bukan "tidak ada kabar" —         │
  │      itu yang membuat detektor stale dapat dipercaya (Bagian 11)         │
  │   1. SWEEP   ZRANGEBYSCORE expiries:{ev} 0 now  → VERIFIKASI ke Aurora   │
  │              sebelum meng-expire (Bagian 7)                              │
  │   2. ADVANCE bila paused → LEWATI SELURUH LANGKAH INI                    │
  │              (tanpa ini, banner pause tampil sementara slot              │
  │               terus dibagikan — pelanggaran NFR-02)                      │
  │              EVALSHA advance_now_serving.lua                             │
  │              → batch INSERT bookings(status='GRANTED', granted_at=now()) │
  │                booking_expires_at = granted_at + BOOKING_TTL             │
  │              → ZADD expiries:{ev} booking_expires_at booking_id          │
  │              → gagal INSERT setelah retry: DECRBY slots:open,            │
  │                SADD regrant, JANGAN publish grant (Bagian 6)             │
  │   3. PUBLISH sse-broadcast / sse-grant / sse-sold-out                    │
  │   4. REKONSILIASI tiap tick ke-20 (≈10 detik) — Bagian 6                 │
  │   5. bila state=SOLD_OUT dan now > sold_out_at +                         │
  │      SOLD_OUT_HOLD_SECONDS → SET state CLOSED,                           │
  │      PUBLISH sse-sold-out (Bagian 6)                                     │
  │   6. DRAIN qpersist:{ev} → queue_entries (batch, TERAKHIR,               │
  │      beranggaran QPERSIST_DRAIN_BUDGET_MS) — Bagian 6                    │
  └────────────────────────────────┬─────────────────────────────────────────┘
   ┌───────────────────────────────▼──────────────────────────────────────────┐
   │  REDIS — FAST PATH · PUB/SUB · RATE LIMIT · BREAKER · LEASE              │
   │  QUEUE    queue:{ev}:issued · now_serving · state · slots:open:{ev}      │
   │           token:{uuid} (ticket_no, event_id, account_id?,               │
   │                         device_hash?, rebinds?)                         │
   │             EX SALE_WINDOW_SECONDS  -- kunci yang DIANDALKAN recovery;   │
   │             di-DEL pada EXPIRED/ABANDONED/CONFIRMED, TIDAK pada LAPSED   │
   │           settled:{uuid} (booking_id) EX SETTLED_TOMBSTONE_SECONDS       │
   │             -- nisan CONFIRMED; menjawab §3.3, tidak memulihkan apa pun  │
   │           acct:{ev}:{account_id} NX   -- BR-07, hanya user login         │
   │           qpersist:{ev} (LIST) -- buffer tulis durabel; panjang = RPO    │
   │           by_ticket (ZSET score=ticket_no) · regrant:{ev}               │
   │             -- regrant = INSERT grant GAGAL, bukan no-show (Bagian 7)    │
   │           grant:{token} EX GRANT_TOKEN_TTL_SECONDS (1080)                │
   │  BOOKING  stock:{ev}:{cat} · nik:used:{ev}:{hmac} NX                     │
   │           nik:valid:{hmac} EX 86400   -- cache verifikasi Dukcapil       │
   │  QUEUECAP queue:{ev}:capacity · depth = issued − now_serving             │
   │           expiries:{ev} (ZSET) — PENJADWAL, bukan sumber kebenaran       │
   │  CONTROL  ctl:leader (NX EX 5) · ctl:epoch (INCR, fencing token)         │
   │           pause:{ev}:grace_applied NX                                    │
   │  DEPEND   rl:{service} · breaker:{service}:state/failures                │
   │  INGRESS  rl:join:{ip} · rl:sse:{ip} · captcha ← pengganti WAF           │
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
   │     -- granted_at MENENTUKAN booking_expires_at. slot_claimed_at         │
   │     -- hanya dicatat untuk metrik konversi, BUKAN otoritatif.            │
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
   │                                                                          │
   │   -- POSISI ANTRIAN DURABEL (RPO; ditulis async, Bagian 6)               │
   │   queue_entries(event_id, queue_token, ticket_no,                        │
   │                 account_id NULL, joined_at)                              │
   │   PRIMARY KEY (event_id, queue_token)                                    │
   │   UNIQUE (event_id, ticket_no)                                           │
   │   INDEX  (event_id, account_id) WHERE account_id IS NOT NULL             │
   │                                                                          │
   │   event_queue_state(event_id, state, sold_out_at)                        │
   │   -- hanya ditulis saat TRANSISI state; now_serving TIDAK disimpan       │
   │   -- di sini, ia diturunkan saat rebuild (Bagian 6)                      │
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
    RegrantSetSize · SSEWriteBackpressure · OutboxLag · OutboxInlineHitRate
    CaptchaRejected · CaptchaVerifyErrors · CaptchaFailOpenSeconds · Br07JoinRecovered
    SoldOutResumed · SoldOutHoldSeconds · TicketsResoldAfterSoldOut
    PauseTtlGrantedSeconds · PauseTransitions
    CtlLeaderHandovers · CtlHeartbeatAge · StockAdjustRejected · StaleLeaderRejected
    QPersistLag · QPersistAgeSeconds · QPersistBatchFailures
    DukcapilLatency p95 · DukcapilCircuitState · DukcapilThrottled
    NikRejectedRate · DukcapilBlockedSales · PaymentGatewayLatency p95
    PGCircuitState · PGThrottled · AdmissionPausedSeconds · WebhookLag · EmailDLQDepth
```

> ⚠️ **Diubah oleh ADR-006 §8.15** — `booking_items.holder_email` dihapus: e-tiket seluruh booking dikirim ke `bookings.contact_email` (FR-26 diubah).

---

## 5. Halaman Utama dan Jalur Masuk

> ⚠️ **Diubah oleh ADR-006** — bypass FR-05 tidak lagi menjalankan Lua di Lambda join; join tidak lagi menunggu grant sama sekali (ADR-006 §4.1, §8.14): tidak ada jalur langsung-masuk, customer dimasukkan oleh tick admission-ctl berikutnya.

| Requirement | Implementasi |
|---|---|
| FR-01 daftar event | `GET /events` — Lambda + cache CloudFront 30 detik. Halaman utama dapat di-cache; ia tidak punya konsep freshness per-customer |
| FR-02 event aktif | `status = 'available' AND stock > 0` → `clickable: true` |
| FR-03 event sold out | `status = SOLD_OUT` (dari `available == 0`) → `display.enabled = false`, `label_key = sold_out` — Bagian 6 |
| FR-04 ada antrian → Waiting Room | `POST /events/{ev}/queue/join` mengembalikan `{ticket_no, nowServing, streamUrl}` |
| **FR-05 tidak ada antrian → langsung Detail** | `POST /events/{ev}/queue/join` mengecek `issued − now_serving == 0 AND slots:open < MAX AND stock > 0`. Bila benar, Lua memberi grant pada permintaan yang sama dan respons berisi `{granted: true, grantToken}` — klien langsung ke Detail Ticket Page tanpa pernah membuka stream |

Jalur bypass memakai Lua yang sama (`advance_now_serving.lua` dengan `deficit = 1`), sehingga
tidak ada jalur admission kedua yang bisa menyimpang dari yang utama.

---

## 6. Satu Jalur Admission, dan Integritas `slots:open`

> ⚠️ **Diubah oleh ADR-006** — Lua `advance_now_serving.lua` di bawah **digantikan** oleh ADR-006 §3.4 (tanpa `ZREM`, tanpa `regrant`, formula `min(MAX − open, ceil(stock/3) − browsing)`, `now_serving ≤ issued`, pagar CAS). **Jangan menyalin Lua di bagian ini.**

Saat `POST /events/{ev}/queue/join`, dalam urutan ini:

```
1. rate limit  rl:join:{ip}                  -- Redis lokal, murah
2. verifikasi CAPTCHA (token dari klien)     -- panggilan eksternal
3. BR-07, hanya bila pemanggil login:
     SET acct:{ev}:{account_id} {uuid} NX EX SALE_WINDOW_SECONDS
     gagal → BUKAN error. Ambil {uuid} yang sudah ada, terbitkan ulang
             cookie untuk token itu, dan kembalikan keadaannya (200).
4. INCR queue:{ev}:issued → ticket_no
5. HSET token:{uuid} ; ZADD queue:{ev}:by_ticket {ticket_no} {uuid}
     -- untuk pemanggil ANONIM, HSET menyertakan
     --   device_hash = HMAC(DEVICE_PEPPER, header X-Device-Id)
     -- Pemanggil login tidak: account_id sudah mengikat lebih kuat (Bagian 12).
6. RPUSH qpersist:{ev} <queue_token|ticket_no|account_id|joined_at>
     -- satu RPUSH, tetap di jalur cepat Redis. Aurora TIDAK disentuh di
     -- sini: 300.000 join dalam ~60 detik = ~5.000 insert/detik tepat pada
     -- momen beban tertinggi, dan itu akan menjadikan Aurora SPOF baru di
     -- satu-satunya jalur yang sengaja dibuat Redis-only. Penulisannya
     -- di-batch oleh leader admission-ctl (Bagian 6).
```

**Urutan 1 sebelum 2 itu wajib.** Rate limit adalah panggilan Redis lokal; verifikasi CAPTCHA
adalah round trip ke pihak ketiga. Verifikasi dulu lalu membatasi berarti membayar satu
panggilan eksternal untuk setiap percobaan yang akhirnya ditolak — penyerang memilih biaya
kita, bukan kita.

### CAPTCHA di `/join`: tak terlihat, diterbitkan sebelum gerbang buka

Vektor bot yang nyata ada di `/join` — memborong **posisi antrian**, bukan tiket (tiket sudah
dibatasi BR-02). Karena itu CAPTCHA diletakkan di sini, bukan sebelum Detail Ticket Page:
di sana bot sudah mengambil posisinya, dan gesekannya justru memakan jam booking.

> **CAPTCHA interaktif di `/join` akan MEMBALIK keadilan yang ia lindungi.** `ticket_no`
> berasal dari `INCR issued` pada saat join. Manusia yang menghabiskan tiga detik pada teka-
> teki visual sementara bot menembus layanan pemecah dalam 200 ms berarti **bot mendapat
> nomor antrian lebih kecil**. Tantangan interaktif tepat di titik itu menghukum manusia
> dalam urutan antrian.

Dua hal yang membuatnya aman:

1. **Hanya mode tak terlihat / berbasis skor.** Tidak pernah teka-teki interaktif di `/join`.
2. **Token diterbitkan sebelum gerbang buka.** Halaman utama pra-buka (`queue_state:
   NOT_OPEN` + `sale_starts_at`, Bagian 6) mengambil token saat halaman dimuat dan
   menyegarkannya tiap ~60 detik selama menunggu, sehingga pada `sale_starts_at` klien sudah
   memegang token segar. Join hanya membawanya; verifikasi murni server-side, jadi tidak ada
   komponen berkecepatan-manusia yang masuk ke dalam perlombaan. Token CAPTCHA berumur
   pendek dan sekali pakai, jadi loop penyegaran itu **wajib**, bukan opsional.

**Fail-open, berbeda dari Dukcapil.** Provider CAPTCHA adalah dependensi eksternal **ketiga**,
dan ia duduk di endpoint dengan trafik tertinggi. Aturannya:

> **Fail-closed bila yang dipertaruhkan kebenaran; fail-open bila hanya keadilan.**

Dukcapil fail-open akan menerbitkan tiket atas NIK yang belum terverifikasi — kebenaran.
CAPTCHA fail-open hanya kehilangan gesekan terhadap bot: tidak ada stok yang oversell, tidak
ada NIK yang lolos, tidak ada uang yang bergerak. Fail-closed di sini berarti **tidak seorang
pun dapat bergabung** saat provider terganggu — sale mati sebelum dimulai, lebih buruk
daripada gangguan Dukcapil yang setidaknya masih membiarkan antrian terbentuk.

`CaptchaFailOpenSeconds` dialarmkan sehingga jendela tanpa proteksi terlihat, bukan diam.

**Berlaku untuk tamu DAN user login.** Mengecualikan user login tampak masuk akal — mereka
teridentifikasi — tetapi akun hanya berbiaya 5 per IP per jam, sehingga pengecualian itu
menjadikan login sebagai jalur yang ramah bot. Itu persis kebalikan dari celah BR-07 di bawah.

### BR-07: satu keterlibatan aktif per event, hanya untuk user login

`acct:{ev}:{account_id}` memegang satu keterlibatan aktif per akun per event. Ia dilepas saat
keterlibatan itu mencapai keadaan terminal:

| Keadaan | `acct` key | Akibat |
|---|---|---|
| QUEUED, GRANTED, SELECTING, HELD, AWAITING_PAYMENT, PENDING_CANCELLATION | tetap | join kedua memulihkan yang ada, bukan membuat yang kedua |
| **CONFIRMED** | `DEL` | **boleh antre lagi** untuk membeli lagi |
| EXPIRED, LAPSED, ABANDONED | `DEL` | boleh antre lagi |

> **`LAPSED` melepas `acct` tanpa syarat, dan itu benar hanya karena `LAPSED` final.**
> Selama `LAPSED` masih memberi kesempatan kedua, baris ini adalah lubang BR-07: token yang
> sama menunggu grant keduanya sementara akunnya sudah boleh join lagi — dua keterlibatan
> hidup untuk satu akun. Menghapus kesempatan kedua (Bagian 7) menghapus kondisinya: tidak
> ada lagi keadaan "LAPSED tapi masih menunggu", jadi umur `acct` sama dengan umur
> keterlibatannya tanpa percabangan apa pun.
>
> Bila kesempatan kedua itu kelak dipertimbangkan lagi, **baris ini harus dipecah dulu** —
> `DEL acct` hanya boleh terjadi ketika token benar-benar dibuang dari antrian, bukan ketika
> booking-nya terminal.

> **Join kedua memulihkan, tidak menolak.** Menjawab 409 akan menghukum customer yang hanya
> berpindah perangkat — login di ponsel setelah antre di laptop. Karena `acct:{ev}:{id}`
> menyimpan `{uuid}` token yang ada, join kedua dapat menerbitkan ulang cookie untuk token
> itu dan mengembalikan keadaannya. BR-07 tetap ditegakkan: ia tidak mendapat keterlibatan
> kedua. Sebagai efek sampingnya, user login mendapat pemulihan lintas perangkat yang tidak
> tersedia bagi tamu.

> **Antre lagi berarti dari belakang.** `ticket_no` baru datang dari `INCR issued`; posisi
> lama tidak dipulihkan. Kuota tetap dijaga BR-02 — pembelian kedua menuntut NIK yang
> berbeda lagi, karena satu NIK hanya boleh satu kursi per event.

**Backstop di Aurora**, karena `acct` adalah kunci Redis dan dapat hilang pada flush/failover:

```sql
CREATE UNIQUE INDEX br07_one_active_per_event
    ON bookings (event_id, account_id)
 WHERE account_id IS NOT NULL
   AND status IN ('GRANTED','SELECTING','HELD',
                  'AWAITING_PAYMENT','PENDING_CANCELLATION');
```

Indeks parsial ini mengecualikan `account_id IS NULL`, sehingga **customer tamu tidak
terbatas** — sesuai keputusan. Ia juga mengecualikan status terminal, sehingga pembeli yang
sudah `CONFIRMED` bebas memulai pembelian baru. Pola Redis-cepat/Aurora-otoritatif ini sama
dengan `nik:used` terhadap `nik_claims` (Bagian 6).

> **Kelemahan sisa, dinyatakan terang-terangan.** BR-07 hanya dapat ditegakkan terhadap user
> yang teridentifikasi, sehingga strategi optimal pihak yang berniat buruk menjadi **jangan
> login**. Ia membatasi pengguna yang kooperatif dan bukan yang adversarial. Itu dapat
> diterima **hanya** karena CAPTCHA di `/join` kini menjadi proteksi nyata bagi semua orang;
> BR-07 sendirian akan hampir sekadar hiasan.

```lua
-- advance_now_serving.lua   (SCRIPT LOAD sekali, dipanggil via EVALSHA)
-- KEYS[1]=now_serving KEYS[2]=slots:open KEYS[3]=state KEYS[4]=regrant KEYS[5]=by_ticket
-- KEYS[6]=sold_out_at   KEYS[7]=resume_flag
-- ARGV[1]=MAX_CONCURRENT_SLOTS  ARGV[2]=remaining_stock  ARGV[3]=now (epoch detik)
-- ARGV[2] adalah SISA STOK TOTAL seluruh kategori, bukan per kategori: admission-ctl
-- tidak tahu kategori apa yang akan dipilih customer yang belum masuk.

local stock = tonumber(ARGV[2])
local state = redis.call('GET', KEYS[3])

if stock <= 0 then
  -- SEMENTARA, bukan terminal. CLOSED disetel admission-ctl setelah
  -- SOLD_OUT_HOLD_SECONDS, bukan di sini -- lihat Bagian 6.
  if state == 'OPEN' then
    redis.call('SET', KEYS[3], 'SOLD_OUT')
    redis.call('SET', KEYS[6], ARGV[3])        -- sold_out_at
  end
  return {0, 0, 'SOLD_OUT', {}}
end

-- Stok kembali (booking kedaluwarsa melepas kursinya): lanjutkan admission.
-- by_ticket dan now_serving tidak disentuh, jadi FCFS melanjutkan dari posisi
-- yang sama alih-alih membuat pendatang baru bersaing dengan yang menunggu.
if state == 'SOLD_OUT' then
  redis.call('SET', KEYS[3], 'OPEN')
  redis.call('DEL', KEYS[6])
  redis.call('SET', KEYS[7], 1)                -- tanda: broadcast state OPEN
end
if state == 'CLOSED' then return {0, 0, 'CLOSED', {}} end

local open    = tonumber(redis.call('GET', KEYS[2]) or 0)
local target  = math.min(tonumber(ARGV[1]), stock)
local deficit = target - open
if deficit <= 0 then return {0, 0, 'NOOP', {}} end

local tokens = {}
-- regrant lebih dulu: token di sini sudah dilewati watermark dan INSERT-nya
-- gagal, jadi ia belum pernah benar-benar dilayani. TIDAK ada no-show di sini —
-- LAPSED bersifat final sejak kesempatan kedua dihapus (Bagian 7).
local regranted = redis.call('SPOP', KEYS[4], deficit)
for i = 1, #regranted do tokens[#tokens + 1] = regranted[i] end
if #regranted > 0 then
  redis.call('INCRBY', KEYS[2], #regranted)
  deficit = deficit - #regranted
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

> ⚠️ **Diubah oleh ADR-006** — kompensasi `DECRBY slots:open` + `SADD regrant` digantikan oleh pending grant yang diulang tanpa batas; INSERT idempoten via `UNIQUE (event_id, queue_token)` (ADR-006 §2).

Prasyarat rekonsiliasi (tanpa baris ini `COUNT(*)` tidak melihat slot granted-unclaimed,
sehingga rekonsiliasi justru menyebabkan over-admission) dan penutup celah rebuild.

**`account_id` ditulis di sini, bukan di `POST /bookings`.** `token:{uuid}` sudah memuat
`account_id` sejak join (langkah 3 dan 5 di atas), jadi INSERT grant mengisinya langsung:

```
INSERT INTO bookings (id, event_id, grant_token, account_id, status, granted_at, …)
VALUES (…, HGET token:{uuid} account_id, 'GRANTED', now(), …)
     -- NULL untuk tamu; terisi untuk customer login, tanpa bertanya pada klien
```

> **Atribusi tidak boleh bergantung pada token yang dibawa klien saat checkout.** Kontrak
> v2.3 menerima `Authorization` opsional pada `POST /bookings` justru untuk mengisi kolom ini.
> Itu rapuh dengan cara yang gagal diam-diam: bila sesi akun mati di ruang tunggu — dan
> sebelum `ACCESS_TOKEN_TTL_SECONDS` diturunkan ulang itu terjadi pada hampir seluruh antrian
> — klien tidak mengirim apa pun, baris tertulis `account_id = NULL`, dan pembelian customer
> login tercatat sebagai pembelian tamu. Tanpa error, tanpa 401, tanpa jejak.
>
> Keterlibatan tahu identitasnya sejak join, ketika sesi masih pasti segar. Mengambilnya dari
> sana membuat atribusi **tidak bergantung pada jam klien sama sekali**, dan membuat
> `Authorization` pada §4/§5 tidak punya pekerjaan lagi — kontrak v2.4 menghapusnya.
>
> Efek samping: indeks parsial `ON bookings (event_id, account_id) WHERE account_id IS NOT
> NULL` mulai mencakup booking customer login yang sebelumnya NULL, sehingga penegakan BR-07
> justru **menguat**.

**Kompensasi bila batch INSERT gagal:** retry hingga `GRANT_INSERT_MAX_RETRIES`; bila tetap
gagal → `DECRBY slots:open`, `SADD regrant`, **jangan publish** `sse-grant`, emit
`GrantInsertFailures`. Urutan wajib **INSERT dulu, PUBLISH kemudian** — mem-publish lebih
dahulu berarti memberi tahu customer gilirannya tiba untuk slot yang kemudian dikompensasi,
pelanggaran NFR-02 "Jangan PHP" paling langsung.

### Kepemilikan `slots:open`

> ⚠️ **Diubah oleh ADR-006** — `slots:open:{ev}` dihapus. `open` dan `browsing` dihitung dari Aurora tiap tick; tidak ada komponen yang men-DECR penghitung antrian (ADR-006 §3).

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
| GRANTED → LAPSED | `rusuh-admission-ctl` | ✅ Terminal, final — tidak ada kesempatan kedua (Bagian 7) |
| apa pun → ABANDONED | `booking-service` | ✅ Terminal + `DEL nik:used` + `DEL acct:{ev}:{id}` |

Double-DECR → `slots:open` negatif → `deficit = target − (−n)` → over-admission.

### Rekonsiliasi

> ⚠️ **Diubah oleh ADR-006** — rekonsiliasi `slots:open` tidak berlaku lagi karena penghitungnya dihapus; `COUNT` yang sama kini dipakai langsung tiap tick untuk admission (ADR-006 §3). Rekonsiliasi stok di bawah tetap berlaku.

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
      -- WAJIB IMUTABEL SELAMA SALE. Seluruh perhitungan drift diukur terhadap
      -- angka ini; mengubahnya di tengah sale menggeser garis dasar dan membuat
      -- koreksi berikutnya salah arah. Perubahan kapasitas hanya di luar jendela
      -- sale, dan rekonsiliasi harus di-reset (last_drift = 0) sesudahnya.
    − SELECT COUNT(*) FROM booking_items bi JOIN bookings b ON b.id = bi.booking_id
        WHERE bi.category = $cat
          AND b.status IN ('HELD','AWAITING_PAYMENT','CONFIRMED')     -- WRITER
  R = GET stock:{ev}:{cat}
  current_drift = R - E ;  emit StockDrift

  if R is MISSING (bukan nol):
      SET stock:{ev}:{cat} E        -- SEGERA, tanpa menunggu konfirmasi kedua
      emit StockKeyRestored
      -- Aturan dua-tick ada untuk mencegah koreksi atas drift semu dari lag
      -- replika. Kunci yang HILANG bukan drift — ia menolak setiap hold sampai
      -- ditanam kembali, jadi menunggu 60 detik lagi berarti 60 detik penjualan
      -- mati demi kehati-hatian yang tidak berlaku di sini.
      continue

  if |current_drift| >= STOCK_DRIFT_THRESHOLD and |last_drift| >= STOCK_DRIFT_THRESHOLD:
      -- COMPARE-AND-ADJUST, bukan INCRBY buta. Lua membaca ulang stok dan
      -- hanya menerapkan delta bila nilainya MASIH sama dengan R yang diamati.
      -- Dengan begitu dua eksekusi bersamaan tidak mengoreksi 2x, bahkan bila
      -- leader lease kalah balapan (Bagian 11).
      EVALSHA adjust_if_unchanged.lua  KEYS=stock:{ev}:{cat}
                                       ARGV=R, -current_drift
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

### Dua sumbu, dua field: `status` dan `queue_state`

`GET /events` memisahkan **stok** dari **izin masuk**, karena keduanya bergerak sendiri-sendiri
dan pada puncak sale sering berlawanan.

`status` — hanya stok:

| Nilai | Kondisi |
|---|---|
| `AVAILABLE` | `available > threshold` |
| `SELLING_OUT_FAST` | `0 < available <= threshold` |
| `SOLD_OUT` | `available == 0` |

`threshold = capacity × SELLING_FAST_THRESHOLD_PCT / 100` (bawaan **20**). Ambang persentase,
bukan laju penjualan: laju memerlukan jendela waktu yang harus ditentukan, lebih berisik, dan
lebih sulit dipertanggungjawabkan dalam tender.

`queue_state` — hanya izin masuk:

| Nilai | Kondisi | Tombol |
|---|---|---|
| `NOT_OPEN` | `queue:{ev}:state == SCHEDULED` | Nonaktif |
| `FULL` | `depth >= QUEUE_CAPACITY` | Nonaktif, "Antrian penuh" |
| `DIRECT` | `depth == 0` dan masih ada slot bebas | "Beli Tiket" — FR-05 |
| `OPEN` | sisanya | "Masuk Antrian" |

> **`CLOSED` dihapus.** Ia didefinisikan sebagai "event sold out", jadi ia melaporkan urusan
> field lain — persis percampuran yang pemisahan ini dimaksudkan untuk menghentikan. Sold-out
> kini hanya dikatakan oleh `status`.
>
> **`DIRECT` ditambahkan.** FR-05 menuntut "jika tidak ada antrian → langsung ke Detail Ticket
> Page", dan sebelumnya `OPEN` membawa dua arti sekaligus meskipun label tombolnya berbeda.

**Mengapa dua field, bukan satu enum.** Pada puncak sale kombinasi ini normal, bukan tepi:

| `status` | `queue_state` | Artinya |
|---|---|---|
| `AVAILABLE` | `FULL` | tiket masih ada, antrian sudah 300.000 |
| `SELLING_OUT_FAST` | `FULL` | tiket hampir habis, antrian sudah 300.000 |

Satu enum harus memilih salah satu kata. Pilih `QUEUE_FULL` dan customer tidak tahu masih ada
stok, sehingga tidak tahu apakah layak mencoba lagi — padahal halaman utama memang dirancang
untuk di-refresh manual. Pilih `SELLING_OUT_FAST` dan ia menekan tombol yang pasti ditolak.

### `display`: server memutuskan, klien merender

`status` dan `queue_state` adalah fakta yang dapat dibaca mesin — aplikasi mobile, analitik,
dan integrasi pihak ketiga membutuhkan keduanya. `display` adalah jawaban yang sudah
diselesaikan server, sehingga frontend tidak memutuskan apa pun:

```
-- urutan ini milik SERVER, bukan frontend
status == SOLD_OUT         → enabled=false  action=NONE           label=sold_out
queue_state == NOT_OPEN    → enabled=false  action=NONE           label=coming_soon
queue_state == FULL        → enabled=false  action=NONE           label=queue_full
queue_state == DIRECT      → enabled=true   action=GO_TO_TICKETS  label=buy_now
queue_state == OPEN        → enabled=true   action=JOIN_QUEUE     label=join_queue

badge_key = selling_out_fast  bila status == SELLING_OUT_FAST DAN display.enabled
          = null              selainnya
```

Satu syarat badge itu menggantikan aturan khusus "sembunyikan saat FULL": lencana urgensi
hanya tampil ketika tombolnya memang dapat ditekan, sehingga `FULL`, `NOT_OPEN` dan `SOLD_OUT`
tercakup sekaligus. Menampilkan "hampir habis" kepada 300.000 orang yang tidak dapat masuk
hanya menambah tekanan pada perilaku yang justru ingin diserap ruang tunggu.

`action` ∈ `JOIN_QUEUE` | `GO_TO_TICKETS` | `NONE` — rute pun bukan keputusan frontend.

**Label dan badge adalah kunci, bukan teks.** FR-32 menuntut dua bahasa; bila server mengirim
teks jadi, ia harus melokalkannya, dan itu memasukkan `Accept-Language` ke dalam cache key
CDN — memecah satu respons yang di-cache 30 detik untuk 300.000 orang menjadi banyak varian.
Kunci + tabel terjemahan di frontend adalah pencarian, bukan logika.

> **Halaman utama menampilkan waktu absolut, bukan hitung mundur.** Hitung mundur pra-buka
> berarti frontend membandingkan `sale_starts_at` dengan jam — logika di frontend, dan keliru
> juga karena respons ini di-cache CDN 30 detik. Label pra-buka berbunyi "Buka 10 Nov 09:00".
> Hitung mundur hidup tetap berada di ruang tunggu, tempat `serverTime` datang pada setiap
> payload SSE.

Menampilkannya di halaman utama berarti customer melihat antrian penuh **sebelum** mengklik,
bukan sesudah. `GET /events` di-cache CDN 30 detik, sehingga `depth` yang ditampilkan adalah
perkiraan; penegaknya tetap pemeriksaan atomik saat join.

### `queue:{ev}:state` dan pembukaan sale

```
SCHEDULED  ← nilai AWAL. Sale belum dibuka; join ditolak 409.
   │
   │  job terjadwal pada sale_starts_at  (EventBridge, satu kali per event)
   ▼
OPEN   ◄───────────────┐   ← antrian menerima join, admission berjalan
   │                  │
   │ stok <= 0        │ stok > 0 lagi (booking kedaluwarsa
   ▼                  │ melepas kursinya) — REVERSIBEL
SOLD_OUT ─────────────┘   ← stream TETAP terbuka; admission lanjut
   │                        begitu stok kembali
   │  sold_out_at + SOLD_OUT_HOLD_SECONDS berlalu
   ▼
CLOSED     ← TERMINAL. sse-sold-out + 410 Gone; klien menutup stream.
```

> **Ini menutup celah, bukan menambah fitur.** Sebelumnya `state` hanya punya dua nilai,
> nilai awalnya tidak pernah dinyatakan, dan tidak ada apa pun yang membukanya — sale tidak
> punya permulaan yang terdefinisi. Untuk sistem flash sale itu justru layar yang paling
> padat: 300.000 orang berkumpul **sebelum** gerbang dibuka, bukan sesudah.

### `SOLD_OUT` reversibel, `CLOSED` terminal

Stok kembali dari booking yang kedaluwarsa (BR-04, BR-06), dan `ExpiryRate` dipantau justru
karena angkanya tidak nol. Membuat sold-out satu arah berarti **setiap kursi yang dilepas
setelah momen itu tidak akan pernah terjual** — kehilangan pendapatan yang langsung
sebanding dengan expiry rate.

Karena itu `SOLD_OUT` adalah keadaan **sementara**, dan keadaan terminalnya dipisah menjadi
`CLOSED`:

| | `SOLD_OUT` | `CLOSED` |
|---|---|---|
| Arti | tidak ada stok **saat ini** | tidak akan ada stok lagi |
| Stream SSE | tetap terbuka, broadcast berlanjut | `sse-sold-out` lalu 410 Gone |
| Admission | lanjut otomatis begitu stok kembali | berhenti |
| Reversibel | ✅ → `OPEN` | ❌ |

**Jendela `SOLD_OUT` punya batas yang diturunkan, bukan ditebak.** Tidak ada hold yang dapat
kedaluwarsa lebih lambat dari satu perjalanan penuh setelah hold terakhir dibuat, sehingga
setelah `BOOKING_TTL + PAYMENT_TTL + PAYMENT_GRACE` (930 detik) terlewat dari momen sold-out,
tidak ada lagi kursi yang mungkin kembali:

```
SOLD_OUT_HOLD_SECONDS = BOOKING_TTL_SECONDS + PAYMENT_TTL_SECONDS
                      + PAYMENT_GRACE_SECONDS = 930
```

Setelah itu admission-ctl menyetel `state = CLOSED`, mem-broadcast `sse-sold-out`, dan stream
menjawab 410 untuk reconnect berikutnya.

> **Klaim "tidak ada badai reconnect" tetap utuh, dan justru lebih kuat.** Badai itu berasal
> dari 300.000 klien yang **diputus** lalu mencoba kembali. Di `SOLD_OUT` mereka tidak diputus
> sama sekali — jadi tidak ada yang perlu reconnect. Pemutusan hanya terjadi sekali, di
> `CLOSED`, dan di sana 410 menghentikan retry bawaan `EventSource` secara permanen seperti
> sebelumnya.
>
> Harganya adalah memegang koneksi hingga 930 detik lebih lama pada event yang sold out.
> Itu biaya egress dan task ECS yang nyata, dan ia dibayar untuk menjual kembali kursi yang
> dilepas — sebanding dengan `ExpiryRate`.

**FCFS tetap utuh melewati batas sold-out.** Token tidak dibuang saat `SOLD_OUT`: `by_ticket`
tidak disentuh, `now_serving` tidak di-reset, dan saat stok kembali admission melanjutkan dari
posisi yang sama. Customer di posisi 5 ketika sold-out terjadi tetap dilayani sebelum posisi
6 — bukan bersaing ulang dengan pendatang baru.

**Tidak ada `Session_TTL_Antrian`.** Yang melepas posisi hanyalah grace period setelah slot
diberikan: customer yang tidak masuk ke Detail Ticket Page sebelum `booking_expires_at`
(`granted_at` + 300 detik) kehilangan
slotnya ke antrian berikutnya (maksimum dua kali re-admit), dan menutup tab tidak melepas
posisi. Memberi antrian batas waktu absolut tidak menambah proteksi di atas itu — ia hanya
menghukum customer yang menunggu lama karena antrian memang panjang.

### `regrant:{ev}` dan pause/resume

> ⚠️ **Diubah oleh ADR-006** — `regrant:{ev}` dihapus. Token yang INSERT-nya gagal tetap di `by_ticket` sebagai pending grant dan diulang tiap tick (ADR-006 §2.3). Aturan pause di bagian ini tetap berlaku untuk ADVANCE dan RECOVER.

`regrant:{ev}` memegang token yang **kita** gagal layani: token yang sudah dipilih untuk
di-grant, tetapi batch INSERT `bookings`-nya gagal setelah `GRANT_INSERT_MAX_RETRIES`. Slot
dikembalikan (`DECRBY slots:open`) dan token masuk sini, karena alternatifnya adalah
menghapusnya dari antrian tanpa sepengetahuannya — customer itu tidak akan pernah dilayani
dan tidak akan pernah tahu.

> **Ia TIDAK memberi kesempatan kedua kepada no-show.** Set ini dulu bernama `readmit:{ev}`
> dan melayani dua hal sekaligus: kompensasi kegagalan server *dan* kesempatan kedua bagi
> customer yang `LAPSED` — slot diberikan, Detail Ticket Page tidak pernah dibuka. Yang kedua
> **dihapus**; lihat keputusannya di bawah. Namanya diganti supaya artinya tidak dapat
> disalahpahami lagi dan supaya jalur `LAPSED` tidak diam-diam dipasang kembali.
>
> Perbedaannya bukan soal selera. Token di `regrant` **belum pernah mendapat gilirannya** —
> ia dipilih, lalu Aurora gagal. Token `LAPSED` **sudah** mendapat gilirannya dan membakar
> seluruh 300 detiknya. Itulah mengapa `regrant` dikuras lebih dulu, dan mengapa jalur
> `LAPSED` tidak boleh ada di set yang sama.

#### Keputusan: `LAPSED` bersifat final, tanpa kesempatan kedua

Sebelumnya `LAPSED` memberi satu grant kedua. Itu **dihapus**: `LAPSED` kini terminal
sepenuhnya, seperti `EXPIRED` dan `ABANDONED`.

| | Alasannya |
|---|---|
| **Keadilan FCFS** | Pengurasan `readmit` lebih dulu *mengurangi* `deficit` yang tersisa untuk antrian. Customer yang sudah mendapat giliran dan membakar 300 detiknya karena itu dilayani **mendahului** orang di posisi 150.000 yang menunggu satu jam dan belum pernah mendapat giliran sama sekali. FCFS memberi satu giliran, bukan giliran tanpa batas |
| **Biaya slot** | Satu no-show membakar 300 detik penuh jam booking; dengan kesempatan kedua ia menahan slot hingga 600 detik tanpa pernah memilih tiket — pada beban 300.000 orang memperebutkan 10.000 tiket |
| **Tidak dapat dibedakan** | "OS menangguhkan push saya" dan "saya tutup laptop lalu pergi makan" menghasilkan `LAPSED` yang identik. Membayari yang pertama berarti ikut memberi hadiah pada yang kedua, dan tidak ada data yang mengatakan mana yang dominan |
| **Tanggung jawab customer** | Menjaga perangkat tetap menyala dan notifikasi tetap aktif selama perang tiket ada di sisi customer. Sistem menyiarkan posisi tiap detik dan mengirim `grant` begitu slot aman; ia tidak dapat menjamin OS pihak ketiga meneruskannya |

> **Harganya nyata dan diterima**, bukan diabaikan: Bagian 7 sudah mencatat bahwa customer
> mobile yang melewatkan push kehilangan sebagian waktu memilihnya. Tanpa `readmit`, ia
> kehilangan **seluruh** slotnya. Mitigasi yang tersisa hanya UX — peringatan sebelum gerbang
> buka agar aplikasi tetap di depan dan notifikasi diizinkan (Bagian 17 #13) — dan itu
> peringatan, bukan jaminan.
>
> `LapsedRate` (FR-35) karena itu berhenti menjadi tuas penyetelan dan menjadi **metrik
> murni**: ia mengukur berapa banyak slot yang terbakar no-show. Bila angkanya tinggi pada
> rehearsal T-2 hari, tuasnya `MAX_CONCURRENT_SLOTS` (Bagian 17 #5) — bukan mengembalikan
> kesempatan kedua.

> **Biaya slot per no-show, untuk catatan.** Dulu satu kali lewat hanya membakar 120 detik
> jendela klaim; dengan anchor tunggal ia membakar 300 detik penuh jam booking. Itulah ukuran
> yang hilang setiap kali sebuah slot di-grant dan tidak dipakai — dan ukuran itu, bukan
> jumlah kesempatannya, yang membuat kesempatan kedua tidak terbayar: setiap pengulangan
> menggandakannya pada beban di mana 300.000 orang memperebutkan 10.000 tiket.

```
lapse:         ZREM by_ticket                       -- TERMINAL, tanpa kecuali
               DEL  grant:{token}                   -- kredensialnya dicabut di sini:
                    -- EX-nya 1080 sementara TTL booking 300, jadi tanpa DEL ini
                    -- grant mati masih lolos verifikasi ~780 detik lagi
               DEL  acct:{ev}:{account_id}          -- boleh antre lagi (BR-07)
               DEL  token:{uuid}                    -- Bagian 12: terminal-kegagalan
               -- TIDAK ADA SADD regrant. LAPSED sudah mendapat gilirannya.

grant INSERT   -- jalur kegagalan SERVER, bukan no-show:
gagal:         DECRBY slots:open ; SADD regrant:{ev} {uuid}
               -- token ini BELUM pernah dilayani; ia tetap di antrian.

sale ditutup:  EXPIRE regrant:{ev} 86400 ; EXPIRE by_ticket 86400
post-sale job: DEL regrant:{ev} by_ticket
```

> **Slot yang terisi dari `regrant` tidak memajukan `now_serving`.** Lua menguras `regrant`
> lebih dulu dan mengurangi `deficit` sebesar jumlah itu, sehingga `INCRBY now_serving` hanya
> berjalan untuk sisanya. Ini benar — token di `regrant` sudah dihitung sebagai "dilayani"
> ketika watermark melewatinya; INSERT-nya yang gagal, bukan gilirannya. Konsekuensinya
> customer yang menunggu kadang melihat slot terbebas tanpa posisinya bergerak. Perlu
> diketahui tim dukungan sebelum hari-H.
>
> Sejak kesempatan kedua untuk `LAPSED` dihapus, set ini hanya terisi oleh kegagalan server,
> jadi dalam operasi normal ia kosong dan `deficit` seluruhnya menjadi milik antrian.

### Apa yang dihentikan `paused`, dan apa yang tetap jalan

> ⚠️ **Dihapus oleh ADR-006 §4.3** — pause admission dihapus seluruhnya (BR-13a, BR-22). Breaker tetap fail closed, tetapi tidak menghentikan antrian; tidak ada grace +60 detik dan tidak ada field `paused`.

`paused` bukan sekadar field untuk banner. Ia **melewati seluruh langkah ADVANCE** pada tick
admission-ctl (Bagian 4 langkah 2):

| Saat `paused = true` | Jalan? | Alasan |
|---|---|---|
| ADVANCE — `INCRBY now_serving` + grant baru | ❌ | tidak ada yang dapat menyelesaikan pembelian |
| Pengurasan `regrant` | ❌ | ia juga memasukkan orang ke funnel yang sama |
| SWEEP — expiry booking dan pembayaran | ✅ | stok yang tertahan harus tetap dilepas |
| Broadcast SSE + heartbeat | ✅ | customer harus melihat bannernya dan posisinya |
| REKONSILIASI `slots:open` | ✅ | drift tidak berhenti hanya karena admission berhenti |

Karena pengurasan `regrant` ada **di dalam** `advance_now_serving.lua`, gerbangnya diletakkan
di admission-ctl — tick melewatkan `EVALSHA` sepenuhnya — bukan sebagai `ARGV` baru.
Keputusan pause tidak sensitif terhadap balapan: sebuah tick menjalankan advance atau tidak.

> **SWEEP sengaja TIDAK di-pause.** `booking_expires_at` adalah timestamp absolut yang
> tersimpan; mem-pause sweep hanya menghentikan **penagihannya**, bukan jamnya. Saat pause
> berakhir, tenggat itu sudah lewat dan sweep tetap meng-expire mereka — lebih lambat, pada
> saat yang tidak dapat diduga customer, dan dalam satu gelombang besar. Selama itu
> `slots:open` tetap terisi booking mati sehingga admission tidak dapat dimulai bersih
> meskipun breaker sudah CLOSED.
>
Perbaikan yang benar adalah **menggeser tenggatnya**, bukan berhenti menagih — dan
penggeseran itu dilakukan **saat pause dimulai**, bukan saat berakhir:

```
transisi → paused (sekali per pause, bukan tiap tick):
  SET pause:{ev}:grace_applied 1 NX EX SALE_WINDOW_SECONDS
    gagal → grace sudah diberikan untuk pause ini; LEWATI
    -- Guard ini berdiri sendiri, terlepas dari leader lease: dua task yang
    -- sama-sama mendeteksi transisi akan memberi +120 detik, bukan +60.
  UPDATE bookings
     SET booking_expires_at = booking_expires_at + PAUSE_TTL_GRACE_SECONDS,
         payment_expires_at = payment_expires_at + PAUSE_TTL_GRACE_SECONDS
   WHERE event_id = $ev
     AND status IN ('GRANTED','SELECTING','HELD',
                    'AWAITING_PAYMENT','PENDING_CANCELLATION')
  ZADD expiries:{ev} <tenggat baru> untuk tiap baris yang terkena
  emit PauseTtlGrantedSeconds
```

**Mengapa saat mulai, bukan saat selesai.** SWEEP tetap berjalan selama pause. Customer yang
tenggatnya lewat **di tengah** pause sudah di-expire sebelum resume tiba, sehingga perpanjangan
pada saat resume datang terlambat bagi justru orang-orang yang paling membutuhkannya.
Menggeser di muka membuat sweep berjalan apa adanya tanpa perlakuan khusus, dan siapa pun yang
akan kedaluwarsa dalam `PAUSE_TTL_GRACE_SECONDS` pertama terselamatkan.

Konsekuensinya jelas dan dapat diuji: **sebuah pause membeli setiap customer yang sedang di
dalam tepat `PAUSE_TTL_GRACE_SECONDS` tambahan waktu — tidak kurang, tidak lebih.** Gangguan
yang lebih panjang dari itu tetap memakan waktu mereka; itu batas yang disengaja, bukan
kelalaian. Bila sebuah gangguan melampauinya secara material, keputusannya naik ke operator:
menunda sale, bukan memperpanjang tanpa batas.

> **Ruang TTL token sudah cukup.** Perjalanan maksimum 930 detik + 60 detik grace = 990,
> masih di bawah `GRANT_TOKEN_TTL_SECONDS` (1080). Tidak ada perubahan yang diperlukan di
> sana — margin menyusut dari 150 ke 90 detik.

> **Pause yang berkedip memperbanyak perpanjangan.** Setiap transisi masuk ke `paused`
> memberi satu grace lagi. Pembatas alaminya adalah `PG_BREAKER_OPEN_SECONDS`: breaker tidak
> membuka-menutup lebih cepat dari jendelanya sendiri. `PauseTransitions` dialarmkan agar
> gangguan yang berkedip terlihat sebagai gangguan, bukan sebagai customer yang beruntung.

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

> ⚠️ **Dihapus oleh ADR-006 §4.3** — pause admission dihapus seluruhnya (BR-13a, BR-22). Breaker tetap fail closed, tetapi tidak menghentikan antrian; tidak ada grace +60 detik dan tidak ada field `paused`.

### Dua TTL yang berurutan, bukan tumpang tindih

BR-03 memberi 5 menit di Detail Ticket Page. BR-05 memberi **10 menit sejak booking dibuat** —
dan "booking dibuat" adalah saat customer menekan tombol bayar: kursi dipilih, NIK diisi, stok
di-hold (FR-16). Sebelum itu tidak ada booking, hanya sesi pemilihan. Karena itu keduanya
**berurutan**, masing-masing dengan anchor sendiri:

```
booking_expires_at = granted_at + BOOKING_TTL_SECONDS       (300)   ← slot diberikan
payment_expires_at = held_at         + PAYMENT_TTL_SECONDS   (600)   ← tekan tombol bayar
```

Setiap customer karena itu mendapat **10 menit penuh** di halaman pembayaran, berapa pun lama
ia memilih kursi — sesuai wireframe tender.

**Perjalanan maksimum karena itu 17 menit, bukan 15**, dan umur `grantToken` harus
diturunkan darinya, bukan ditulis tetap:

```
booking TTL     300 detik   GRANT → tekan Pay
payment TTL     600 detik   tekan Pay → batas bayar
payment grace    30 detik   batas bayar → pembatalan final
                            ─────────
                             930 detik = 15,5 menit

GRANT_TOKEN_TTL_SECONDS = BOOKING_TTL_SECONDS + PAYMENT_TTL_SECONDS
                        + PAYMENT_GRACE_SECONDS + margin = 1080
```

### Umur sesi akun diturunkan dari interval yang lain

`GRANT_TOKEN_TTL_SECONDS` membatasi `granted_at → batas bayar`. **Sesi akun tidak dimulai di
sana.** Ia dimulai saat login, yang terjadi *sebelum* antrian, sehingga ia harus mencakup
seluruh penantian juga:

```
sale window     SALE_WINDOW_SECONDS   login → giliran tiba
journey + pause 990 detik             930 + 60 (grace pause BR-22)
margin          150 detik             sama dengan margin grant di atas

ACCESS_TOKEN_TTL_SECONDS = SALE_WINDOW_SECONDS + 990 + 150
                         = 21600 + 1140 = 22740   (6 jam 19 menit)
```

> **1080 adalah angka yang salah untuk sesi akun, dan alasannya instruktif.** Kontrak v2.2
> menyetelnya ke 1080 karena 1080 diturunkan dari perjalanan — benar untuk grant, dan
> kebetulan saja untuk sesi. Pada `ratePerMin` 318 dan `QUEUE_CAPACITY` 300.000, customer di
> posisi terakhir menanti sekitar **15,7 jam**. Sesi 18 menit mati di ruang tunggu untuk
> hampir seluruh antrian, dan yang rusak bukan ruang tunggunya — stream memakai cookie — tapi
> `GET /accounts/me/saved-niks` pada Detail Ticket Page. Daftar NIK Teman 401 tepat ketika
> customer membutuhkannya, memaksa NIK diketik manual, memaksa panggilan Dukcapil, dan
> Dukcapil *fail closed* (BR-13) — gangguan Dukcapil kemudian memblokir booking yang
> seharusnya lolos dari bukti tersimpan, di dalam TTL booking 300 detik.
>
> **`SALE_WINDOW_SECONDS` karena itu sekarang memikul dua TTL** dan tetap belum punya nilai
> yang diputuskan (Bagian 17 #10). 21600 adalah *fallback bila env tidak didefinisikan*,
> bukan keputusan.
>
> **Harganya: jendela pencabutan 6 jam 19 menit.** Access token adalah JWT stateless; tidak
> ada store yang dibaca saat verifikasi, jadi umurnya *adalah* jendela pencabutannya. Pada
> 1080 detik itu dapat diterima. Pada 22740 detik ia memuat `nik_verified` — artinya
> **otorisasi** yang ter-cache enam jam, bukan hanya identitas. Pasangan yang semestinya
> menyertainya adalah mengeluarkan `nik_verified` dari claim dan memeriksanya ke basis data;
> claim itu ada justru untuk menghindari satu pembacaan per permintaan di bawah flash crowd,
> jadi pertukarannya nyata dan belum diputuskan (Bagian 17 #11).
>
> **Alternatif yang ditolak.** Mempertahankan 1080 dan me-refresh di ruang tunggu lebih aman
> (jendela tetap 18 menit, refresh token tersimpan dan dapat dicabut), tetapi menuntut klien
> menjadwalkan refresh dan memeriksa sisa umur saat `grant` — pekerjaan klien yang, bila
> dilewatkan, gagal secara diam-diam. Pilihan ini memindahkan jaminan ke server.

**Tidak ada grace antrian.** Jam booking mulai berdetak saat slot diberikan, bukan saat
customer membuka Detail Ticket Page. Satu jam, satu anchor, tanpa jendela terpisah untuk
mengklaim.

> **Harganya dibayar customer mobile.** Ponsel dengan layar terkunci yang melewatkan push
> `grant` dan baru membuka aplikasi empat menit kemudian hanya punya sisa satu menit untuk
> memilih kursi dan mengisi NIK. Sebelumnya 120 detik pertama adalah jendela klaim terpisah
> yang tidak memotong waktu memilih. Ini konsekuensi yang diterima, dan sejak kesempatan
> kedua atas `LAPSED` dihapus (Bagian 6) **tidak ada mitigasi teknis yang tersisa**: push
> yang terlewat sampai 300 detik habis berarti slot itu hilang. Yang tinggal hanya UX —
> peringatan sebelum gerbang buka agar aplikasi tetap di depan dan notifikasi diizinkan
> (Bagian 17 #13). Menjaga perangkat tetap menyala selama perang tiket ada di sisi customer;
> sistem tidak dapat menjamin OS pihak ketiga meneruskan push-nya.

> Angka tetap `EX 900` berlaku pada model anchor-tunggal sebelumnya dan **tidak lagi**:
> customer yang memakai hampir seluruh grace dan seluruh TTL booking akan kehilangan
> token-nya dengan sisa dua menit jendela pembayaran yang masih sah — 401 saat polling
> status, mungkin sambil berdiri di ATM dengan transfer sudah terkirim. Menurunkannya dari
> ketiga konstanta membuat perubahan `PAYMENT_TTL_SECONDS` kelak tidak diam-diam
> memendekkannya lagi.

| Transisi | Kolom Aurora (di dalam transaksi) | ZSET (penjadwal) |
|---|---|---|
| GRANT | `granted_at = now()`, `booking_expires_at = granted_at + 300` | `ZADD expiries booking_expires_at booking_id` |
| CLAIM → SELECTING | `slot_claimed_at = now()` — **tenggat tidak diubah**, ia sudah ditetapkan saat GRANT | `ZADD expiries (booking_expires_at)` |
| **TEKAN PAY → HELD** | `held_at = now()`, **`payment_expires_at = held_at + 600`** | `ZADD expiries (payment_expires_at)` |
| HELD → AWAITING_PAYMENT | *(tidak menulis deadline baru)* | *(skor tidak berubah)* |
| Terminal apa pun | `status = <terminal>` | `ZREM expiries booking_id` + kredensial antrian (di bawah) |

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
menjadi 15,5 menit, dan `GRANT_TOKEN_TTL_SECONDS` harus mencakupnya.

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
               WHEN 'GRANTED'          THEN row.booking_expires_at
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
    GRANTED          → LAPSED  : DECR slots:open ; DEL grant:{token}
                                 ; DEL acct ; DEL token:{uuid}   -- final
                                 -- LAPSED = slot diberikan, Detail Ticket Page tidak
                                 -- pernah dibuka. Dibedakan dari EXPIRED (dibuka tapi
                                 -- tidak selesai memilih) agar FR-35 tetap dapat
                                 -- memisahkan keduanya; tenggatnya kini sama.
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

**Dua kode kegagalan baru, dan keduanya BUKAN "habis".** `STOCK_UNINITIALIZED` berarti kunci
stok hilang dari Redis — flush atau failover — dan `STOCK_NEGATIVE` berarti guard terakhir
menyala. Keduanya dijawab **503**, bukan 409, karena stok sebenarnya tidak diketahui; keduanya
dialarmkan (`StockKeyMissing`, `StockNegativeGuard`) dan keduanya menuntut rekonsiliasi
segera, bukan pada tick berikutnya.

> **Gagal hold TIDAK menghilangkan slot.** `NO_STOCK`, `NIK_TAKEN` atau `QTY_EXCEEDED` →
> customer tetap `SELECTING` dengan sisa TTL; UI menampilkan pesan spesifik; pemilihan ulang
> kategori atau perbaikan NIK diizinkan. (NFR-02)

```lua
-- hold_booking.lua
-- KEYS[1..m]      = stock:{ev}:{cat}       satu per KATEGORI yang dipilih
-- KEYS[m+1..m+q]  = nik:used:{ev}:{hmac}   satu per KURSI
-- ARGV[1]=m  ARGV[2]=MAX_TICKETS_PER_TRANSACTION  ARGV[3]=NIK_USED_TTL_SECONDS
-- ARGV[4..3+m]    = qty per kategori, urutannya sama dengan KEYS[1..m]
--
-- Seluruh kunci memakai hash tag {ev} sehingga berada pada slot yang sama di
-- Redis cluster mode -- prasyarat agar satu skrip boleh menyentuh semuanya.
local m    = tonumber(ARGV[1])
local maxq = tonumber(ARGV[2])

-- Kuota dihitung LINTAS kategori: FR-13 membatasi 3 tiket per TRANSAKSI,
-- bukan 3 per kategori. 1 Premium + 1 Festival + 1 VIP = 3, sah.
local total = 0
for i = 1, m do
  local q = tonumber(ARGV[3 + i])
  if q == nil or q < 1 then return {0, 'QTY_INVALID', i - 1} end
  total = total + q
end
if total > maxq then return {0, 'QTY_EXCEEDED', -1} end                       -- FR-13
if #KEYS - m ~= total then return {0, 'NIK_COUNT_MISMATCH', -1} end           -- BR-02

-- FASE 1 -- periksa SEMUA kategori sebelum mengurangi satu pun.
-- Inilah yang membuat hold multi-kategori all-or-nothing: tidak pernah ada
-- keadaan sebagian kategori ter-hold dan sebagian gagal.
for i = 1, m do
  local raw = redis.call('GET', KEYS[i])
  if raw == false then
    -- Kunci HILANG, bukan nol -- Redis flush atau failover. Menjawab NO_STOCK
    -- di sini akan mengumumkan "habis" untuk event yang stoknya masih penuh,
    -- dan menolak SELURUH penjualan sampai rekonsiliasi menanamnya kembali.
    return {0, 'STOCK_UNINITIALIZED', i - 1}
  end
  if tonumber(raw) < tonumber(ARGV[3 + i]) then
    return {0, 'NO_STOCK', i - 1}                                             -- FR-14
  end
end

-- FASE 2 -- kunci setiap NIK; satu bentrok membatalkan semuanya
local claimed = {}
for i = m + 1, #KEYS do
  if redis.call('SET', KEYS[i], 1, 'NX', 'EX', tonumber(ARGV[3])) then
    -- EX membatasi kebocoran bila transaksi Aurora rollback setelah titik ini:
    -- NIK terkunci paling lama selama sale, bukan selamanya (Bagian 6)
    claimed[#claimed + 1] = KEYS[i]
  else
    for j = 1, #claimed do redis.call('DEL', claimed[j]) end
    return {0, 'NIK_TAKEN', i - m - 1}    -- indeks NIK yang bentrok
  end
end

-- FASE 3 -- kurangi stok. Fase 1 sudah menjamin cukup dan skrip ini atomik,
-- jadi tidak ada yang dapat menyelip di antaranya. Guard di bawah murni
-- pertahanan terakhir: Redis tidak punya CHECK (stock >= 0).
for i = 1, m do
  local left = redis.call('DECRBY', KEYS[i], tonumber(ARGV[3 + i]))
  if left < 0 then
    for j = 1, i          do redis.call('INCRBY', KEYS[j], tonumber(ARGV[3 + j])) end
    for j = 1, #claimed   do redis.call('DEL', claimed[j]) end
    return {0, 'STOCK_NEGATIVE', i - 1}
  end
end
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
- **Pepper disimpan sebagai secret yang dikelola KMS**, diambil dan didekripsi oleh service,
  lalu HMAC-SHA256 dihitung **di dalam proses** (keputusan 30 Oktober 2026 — lihat di bawah).
- **Pepper tidak dapat dirotasi sama sekali, bukan hanya "tidak selama sale".** Rotasi
  mengubah seluruh hmac, sehingga `nik_claims` dan `nik:used` yang ada menjadi tidak dapat
  dicocokkan dan BR-02 berhenti ditegakkan secara senyap. Memperbaikinya menuntut **re-hash**
  seluruh klaim lama — dan itu menuntut NIK **plaintext**, yang sengaja tidak pernah disimpan.
  Jadi backfill bukan sekadar pekerjaan berat; ia **tidak mungkin** tanpa mengumpulkan ulang
  NIK dari setiap customer.

> **Versi sebelumnya dokumen ini keliru.** Ia menginstruksikan "klaim lama harus di-backfill
> dengan hmac baru" seolah rotasi hanyalah soal penjadwalan. Dengan hanya hash yang tersimpan,
> instruksi itu tidak dapat dijalankan.

**Konsekuensinya: pepper bersifat permanen.** Ia tidak boleh hilang — kehilangannya membuat
setiap `nik_hmac` tidak dapat dicocokkan dan BR-02 berhenti bekerja selamanya — dan ia tidak
boleh bocor, karena **rotasi tidak tersedia sebagai remediasi**. Biasanya secret yang bocor
dirotasi; di sini tidak bisa.

### Penyimpanan pepper: secret yang dikelola KMS (bukan `GenerateMac`)

Dua opsi dipertimbangkan:

| | Pepper sebagai secret KMS (**dipilih**) | KMS HMAC key + `GenerateMac` |
|---|---|---|
| Letak kunci | didekripsi ke memori proses | tidak pernah keluar dari HSM |
| Service dikompromikan | pepper ikut terambil → enumerasi seluruh NIK tersimpan **offline dan senyap** | hanya dapat meminta KMS mem-hash — ter-IAM, ter-rate-limit, tercatat CloudTrail |
| Pepper hilang | **dapat dipulihkan** dari backup secret | tidak dapat diekspor → BR-02 rusak permanen |
| Latensi per hash | nol | satu panggilan KMS (~5–10 ms) |

**Dipilih: secret yang dikelola KMS.** Alasan utamanya bukan kesederhanaan melainkan
**keterpulihan**: karena rotasi tidak mungkin, kehilangan pepper adalah kegagalan permanen
atas BR-02 — dan opsi ini satu-satunya yang memungkinkan pepper di-backup dan dipulihkan.
`GenerateMac` melindungi kerahasiaan lebih baik tetapi membuat kunci tidak dapat diekspor,
yang menukar satu kegagalan permanen dengan kegagalan permanen yang lain.

> **Risiko yang diterima secara sadar:** karena pepper berada di memori proses, kompromi pada
> `nik-verify` atau `booking-service` memberi penyerang kemampuan **enumerasi offline** atas
> seluruh NIK tersimpan — ruang NIK yang valid cukup kecil untuk itu (provinsi, kabupaten dan
> tanggal lahir terkode di dalamnya). Dan karena rotasi tidak tersedia, kebocoran itu **tidak
> dapat diremediasi**. Yang menahannya hanyalah kontrol akses pada secret itu dan pada kedua
> service tersebut; `GenerateMac` adalah jalur hardening sebelum produksi, bukan POC.

### Persetujuan pemrosesan data pribadi

NIK milik **setiap penonton**, bukan hanya pembeli, dan dikirim ke API pemerintah eksternal —
transfer data pribadi pihak ketiga di bawah UU PDP 27/2022. Dua persetujuan terpisah dicatat
di checkout, dan keduanya wajib sebelum stok di-hold:

1. Pembeli menyatakan memiliki izin tiap pemegang NIK untuk menyerahkan datanya
2. Pembeli menerima syarat dan ketentuan pembelian

`terms_version` dan `terms_accepted_at` merekam penerimaan, sehingga sengketa dapat dirujuk
ke teks yang benar-benar berlaku saat itu. Klausa memerlukan **sign-off legal** sebelum rilis.

### Retensi PII: tanpa batas waktu, dihapus atas permintaan

**Keputusan (30 Oktober 2026): POC tidak menerapkan batas retensi berbasis waktu.** Data
pribadi disimpan sampai pemilik datanya meminta penghapusan, dan penghapusan dijalankan
**manual oleh tim operasional** — tidak ada endpoint self-service di POC.

Tiga batasan harus ikut bersama keputusan itu, karena permintaan hapus tanpa syarat tidak
kompatibel dengan skema ini.

**1. Ada LANTAI waktu: tidak dapat dihapus sebelum event berlangsung.** Dua kolom NIK sedang
menegakkan aturan bisnis selama event belum lewat:

| Kolom | Mengapa tidak boleh hilang lebih awal |
|---|---|
| `nik_claims.nik_hmac` | `UNIQUE (event_id, nik_hmac)` **adalah** penegak BR-02. Menghapusnya membuat NIK yang sudah punya kursi dapat membeli lagi di event yang sama |
| `tickets.nik_hmac` | pemeriksaan di gerbang mencocokkan KTP penonton terhadap hash ini. Menghapusnya membuat tiket tidak dapat divalidasi di pintu |

Permintaan hapus yang masuk **sebelum** event karena itu dijadwalkan, bukan dieksekusi: data
dihapus setelah event berlangsung dan jendela sengketa tertutup. Pemintanya diberi tahu kapan.

**2. `consent_records` dikecualikan.** Ia adalah bukti bahwa persetujuan pernah diberikan —
menghapusnya menghancurkan catatan kepatuhan yang justru dituntut UU PDP. Ia dipertahankan
atas dasar kewajiban hukum, bukan atas dasar persetujuan, dan karena itu di luar cakupan
penghapusan atas permintaan.

**3. NIK pihak ketiga tidak punya jalur self-service.** `saved_niks` memuat NIK **orang lain**
beserta labelnya. Pemilik akun dapat menghapus entrinya sendiri; orang yang NIK-nya disimpan
tidak punya akun, tidak dapat login, dan tidak punya cara meminta apa pun. Permintaan dari
pihak itu ditangani tim operasional — dan ini titik terlemah desain terhadap UU PDP, dicatat
sebagai diterima untuk POC.

> **`DELETE` bukan penghapusan sampai jendela PITR terlewat.** Aurora menyimpan baris yang
> dapat dipulihkan selama masa retensi backup, sehingga permintaan hapus baru benar-benar
> terpenuhi setelah jendela itu berakhir. Angka itu harus dinyatakan ke pemintanya, bukan
> disembunyikan.

> **Konsekuensi yang diterima: data bertumpuk tanpa batas.** Tanpa purge berbasis waktu,
> `queue_entries` tumbuh 300.000 baris per event selamanya, dan setiap baris PII yang disimpan
> memperbesar dampak bila terjadi kebocoran. Purge berjadwal adalah tindak lanjut sebelum
> produksi, bukan POC.

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

> ⚠️ **Dihapus oleh ADR-006 §4.3** — pause admission dihapus seluruhnya (BR-13a, BR-22). Breaker tetap fail closed, tetapi tidak menghentikan antrian; tidak ada grace +60 detik dan tidak ada field `paused`.

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
CAPTCHA_PROVIDER            CAPTCHA_SECRET_KEY       CAPTCHA_SITE_KEY
CAPTCHA_VERIFY_TIMEOUT_MS=800    CAPTCHA_FAIL_OPEN=true
  -- FAIL_OPEN=true disengaja: CAPTCHA melindungi keadilan, bukan kebenaran.
QPERSIST_BATCH_SIZE=1000         -- baris per batch INSERT queue_entries
QPERSIST_DRAIN_BUDGET_MS=200     -- anggaran drain per tick; drain adalah langkah
  -- TERAKHIR agar dapat dipotong tanpa mengancam lease (Bagian 6, Bagian 11)
CTL_LEADER_LEASE_SECONDS=5       -- TTL ctl:leader; diperpanjang tiap tick 500 ms.
  -- Failover ≤ lease + 1 tick ≈ 5,5 detik. 10× margin terhadap periode tick.
SSE_STALE_AFTER_SECONDS=30       -- keheningan sse-broadcast yang dibaca sebagai kegagalan
  -- = 3 × BROADCAST_HEARTBEAT_SECONDS, dan > handover lease 5,5 detik sehingga
  -- handover normal tidak pernah mencapai indikator customer (Bagian 11).
PAUSE_TTL_GRACE_SECONDS=60       -- tambahan waktu bagi customer yang sedang di dalam
  -- saat admission di-pause. Digeser SAAT PAUSE MULAI, sekali per pause (Bagian 6).
SALE_WINDOW_SECONDS=21600   -- FALLBACK bila env tidak didefinisikan (6 jam), BUKAN
  -- keputusan: nilainya milik Bagian 17 #10. TTL acct:{ev}:{id} dan token:{uuid};
  -- membatasi kebocoran BR-07. DUA TTL lain diturunkan darinya, jadi menaikkannya
  -- menaikkan jendela pencabutan access token juga.
ACCESS_TOKEN_TTL_SECONDS=22740  -- = SALE_WINDOW + 990 + 150. DITURUNKAN: sesi dimulai
  -- saat login, sebelum antrian, jadi ia harus mencakup penantian + perjalanan.
  -- BUKAN sama dengan GRANT_TOKEN_TTL_SECONDS; keduanya mengukur interval berbeda.
SETTLED_TOMBSTONE_SECONDS=3600  -- umur settled:{uuid} setelah CONFIRMED (Bagian 12).
  -- Cukup untuk email e-tiket tiba dan customer memeriksanya; tidak memulihkan apa pun.
DEVICE_PEPPER               -- HMAC device_id tamu (Bagian 12). TERPISAH dari pepper NIK
  -- dan BERBEDA sifatnya: boleh dirotasi ANTAR sale, tidak pernah DI TENGAH sale —
  -- rotasi di tengah membatalkan setiap pengikatan yang hidup.
MAX_DEVICE_REBINDS=1        -- Bagian 17 #12 — DIHAPUS oleh ADR-006 §8.9 (tanpa rebind)
  -- MAX_READMITS DIHAPUS: LAPSED tidak lagi memberi kesempatan kedua sama sekali,
  -- jadi tidak ada batas untuk disetel. regrant:{ev} hanya terisi oleh kegagalan
  -- INSERT grant, yang sudah dibatasi GRANT_INSERT_MAX_RETRIES (Bagian 6, Bagian 7).
SELLING_FAST_THRESHOLD_PCT=20    -- ambang badge "hampir habis" (Bagian 6)
SOLD_OUT_HOLD_SECONDS=930        -- = BOOKING_TTL + PAYMENT_TTL + PAYMENT_GRACE.
  -- Lama SOLD_OUT tetap reversibel. Diturunkan, bukan ditebak: tidak ada hold yang
  -- dapat kedaluwarsa lebih lambat dari satu perjalanan penuh (Bagian 6).
MAX_TICKETS_PER_TRANSACTION=3   -- total lintas kategori, bukan per kategori
NIK_USED_TTL_SECONDS        -- durasi sale + margin; membatasi kebocoran nik:used
BOOKING_TTL_SECONDS=300     PAYMENT_TTL_SECONDS=600
PAYMENT_GRACE_SECONDS=30    QUEUE_CAPACITY=300000    MAX_SAVED_NIKS=5
ACCOUNT_RATE_LIMIT_PER_IP_HOUR=5                     POLICY_VERSION
  -- booking dari granted_at; pembayaran dari held_at (berurutan, maks 15,5 menit)
GRANT_INSERT_MAX_RETRIES=3  BROADCAST_HEARTBEAT_SECONDS=10
MAX_TICKETS_PER_TRANSACTION=3                        TERMS_VERSION
MAX_PAYMENT_ATTEMPTS=5      -- tiap percobaan menerbitkan VA/QR baru di Midtrans
MIN_RETRY_REMAINING_SECONDS=60  -- di bawah ini, ganti metode ditolak
GRANT_TOKEN_TTL_SECONDS=1080    -- = BOOKING_TTL + PAYMENT_TTL + PAYMENT_GRACE + margin.
  -- DITURUNKAN, bukan konstanta lepas: perjalanan maksimum 120+300+600 = 1020 detik.
```

**Payment Gateway**: Midtrans **Core API**. Probe half-open read-only; breaker terbuka →
**pause admission**. **Dua** metode dibuka — `bank_transfer` (BCA/BNI/BRI/CIMB saja) dan
`gopay`; rinciannya di `payment-method/`.

> **Kartu kredit dan OTC dikeluarkan dari POC karena alasan yang sama: keduanya tidak dapat
> menghormati TTL pembayaran di sisi gateway.** `custom_expiry` tidak berlaku untuk kartu
> (dinyatakan eksplisit oleh Midtrans), sehingga BR-05 hanya ditegakkan sweep kita tanpa
> dukungan gateway; OTC menuntut customer datang ke gerai fisik, yang tidak mungkin dalam 10
> menit, dan Indomaret termasuk kanal yang dapat reversal. Dua metode yang tersisa keduanya
> menerima `custom_expiry`, jadi **setiap** metode yang hidup menutup jendelanya pada detik
> yang sama dengan jendela kita. Keluarnya kartu juga mengeluarkan cakupan PCI-DSS dan alur
> tokenisasi di browser.

> **Permata, Mandiri Bill dan Indomaret sengaja tidak dibuka.** Midtrans mendokumentasikan
> `settlement → deny` (reversal, 1–5 menit, dana kembali ke customer) hanya untuk ketiganya.
> Menutup mereka berarti kelas kegagalan itu tidak dapat terjadi, dan tidak perlu handler
> yang menarik kembali tiket yang sudah terbit dan e-tiket yang sudah terkirim.

**PCI-DSS (NFR-07): keluar dari cakupan POC, dan itu harus dijaga.** Dengan kartu kredit
tidak dibuka, tidak ada nomor kartu yang menyentuh sistem ini sama sekali — bukan karena
ditokenisasi di browser, melainkan karena tidak pernah diminta. VA dan GoPay keduanya
menyelesaikan pembayaran di luar halaman kita.

> **Bila kartu ditambahkan sebelum produksi**, klaim itu menjadi bersyarat lagi: nomor kartu
> harus bergerak langsung dari browser ke Midtrans lewat Midtrans.js dengan backend hanya
> menerima `token_id`, dan **tidak satu endpoint pun boleh menerima field nomor kartu**. Satu
> form yang mengirimkannya ke backend kita akan memperluas cakupan PCI-DSS secara drastis.

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
    SELECT id, status FROM bookings WHERE grant_token = $grantToken FOR UPDATE
    status sudah 'HELD' atau lebih jauh → COMMIT ; kembalikan booking yang ADA, 200
    status ≠ 'SELECTING'                → COMMIT ; 409

    EVALSHA hold_booking.lua            -- stok (semua kategori) + NIK, satu skrip
    gagal → COMMIT (tanpa perubahan) ; kembalikan NO_STOCK / NIK_TAKEN / QTY_EXCEEDED
            STOCK_UNINITIALIZED / STOCK_NEGATIVE → 503 + alarm (Bagian 8)
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
    INCRBY stock:{ev}:{cat} qty         -- satu INCRBY per kategori yang di-hold
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
  1. Validasi grantToken (header `X-Grant-Token`)
  2. BEGIN
       SELECT status, payment_ref, payment_attempts
         FROM bookings
        WHERE id = $1 AND grant_token = $grantToken    -- KEPEMILIKAN, bukan hanya
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
              payment_method = $method        -- bank_transfer | gopay
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
     WHERE id = $1 AND grant_token = $grantToken       -- 1. KEPEMILIKAN
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

**TTL tidak ikut mundur.** `custom_expiry.order_time` diisi `held_at`, bukan waktu request,
sehingga VA baru mewarisi sisa jendela yang benar: anchor 10:15 + 10 menit tetap berakhir
10:25 walau charge kedua terjadi pukul 10:23. Seandainya `order_time` diisi `now()`, setiap
pergantian metode akan memberi 10 menit baru dan TTL pembayaran tidak pernah habis.

Bila gateway timeout: **jangan kirim ulang charge** — biarkan `payment_ref` NULL dan serahkan
ke charge reconciler (Bagian 8).

### Langkah 2 — Webhook: commit, lalu terbitkan inline

> ⚠️ **Diubah oleh ADR-006** — langkah inline `DECR slots:open` + `EVALSHA advance_now_serving.lua` + `PUBLISH sse-broadcast / sse-grant` **dihapus tanpa pengganti**: tempat yang bebas terpakai pada tick admission-ctl berikutnya (≤ 500 ms). Hanya admission-ctl yang memberi grant (ADR-006 §4.1). Berlaku juga untuk konsumer outbox (Langkah 4).

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

> ⚠️ **Diubah oleh ADR-006 §8.15** — `ticket.issue` mengirim ke `contact_email` (bukan `booking_items.holder_email`), dan `booking.expired` hanya ke `contact_email`.

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

> ⚠️ **Diubah oleh ADR-006 §8.8** — `GET /bookings/{id}/tickets` dihapus. Jalur NFR-10 kini `GET /bookings/{id}` saat `PAID`: `tickets[].eticket_url` sudah membawa `?t={eticket_token}`, jadi e-tiket dapat dibuka tanpa akun dan tanpa email.

> **Booking `CONFIRMED` sejak pembayaran tercatat di Aurora.** Kegagalan SES tidak mengubah
> status booking — sesuai NFR-10.

```
SES gagal → SQS retry backoff → N percobaan → DLQ → alarm ops
GET /bookings/{id}           ← e-tiket SELALU dapat dibuka: eticket_url bertoken ?t= (ADR-006 §8.8)
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
Scale in  : DINONAKTIFKAN selama sale aktif
Min / Max : 0 → N / 60 · Cooldown out 60 detik

"sale aktif" = queue:{ev}:state ∈ { OPEN, SOLD_OUT }
"selesai"    = queue:{ev}:state == CLOSED   → desired_count = 0
```

### Tiga hal berbeda yang sering dikira satu

Pertanyaan "berapa instance" sebenarnya tiga pertanyaan, dan masing-masing dijawab mekanisme
yang berbeda. Menyelesaikan satu dengan alat yang lain adalah kesalahan yang paling mudah
dibuat di sini:

| | Masalahnya | Alatnya | Angkanya |
|---|---|---|---|
| **Warm floor** | cold start, image rusak, IAM salah, Redis tak terjangkau | `desired_count` minimum sebelum sale | 3 task, T-2 sampai T-6 **jam** |
| **Pre-scale** | 300.000 koneksi datang dalam ~60 detik | `desired_count = N` terjadwal | 20–43 task, **T-30 menit** |
| **HA** | satu task mati di tengah sale | **leader lease**, bukan jumlah instance | 2 task (1 leader + 1 standby) |

**Autoscaling reaktif tidak dapat melayani flash crowd, dan itu bukan kelemahan konfigurasi.**
Ramp di T=0 adalah ~5.000 koneksi/detik, sementara satu task Fargate butuh puluhan detik untuk
hidup dan cooldown scale-out 60 detik. Bebannya datang lebih cepat daripada loop apa pun yang
bereaksi terhadapnya. Karena itu `desired_count = N` **dijadwalkan**, dan autoscaling hanya
berfungsi sebagai margin di atas armada yang sudah di-pre-scale — bukan sebagai mekanisme yang
menangani kerumunan.

**Warm floor dihitung dalam jam, bukan hari — dan alasannya bukan biaya.** Sebelum
`sale_starts_at`, join dijawab 409 dan stream menolak `SCHEDULED`, sehingga **tidak ada trafik
sama sekali**. Task yang menyala dua hari dalam keadaan idle tidak menjalankan satu pun jalur
kode yang dipedulikan. Kegagalan yang memang butuh jendela panjang — pertumbuhan memori pada
proses SSE berumur panjang, connection table yang membengkak, volume log, rotasi token — hanya
muncul **di bawah beban**, dan idle tidak memunculkannya.

Biayanya nyata meski bukan penentu:

| Jendela warm floor | 3 task SSE | 2 task ctl | Per event | 2 event/bulan |
|---|---|---|---|---|
| T-30 menit | $0.15 | $0.02 | **$0.17** | $0.34 |
| **T-6 jam** | $1.78 | $0.30 | **$2.08** | $4.16 |
| T-2 hari | $14.21 | $2.37 | **$16.58** | $33.16 |

T-2 hari adalah **+46% atas biaya variabel per event (~$36)** dan +5% atas total bulanan —
dibayar untuk task yang tidak melakukan apa pun.

> **Yang sebenarnya dibeli oleh lead time dua hari adalah WAKTU UNTUK MEMPERBAIKI, bukan waktu
> menyala.** Jadwalkan **rehearsal** di T-2 hari — ramp sintetis 300.000 join, fan-out SSE,
> sold-out lalu resume — kemudian turunkan kembali ke floor. Itu menemukan masalah dengan sisa
> waktu untuk membenahinya, dan beberapa pemeriksaan di T-2 menit sebaiknya pindah ke sana.
> Skenario dan thresholdnya adalah Bagian 17 #9.

> **`IdleTaskCount > 0` harus mengecualikan jendela warm floor dan rehearsal.** Kalau tidak,
> alarm itu menyala pada operasi yang normal, dan tim akan belajar mengabaikannya — yang
> menghapus gunanya pada kasus yang sebenarnya, yaitu task tertinggal setelah sale.

### `admission-ctl`: jumlah instance bukan jawabannya

> ⚠️ **Diubah oleh ADR-006** — leader lease + standby **ditunda ke produksi** (ADR-006 §9). POC menjalankan 1 task; ECS mengganti task yang mati, pending grant mencegah kehilangan data.

`rusuh-admission-ctl` adalah **control loop**, bukan request handler, dan karena itu
`desired_count` dinaikkan ke **2 — 1 leader + 1 standby — dengan leader lease**:

```
AMBIL (bila belum punya):
  SET ctl:leader {task_id} NX EX CTL_LEADER_LEASE_SECONDS
  berhasil → my_epoch = INCR ctl:epoch        -- fencing token, monoton

PERPANJANG (tiap tick, WAJIB bersyarat — Lua, atomik):
  if redis.call('GET', KEYS[1]) == ARGV[1]
    then redis.call('PEXPIRE', KEYS[1], ARGV[2]) ; return 1
    else return 0                              -- kepemimpinan hilang
  end

LEPAS (shutdown, WAJIB bersyarat — Lua, atomik):
  if redis.call('GET', KEYS[1]) == ARGV[1]
    then redis.call('DEL', KEYS[1]) end

bila BUKAN leader → lewati seluruh tick (standby diam, tetap hangat)
```

> **`PEXPIRE` dan `DEL` tanpa syarat adalah bug, bukan penyederhanaan.** `EXPIRE` buta
> memperpanjang lease yang mungkin sudah bukan milik kita; `DEL` buta menghapus lease task
> lain dan mengundang pemimpin kedua secara aktif. Keduanya **wajib** melalui Lua yang
> memeriksa pemilik dalam operasi yang sama.

Task ketiga menambah biaya tanpa menambah ketersediaan: hanya satu yang pernah bertindak.
Failover turun dari 30–90 detik (deteksi ECS + start task + health check) menjadi
≤ TTL lease + satu tick — sekitar **5,5 detik**.

### Lease ganda tidak dicegah — ia dibuat tidak berbahaya

Ini perlu dinyatakan tanpa basa-basi: **lock berbasis waktu di Redis tidak dapat menjamin
pemegang tunggal.** Ada dua jalan menuju dua pemegang sekaligus, dan tidak satu pun dapat
ditutup oleh Redis:

| Jalan | Urutannya |
|---|---|
| **Holder tertahan** | A memegang lease EX 5 → A tertahan (GC, query SWEEP lambat, CPU throttle) → lease kedaluwarsa sendiri di t=5 → B mengambilnya secara sah → A bangun di t=7 **masih mengira ia leader** dan bertindak |
| **Failover Redis** | Replikasi ElastiCache bersifat **asinkron**; failover primary di antara `SET` dan replikasinya membuat key itu tidak ada di primary baru → B mengambil lease yang menurut A masih dipegangnya |

Redis tidak punya cara memberi tahu holder yang tertahan untuk berhenti, dan tidak dapat
membuat replikasinya sinkron. Karena itu strateginya **bukan** mencegah dua pemegang,
melainkan membuat keadaan itu tidak merusak apa pun:

```
sebelum tiap langkah non-idempoten:
  masih pemilik?        GET ctl:leader == {task_id}
  sisa TTL cukup?       PTTL ctl:leader > anggaran langkah ini
  bawa {my_epoch} ke dalam write yang dijaga
```

**`ctl:epoch` adalah fencing token.** Ia naik monoton pada setiap pengambilan lease, sehingga
leader yang tertahan membawa epoch yang **lebih kecil** daripada leader yang sah. Write yang
dijaga menolak epoch yang basi dan `StaleLeaderRejected` dialarmkan — artinya dua pemegang
terlihat sebagai peristiwa, bukan sebagai kerusakan senyap.

> **Untuk dua operasi non-idempoten hari ini, CAS di sumber dayanya sudah cukup sendiri**
> — compare-and-adjust pada rekonsiliasi stok dan `grace_applied NX` pada pause grace
> menolak aktor kedua tanpa perlu melihat epoch sama sekali. Epoch ditambahkan karena ia
> bersifat **umum**: operasi non-idempoten berikutnya yang ditambahkan seseorang tidak
> otomatis menjadi lubang baru, dan alarmnya memberi tahu bahwa lease ganda benar-benar
> terjadi di produksi.

**Urutan klaim yang benar, dan ini yang harus dipegang saat membaca sisa dokumen:** lease
menjawab *siapa yang sebaiknya bekerja* — ketersediaan dan efisiensi. Pengaman di sumber daya
menjawab *apa yang terjadi bila dua tetap bekerja* — korektifitas. Bila lease dihapus besok,
sistem tetap benar (hanya boros dan berisik). Bila pengamannya dihapus dan hanya lease yang
tinggal, sistem menjadi **mungkin** benar — dan "mungkin" bukan sifat yang pantas dipegang
oleh stok tiket.

> **Jaminan sungguhan menuntut store lain.** Fencing token yang diperiksa oleh setiap write,
> atau lock di store linearizable (DynamoDB conditional write, etcd, Zookeeper). Berlebihan
> untuk POC ini karena pengaman di sumber daya sudah menutupnya — tetapi itulah jawabannya
> bila kelak ada operasi yang tidak dapat dibuat idempoten.

**Apa yang sebenarnya tidak aman bila dijalankan ganda.** Ini perlu dinyatakan tepat, karena
sebagian besar loop ternyata sudah terlindungi dan hanya dua operasi yang benar-benar rapuh:

| Operasi | Aman dijalankan ganda? | Mengapa |
|---|---|---|
| SWEEP | ✅ sudah | `UPDATE … WHERE id = $1 AND status = row.status` adalah CAS; yang kalah mendapat 0 baris dan berhenti sebelum efek sampingnya (Bagian 7) |
| ADVANCE | ✅ sudah | Lua atomik dan `deficit` dihitung dari `slots:open` yang dibaca ulang; eksekusi kedua melihat angka yang sudah naik |
| `SOLD_OUT → CLOSED` | ✅ praktis | `SET state CLOSED` idempoten; `sse-sold-out` ganda tidak berbahaya karena klien menutup pada yang pertama |
| **Rekonsiliasi stok** | ❌ | `INCRBY -current_drift` tidak idempoten — dua eksekusi mengoreksi 2× |
| **Pause TTL grace** | ❌ | `UPDATE … + PAUSE_TTL_GRACE_SECONDS` dua kali memberi +120 detik, bukan +60 |

> **Koreksi atas pernyataan sebelumnya di dokumen ini.** Versi terdahulu menyebut SWEEP ikut
> rusak bila dijalankan ganda. Itu berlebihan: guard CAS-nya sudah ada, dan komentarnya
> ("pihak lain mendahului") memang mengantisipasi aktor bersamaan. Yang benar-benar rapuh
> hanya dua baris di tabel di atas.

**Kedua operasi rapuh itu diberi pengaman sendiri, bukan hanya mengandalkan lease.** Lease
adalah kunci berbasis waktu, dan kunci berbasis waktu dapat kalah balapan: tick yang tertahan
lebih lama dari TTL-nya (GC pause, query SWEEP yang lambat) kehilangan kepemimpinan sementara
masih bertindak. Karena itu:

| Operasi | Pengaman mandiri |
|---|---|
| Rekonsiliasi stok | `adjust_if_unchanged.lua` — compare-and-adjust: delta hanya diterapkan bila stok **masih** sama dengan nilai yang diamati (Bagian 6) |
| Pause TTL grace | `SET pause:{ev}:grace_applied NX` — satu pemberian per pause, siapa pun yang menjalankannya (Bagian 6) |

Dengan keduanya terpasang, lease menjadi **optimasi ketersediaan**, bukan satu-satunya hal
yang berdiri antara sistem dan kerusakan stok. Itu urutan yang benar: korektifitas dijaga
operasinya sendiri, ketersediaan dijaga lease.

> **TTL lease 5 detik, diperpanjang tiap tick 500 ms.** Sepuluh kali margin: kehilangan
> sepuluh perpanjangan berturut-turut berarti task itu memang sudah tidak sehat. Lebih pendek
> mempercepat failover tetapi memicu handover pada hiccup Redis sesaat; `CtlLeaderHandovers`
> dialarmkan agar handover yang sering terbaca sebagai gejala, bukan sebagai hal normal.

### Pengawasan: lease memulihkan, pengawas membuktikan

Matinya `admission-ctl` tidak terlihat siapa pun kecuali ada yang mengawasinya. Ia yang
mem-publish `sse-broadcast` termasuk heartbeat 10 detik, sedangkan `status {stale:true}`
selama ini hanya menyala ketika **sse-server kehilangan Redis**. Bila ctl mati, Redis sehat
dan sse-server sehat: tidak ada event `stale`, `: keepalive` 15 detik tetap datang sehingga
koneksi tampak hidup, dan angka antrian hanya membeku.

Leader lease (di atas) **memulihkan** penyebab yang paling sering — task leader mati atau
diganti ECS — dan memulihkannya dalam ≤ 5,5 detik, jauh di bawah ambang deteksi 30 detik,
sehingga indikatornya tidak pernah punya waktu untuk muncul. Tetapi lease adalah mekanisme
yang **kegagalannya sendiri senyap**:

| Penyebab beku | Dipulihkan lease? |
|---|---|
| Leader crash / diganti ECS / rolling deploy | **Ya** — standby ≤ 5,5 detik |
| Leader hidup, lease diperpanjang, **tick loop macet** | Tidak — standby tidak pernah terpicu |
| **Kedua task hilang** (image buruk, kehilangan AZ, `desired_count` turun ke 0 terhadap event yang masih hidup) | Tidak — tidak ada yang mengambil lease |
| Redis tak terjangkau dari ctl | Tidak — tetapi jalur kehilangan-Redis di sse-server sudah menyalakan `stale` |
| Pub/Sub putus ke **satu** task sse-server | Tidak relevan — ctl sehat; hanya klien task itu yang basi |

Pengawasan karena itu bukan duplikat lease. Ia yang **membuktikan lease bekerja** — justru
pada situasi di mana paling sulit menyadari bahwa ia tidak bekerja.

**Diputuskan: dua penutup, keduanya di backend.**

| Penutup | Di mana | Yang hanya ia tangkap |
|---|---|---|
| sse-server mendeteksi keheningan `sse-broadcast` → `status {stale:true, reason}` | sse-server, memakai event `status` yang sudah ada | Pub/Sub putus ke satu task |
| Metrik `CtlHeartbeatAge` + dua alarm | sse-server mengemisi, CloudWatch mengalarmkan | Leader macet, kedua task hilang |

Keduanya **tidak menambah kode klien sama sekali**: `setStaleIndicator` sudah terpasang pada
event `status` (Bagian 11, handler klien). Pengawas berbasis timer **di klien** sengaja
**ditunda** — setelah lease terpasang, yang hanya ia tangkap adalah sse-server yang macet
sendiri atau TCP half-open, dan itu tidak sebanding dengan menaruh timer liveness di
frontend.

**Detektor keheningan di sse-server.** Per event yang punya subscriber, in-memory:

```
last_broadcast_at[ev] ← disetel pada SETIAP sse-broadcast yang diterima
                        (inisialisasi: waktu task mulai, bukan nol)

tiap 1 detik:
  age      := now − last_broadcast_at[ev]
  expected := queue:{ev}:state ∈ { OPEN, SOLD_OUT }     -- cache ~1 detik

  expected dan age > SSE_STALE_AFTER_SECONDS → status {stale:true,
                                              reason:"no_admission_updates"}
  sebelumnya stale, sekarang tidak           → status {stale:false}
```

Dikirim **hanya pada transisi**, tidak diulang. Nol trafik Redis tambahan —
`last_broadcast_at` adalah efek samping dari subscription yang sudah dipegang. Dan
cakupannya tepat: ia menyala ketika klien **task ini** basi, baik karena ctl mati maupun
karena subscription task ini sendiri yang putus.

`expected` ada supaya keheningan yang sah tidak dibaca sebagai kegagalan: pada `SCHEDULED`
dan `CLOSED` tidak ada broadcast yang diharapkan. Bila sse-server kehilangan Redis sehingga
`expected` tak terbaca, pertahankan nilai terakhir dan biarkan jalur kehilangan-Redis yang
memiliki `stale` — keduanya tidak pernah bersaing.

**`CtlHeartbeatAge` dan dua alarmnya.** Diemisi **sse-server**, bukan `admission-ctl`:
proses yang mati tidak dapat melaporkan umurnya sendiri, dan sse-server sudah memegang
angkanya. Nilai = `now − last_broadcast_at`, per event, tiap 10 detik. Statistiknya yang
menentukan arti, dan itu dua insiden yang berbeda:

| Statistik lintas task | > 30 detik berarti | Tindakan |
|---|---|---|
| **Minimum** | tidak ada task yang menerima → `admission-ctl` memang mati | **page** |
| **Maximum** | subscription satu task putus → hanya klien task itu | ticket |

> **Ambang 30 detik = tiga heartbeat terlewat** pada `BROADCAST_HEARTBEAT_SECONDS=10`. Satu
> publish yang ter-jitter bukan gangguan; tiga yang hilang berturut-turut adalah kegagalan.
> Ambang itu juga menyerap handover leader dengan nyaman (≤ 5,5 detik), sehingga handover
> normal tidak pernah mencapai indikator customer — dan itulah alasan ia tidak dibuat lebih
> rapat.

> **Deteksi, bukan pemulihan.** Customer mengetahui antriannya membeku; antriannya tetap
> membeku. Pemulihannya ada di lease. Satu kasus tidak punya pemulihan otomatis sama sekali
> — leader yang masih memperpanjang lease-nya di satu thread timer sementara tick loopnya
> macet — dan untuk itu alarm yang membangunkan operator adalah **keseluruhan** mitigasinya.
> Itu argumen terkuat bagi `CtlHeartbeatAge`, lebih kuat daripada redundansi.

> **`SOLD_OUT` BUKAN akhir sale, dan ini tidak boleh dibaca keliru.** Sejak `SOLD_OUT`
> dibuat reversibel (Bagian 6), stok nol hanyalah keadaan sementara: kursi kembali dari
> booking yang kedaluwarsa, admission melanjutkan dari posisi antrian yang sama, dan klien
> **tetap tersambung** alih-alih diputus. Menurunkan `desired_count` ke 0 pada momen sold-out
> karena itu akan:
>
> | Akibat | Mengapa |
> |---|---|
> | Tidak ada yang menyetel `CLOSED` | transisinya ada di tick admission-ctl langkah 5 |
> | Kursi yang dilepas tidak terjual | tidak ada admission yang berjalan untuk melayaninya |
> | ~300.000 koneksi SSE kehilangan servernya | mereka sengaja dipertahankan selama `SOLD_OUT` |
>
> Kedua service karena itu **wajib tetap menyala hingga `SOLD_OUT_HOLD_SECONDS` (930 detik)
> berlalu dari `sold_out_at`**, bukan sampai stok menyentuh nol. Dampak biayanya ~15,5 menit
> ekstra per event yang sold out — sekitar 13% pada jendela 2 jam, dan itu harga untuk
> menjual kembali kursi yang dilepas.

> `desired_count = 0` setelah sale bukan optimasi opsional: 20 task yang tertinggal menyala
> berbiaya ~$425/bulan tanpa ada yang menyadarinya. Alarm `IdleTaskCount > 0` di luar jendela
> sale — ambangnya memakai `state == CLOSED`, bukan stok nol.

### Pre-Sale Checklist

| Waktu | Aksi |
|---|---|
| **T-14 hari** | Load test single-task → kapasitas koneksi per task |
| **T-7 hari** | Service Quota: Lambda concurrent executions → 6.000 (lead time 3–5 hari kerja) |
| **T-7 hari** | Kuota dan rate limit provider CAPTCHA dikonfirmasi untuk **300.000 verifikasi dalam ~60 detik**. Pada volume ini pilihan provider menjadi penentu biaya, bukan selera: harga per-assessment akan setara atau melampaui seluruh biaya variabel per event (~$36) |
| **T-7 hari** | Service Quota: Fargate On-Demand vCPU → 120 |
| **T-7 hari** | Cek burst concurrency region; kuota API Gateway (default 10.000 RPS) |
| **T-7 hari** | Konfirmasi `DUKCAPIL_RATE_LIMIT_RPS` riil vs kebutuhan ~40/s burst **dan SLA ketersediaannya** — kini ia SPOF |
| **T-7 hari** | Daftarkan URL `/payments/webhook`; uji tanda tangan dan resource policy IP |
| **T-7 hari** | Klausa T&C + persetujuan PDP final; sign-off legal; `TERMS_VERSION` dipasang |
| T-1 hari | Konfirmasi semua kuota naik di AWS Console |
| T-30 menit | ECS `desired_count` = N; Provisioned Concurrency `queue-service`, `nik-verify`, `payment-api`. Cukup awal karena halaman pra-buka tidak memakai ECS sama sekali — join ditolak 409 dan stream menolak `SCHEDULED`, sehingga 300.000 orang yang menunggu dilayani Lambda + CloudFront |
| T-10 menit | `rusuh-admission-ctl` healthy (**kedua task**); tepat **satu** pemegang `ctl:leader`; subscriber Redis aktif; kedua breaker tertutup; `SlotsOpenDrift` = 0 |
| T-5 menit | Semua task healthy; `OutboxLag` = 0 |
| Pasca-sale | Turunkan `desired_count` ke 0 **hanya setelah `state == CLOSED`** — bukan saat stok menyentuh nol (Bagian 11) |
| T-2 menit | Smoke test: 100 koneksi SSE + admission + grace lapse + NIK valid + NIK ditolak + **Dukcapil down → pause** + charge sandbox + **charge ganda bersamaan → satu charge, dua respons identik** + **webhook diputar ulang → 200 tanpa efek** + **webhook settled setelah EXPIRED → payment.refund** + rekonsiliasi `slots:open` + **rollback Aurora setelah Lua berhasil → stok dan NIK kembali** + **NIK yang sama di dua booking → ditolak** + **flush `nik:used` → rekonsiliasi memulihkan dari `nik_claims`** + resume + sold-out 410 + bypass antrian kosong + **provider CAPTCHA mati → join tetap lolos (fail-open) + alarm menyala** + **join kedua user login → memulihkan token yang sama, bukan token baru** + **CONFIRMED → join lagi diizinkan** + **leader di-kill → standby mengambil alih ≤ 5,5 detik, `now_serving` lanjut** + **KEDUA task ctl di-kill → `status {stale:true, reason:"no_admission_updates"}` sampai ke klien ≤ 30 detik, `CtlHeartbeatAge` Minimum mem-page, dan klien TIDAK menutup stream maupun mereset posisi** + **satu task sse-server diputus dari Pub/Sub → hanya klien task itu yang stale, `CtlHeartbeatAge` Maximum naik tanpa Minimum naik** + **dua task dipaksa aktif → rekonsiliasi tetap mengoreksi sekali (compare-and-adjust menolak yang kedua)** + **leader di-SIGSTOP melewati TTL lease → standby mengambil alih, dan write leader lama ditolak saat ia bangun (`StaleLeaderRejected` naik)** + **flush Redis setelah 1.000 join → rebuild dari `queue_entries` memulihkan `by_ticket`, `issued` dan `now_serving`; tidak ada token yang di-grant dua kali** |
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

> ⚠️ **Diubah oleh ADR-006** — tidak ada lagi event `grant`. Handler memanggil `GET /queue/me` saat `nowServing ≥ ticket_no` pada payload apa pun, termasuk snapshot setelah reconnect (ADR-006 §4.2).

```javascript
const es = new EventSource(url, { withCredentials: true });

es.addEventListener('message', e => {
  const { nowServing, ratePerMin, paused, serverTime } = JSON.parse(e.data);
  syncClock(serverTime);                                 // BR-10
  render(myTicketNo - nowServing, ratePerMin, paused);   // paused → banner
});

es.addEventListener('grant', async () => {
  const { grantToken } = await fetch(`/events/${ev}/queue/me`).then(r => r.json());
  goToDetailTicketPage(grantToken);
});

// TIDAK berubah oleh detektor keheningan (Bagian 11): `reason` hanya diagnostik,
// satu indikator melayani kedua pemicu. JANGAN close(), reset posisi, atau
// reconnect sendiri — 300.000 klien reconnect pada satu sinyal bersama adalah
// badai yang dihindari seluruh desain ini.
es.addEventListener('status', e => setStaleIndicator(JSON.parse(e.data).stale));

es.addEventListener('sold-out', e => {
  showSoldOutScreen(JSON.parse(e.data));
  es.close();      // WAJIB — tanpa ini EventSource reconnect otomatis
});                //  dan 300.000 klien membanjiri ALB setelah sold out

es.onerror = () => { /* biarkan retry bawaan; 410 menghentikannya sendiri */ };
```

---

## 12. Autentikasi, Identitas, Kontak, dan Re-entrancy

> ⚠️ **Diubah oleh ADR-006** — baris yang menyebut `regrant` (RTO, rebuild) dibaca sebagai pending grant (ADR-006 §2.6). Masuk normal kini selalu lewat `/queue/me` setelah `nowServing ≥ ticket_no` (ADR-006 §4.2).

Stream-ticket sekali pakai **merusak reconnect bawaan** `EventSource`: percobaan ulang ke URL
yang sama menemui ticket yang sudah dipakai → `401` → menurut HTML Living Standard status
non-200 membuat user agent **mengakhiri koneksi permanen**. Itulah mengapa autentikasi stream
memakai cookie/header — dan mengapa perilaku yang sama dipakai **dengan sengaja** sebagai
`410 Gone` pada sold-out (Bagian 4).

| Platform | Mekanisme | Token di URL? |
|---|---|---|
| Web | Cookie `HttpOnly; Secure; SameSite=Lax; Domain=.my-rusuh.my.id`; `withCredentials: true` | ❌ |
| Mobile | `react-native-sse` custom header → `X-Queue-Token` | ❌ |

| Kelas | Pengikatan `grantToken` | Requirement |
|---|---|---|
| **Login** | `account_id` | FR-29 opsional |
| **Anonim** | `queue_token` + `device_id` (di bawah) | FR-29, FR-30 |

### Pengikatan device untuk grant anonim

> ⚠️ **Diubah oleh ADR-006 §8.9** — **tanpa rebind.** Baris "Toleransi" dan paragraf "Satu rebind, dan itu sengaja
> longgar" di bawah tidak berlaku: token tamu hanya berlaku dari perangkat yang menerimanya, tidak cocok → `401
> DEVICE_MISMATCH`. `MAX_DEVICE_REBINDS`, field `rebinds` dan `DeviceRebindRate` dihapus. Token akun dicocokkan ke
> `account_id`.

Versi sebelumnya dokumen ini menulis "device fingerprint" tanpa mendefinisikan apa pun: tidak
ada header, tidak ada algoritma, tidak ada perilaku saat fingerprint berubah. Sebagai kontrol
keamanan ia tidak dapat diimplementasikan, padahal ia satu-satunya yang berdiri antara customer
dan menjual slot antriannya — dan di antrian 300.000 untuk 10.000 tiket, slot punya harga pasar.

**Yang dipakai: `device_id` acak yang dibuat klien, bukan fingerprint pasif.**

| | Keputusan |
|---|---|
| Bentuk | 128 bit acak, dibuat klien sekali, disimpan di `localStorage` (web) / secure store (mobile) |
| Transport | header `X-Device-Id`, pada `POST …/queue/join` dan pada setiap endpoint ber-grant |
| Penyimpanan | `HSET token:{uuid} device_hash HMAC(DEVICE_PEPPER, device_id)` — nilai mentah tidak pernah disimpan |
| Pemeriksaan | setiap panggilan ber-grant anonim; tidak cocok → `401 DEVICE_MISMATCH` |
| Toleransi | `HINCRBY token:{uuid} rebinds 1`, maksimum `MAX_DEVICE_REBINDS`; melebihi → tolak |
| Umur | dihapus bersama keterlibatannya (bagian terminal di bawah) |
| Login | **tidak** ditegakkan — `account_id` sudah mengikat lebih kuat |

> **Mengapa bukan fingerprint pasif.** Canvas, WebGL, daftar font dan sejenisnya rapuh —
> pembaruan browser mengubahnya — dan memanen karakteristik perangkat tanpa pemberitahuan
> adalah pemrosesan data pribadi yang tidak punya dasar di persetujuan yang kita minta
> (NFR-11, UU PDP). Nilai acak yang dibuat klien tidak mengungkap apa pun tentang perangkat;
> ia hanya rahasia ber-scope perangkat, dan itu cukup untuk tugasnya.
>
> **Mengapa di-hash saat istirahat.** `device_id` adalah quasi-identifier: ia menautkan
> kunjungan satu perangkat. Peppered HMAC membuat dump Redis tidak menghasilkan korelator
> lintas-event. Berbeda dari pepper NIK, `DEVICE_PEPPER` **boleh dirotasi antar sale** —
> tetapi tidak pernah di tengah sale, karena rotasi membatalkan setiap pengikatan yang hidup.
>
> **Satu rebind, dan itu sengaja longgar.** Menghapus data situs, incognito, atau ganti
> browser menghapus `device_id` yang sah. Pengikatan keras berarti customer itu kehilangan
> slotnya — menghukum perilaku normal demi kontrol yang dapat dilewati siapa pun yang mau
> menyalin satu nilai `localStorage`. Satu rebind yang tercatat membalik pertukarannya:
> transfer kasual tetap gagal pada percobaan ketiga, kecelakaan tidak. Metrik
> `DeviceRebindRate` memberi tahu tim bila angkanya salah (Bagian 17 #12).
>
> **Non-goal, dinyatakan terang.** Ini menghentikan "jual slot antrian saya" — pasar WhatsApp,
> screenshot token. Ia **tidak** menghentikan penyerang yang menyalin `localStorage` korban,
> dan tidak berpura-pura demikian. Kontrol untuk itu adalah pengikatan akun (FR-29), dan
> itulah mengapa login tidak memerlukan pemeriksaan ini sama sekali.

### Posisi antrian durabel: buffer di Redis, tulis ter-batch ke Aurora

> ⚠️ **Diubah oleh ADR-006** — `qpersist` + `queue_entries` + rebuild **ditunda ke produksi** (ADR-006 §9). POC bergantung pada Redis Multi-AZ.

Posisi antrian yang belum di-grant adalah satu-satunya state yang dulu tidak punya salinan
durabel: `ticket_no` hanya ada di `token:{uuid}` dan `by_ticket`, sementara baris Aurora baru
lahir saat GRANT. Pada antrian 300.000 dengan 10.000 tiket itu berarti ~290.000 posisi yang
akan hilang total bila Redis hilang.

**Insert sinkron saat join adalah implementasi yang jelas dan salah.** 300.000 join dalam ~60
detik adalah ~5.000 insert/detik ke satu tabel, tepat pada instan beban tertinggi sistem. Ia
akan menaruh Aurora di jalur kritis flash crowd, menciptakan contention tepi-kanan B-tree
(karena `ticket_no` monoton), menahan satu koneksi RDS Proxy per join pada ~5.000 Lambda
bersamaan, dan membuat **join gagal ketika Aurora lambat** — SPOF baru pada satu-satunya jalur
yang sengaja dibuat Redis-only justru demi ketahanan.

Karena itu penulisannya **di-buffer**:

```
join            : RPUSH qpersist:{ev} <...>          -- O(1), jalur cepat
leader tiap tick: LRANGE qpersist:{ev} 0 N-1         -- baca, JANGAN pop
                  INSERT INTO queue_entries (...) VALUES (...), (...)
                    ON CONFLICT (event_id, queue_token) DO NOTHING
                  COMMIT
                  LTRIM qpersist:{ev} N -1           -- buang HANYA setelah commit
```

**Baca-lalu-trim, bukan pop-lalu-tulis.** Pop lebih dulu akan kehilangan batch itu bila insert
gagal. Dengan trim sesudah commit, kegagalan berarti batch yang sama dibaca ulang pada tick
berikutnya, dan `ON CONFLICT DO NOTHING` membuat pengulangan tidak berbahaya — at-least-once
dengan sink yang idempoten, pola yang sama dengan outbox di Bagian 9.

> **Drain adalah langkah TERAKHIR pada tick, dan beranggaran waktu.** Tick yang tertahan lebih
> lama dari TTL lease kehilangan kepemimpinan sementara masih bertindak (Bagian 11). Drain
> karena itu dibatasi `QPERSIST_DRAIN_BUDGET_MS` dan diletakkan paling akhir, sehingga ia dapat
> dipotong tanpa memengaruhi admission atau sweep. Tertinggal sebentar jauh lebih murah
> daripada kehilangan lease.

**Panjang buffer ADALAH RPO, dan ia paling buruk tepat saat badai join.** Pada ~5.000 join/detik
masuk dan kapasitas drain ~2.000/detik (1.000 per batch, 2 tick/detik), antrian buffer tumbuh
~3.000/detik selama badai 60 detik — memuncak di ~180.000 entri — lalu terkuras dalam ~90
detik sesudahnya. Artinya jendela kehilangan data terbesar justru ketika data paling banyak:

| Fase | Jendela RPO |
|---|---|
| Badai join (T+0 s/d ~T+2,5 menit) | hingga ~90 detik join |
| Steady state sesudahnya | < 1 detik |

`QPersistLag` (panjang list) dan `QPersistAgeSeconds` (umur entri tertua) dialarmkan, karena
keduanya **adalah** angka RPO yang berjalan — bukan proksi untuknya. Bila benchmark menuntut
jendela lebih sempit, naikkan batch dan ganti multi-row INSERT dengan `COPY` ke tabel temp
lalu `INSERT … SELECT … ON CONFLICT`; `COPY` menangani 5.000 baris tanpa kesulitan.

> **`queue_entries` bersifat TULIS-SAJA pada operasi normal.** Ia tidak pernah dibaca kecuali
> saat rebuild. Seluruh jalur `ticket_no` tetap Redis sepenuhnya:
>
> | Operasi | Sumber | Aurora? |
> |---|---|---|
> | Menerbitkan `ticket_no` | `INCR queue:{ev}:issued` | tidak |
> | Menyimpannya | `HSET token:{uuid}` + `ZADD by_ticket` | tidak |
> | Menampilkan posisi | `ticket_no − nowServing`, keduanya dari Redis | tidak |
> | Pemulihan, perangkat sama | cookie → `HGET token:{uuid}` | tidak |
> | Pemulihan, lintas perangkat | JWT → `acct:{ev}:{account_id}` → `HGET` | tidak |
> | Grant | `by_ticket` di dalam Lua | ditulis sesudahnya, tidak dibaca |
> | Rebuild setelah Redis hilang | `queue_entries` | **hanya di sini** |
>
> Konsekuensinya penting dan mudah disalahpahami: **lag drain tidak punya efek fungsional
> apa pun.** Customer yang barisnya belum mendarat tetap punya posisi yang berfungsi penuh —
> terlihat, dapat dipulihkan, dapat di-grant. Lag hanya menentukan apa yang bertahan bila
> Redis hilang total, bukan apa yang bekerja hari ini. Tidak ada pembacaan yang bisa basi,
> karena tidak ada pembacaan.

> **Jendela ~90 detik selama badai join DITERIMA** (keputusan 30 Oktober 2026). Multi-row
> INSERT dipertahankan; `COPY` tetap menjadi jalur eskalasi bila benchmark T-14 menunjukkan
> lag jauh melampaui perkiraan. `QPersistAgeSeconds` adalah angka yang diawasi terhadap
> keputusan ini.

### RTO: antrian membeku, tidak kehilangan data — diterima

Target RTO diterima sebagai **perilaku, bukan angka** (keputusan 30 Oktober 2026): selama
failover dependensi, antrian **berhenti maju** dan tidak ada yang kehilangan posisinya di luar
jendela RPO yang sudah diterima. Tidak ada target yang lebih ketat, dan karena itu **tidak ada
perubahan infrastruktur**.

| Kegagalan | Antrian membeku selama | Perilaku |
|---|---|---|
| Handover leader `admission-ctl` | ≤ 5,5 detik | `now_serving` berhenti sesaat; tidak ada yang hilang |
| Kedua task ctl mati | 30–90 detik (penggantian ECS) | idem, lebih lama |
| Failover ElastiCache | puluhan detik | state dari replika, atau rebuild dari `queue_entries` |
| Failover Aurora | puluhan detik | INSERT grant gagal → kompensasi `DECRBY slots:open` + `regrant`, **tanpa** publish `grant` (Bagian 6). Admission berhenti alih-alih salah memberi slot |

Yang membuat ini dapat diterima adalah bahwa setiap jalur kegagalan berhenti **aman**: tidak
ada grant yang terbit tanpa slot, tidak ada stok yang oversell, tidak ada posisi yang
diserobot. Sistem menunggu, lalu melanjutkan dari tempat yang sama.

> **Pembekuan itu HARUS terlihat customer, dan itu menuntut pengawasan.**
> Antrian yang membeku tanpa tanda apa pun tidak dapat dibedakan dari halaman yang rusak —
> angka yang tidak bergerak dengan koneksi yang tampak sehat. Menerima "antrian membeku"
> karena itu memaksa keputusan yang menyertainya: detektor keheningan di sse-server plus
> `CtlHeartbeatAge` dengan dua alarmnya, keduanya **sudah diputuskan** (Bagian 11). Leader
> lease memulihkan penyebab yang paling sering; pengawaslah yang menangkap sisanya dan
> membuktikan lease benar-benar menyala.

> **Harga yang dibayar customer yang sedang di dalam: jam mereka tetap berjalan.**
> `PAUSE_TTL_GRACE_SECONDS` hanya menyala pada transisi `paused`, yang digerakkan **breaker**
> (payment gateway / Dukcapil). Failover Redis atau Aurora bukan peristiwa breaker, sehingga
> customer yang sedang `SELECTING` saat failover 45 detik kehilangan 45 detik dari 300 detiknya
> tanpa penggantian. Mitigasi termurahnya memakai mesin yang sudah ada: perlakukan **N tick
> `admission-ctl` yang gagal berturut-turut** sebagai transisi `paused`, sehingga grace yang
> sama diberikan. Belum diputuskan — diterima apa adanya untuk saat ini.

> **RPO data berbayar sudah nol, dan itu bukan hal yang perlu diputuskan.** `CONFIRMED`,
> `tickets`, `nik_claims` dan baris pembayaran hidup di Aurora, yang mereplikasi setiap commit
> ke enam salinan di tiga AZ pada lapisan storage sebelum mengakui transaksinya. Transaksi
> yang sudah commit karena itu bertahan terhadap kehilangan satu AZ tanpa kehilangan data.
> Yang tersisa untuk diputuskan hanyalah jendela posisi antrian di atas dan target RTO —
> Bagian 17 #7.

### Rebuild Redis dari Aurora setelah kehilangan total

Tabel saja tidak memberi pemulihan — prosedurnya harus ada dan dilatih:

```
1. state, sold_out_at  ← event_queue_state
2. issued              ← MAX(ticket_no) FROM queue_entries WHERE event_id = $ev
3. now_serving         ← MAX(qe.ticket_no)
                            FROM bookings b
                            JOIN queue_entries qe ON qe.queue_token = b.queue_token
                           WHERE b.event_id = $ev
4. by_ticket           ← ZADD dari queue_entries WHERE ticket_no > now_serving
5. token:{uuid}        ← HSET per baris queue_entries
6. slots:open, stock, nik:used ← rekonsiliasi yang sudah ada (Bagian 6)
```

> **`now_serving` sengaja TIDAK dipersistenkan.** Ia diturunkan dari nomor antrian tertinggi
> yang punya baris `bookings` — dan itu **lebih benar** daripada nilai tersimpan, yang bisa
> tertinggal satu tick dan menyebabkan token yang sudah di-grant di-grant ulang. Turunannya
> mengoreksi dirinya sendiri: apa pun yang sudah pernah di-grant punya baris, jadi
> watermarknya tepat. Token di `regrant` tidak memajukannya, dan itu benar — watermark sudah
> melewatinya; yang gagal adalah INSERT-nya, bukan gilirannya.

### Pemulihan antrian: dua lapis, dan yang kedua harus berupa READ

Customer menutup tab, berganti perangkat, atau aplikasinya di-install ulang. Posisi antrian
tidak boleh hilang karena itu (BR-09, NFR-01). Dua lapis menanganinya:

| Lapis | Berlaku saat | Mekanisme |
|---|---|---|
| **Cookie / header** | perangkat yang sama — tab ditutup, refresh, kembali dari aplikasi bank | `queue_token` → `HGET token:{uuid}` → state + posisi |
| **Identitas akun** | perangkat berbeda, cookie dibersihkan, incognito, app di-install ulang | identity token (JWT) → `acct:{ev}:{account_id}` → uuid token → state yang sama |

**Lapis kedua WAJIB lewat `GET …/queue/me`, bukan `POST …/queue/join`.** Keduanya secara
teknis dapat memulihkan — join memang mengembalikan keterlibatan yang ada (BR-07) — tetapi
join adalah jalur tulis yang membawa **CAPTCHA** dan dua rate limit (`rl:join:{ip}` 20 per 5
menit, plus 3 join per customer per menit). Memaksa customer yang sekadar kembali untuk
melewati semua itu berarti:

- ia harus menyediakan token CAPTCHA segar hanya untuk mengambil posisi yang sudah dimilikinya
- di belakang NAT kantor atau operator seluler, 20 join per 5 menit habis dengan mudah — dan
  yang menghabiskannya justru orang-orang yang sedang reconnect
- pemulihan diperlakukan sebagai serangan

Karena itu `GET …/queue/me` menerima **identity token ATAU cookie**, dan menjadi satu-satunya
jalur pemulihan yang disarankan. `POST …/queue/join` tetap memulihkan sebagai jaring pengaman
BR-07, bukan sebagai jalur normal.

> **Umur kunci adalah bagian dari kontrak pemulihan, bukan detail implementasi.**
> `token:{uuid}` diberi `EX SALE_WINDOW_SECONDS` — tanpa TTL ia membocorkan 300.000 hash per
> event yang tidak pernah direklamasi; dengan TTL yang lebih pendek dari masa tunggu,
> pemulihan rusak diam-diam. `Max-Age` cookie **diturunkan dari nilai yang sama**, bukan
> dipatok 7200 detik, supaya cookie dan kunci yang dirujuknya tidak pernah kedaluwarsa pada
> waktu yang berbeda.

> **Kedua lapis ini mengandaikan Redis masih ada.** `token:{uuid}` dan `by_ticket` keduanya
> hanya hidup di Redis, sehingga pada kehilangan Redis total cookie menunjuk ke sesuatu yang
> tidak ada lagi dan customer yang belum di-grant kehilangan posisinya — login maupun tamu.
> Itu pertanyaan RPO, bukan pertanyaan pemulihan sesi; lihat Bagian 17 #7.

### Empat kredensial, empat slot

Perjalanan ini memakai empat kredensial yang berbeda jenis, dan **tidak ada satu slot
transport pun yang membawa lebih dari satu**:

| Kredensial | Slot | Bentuk | Membuktikan | Dapat dicabut |
|---|---|---|---|---|
| Access token (sesi akun) | `Authorization: Bearer` | JWT HS256, `ACCESS_TOKEN_TTL_SECONDS` | siapa pemegangnya | ❌ — stateless, tunggu TTL |
| Grant token | `X-Grant-Token` | opaque `gt_…`, `grant:{token}` di Redis | pemegangnya punya slot | ✅ — satu `DEL` |
| Queue token (ruang tunggu) | cookie `queue_token` / `X-Queue-Token` | opaque `qt_…`, `token:{uuid}` di Redis | posisinya dalam antrian | ✅ |
| E-ticket token | query `?t=` pada tautan email | HMAC bertanda tangan, satu `ticket_id`, sampai tanggal event | hak baca **satu** tiket | hanya dengan rotasi kunci |

> **E-ticket token ada karena FR-26 menuntutnya.** Tiket dikirim ke *masing-masing pemegang
> NIK*, dan pemegang NIK kedua dan ketiga biasanya bukan pemilik akun — sering bukan pembeli
> sama sekali. Rute e-tiket yang hanya menerima sesi akun berarti tombol di email mereka
> menjawab 401, yaitu kegagalan jalur pengiriman utama requirement itu. Token ini tidak
> membawa identitas akun, tidak memberi akses ke apa pun selain satu tiket, dan meneruskan
> emailnya memang meneruskan tiketnya — sama seperti e-tiket mana pun; pemeriksaan identitas
> di gerbang venue di luar cakupan (Bagian 8).

### Kapan kredensial antrian dihapus

Keduanya punya TTL, tetapi TTL saja berarti kredensial tetap hidup lama setelah ia berhenti
berarti apa pun. Penghapusan aktif pada transisi terminal — `admission-ctl` dan webhook sudah
memiliki transisi itu (Bagian 6):

| Keadaan | `token:{uuid}` | Mengapa |
|---|---|---|
| **grant diterbitkan** | **tetap** | §3.3 memulihkan keadaan *setelah* grant: `GRANTED`, `SELECTING`, `AWAITING_PAYMENT`. Pemicunya yang paling umum adalah "kembali dari aplikasi bank" — menghapus di sini persis kegagalan NFR-02 |
| EXPIRED, **LAPSED**, ABANDONED | `DEL` | Terminal-kegagalan: tidak ada apa pun untuk kembali ke sana. `grant:{token}` dan `acct:{ev}:{account_id}` dilepas di titik yang sama (Bagian 6, Bagian 7) |
| **CONFIRMED** | `DEL` + `SET settled:{uuid}` | Terminal-keberhasilan: justru **ada** sesuatu yang bernilai untuk kembali ke sana |

**Tombstone, bukan jendela perpanjangan.** `settled:{uuid}` memuat `booking_id` dan tidak
lebih — bukan `ticket_no`, bukan `account_id`, bukan `device_hash`. Ia menjawab §3.3 dengan
`CONFIRMED` sehingga frontend merutekan ke halaman tiket, tanpa menyimpan kredensial yang
dapat memulihkan apa pun. Umurnya `SETTLED_TOMBSTONE_SECONDS`.

> **Yang dicegahnya adalah sebuah kebohongan, bukan kebocoran.** Tanpa tombstone, tamu yang
> me-refresh di celah antara pembayaran lunas dan email e-tiket tiba memanggil §3.3 dengan
> token mati, dan pemetaan frontend untuk token tak terselesaikan adalah **layar
> kedaluwarsa** — memberi tahu orang yang baru saja membayar bahwa sesinya habis. Itu
> "jangan PHP" dalam bentuk paling murni, dan ia salah secara faktual, bukan hanya buruk.
>
> **Cookie dibersihkan pada permintaan berikutnya, bukan saat transisi.** Transisi terminal
> terjadi di `admission-ctl` atau webhook, bukan di dalam permintaan customer, jadi tidak ada
> respons untuk menumpangi. §3.3 yang menjawab dari tombstone mengirim
> `Set-Cookie: queue_token=; Max-Age=0` bersamaan.
>
> **Setelah tombstone kedaluwarsa, email adalah rutenya** — ditambah mendaftar dengan NIK yang
> sama, yang mewarisi tiket lewat klausa kepemilikan (*pembeli **atau** pemegang NIK
> terdaftar*). `contact_email` dan `holder_email` wajib di `POST /bookings`, jadi kanal itu
> selalu ada; itulah yang membuat penghapusan ini dapat diterima, dan juga mengapa token
> e-ticket di atas wajib ada.
>
> ⚠️ **Diubah oleh ADR-006 §8.15** — hanya `contact_email` yang wajib di `POST /bookings`; kanal email tetap selalu ada.

> **Mengapa ini perlu dinyatakan.** Sampai kontrak v2.2 ketiganya dikirim di
> `Authorization: Bearer` dan ADR ini menyebut yang kedua `accessToken` — nama kredensial
> pertama. Dua di antaranya bahkan tidak bisa sama: klausa kepemilikan di bawah mengikat
> Bearer ke kolom `queue_token`, sementara `GET /users/me/tickets` hanya mungkin berarti JWT
> akun. Sejak kontrak v2.2 keduanya juga berumur 1080 detik, sehingga pembeda runtime
> terakhir pun hilang.
>
> **Middleware memilih per rute, bukan per token.** Ia tidak mencoba memverifikasi tanda
> tangan lalu jatuh ke pencarian Redis. Fallback semacam itu membuat grant tamu lolos di
> `GET /users/me/tickets`, atau JWT akun lolos pada `booking_id` yang bukan miliknya — tepat
> yang dicegah bagian berikut.
>
> **Grant wajib, sesi opsional, pada §4/§5 kontrak.** Tamu boleh membeli (FR-02) dan tidak
> punya sesi; yang dimilikinya adalah slot. Sesi tetap diterima di atas grant karena itulah
> satu-satunya cara `account_id` terisi saat booking dibuat — dan tanpa itu booking customer
> login tidak dapat muncul di `GET /users/me/tickets` tanpa menebak.

### Kepemilikan: token yang valid ≠ token untuk booking ini

`grantToken` yang sah membuktikan pemegangnya punya slot, **bukan** bahwa slot itu milik
`booking_id` di URL. Keduanya harus diikat, pada **setiap** endpoint yang menerima
`booking_id` atau `ticket_id` dari path:

```sql
WHERE id = $1 AND grant_token = $grantToken
```

Tanpa ikatan ini siapa pun yang memegang satu token sah dapat membaca booking orang lain,
membatalkan pembayaran orang lain lewat `/retry`, atau melakukan charge atas booking orang
lain. `booking_id` berbentuk `RB-00912` — berurutan dan mudah ditebak, dan pada 300.000
customer bersamaan enumerasinya sepele.

| Endpoint | Terikat |
|---|---|
| `GET /bookings/{id}` | ✅ |
| ~~`GET /bookings/{id}/tickets`~~ (dihapus, ADR-006 §8.8) | — |
| `GET /payments/{id}/methods` | ✅ |
| `POST /payments/{id}` | ✅ |
| `POST /payments/{id}/retry` | ✅ |
| `GET /payments/{id}/status` | ✅ |

Bila `Authorization` ikut dikirim, `account_id` menjadi **konjungsi kedua**, bukan pengganti:
grant tetap yang menjaga akses. Tamu tidak mengirimnya dan booking-nya tersimpan dengan
`account_id = NULL`, terikat lewat NIK saja.

**Tidak cocok → `404`, bukan `403`.** `403` mengonfirmasi bahwa booking itu ada, yang sudah
cukup untuk memetakan rentang id yang aktif. Pemeriksaan ini dijalankan **sebelum** validasi
status apa pun, agar kode respons tidak pernah membocorkan keadaan booking orang lain.

### Re-entrancy (BR-09)

`GET /events/{ev}/queue/me` adalah satu-satunya endpoint yang dipanggil klien saat kembali, apa pun
keadaannya, dan ia mengembalikan **token yang sama** selama TTL belum habis:

| Status booking | Respons | Halaman tujuan |
|---|---|---|
| tidak ada | `{queued, ticketNo, nowServing}` | Waiting Room |
| GRANTED | `{granted, grantToken}` | Detail Ticket |
| SELECTING / HELD | `{grantToken, bookingId, bookingExpiresAt, serverTime}` | Detail Ticket, countdown dilanjutkan |
| AWAITING_PAYMENT | `{grantToken, bookingId, paymentExpiresAt, paymentRef, serverTime}` | Payment, countdown dilanjutkan |
| CONFIRMED | `{bookingId}` | Halaman tiket |
| EXPIRED / LAPSED | `{expired: true}` | Layar expired |

`grantToken` disimpan di `grant:{token} EX GRANT_TOKEN_TTL_SECONDS` sehingga refresh, ganti tab, atau kembali
dari aplikasi bank tidak pernah menerbitkan token baru maupun memindahkan customer.

### `contact_email`, `holder_email`, dan penerimaan T&C

> ⚠️ **Diubah oleh ADR-006 §8.15** — `holder_email` dihapus; hanya `contact_email` (pembeli) yang dikumpulkan dan seluruh e-tiket dikirim ke sana. `contact_email` wajib sejak `HELD` (diisi pada `POST /bookings`), bukan sejak `SELECTING`: halaman pilih tiket sudah memindahkan booking ke `SELECTING` sebelum customer mengisi apa pun (migrasi 0008).
>
> ⚠️ **Diubah oleh ADR-006 §8.16** — `terms_version`, `terms_accepted_at` dan `TERMS_VERSION` dihapus; persetujuan dicatat saat registrasi.

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

> ⚠️ **Diubah oleh ADR-006** — `/queue/serving` tanpa query string, TTL CDN 1 detik, respons menambah `poll_tiers` (ADR-006 §8.5). Klien mobile tidak menunggu event `grant` (ADR-006 §4.2).

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

`LAPSED` bersifat **final** — konsekuensinya kehilangan slot, bukan penundaan. Kesempatan
kedua dihapus di Bagian 7: ia dilayani mendahului orang yang belum pernah mendapat giliran,
dan ia tidak dapat membedakan push yang tertahan OS dari customer yang pergi. Menjaga
perangkat menyala selama perang tiket ada di sisi customer.

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

> ⚠️ **Diubah oleh ADR-006 §8.14** — FR-05 dipenuhi tanpa jalur langsung-masuk: join tidak menunggu dan tidak menjawab `granted`; bila ada tempat kosong, customer menerima giliran dalam satu tick (≤ 500 ms) lewat stream + `/queue/me`. Selain itu, FR-08 oleh `nowServing ≥ ticket_no` → `/queue/me` (bukan `event: grant`), FR-09 oleh ADR-006 §6.

> ⚠️ **Diubah oleh ADR-006 §8.15** — FR-26 diubah: e-tiket ke email pembeli (`contact_email`), bukan ke tiap pemegang NIK.

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
| FR-13, FR-14 | Maks 3 **lintas kategori**; tolak melebihi stok | Bagian 8 `hold_booking.lua` — kuota dijumlahkan atas seluruh kategori, stok diperiksa untuk semuanya sebelum satu pun dikurangi (all-or-nothing) |
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
| NFR-05 | FCFS, tidak dapat dilangkahi | Bagian 6 `INCR` + `by_ticket`, CAPTCHA tak terlihat + rate limit per IP di `/join`, dan BR-07 untuk user login — sisa celahnya (tanpa WAF, tamu tak terbatas) di Bagian 16 |
| NFR-06 | p95 < 1 detik | Bagian 10 |
| NFR-07 | Token proteksi; PCI-DSS via gateway | Bagian 12; Bagian 8 — kartu tidak dibuka di POC, sehingga tidak ada data kartu yang masuk sistem. Syaratnya bila kartu ditambahkan ada di Bagian 8 |
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

BR-07 **dipulihkan** sebagai batas satu keterlibatan aktif per event untuk user login —
alasannya okupansi slot, bukan scalping (Bagian 8); tamu tidak terbatas, konsekuensinya
ada di Bagian 16.

---

## 15. Estimasi Biaya @ 300.000

| Komponen | Per sale event |
|---|---|
| ECS Fargate SSE (43 task × 2,5 jam @7.000 conn/task) | ~$10.61 |
| ECS Fargate admission-ctl (2 task: leader + standby) | ~$0.12 |
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

> ⚠️ **Diubah oleh ADR-006** — baris "Batch INSERT GRANTED gagal" dan "`regrant` livelock" digantikan oleh pending grant yang diulang tanpa batas + alarm 5 detik (ADR-006 §2.3).

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
| **Tanpa WAF, proteksi bot di `/join` tidak lengkap** | Kini tiga lapis: rate limit Redis per IP (20 / 5 menit), **CAPTCHA tak terlihat**, dan **BR-07** untuk user login. Yang masih hilang: managed rule, reputasi IP, dan mitigasi DDoS L7 — **degradasi NFR-05 yang diterima untuk POC**, harus dipulihkan sebelum produksi |
| **Customer tamu tidak dibatasi BR-07** | Tidak ada identitas stabil untuk ditegakkan; batas tamu hanya CAPTCHA + rate limit per IP. Konsekuensinya strategi optimal penyerang adalah tidak login, sehingga BR-07 membatasi pengguna kooperatif — diterima, dengan CAPTCHA sebagai proteksi sebenarnya (Bagian 8) |
| **Provider CAPTCHA tidak tersedia** | **Fail-open** dengan alarm `CaptchaFailOpenSeconds`. Fail-closed akan mematikan sale sebelum dimulai, dan CAPTCHA melindungi keadilan, bukan kebenaran — tidak ada oversell, NIK, atau uang yang terpengaruh (Bagian 8) |
| `acct:{ev}:{id}` hilang pada flush/failover | Join ganda lolos di Redis, tetapi indeks parsial `br07_one_active_per_event` di Aurora menolaknya pada GRANT (Bagian 8) |
| Okupansi slot naik ke maks 15 menit (5 + 10) | Hanya ekor yang terpengaruh: median selesai 2–4 menit. Slot yang ditinggalkan terkunci 15 menit, bukan 10 — dipantau lewat `ExpiryRate` per tahap |
| Crash antara COMMIT dan `ZADD` | Deadline otoritatif di Aurora; sweep me-rescore; `ExpiryRescored` dipantau (Bagian 7) |
| Klien lama tanpa `es.close()` pada sold-out | Server menjawab `410 Gone`; `SoldOutStreamRejections` dipantau |
| Jalur fallback outbox p95 ≈ 1107ms | `OutboxInlineHitRate`; turunkan interval publisher bila sering |
| Batch INSERT GRANTED gagal | Retry → `DECRBY` + `regrant` + tidak publish (Bagian 6) |
| Kuota Dukcapil menjadi plafon throughput | Konfirmasi T-7 hari; uji di rehearsal |
| Payment gateway tidak tersedia | Breaker fail closed + pause admission + resume otomatis. Pause melewatkan ADVANCE, termasuk pengurasan `regrant` (Bagian 6) |
| Double-charge saat timeout gateway | Kunci per-percobaan `{booking_id}:{attempt}` + charge reconciler read-only (Bagian 8); charge tidak pernah dikirim ulang |
| Outbox at-least-once merusak `slots:open` | Lease guard (Bagian 9) + rekonsiliasi (Bagian 6) |
| **Rollback Aurora setelah `hold_booking.lua` berhasil** | Redis di luar transaksi: stok dan NIK bocor, dan jalur expiry `SELECTING` tidak mengembalikannya. Kompensasi eksplisit (Bagian 8) + rekonsiliasi stok tiap 60 detik (Bagian 6) + `HoldCompensated` dialarmkan |
| `nik:used` bocor di Redis | Direkonsiliasi terhadap `nik_claims` tiap 60 detik + `EX` sepanjang sale (Bagian 6) |
| Posisi antrian yang belum di-grant hilang saat Redis hilang | **Ditutup**: `queue_entries` ditulis ter-batch oleh leader; rebuild di Bagian 6. Sisa eksposurnya adalah lag drain — paling lebar tepat saat badai join, `QPersistLag` dialarmkan (Bagian 17 #7) |
| Drain `qpersist` tidak mengejar laju join | Buffer tumbuh dan RPO melebar; tidak ada data yang rusak. Naikkan `QPERSIST_BATCH_SIZE` atau pindah ke `COPY`. `QPersistAgeSeconds` adalah angka RPO yang berjalan |
| Batch insert gagal di tengah drain | `LTRIM` hanya setelah COMMIT, jadi batch yang sama dibaca ulang pada tick berikutnya; `ON CONFLICT DO NOTHING` membuat pengulangan tidak berbahaya |
| **Redis flush / failover menghapus `nik:used`** | BR-02 tetap ditegakkan `UNIQUE (event_id, nik_hmac)` di Aurora; rekonsiliasi memulihkan kunci Redis dan `NikClaimRestored` dialarmkan |
| **Pepper dirotasi, kapan pun** | Seluruh hmac berubah dan BR-02 gagal senyap; backfill **tidak mungkin** karena menuntut NIK plaintext yang tidak disimpan. Pepper diperlakukan permanen (Bagian 8) |
| **Pepper hilang** | Setiap `nik_hmac` tidak dapat dicocokkan dan BR-02 berhenti bekerja selamanya. Mitigasi: pepper disimpan sebagai secret yang dapat di-backup — alasan utama `GenerateMac` ditolak (Bagian 8) |
| **Pepper bocor** | Enumerasi offline seluruh NIK tersimpan; ruang NIK valid cukup kecil. **Tidak dapat diremediasi** karena rotasi tidak tersedia — diterima untuk POC; hardening `GenerateMac` sebelum produksi (Bagian 8) |
| Rekonsiliasi memakai drift basi | `current_drift`, bukan `last_drift` (Bagian 6) |
| **Redis flush menghapus `stock:{ev}:{cat}`** | `GET` mengembalikan `false`, bukan 0 — skrip menjawab `STOCK_UNINITIALIZED` (503) alih-alih mengumumkan sold-out, dan rekonsiliasi menanam ulang **segera** tanpa aturan dua-tick (Bagian 6, 8) |
| **`stock` menjadi negatif** | Redis tidak punya `CHECK (stock >= 0)`; Fase 3 memeriksa hasil `DECRBY` dan mengembalikan seluruh pengurangan plus kunci NIK bila negatif, lalu `StockNegativeGuard` dialarmkan (Bagian 8) |
| **Hold multi-kategori sebagian berhasil** | Fase 1 memeriksa seluruh kategori sebelum Fase 3 mengurangi satu pun, dalam satu skrip atomik (Bagian 8) |
| Kapasitas kategori diubah saat sale berjalan | Garis dasar drift bergeser dan koreksi salah arah — kapasitas **imutabel selama sale**, perubahan hanya di luar jendela + reset `last_drift` (Bagian 6) |
| Webhook duplikat dari gateway | `payment_events.gateway_event_id UNIQUE`; selalu 200 |
| **Pembayaran tiba setelah booking EXPIRED** | Stok sudah dijual ulang. Webhook `settled` terhadap baris EXPIRED memicu `payment.refund` + alarm `LatePaymentRefund` (Bagian 9) |
| Customer terkunci setelah charge ditolak | Jalur deny menaikkan `payment_attempts` sendiri → `order_id` berikutnya sudah baru (Bagian 9) |
| **`booking_id` dapat ditebak; token sah dipakai atas booking orang lain** | Setiap endpoint ber-`booking_id` mengikat `queue_token`; tidak cocok → `404` sebelum pemeriksaan status apa pun (Bagian 12) |
| **Entri NIK tersimpan tanpa bukti verifikasi** | Mustahil: `CHECK (verified_at IS NOT NULL)` + Dukcapil dipanggil sebelum INSERT, bukan sesudah (Bagian 8) |
| Status NIK berubah di Dukcapil setelah tersimpan | **Diterima.** Bukti verifikasi permanen, tanpa revalidasi berkala. Tidak berkonsekuensi selama gate check-in di luar lingkup (Bagian 8) |
| Stok dilepas lalu callback sukses tiba | Stok **tetap** di-hold selama `PENDING_CANCELLATION`; pelepasan hanya setelah grace berakhir (Bagian 7) |
| Refund disputed tidak otomatis di POC | Ditandai `payment_disputed`, customer diberi tahu lewat email, operator merekonsiliasi lewat `GET /admin/reconciliation/disputed` (Bagian 9) |
| Antrian menerima lebih banyak orang daripada yang bisa dilayani | `QUEUE_CAPACITY` + `QUEUE_FULL` fail fast; state antrian terlihat di halaman utama sebelum customer mengklik (Bagian 6) |
| **`grantToken` kedaluwarsa sebelum jendela pembayaran habis** | Umur token diturunkan dari GRACE + BOOKING_TTL + PAYMENT_TTL, bukan konstanta lepas (Bagian 7) |
| Ganti metode meninggalkan dua instrumen hidup | `/retry` membatalkan transaksi lama di Midtrans **sebelum** COMMIT; gagal cancel → attempt tidak naik (Bagian 9) |
| Customer menerbitkan puluhan VA dengan ganti metode berulang | `MAX_PAYMENT_ATTEMPTS` → 429 (Bagian 8) |
| **VA tetap dapat dibayar setelah stok dijual ulang** | `custom_expiry` menyamai TTL saat charge + `POST /v2/{order_id}/expire` dari sweep (Bagian 7, Bagian 9) |
| Form kartu kelak mengirim nomor kartu ke backend | Runtuhkan NFR-07 dan perluas cakupan PCI-DSS. Tidak ada endpoint menerima field kartu (Bagian 8) |
| Outbox retry menerbitkan QR ganda | `UNIQUE (booking_item_id)` pada `tickets` (Bagian 4) |
| Submit ulang `/bookings` menolak NIK sendiri | Guard `FOR UPDATE` mengembalikan booking yang ada (Bagian 8) |
| Task ECS tertinggal menyala setelah sale | Alarm `IdleTaskCount > 0` di luar jendela sale (Bagian 11) |
| **ECS diturunkan ke 0 pada momen sold-out, bukan pada `CLOSED`** | `SOLD_OUT` reversibel: kursi yang dilepas tidak akan terjual, `CLOSED` tidak pernah disetel, dan 300.000 koneksi yang sengaja dipertahankan kehilangan servernya. Definisi "selesai" dipatok ke `state == CLOSED` di Bagian 11 |
| Kuota Fargate vCPU / Lambda concurrency | Service Quota Request T-7 hari |
| **Tick tertahan lebih lama dari TTL lease** — leader kehilangan kunci sementara masih bertindak | Kedua operasi non-idempoten punya pengaman mandiri: compare-and-adjust pada rekonsiliasi stok, `grace_applied NX` pada pause grace. Lease menjaga ketersediaan, bukan satu-satunya penjaga korektifitas (Bagian 11) |
| **Failover Redis membuat lease dipegang dua task** | Replikasi ElastiCache asinkron, jadi `SET` dapat hilang saat failover primary. Tidak dapat dicegah — ditutup oleh pengaman di sumber daya + fencing token `ctl:epoch`, dengan `StaleLeaderRejected` dialarmkan (Bagian 11) |
| `EXPIRE` / `DEL` buta pada lease | Keduanya wajib lewat Lua bersyarat (`GET == task_id` dalam operasi yang sama); `DEL` buta akan menghapus lease task lain dan justru mengundang pemimpin kedua (Bagian 11) |
| Handover leader terlalu sering karena hiccup Redis | TTL 5 detik memberi 10× margin atas tick 500 ms; `CtlLeaderHandovers` dialarmkan |
| Kedua task ctl mati bersamaan | ECS mengganti keduanya; selama itu antrian membeku. **Lease tidak menolong** — tidak ada yang mengambilnya. Detektor keheningan sse-server menyalakan `status {stale:true}` ke customer, dan `CtlHeartbeatAge` statistik **Minimum** mem-page operasi (Bagian 11) |
| **Leader hidup tetapi tick loop macet** | Lease tetap diperpanjang oleh thread timer, jadi standby tidak pernah terpicu dan tidak ada pemulihan otomatis sama sekali. Satu-satunya mitigasi adalah alarm `CtlHeartbeatAge` yang membangunkan operator (Bagian 11) |
| ALB ramp 5.000 conn/s di T=0 | Uji pada ramp |
| Mobile background tidak menerima grant | **Kerugian yang diterima, tanpa mitigasi teknis.** Tanpa grace antrian jam booking 300 detik sudah berjalan saat push terlewat; bila habis, slot hilang dan tidak ada kesempatan kedua (Bagian 6). Menjaga perangkat menyala ada di sisi customer; yang tersisa adalah peringatan pra-gerbang di FE (Bagian 17 #13) |
| Redis Pub/Sub kehilangan pesan saat failover | Resync + re-broadcast; `status {stale:true}`; heartbeat 10 detik. Bila yang putus hanya **satu** task sse-server, `admission-ctl` sehat dan hanya detektor per-task yang menangkapnya — `CtlHeartbeatAge` statistik **Maximum** (Bagian 11) |
| `regrant` livelock | Dibatasi `GRANT_INSERT_MAX_RETRIES`; `EXPIRE` + job post-sale. Tidak ada jalur no-show yang dapat mengisinya berulang |
| Kontensi kategori saat tekan Pay | Aturan gagal-hold: slot tidak hilang (Bagian 8) |
| NIK terkunci oleh booking kedaluwarsa | `DEL nik:used` pada jalur expiry (Bagian 8) |
| PII pihak ketiga ke API eksternal | Persetujuan dicatat terpisah dari T&C; HMAC + KMS; retensi di Bagian 8 |
| **NIK pihak ketiga di `saved_niks` tidak punya jalur hapus self-service** | Pemilik akun dapat menghapus; orang yang NIK-nya disimpan tidak punya akun untuk memintanya. Ditangani operasional — **titik terlemah terhadap UU PDP**, diterima untuk POC (Bagian 8) |
| Permintaan hapus masuk sebelum event | Dijadwalkan, bukan dieksekusi: `nik_claims` menegakkan BR-02 dan `tickets.nik_hmac` dipakai di gerbang. Dihapus setelah event + jendela sengketa (Bagian 8) |
| PII bertumpuk tanpa batas tanpa purge berjadwal | Diterima untuk POC; `queue_entries` tumbuh 300.000 baris/event. Purge berjadwal adalah tindak lanjut sebelum produksi (Bagian 8) |

---

## 17. Yang Masih Terbuka

> ⚠️ **Diubah oleh ADR-006** — #5 (formula admission) dan #10 (`SALE_WINDOW_SECONDS` untuk POC) diputuskan di ADR-006 §3 dan §8.2; nilai `MAX_CONCURRENT_SLOTS` tetap terbuka. Butir terbuka baru ada di ADR-006 §10.

| # | Item | Pemilik |
|---|---|---|
| 1 | `DUKCAPIL_RATE_LIMIT_RPS` dan **SLA ketersediaan** Dukcapil | **Pemilik tender / Dukcapil** |
| 2 | Biaya per panggilan Dukcapil | **Pemilik tender** |
| 3 | Cap antrian / pesan jujur saat 300.000 : 10.000 tiket | **Pemilik tender** |
| 4 | Kapasitas koneksi per ECS task | Tim — **T-14 hari** |
| 5 | `MAX_CONCURRENT_SLOTS` + maksimum okupansi slot | Tim |
| 6 | Daftar skenario load test + threshold, **termasuk rehearsal T-2 hari** (ramp sintetis 300.000 join, fan-out SSE, sold-out lalu resume) — Bagian 11 | Tim + QA |
| 7 | Aksesibilitas (`aria-live` pada angka antrian dan banner pause) | Tim FE |
| 8 | Kuota provider CAPTCHA untuk 300.000 verifikasi dalam ~60 detik | Tim — **T-7 hari** |
| 9 | Alamat IP webhook Midtrans untuk allowlist | Tim — **T-7 hari** |
| 10 | **`SALE_WINDOW_SECONDS`** — belum pernah punya nilai, dan kini memikul TTL `acct`, TTL `token:{uuid}`, **dan** `ACCESS_TOKEN_TTL_SECONDS`. Fallback 21600 dipakai bila env tidak didefinisikan | Tim + **Pemilik tender** (bergantung jam penjualan) |
| 11 | Apakah `nik_verified` dikeluarkan dari claim access token. Pada 22740 detik claim itu adalah otorisasi ter-cache 6 jam; mengeluarkannya berarti satu pembacaan basis data per permintaan di bawah flash crowd | Tim |
| 13 | Peringatan pra-gerbang di FE: jaga aplikasi tetap di depan, izinkan notifikasi, jangan andalkan push saat layar terkunci. Ini **satu-satunya** mitigasi yang tersisa bagi customer mobile sejak kesempatan kedua atas `LAPSED` dihapus (Bagian 7), dan ia peringatan — bukan jaminan | Tim FE |
| 12 | ⚠️ **Diputuskan oleh ADR-006 §8.9: tanpa rebind.** **`MAX_DEVICE_REBINDS`**, dan apakah pengikatan device ditegakkan di web — menghapus storage browser itu lumrah, dan tiap rebind yang ditolak adalah slot yang hilang bagi customer yang sah | Tim + QA |

**Diputuskan dan ditutup**: pengawasan kesegaran antrian — **sse-server** mendeteksi
keheningan `sse-broadcast` > `SSE_STALE_AFTER_SECONDS` (30 detik, = 3 heartbeat) selama
`state ∈ {OPEN, SOLD_OUT}` dan menyalakan `status {stale:true, reason}` lewat event yang
sudah ada, sementara `CtlHeartbeatAge` diemisi sse-server dengan dua alarm (**Minimum** >
30 detik mem-page = ctl mati; **Maximum** = subscription satu task putus); pengawas
berbasis timer **di klien ditunda** karena setelah leader lease yang tersisa hanya
sse-server macet sendiri atau TCP half-open, dan nol kode klien lebih berharga; pepper
NIK disimpan sebagai secret yang dikelola KMS dan
di-HMAC di dalam proses — `GenerateMac` ditolak karena kunci yang tidak dapat diekspor
menukar satu kegagalan permanen dengan yang lain, sementara rotasi pepper memang tidak
mungkin; retensi PII — tanpa batas waktu, dihapus manual atas permintaan
pemilik data, dengan lantai "tidak sebelum event berlangsung" karena `nik_claims` menegakkan
BR-02 dan `tickets.nik_hmac` memvalidasi di gerbang; RTO diterima sebagai perilaku — **antrian membeku, tidak
kehilangan data** — tanpa perubahan infrastruktur; RPO — nol untuk data berbayar, dan
jendela ~90 detik selama
badai join diterima untuk posisi antrian; posisi antrian dipersistenkan ke `queue_entries`
lewat buffer
Redis yang dikuras ter-batch oleh leader (bukan insert sinkron saat join, yang akan
menaruh Aurora di jalur kritis flash crowd); `admission-ctl` berjalan **2 task, 1 leader + 1 standby, dengan
leader lease** — perpanjangan dan pelepasannya bersyarat lewat Lua, dengan fencing token
`ctl:epoch`; lease ganda **tidak dicegah melainkan dibuat tidak berbahaya** oleh pengaman
di sumber daya (compare-and-adjust pada rekonsiliasi stok, `grace_applied NX` pada pause
TTL grace); transport SSE; Dukcapil fail-closed dengan pause admission penuh;
`SOLD_OUT` reversibel selama 930 detik dengan `CLOSED` sebagai keadaan terminal;
`PAUSE_TTL_GRACE_SECONDS` digeser saat pause **mulai**, bukan saat berakhir; dua metode
pembayaran (`bank_transfer` + `gopay`) — kartu kredit dan OTC keluar karena tidak dapat
menghormati TTL di sisi gateway; CAPTCHA tak terlihat di `/join`, fail-open; BR-07 dipulihkan
sebagai satu keterlibatan aktif per event untuk user login; penghapusan WAF untuk POC; TTL
berurutan dengan anchor `granted_at` dan `held_at` **tanpa grace antrian** (perjalanan maksimum
15,5 menit); stok Redis-otoritatif dengan rekonsiliasi Aurora; hold multi-kategori
all-or-nothing; target kapasitas 300.000; Dukcapil memvalidasi NIK saja; penomoran kursi tidak
diimplementasikan di POC — tiket berisi QR code, kategori, dan NIK pemegang; push notification
TIDAK dibangun di POC; `Session_TTL_Antrian` TIDAK dipakai — slot yang diberikan tapi tidak
diklaim direbut kembali oleh TTL Booking itu sendiri; refund disputed dicatat dan
direkonsiliasi manual, tidak otomatis; bukti verifikasi NIK berlaku permanen tanpa revalidasi
berkala; kontrak API per endpoint — `api-contracts.md` v2.1 berstatus Accepted dan mencakup
seluruh endpoint.

---

## Referensi

- `requirement/requirements.md` v1.0 (Finalized) — FR-01..35, NFR-01..10, BR-01..10
- `Tender Rusuh Booking Ticket.pdf` — baseline
- `requirement/update-requirement-rusuh-booking-tiket.md` — login opsional, BR-01/BR-02, BR-07
- `payment-method/` — dua metode Midtrans Core API, diagram alur dan tautan sumber
- [Midtrans Core API](https://docs.midtrans.com/docs/custom-interface-core-api) ·
  [Transaction Status Cycle](https://docs.midtrans.com/docs/transaction-status-cycle) ·
  [Signature notifikasi](https://docs.midtrans.com/reference/receiving-notifications) ·
  [custom_expiry](https://docs.midtrans.com/docs/coreapi-advanced-features)
- ADR-001 — Teknologi Antrian untuk Virtual Waiting Room *(keputusan berlaku; skema key
  dan TTL di dalamnya digantikan dokumen ini)*
- ADR-002 — Mekanisme Penentuan Posisi Antrian per Customer *(model counter berlaku;
  ilustrasi polling + CDN di dalamnya digantikan SSE pada dokumen ini)*
- ADR-003 — Mekanisme Pengiriman Update Posisi Antrian ke Client *(superseded by ADR-005)*
- ADR-004 — Pemilihan Compute Service *(superseded by ADR-005)*
- ADR-006 — Amandemen Antrian *(mengamandemen ADR-005)*
- [HTML Living Standard: Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html)
  — `retry:`, `Last-Event-ID`, dan perilaku *fail the connection* pada status non-200
