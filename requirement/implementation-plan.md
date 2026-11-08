# Rencana Implementasi — Rusuh War Ticketing Backend

**Versi**: 2.1
**Tanggal**: 3 November 2026 (rev. 2.1)
**Berdasarkan**: ADR-005 (Accepted), API Contracts **v2.5**
**Branch referensi**: `origin/feature/user-notif-nikverify-outboxpublisher`
**Modul Go**: `github.com/PTServerJanganTumbang-BookingTicket/be-rusuh-war-ticketing`
**Menggantikan**: implementation-plan v1.0

> ⚠️ **Diubah oleh ADR-006 §8.10** — status antrian `SOLD_OUT` dibaca **`FULL_BOOKED`** di seluruh dokumen ini
> (`sold_out_at` → `full_booked_at`, `SOLD_OUT_HOLD_SECONDS` → `FULL_BOOKED_HOLD_SECONDS`, `rediskeys.SoldOutAt` →
> `FullBookedAt`, kolom `event_queue_state.sold_out_at` → `full_booked_at`). `CLOSED` dan `sold-out` tidak berubah. `StatusSoldOut`
> pada halaman utama (sumbu stok, FR-03c) belum berubah.
>
> ⚠️ **Diamandemen oleh ADR-006 (5 November 2026, antrian), kontrak v2.7.** Bagian berikut
> **belum** ditulis ulang dan harus dibaca bersama ADR-006 serta
> `be-rusuh-war-ticketing/docs/04-queue/` sampai rewrite per bagian dilakukan saat implementasi:
>
> | Bagian | Yang berubah |
> |---|---|
> | §3.1 `pkg/rediskeys` | hapus `regrant:{ev}`, `slots:open:{ev}`, kanal `sse-grant`; `ctl:leader`, `ctl:epoch`, `qpersist:{ev}` ditunda ke produksi |
> | §5.6 Abandon, §10.5 webhook inline, §13.2 outbox | `DECRBY slots:open` + `EVALSHA advance_now_serving.lua` + `PUBLISH` **dihapus**; tick admission-ctl berikutnya memakai tempat yang bebas |
> | §7.2 Join | bypass Lua → tunggu ≤ 1 detik untuk tick berikutnya (baca saja); tanpa `RPUSH qpersist` |
> | §7.3 `/me` | cek ulang di writer; status pending; masuk normal lewat `nowServing ≥ ticket_no` |
> | §7.4 `/serving` | tanpa query string, `poll_tiers` |
> | §8 sse-server | tanpa peta token → koneksi, tanpa `sse-grant` |
> | §9.6 tick admission-ctl | RECOVER → SNAPSHOT → COUNT (Aurora) → ADVANCE → FINISH → BROADCAST; `rate_per_min` per ADR-006 §6 |
> | §9.2 lease, §9.7 drain qpersist | **ditunda ke produksi** (ADR-006 §9); POC 1 task, deploy min 0 % / max 100 % |
> | §9.8 rekonsiliasi | rekonsiliasi `slots:open` dihapus; rekonsiliasi stok tetap |
> | §14.1 Lua | diganti ADR-006 §3.4; **satu** pemanggil (admission-ctl) |
> | §14.2 `hold_booking.lua` | tidak menyentuh penghitung slot apa pun |
> | §18 kepemilikan | baris `slots:open` dihapus |
> | **Semua bagian yang menyebut pause** (`paused`, gerbang ADVANCE, `PAUSE_TTL_GRACE_SECONDS`, `pause:{ev}:grace_applied`, BR-13a, BR-22, metrik `PauseTransitions` / `PauseTtlGrantedSeconds`) | **dihapus** (ADR-006 §4.3). Breaker tetap fail closed, antrian tetap maju |
> | §8 payload SSE | `{nowServing, state, ratePerMin, serverTime}`; event `status {state}` tiap transisi; indikator basi = event `stale` (ADR-006 §4.2) |
> | §5 booking-service: `GET /bookings/{id}/tickets`, `get_booking_tickets.go`, `booking_tickets.go`, `GetBookingTickets`, §5.5 usecase kedua | **dihapus** (ADR-006 §8.8). `GetBooking` saat `CONFIRMED` menerbitkan `eticket_url` bertoken lewat `pkg/eticket.Mint`; §3.5 pemakai `pkg/grantauth` menjadi delapan |
> | §3.1 `pkg/rediskeys`, semua bagian yang menyebut `expiries:{ev}`, `queue:{ev}:state`, `:issued`, `:full_booked_at`, `:resume` | **ADR-006 §8.12 (8 Nov 2026):** `state`, `issued`, `full_booked_at` (epoch detik) menjadi field hash `queue:{ev}`; `queue:{ev}:resume` dan `expiries:{ev}` dihapus. Sweep tenggat membaca Aurora (indeks parsial pada `booking_expires_at`); `ZADD expiries` setelah `HELD` / `AWAITING_PAYMENT` tidak ada lagi |
> | §9.5 `HSET grant:{token}`, §3.5 `pkg/grantauth` `Grant`, §8 (kredensial) | **ADR-006 §8.13 (8 Nov 2026):** field `event_id`, `booking_id`, `queue_token` + `account_id` (login) **atau** `device_hash` (tamu); tanpa field `token`, tanpa `rebinds` / `MaxDeviceRebinds` (§8.9). TTL = `GRANT_TOKEN_TTL_SECONDS` (1080) dari `granted_at`, bukan 300 |
> | §7.2 Join | respons membawa `queue_token` (untuk mobile); tanpa `estimated_wait_minutes` — estimasi dihitung frontend (ADR-006 §6.2) |

> **Rev 2.1** — v2.0 ditulis terhadap kontrak v2.4, yang naik ke v2.5 pada sesi yang sama.
> Tiga temuan §21.3 sudah diperbaiki di sana dan ditandai selesai; `pkg/eticket` ditambahkan
> karena token e-tiket §6.2 tidak punya pemilik di v2.0 sementara FR-26 bergantung padanya.

---

## 0. Catatan Revisi v1.0 → v2.0

v1.0 memparafrase mekanisme yang sudah diselesaikan ADR-005 dan kehilangan alasannya di
jalan. Dokumen ini **mengutip** pseudocode dan Lua dari ADR-005 apa adanya, bukan
menuliskannya ulang. Yang berubah secara material:

| # | v1.0 | v2.0 |
|---|---|---|
| 1 | Kode mengacu `bookings.grant_token`; tidak ada migration yang membuatnya | §2.1 memutuskan dua kolom terpisah; `0006` membuat `grant_token` + unique index |
| 2 | `POST /bookings` **INSERT** baris baru status HELD | §5 **UPDATE** baris GRANTED yang sudah ada; §5.3 mendefinisikan transisi `GRANTED → SELECTING` yang sebelumnya tidak ada pemiliknya |
| 3 | Sweep hanya menangani GRANTED/SELECTING; entri basi di-`ZREM` | §9.3 lima status lengkap + **rescore** (`ZADD`), sesuai ADR §7. BR-05 kini benar-benar ditegakkan |
| 4 | `payment-api` menulis ulang `held_at` saat charge | §10.4 tidak menyentuh `held_at`; `custom_expiry.order_time = held_at` |
| 5 | `hold_booking.lua` ditulis ulang (rusak: tanpa `STOCK_UNINITIALIZED`, tanpa guard negatif, tanpa kuota, ARGV tidak konsisten) | §14.2 memakai skrip ADR §8 **verbatim** |
| 6 | `advance_now_serving.lua` tidak pernah dilampirkan; ditaruh hanya di admission-ctl | §14.1 melampirkannya di `pkg/redisscript` — dipakai queue-service (FR-05) **dan** admission-ctl |
| 7 | Drain qpersist memakai `LPOP` | §9.7 `LRANGE → INSERT → COMMIT → LTRIM`, batch 1000 (ADR §12) |
| 8 | Webhook tidak punya blok inline publish | §10.5 langkah 3 lengkap dengan lease guard dan urutan `published_at` sebelum `complete_lease` |
| 9 | `outbox-publisher` ditandai selesai | §13.2 menambah efek samping per tipe; `pkg/events` menambah `payment.confirmed` |
| 10 | 2 paket bersama (`ratelimit`, `breaker`) | §3: 8 paket baru + 2 perluasan. `grantauth`, `outbox`, `rediskeys`, `redisscript`, `bookingstate`, `sseevent` |
| 11 | 5 service baru | 7 service baru: ditambah `sale-ops` (§11) dan `metrics-scraper` (§12) |
| 12 | §12 CI/CD mengacu matrix `test-services` yang tidak ada | §17 ditulis terhadap `cicd.yml` yang sebenarnya: paths-filter, 5+2 job deploy, jalur ECS, coverage gate 80% |
| 13 | Kapasitas kategori didefinisikan di dua tabel | §2.2 satu tabel: `ticket_categories` |
| 14 | `tickets` tidak punya `booking_item_id`; guard QR ganda ADR §16 tidak dapat dibuat | `0006` menambahnya + `UNIQUE` |
| 15 | Daftar direktori service baru tanpa scaffold | §15 mendaftar 10 file wajib per service |

**Komponen yang di v1.0 tidak ada sama sekali** dan kini punya pemilik: job pembuka sale,
seeding Redis, `metrics-scraper` (FR-33/34/35, NFR-09), pause TTL grace, fencing token
`ctl:epoch`, charge reconciler, rekonsiliasi `nik:used` ↔ `nik_claims`, `DELETE nik_claims`
saat expiry, nisan `settled:{uuid}`, `CtlHeartbeatAge`, ~~`GET /bookings/{id}/tickets`~~ (dihapus, ADR-006 §8.8),
`GET /admin/reconciliation/disputed`, `rl:sse:{ip}`, `server_time` pada tiap respons REST,
penghitung rebind device, transisi `ABANDONED`, penolakan join saat `SCHEDULED`.

---

## 1. Ringkasan Status

### Sudah selesai (branch `user-notif-nikverify-outboxpublisher`)

| Service | Endpoint / fungsi | Catatan |
|---|---|---|
| `services/user` | `POST /auth/magic-link`, `/magic-link/resend`, `GET /auth/verify`, `GET /auth/verify-email`, `POST /auth/register`, `POST /auth/refresh`, `GET`/`POST` `/accounts/me/saved-niks`, `DELETE /accounts/me/saved-niks/{id}`, `GET /users/me/tickets`, `GET /tickets/{ticket_id}/eticket` | Rotasi refresh token, rate limiter Redis, klien nik-verify, outbox. **Butuh perubahan**: §13.7 — §6.2 hari ini hanya menerima sesi akun, sehingga tautan e-tiket pemegang NIK menjawab 401 (FR-26) |
| `services/nik-verify` | `POST /nik/validate` | Fail-closed, Dukcapil, cache 24j, breaker Redis, bukti tersimpan (registered + saved), token bucket. **Butuh perubahan**: §13.4 |
| `services/outbox-publisher` | Loop Postgres → SQS | Lease Redis, batch, at-least-once. **BELUM selesai**: hanya meneruskan ke SQS, belum menjalankan efek samping per tipe (§13.2) |
| `services/notification` | SQS consumer → SES | Dedup Redis per `outbox_id`, template HTML+text, 5 tipe. **Butuh perubahan**: §13.3, dan §13.7 — `ETicketLink` hari ini tidak bertoken, sehingga tombol di email pemegang NIK menjawab 401 (FR-26) |
| `services/booking` | `POST /bookings`, `GET /bookings/{id}` | **Skeleton** — repo in-memory, lookup kategori hardcoded di `domain/valueobject/category.go`, tanpa DB/Redis/NIK guard |
| `pkg/` | apinizer, apperror, dotenv, envutil, events, httpserver, idgen, logger, pgutil, redisutil, token | Ada. `events` dan `apinizer` butuh perluasan (§13.1, §3.9) |
| `migrations/` | 0001–0004 | 0004 menulis bahwa ia akan menyusut begitu queue/booking punya migration sendiri — itu terjadi di §4 |

### Belum dibangun

| # | Artefak | §  |
|---|---|---|
| 1 | `pkg/rediskeys` — skema kunci Redis terpusat | 3.1 |
| 2 | `pkg/ratelimit` — token bucket generik | 3.2 |
| 3 | `pkg/breaker` — circuit breaker generik | 3.3 |
| 4 | `pkg/redisscript` — loader EVALSHA + 3 skrip Lua | 3.4 |
| 5 | `pkg/grantauth` — middleware grant + device binding | 3.5 |
| 6 | `pkg/outbox` — record + writer dalam transaksi | 3.6 |
| 7 | `pkg/bookingstate` — enum status + transisi legal + kepemilikan `slots:open` | 3.7 |
| 8 | `pkg/sseevent` — nama event + bentuk payload | 3.8 |
| 9 | `pkg/eticket` — token e-tiket pemegang NIK (FR-26) | 3.10 |
| 10 | Migration 0005 — katalog event | 4.1 |
| 11 | Migration 0006 — domain booking | 4.2 |
| 12 | Migration 0007 — indeks BR-07 | 4.3 |
| 13 | `services/booking` — completion | 5 |
| 14 | `services/events` | 6 |
| 15 | `services/queue` | 7 |
| 16 | `services/sse-server` (ECS) | 8 |
| 17 | `services/admission-ctl` (ECS) | 9 |
| 18 | `services/payment-api` | 10 |
| 19 | `services/sale-ops` | 11 |
| 20 | `services/metrics-scraper` | 12 |
| 21 | Perubahan service yang sudah ada | 13 |
| 22 | CI/CD — 7 job deploy, 2 di antaranya jalur ECS baru | 17 |

---

## 2. Keputusan yang Diselesaikan di Dokumen Ini

ADR-005 tidak konsisten pada tiga titik, dan v1.0 mewarisi ketidakkonsistenannya tanpa
memutuskan. Ketiganya diputuskan di sini supaya implementasi punya satu jawaban.

### 2.1 `queue_token` DAN `grant_token` — dua kolom, bukan satu

**Masalahnya.** ADR §4 memberi skema `bookings(id, event_id, queue_token, …)`. ADR §6, §9
dan §12 menulis `INSERT INTO bookings (…, grant_token, …)` dan
`WHERE id = $1 AND grant_token = $grantToken`. Migration `0004` yang sudah ada hanya punya
`queue_token`. v1.0 memakai `grant_token` di tiga tempat tanpa migration yang membuatnya.

**Keputusan: keduanya ada, dan keduanya punya pekerjaan berbeda.** ADR §12 sendiri
mendaftarkan keduanya sebagai **kredensial yang berbeda** pada tabel "Empat kredensial,
empat slot":

| Kolom | Isinya | Menjawab | Siapa yang menulis |
|---|---|---|---|
| `queue_token` | `{uuid}` keterlibatan antrian | keterlibatan mana yang melahirkan booking ini | admission-ctl, saat INSERT GRANTED (dari `by_ticket`) |
| `grant_token` | nilai opaque `gt_…` | kredensial mana yang berhak atas booking ini | admission-ctl, saat INSERT GRANTED (nilai yang sama ditulis ke `grant:{token}`) |

`queue_token` yang dipakai untuk rebuild (`now_serving` diturunkan dari
`MAX(qe.ticket_no) … JOIN queue_entries qe ON qe.queue_token = b.queue_token`, ADR §12) dan
`grant_token` yang dipakai untuk kepemilikan objek pada setiap endpoint §4/§5 kontrak. Satu
kolom tidak dapat melayani keduanya: grant dicabut dengan satu `DEL` saat terminal sementara
`queue_token` harus tetap dapat dicocokkan ke `queue_entries` sesudahnya.

Konsekuensi: `0006` menambah `grant_token TEXT` + `UNIQUE` partial. Query idempotency
`POST /bookings` dan seluruh klausa kepemilikan memakai `grant_token`; rebuild memakai
`queue_token`.

### 2.2 Satu tabel kapasitas: `ticket_categories`

**Masalahnya.** ADR §6 menulis `SELECT capacity FROM event_categories WHERE id = $cat`,
sementara kontrak §2.2 dan §4.1 menuntut kategori dengan `name`, `description`, `price`,
`available`, `max_per_order`. v1.0 membuat **dua** tabel — `ticket_categories` (0005) dan
`event_categories` (0007) — keduanya memegang `capacity`.

**Keputusan: hanya `ticket_categories`.** `capacity` adalah garis dasar yang seluruh
perhitungan drift stok diukur terhadapnya dan ADR §6 menuntutnya **imutabel selama sale**.
Dua tabel berarti dua sumber kebenaran untuk satu angka yang justru tidak boleh bergerak.
Rekonsiliasi stok (§9.8) membaca `ticket_categories.capacity`. `event_categories` **tidak
dibuat**.

### 2.3 `qr_nonce` tetap; `qr_payload` adalah nama yang sama di ADR

**Masalahnya.** ADR §4 menulis `tickets(…, qr_payload, …)`. Migration `0002` yang sudah ada
punya `qr_nonce TEXT NOT NULL`. v1.0 menambah kolom `qr_payload` nullable dan mem-backfill
tanpa menyelesaikan apa pun — `qr_nonce` tetap `NOT NULL`, sehingga setiap INSERT harus
mengisi dua kolom selamanya.

**Keputusan: tidak ada kolom baru.** `qr_nonce` adalah `qr_payload` ADR dengan nama lain,
dan nama yang sudah ada di basis data menang karena ia sudah punya penulis dan pembaca
(user-service `GET /tickets/{id}/eticket`). Pemetaan nama diselesaikan di layer mapper.
`0006` **tidak** menyentuh kolom ini.

Yang `0006` **memang** tambahkan ke `tickets` adalah `booking_item_id` + `UNIQUE`: ADR §16
mendaftarkannya sebagai mitigasi "Outbox retry menerbitkan QR ganda", dan tanpa kolom itu
mitigasi tersebut tidak dapat dibuat.

### 2.4 `GRANTED → SELECTING` dimiliki `GET /events/{event_id}/tickets`

**Masalahnya.** ADR §8 menuntut `POST /bookings` berjalan hanya atas baris berstatus
`SELECTING`, dan ADR §6 mendaftarkan transisi `GRANTED → SELECTING (claim)` sebagai milik
`booking-service`. Tidak ada endpoint yang dinyatakan menjalankannya, dan v1.0 tidak
menambahkannya — sehingga status `SELECTING` tidak pernah tercapai dan `POST /bookings`
tidak pernah dapat berhasil.

**Keputusan: pemuatan Detail Ticket Page adalah claim-nya.**
`GET /events/{event_id}/tickets` (kontrak §4.1) adalah satu-satunya panggilan yang pasti
terjadi tepat ketika customer membuka halaman itu, dan `slot_claimed_at` ADR §7 memang
didefinisikan sebagai "dicatat untuk metrik konversi". Handler-nya menjalankan transisi
secara idempoten (§5.3). Tenggat **tidak** diubah: `booking_expires_at` sudah ditetapkan
saat GRANT.

### 2.5 Nama metrik yang dipakai

v1.0 memakai `StockAdjustRejected` dan `HoldCompensated` untuk drift `slots:open`. Keduanya
milik jalur lain. Yang benar, sesuai ADR §4:

| Jalur | Metrik |
|---|---|
| Drift `slots:open` (§9.6) | `SlotsOpenDrift` |
| Drift stok, koreksi ditolak CAS (§9.8) | `StockDrift`, `StockAdjustRejected` |
| Kunci stok hilang lalu ditanam ulang | `StockKeyRestored` |
| Kompensasi Redis setelah rollback Aurora (§5.5) | `HoldCompensated` |
| `nik:used` dipulihkan dari `nik_claims` | `NikClaimRestored` |

---

## 3. Paket Bersama di `/pkg`

Aturannya: logika yang dipakai lebih dari satu service ada di `/pkg`, bukan di-copy. v1.0
mengidentifikasi dua paket. Delapan yang berikut adalah daftar lengkapnya, diturunkan dari
menghitung pemakai tiap mekanisme di kontrak dan ADR.

Setiap paket ditambahkan ke modul `pkg` yang sudah ada (`pkg/go.mod`) — bukan modul baru.

### 3.1 `pkg/rediskeys` — skema kunci terpusat

**Mengapa pertama.** `services/nik-verify/internal/nik/infrastructure/redis/init.go` memuat
komentar ini:

> `// Keys used by this service. Shared with admission-ctl, which reads`
> `// dukcapil:breaker:state every tick — do not rename without updating it.`

Kunci itu `dukcapil:breaker:state`; ADR §4 menetapkan pola `breaker:{service}:state`.
Selama konstanta kunci hidup di `internal/` masing-masing service, pola seperti ini akan
menyimpang lagi, dan penyimpangannya **senyap** — breaker yang dibaca admission-ctl tinggal
selalu CLOSED dan pause tidak pernah terjadi.

```
pkg/rediskeys/
  keys.go
  keys_test.go
```

```go
package rediskeys

import "fmt"

// Hash tag {ev} WAJIB pada setiap kunci yang disentuh satu skrip Lua: Redis
// cluster mode menempatkan kunci dengan tag yang sama di slot yang sama, dan
// itu prasyarat agar hold_booking.lua boleh menyentuh stok + NIK sekaligus
// (ADR-005 §8).

// Antrian
func QueueIssued(ev string) string   { return fmt.Sprintf("queue:{%s}:issued", ev) }
func QueueState(ev string) string    { return fmt.Sprintf("queue:{%s}:state", ev) }
func QueueCapacity(ev string) string { return fmt.Sprintf("queue:{%s}:capacity", ev) }
func QueueHash(ev string) string     { return fmt.Sprintf("queue:{%s}", ev) } // now_serving, rate_per_min
func ByTicket(ev string) string      { return fmt.Sprintf("queue:{%s}:by_ticket", ev) }
func SlotsOpen(ev string) string     { return fmt.Sprintf("slots:open:{%s}", ev) }
func Regrant(ev string) string       { return fmt.Sprintf("regrant:{%s}", ev) }
func Expiries(ev string) string      { return fmt.Sprintf("expiries:{%s}", ev) }
func QPersist(ev string) string      { return fmt.Sprintf("qpersist:{%s}", ev) }
func SoldOutAt(ev string) string     { return fmt.Sprintf("queue:{%s}:sold_out_at", ev) }
func ResumeFlag(ev string) string    { return fmt.Sprintf("queue:{%s}:resume", ev) }

// Kredensial
func Token(uuid string) string        { return "token:" + uuid }
func Settled(uuid string) string      { return "settled:" + uuid }
func Grant(token string) string       { return "grant:" + token }
func Acct(ev, accountID string) string { return fmt.Sprintf("acct:{%s}:%s", ev, accountID) }

// Booking
func Stock(ev, cat string) string   { return fmt.Sprintf("stock:{%s}:%s", ev, cat) }
func NIKUsed(ev, hmac string) string { return fmt.Sprintf("nik:used:{%s}:%s", ev, hmac) }
func NIKValid(hmac string) string   { return "nik:valid:" + hmac }

// Kontrol
const (
    CtlLeader = "ctl:leader"
    CtlEpoch  = "ctl:epoch"
)
func PauseGraceApplied(ev string) string { return fmt.Sprintf("pause:{%s}:grace_applied", ev) }
func Pause(ev string) string             { return fmt.Sprintf("pause:{%s}", ev) }

// Dependensi eksternal — pola ADR-005 §4, BUKAN pola lama nik-verify.
func BreakerState(service string) string    { return "breaker:" + service + ":state" }
func BreakerFailures(service string) string { return "breaker:" + service + ":failures" }
func DependRateLimit(service string) string { return "rl:" + service }

// Ingress
func JoinRateLimit(ip string) string { return "rl:join:" + ip }
func SSERateLimit(ip string) string  { return "rl:sse:" + ip }
func PayRateLimit(bookingID string) string { return "rl:pay:" + bookingID }

// Outbox
func Processed(outboxID string) string { return "processed:" + outboxID }

// Nama service untuk breaker — satu-satunya tempat string ini ditulis.
const (
    ServiceDukcapil       = "dukcapil"
    ServicePaymentGateway = "payment-gateway"
)
```

**Migrasi nik-verify** (§13.4): `dukcapil:breaker:state` → `breaker:dukcapil:state`,
`dukcapil:breaker:failures` → `breaker:dukcapil:failures`, `dukcapil:ratelimit` →
`rl:dukcapil`, `nik:valid:` tetap. Pola lama tidak dibaca siapa pun selain nik-verify
sendiri hari ini, sehingga migrasi ini aman dilakukan **sebelum** admission-ctl ada —
dan harus dilakukan sebelum itu, karena setelahnya ia menjadi perubahan lintas service.

### 3.2 `pkg/ratelimit` — token bucket Redis generik

**Pemakai**: queue-service (`rl:join:{ip}`), sse-server (`rl:sse:{ip}`), payment-api
(`rl:pay:{booking_id}`), nik-verify (`rl:dukcapil`, sesudah migrasi).

**Perbedaan dari yang sudah ada**: `services/nik-verify/.../redis/limiter.go` mengikat
`KEYS[1]` ke satu konstanta (`KeyRateLimit`) dan menyimpan rps/capacity pada struct,
sehingga satu instance hanya dapat menjaga satu bucket. Versi `/pkg` menerima `key`,
`capacity` dan `rps` sebagai parameter panggilan.

```
pkg/ratelimit/
  ratelimit.go
  token_bucket.lua
  ratelimit_test.go
```

```go
package ratelimit

import (
    "context"
    "time"

    goredis "github.com/redis/go-redis/v9"
)

// Limiter adalah token bucket Redis. Tiap key memelihara bucket sendiri,
// sehingga satu Limiter melayani batas per-IP dan per-booking sekaligus.
type Limiter interface {
    // Allow mencoba mengambil satu token dari bucket milik key.
    // retryAfter bermakna hanya ketika allowed == false.
    Allow(ctx context.Context, key string, capacity, rps float64) (allowed bool, retryAfter time.Duration, err error)
}

func New(client *goredis.Client) Limiter
```

**Skrip Lua.** Diambil dari `nik-verify/.../limiter.go` dengan dua perubahan: `KEYS[1]`
menjadi parameter, dan nilai kembali menjadi dua elemen `{allowed, retry_after_ms}` —
skrip lama hanya mengembalikan `0`/`1`, sehingga `retryAfter` pada interface di atas
**tidak dapat** diturunkan tanpa perubahan ini. `retry_after_ms` dihitung sebagai
`math.ceil((1 - tokens) / refill_per_ms)` pada cabang ditolak; itu satu baris, dan ia
yang membuat respons `429` kontrak dapat memuat `retry_after`.

| Pemanggil | Key | capacity | rps |
|---|---|---|---|
| queue-service (join) | `rl:join:{ip}` | `JOIN_RATE_LIMIT` (20) | `20/(5*60)` |
| sse-server (koneksi) | `rl:sse:{ip}` | `SSE_CONN_RATE_LIMIT` (5) | 5.0 |
| payment-api (initiate) | `rl:pay:{booking_id}` | 3 | `3/600` |
| nik-verify (Dukcapil) | `rl:dukcapil` | `DUKCAPIL_BURST_CAPACITY` | `DUKCAPIL_RATE_LIMIT_RPS` |

> **`fixed_window.lua` tidak dibuat.** v1.0 mendaftarkannya sebagai "alternatif untuk join
> endpoint". ADR §8 menyatakan ingress memakai skrip yang sama dengan dependensi
> (`rate_limit.lua`), jadi tidak ada alternatif untuk dipilih.

### 3.3 `pkg/breaker` — circuit breaker Redis generik

**Pemakai**: nik-verify (Dukcapil), payment-api (payment gateway), admission-ctl (**membaca
keduanya** untuk memutuskan pause — ADR §4 langkah 0).

```
pkg/breaker/
  breaker.go
  breaker_test.go
```

```go
package breaker

import (
    "context"
    "time"

    goredis "github.com/redis/go-redis/v9"
)

type State string

const (
    StateClosed   State = "CLOSED"
    StateOpen     State = "OPEN"
    StateHalfOpen State = "HALF_OPEN"
)

// Breaker adalah circuit breaker Redis per dependensi. Kuncinya mengikuti
// pkg/rediskeys.BreakerState / BreakerFailures.
//
// OPEN berarti dependensinya SEDANG TIDAK DAPAT DIPAKAI (ADR-005 §8). Jangan
// dibaca sebagai "tersedia".
type Breaker interface {
    State(ctx context.Context) (State, error)
    RecordSuccess(ctx context.Context) error
    RecordFailure(ctx context.Context) error
}

// New mengembalikan breaker untuk dependensi bernama service — pakai
// rediskeys.ServiceDukcapil / ServicePaymentGateway, bukan string literal.
func New(client *goredis.Client, service string, failThreshold int, openFor time.Duration) Breaker
```

**Catatan untuk admission-ctl**: ia hanya memanggil `State()`. Ia **tidak** boleh memanggil
`RecordFailure` — breaker milik service yang benar-benar memanggil dependensinya.

### 3.4 `pkg/redisscript` — loader EVALSHA dan skrip Lua

**Mengapa di `/pkg`**: `advance_now_serving.lua` dipakai **dua** service —
admission-ctl (jalur utama) dan queue-service (jalur bypass FR-05, ADR §5: "Jalur bypass
memakai Lua yang sama … sehingga tidak ada jalur admission kedua yang bisa menyimpang dari
yang utama"). Dua kopi skrip itu persis jalur kedua yang ADR berusaha mencegah.

```
pkg/redisscript/
  script.go                     # embed + SCRIPT LOAD sekali, EVALSHA, fallback EVAL
  scripts/advance_now_serving.lua
  scripts/hold_booking.lua
  scripts/adjust_if_unchanged.lua
  scripts/leader_renew.lua
  scripts/leader_release.lua
  script_test.go
```

```go
package redisscript

import (
    "context"
    "embed"

    goredis "github.com/redis/go-redis/v9"
)

//go:embed scripts/*.lua
var files embed.FS

// Registry memuat setiap skrip sekali saat startup dan menyimpan SHA1-nya.
// Run memakai EVALSHA; bila Redis menjawab NOSCRIPT (failover, SCRIPT FLUSH)
// ia memuat ulang lalu mencoba sekali lagi — tanpa ini satu failover
// mematikan admission sampai proses di-restart.
type Registry struct { /* ... */ }

func Load(ctx context.Context, client *goredis.Client) (*Registry, error)

func (r *Registry) Run(ctx context.Context, name string, keys []string, args ...any) *goredis.Cmd
```

Isi ketiga skrip inti ada di §14. `leader_renew.lua` dan `leader_release.lua` ada di §9.2.

### 3.5 `pkg/grantauth` — middleware grant token + pengikatan device

**Pemakai**, dari kontrak: `GET /events/{ev}/tickets`, `POST /nik/validate`,
`POST /bookings`, `GET /bookings/{id}`,
`GET /payments/{id}/methods`, `POST /payments/{id}`, `POST /payments/{id}/retry`,
`GET /payments/{id}/status` — delapan endpoint (sembilan sebelum ADR-006 §8.8) di tiga service. v1.0 tidak
mendaftarkannya sebagai kode bersama sama sekali, dan memberi tiap service "grant.go"
sendiri.

Yang membuatnya wajib bersama bukan jumlah pemakainya, melainkan bahwa aturannya adalah
**aturan keamanan dengan detail yang mudah hilang**: 404-bukan-403, satu rebind yang
tercatat, dan `DEVICE_ID_REQUIRED` ditolak **di join** bukan nanti.

```
pkg/grantauth/
  grantauth.go      # middleware + Grant dalam context
  device.go         # HMAC(DEVICE_PEPPER, device_id) + penghitung rebind
  grantauth_test.go
```

```go
package grantauth

import (
    "context"
    "net/http"

    goredis "github.com/redis/go-redis/v9"
)

// Grant adalah isi grant:{token} di Redis, ditulis admission-ctl saat
// menerbitkan grant (§9.5). Semua field berasal dari token:{uuid}
// keterlibatan — klien tidak pernah menyuplai satu pun dari mereka.
type Grant struct {
    Token      string
    EventID    string
    QueueToken string // uuid keterlibatan
    AccountID  string // kosong untuk tamu
    DeviceHash string // kosong untuk customer login
}

// Middleware memverifikasi X-Grant-Token dan, untuk pemanggil anonim,
// X-Device-Id. Pada sukses ia menaruh Grant di context.
//
// Urutan pemeriksaan, dan urutan ini bagian dari kontraknya:
//   1. X-Grant-Token hilang            → 401 GRANT_TOKEN_MISSING
//   2. HGETALL grant:{token} kosong    → 401 GRANT_TOKEN_INVALID
//   3. Grant.DeviceHash != "" (tamu):
//        X-Device-Id hilang            → 400 DEVICE_ID_REQUIRED
//        HMAC tidak cocok:
//          HINCRBY token:{uuid} rebinds 1
//          hasil <= MAX_DEVICE_REBINDS → lanjut, tulis device_hash baru,
//                                        emit DeviceRebindRate
//          lebih                       → 401 DEVICE_MISMATCH
func Middleware(client *goredis.Client, cfg Config) func(http.Handler) http.Handler

func FromContext(ctx context.Context) (Grant, bool)

type Config struct {
    DevicePepper    []byte
    MaxDeviceRebinds int
}
```

> ⚠️ **Diubah oleh ADR-006 §8.9** — tanpa rebind: `MaxDeviceRebinds`, `HINCRBY rebinds` dan `DeviceRebindRate`
> di atas dihapus; perangkat lain → langsung `401 DEVICE_MISMATCH`. Paragraf berikut historis.
>
> **Satu rebind, dan itu sengaja longgar** (ADR §12). Menghapus data situs, incognito, atau
> ganti browser menghapus `device_id` yang sah; pengikatan keras berarti customer itu
> kehilangan slotnya. `DEVICE_MISMATCH` karena itu menandai percobaan **ketiga**, bukan
> "ganti browser". v1.0 menyebut validasi device tanpa penghitung rebind sama sekali,
> yang menjadikannya pengikatan keras.

**Klausa kepemilikan tidak ada di sini.** Middleware membuktikan pemegang punya slot;
bahwa slot itu milik `booking_id` di path diikat di query tiap repository
(`WHERE id = $1 AND grant_token = $2`, §18). Keduanya dipisah karena yang kedua butuh
basis data dan yang pertama tidak.

### 3.6 `pkg/outbox` — record dan writer dalam transaksi

**Masalahnya.** `pkg/events` memegang konstanta tipe, dan v1.0 menganggap itu cukup
("Outbox event types … ✅ sudah ada"). Yang tidak ada adalah **penulisnya**: satu-satunya
implementasi `INSERT INTO outbox` hidup di
`services/user/internal/user/infrastructure/postgres/outbox.go` — di dalam `internal/` dan
di dalam modul lain, jadi tidak dapat diimpor. v1.0 bahkan menulis signature
`Enqueue(ctx, tx pgx.Tx, events []outboxrecord.Record)` yang mengacu paket tak ada.

**Pemakai**: booking-service, payment-api, admission-ctl, user-service (migrasi).

```
pkg/outbox/
  record.go      # Record + Envelope()
  writer.go      # Enqueue(ctx, tx, ...Record) — satu multi-row INSERT
  lease.go       # claim_lease / complete_lease (ADR §9 langkah 3)
  outbox_test.go
```

```go
package outbox

import (
    "context"

    "github.com/jackc/pgx/v5"

    "github.com/PTServerJanganTumbang-BookingTicket/be-rusuh-war-ticketing/pkg/events"
)

type Record struct {
    ID          string
    AggregateID string
    Type        events.Type
    Payload     any
}

// Enqueue menulis baris outbox DI DALAM transaksi pemanggil. Itu seluruh
// gunanya: perubahan state dan event yang mendeskripsikannya commit bersama,
// sehingga tidak ada booking CONFIRMED tanpa e-tiket terjadwal dan tidak ada
// email untuk booking yang di-rollback (NFR-10).
func Enqueue(ctx context.Context, tx pgx.Tx, records ...Record) error
```

```go
// lease.go — guard yang dipakai inline publisher DAN outbox-publisher.
//
// Flag permanen tanpa lease BOCOR: handler yang mengambil flag lalu crash
// sebelum DECR membuat konsumer berikutnya melewatinya dan slot hilang
// selamanya (ADR-005 §9 langkah 3).
type Lease interface {
    // Claim: SET processed:{id} <workerID> NX EX 30
    Claim(ctx context.Context, id string) (bool, error)
    // Complete: SET processed:{id} "done" EX 86400
    Complete(ctx context.Context, id string) error
    Release(ctx context.Context, id string) error
}
```

> **Urutan wajib**: `UPDATE outbox SET published_at` **sebelum** `Complete`. Dibalik,
> crash di antaranya meninggalkan lease `"done"` 86400 detik dengan `published_at` tetap
> NULL → publisher mengambil baris itu, `Claim` selalu gagal, dan `OutboxLag` naik palsu
> selamanya (ADR §9). Aturan ini ditulis di doc comment `Complete` supaya tidak dapat
> dilewati tanpa membacanya.

### 3.7 `pkg/bookingstate` — status, transisi, dan kepemilikan `slots:open`

**Pemakai**: booking-service, payment-api, admission-ctl, queue-service (pemetaan §3.3
kontrak), user-service (pemetaan status API).

**Mengapa kode, bukan dokumentasi.** ADR §6 memuat tabel "Kepemilikan `slots:open`" dengan
peringatan: "Double-DECR → `slots:open` negatif → `deficit = target − (−n)` →
over-admission." Tabel itu adalah aturan yang dipegang **lima** service; disalin sebagai
komentar ia akan menyimpang.

```
pkg/bookingstate/
  status.go       # enum + IsTerminal + CanTransition
  slots.go        # DecrementsSlot(from, to) bool
  apistatus.go    # pemetaan ke data.status kontrak §5.4 dan state §3.3
  bookingstate_test.go
```

```go
package bookingstate

type Status string

const (
    Granted             Status = "GRANTED"
    Selecting           Status = "SELECTING"
    Held                Status = "HELD"
    AwaitingPayment     Status = "AWAITING_PAYMENT"
    PendingCancellation Status = "PENDING_CANCELLATION"
    Confirmed           Status = "CONFIRMED"
    Expired             Status = "EXPIRED"
    Lapsed              Status = "LAPSED"
    Abandoned           Status = "ABANDONED"
)

// DecrementsSlot mengembalikan true hanya untuk transisi TERMINAL, sesuai
// tabel kepemilikan ADR-005 §6. Lua hanya INCR; DECR dilakukan pemanggil,
// SEBELUM EVALSHA, dan hanya bila UPDATE Aurora mengenai tepat 1 baris.
func DecrementsSlot(from, to Status) bool

// CanTransition menegakkan tabel transisi legal. Dipakai sebagai guard
// sebelum setiap UPDATE berstatus, dan di test sebagai tabel kebenaran.
func CanTransition(from, to Status) bool

func (s Status) IsTerminal() bool // CONFIRMED, EXPIRED, LAPSED, ABANDONED
```

### 3.8 `pkg/sseevent` — nama event dan bentuk payload

**Pemakai**: sse-server (menyiarkan), admission-ctl (mem-publish), payment-api (mem-publish
pada jalur inline webhook).

Tiga pemakai, bukan dua — v1.0 menyebutnya opsional ("dalam satu tim … juga valid") karena
hanya menghitung dua. Dengan payment-api mem-publish `sse-grant` di blok inline (§10.5),
bentuk payload itu dipegang tiga modul terpisah.

```
pkg/sseevent/
  sseevent.go
  sseevent_test.go
```

```go
package sseevent

// Channel Redis Pub/Sub.
const (
    ChannelBroadcast = "sse-broadcast"
    ChannelGrant     = "sse-grant"
    ChannelSoldOut   = "sse-sold-out"
)

// Nama event SSE di sisi klien. Event default (posisi) TIDAK bernama.
const (
    EventGrant     = "grant"
    EventStatus    = "status"
    EventSoldOut   = "sold-out"
    EventReconnect = "reconnect"
)

// Broadcast adalah payload event default DAN snapshot awal. Field-nya persis
// kontrak §3.2 — camelCase, berbeda dari respons REST yang snake_case.
type Broadcast struct {
    EventID    string `json:"eventId,omitempty"` // hanya di Pub/Sub, tidak dikirim ke klien
    NowServing int64  `json:"nowServing"`
    State      string `json:"state"`
    RatePerMin int    `json:"ratePerMin"`
    Paused     bool   `json:"paused"`
    ServerTime string `json:"serverTime"`
}

type GrantMsg struct {
    EventID    string `json:"eventId"`
    QueueToken string `json:"token"`
}

type StatusMsg struct {
    Stale  bool   `json:"stale"`
    Reason string `json:"reason,omitempty"` // redis_disconnected | no_admission_updates
}
```

### 3.9 Perluasan `pkg/apinizer` — `server_time` pada setiap respons

ADR §6: "Tidak ada `GET /time`: setiap respons REST juga membawa `server_time`, sehingga
klien yang belum membuka stream sudah menerima offset jamnya dari panggilan yang memang
harus ia lakukan." BR-10 bergantung padanya, dan kontrak memuatnya pada §3.3, §3.4, §4.1,
§4.3, §4.4, §5.2, §5.3, §5.4.

v1.0 tidak menyebutnya. Diimplementasikan sekali di envelope, bukan delapan kali di DTO:

```go
// pkg/apinizer — tambahan pada struct meta
type Meta struct {
    Code       int         `json:"code"`
    Message    string      `json:"message"`
    ServerTime string      `json:"server_time"`           // BARU — RFC3339 UTC, selalu
    Pagination *Pagination `json:"pagination,omitempty"`
}
```

Satu perubahan di `pkg` mengisinya untuk setiap endpoint di setiap service, termasuk yang
sudah ada. Pemeriksaan kontrak yang menuntut `server_time` di dalam `data` (§4.1
`session.server_time`, §4.3 `payment_ttl.server_time`) tetap diisi DTO masing-masing —
keduanya menyertai tenggat tertentu, bukan jam server secara umum.

### 3.10 `pkg/eticket` — token e-tiket untuk pemegang NIK

**Mengapa ia bersama, bukan milik satu service.** Dua service menyentuh kunci yang sama dan
dari dua sisi: `notification` **menerbitkan** token ke dalam tautan email (dan, sejak ADR-006 §8.8,
`booking-service` ke dalam `eticket_url` pada `GET /bookings/{id}`), `user-service`
**memverifikasinya** pada `GET /tickets/{id}/eticket?t=…` (kontrak §6.2). Menduplikasi HMAC
di dua tempat berarti dua definisi tentang apa yang ditandatangani, dan yang pertama
menyimpang akan membuat setiap tautan yang sudah terkirim gagal — tanpa cara mengetahuinya
selain laporan customer yang tidak bisa membuka tiketnya.

**Mengapa ia harus ada sama sekali.** FR-26 (MUST) mengirim tiket ke **masing-masing
pemegang NIK**. Pemegang kursi kedua dan ketiga umumnya bukan pemilik akun — sering bukan
pembeli. Rute §6.2 yang hanya menerima sesi akun membuat tombol "Buka e-tiket" di email
mereka menjawab **401**, yaitu jalur pengiriman utama requirement itu tidak berfungsi untuk
orang-orang yang requirement itu ditulis untuk mereka. Hal yang sama berlaku bagi pembeli
tamu: setelah §3.3 menghapus kredensial antriannya pada `CONFIRMED`, yang tersisa hanya
email.

```
pkg/eticket/
  eticket.go       Mint(ticketID string, expiresAt time.Time) (string, error)
                   Verify(token, ticketID string, now time.Time) error
  eticket_test.go
```

| Keputusan | Nilai |
|---|---|
| Algoritma | HMAC-SHA256 atas `ticket_id` + `exp` (epoch detik), **stateless** — verifikasi tidak membaca store |
| Bentuk | `base64url(exp) + "." + base64url(mac)`; `ticket_id` tidak ikut di token, ia datang dari path |
| Kunci | `ETICKET_SIGNING_KEY` — **wajib, tanpa default**. Bukan `JWT_SECRET`, bukan `DEVICE_PEPPER`, bukan pepper NIK |
| Lingkup | **satu tiket.** Tidak membuka tiket lain dalam booking yang sama, tidak membuka §6.1, tidak membawa `account_id` |
| Masa berlaku | `event_date + 1 hari` — tiket memang perlu dibuka di gerbang |
| Pencabutan | hanya dengan rotasi kunci, dan rotasi membatalkan **setiap** tautan yang sudah terkirim |

> **`ticket_id` diikat tetapi tidak dibawa.** `Verify` menghitung MAC atas `ticket_id` dari
> path URL, sehingga token dari tiket A yang ditempel pada URL tiket B gagal — tanpa perlu
> menyimpan apa pun. Memasukkan `ticket_id` ke dalam token hanya memperpanjangnya.
>
> **Perbandingan MAC wajib `hmac.Equal`**, bukan `==`: perbandingan byte-demi-byte yang
> keluar lebih awal membocorkan prefiks yang benar, dan token ini dapat dicoba berulang kali
> dari email.
>
> **Kuncinya tidak dirotasi di dalam masa penjualan event aktif.** Rotasi adalah
> satu-satunya cara mencabut, dan ia mencabut semuanya sekaligus. `ETICKET_SIGNING_KEY`
> karena itu masuk daftar secret yang **tidak** ikut rotasi berkala (bandingkan `JWT_SECRET`,
> yang justru harus dirotasi).
>
> **Residual yang dinyatakan terang** (kontrak §6.2): meneruskan email berarti meneruskan
> tiket. Itu benar untuk e-tiket mana pun; BR-02 tetap mengikat kursi pada NIK, bukan pada
> penerima email, dan pemeriksaan identitas di gerbang venue di luar cakupan POC.

**Test**: mint→verify round trip; token tiket A ditolak pada tiket B; token kedaluwarsa
ditolak; MAC yang diubah satu bit ditolak; `exp` yang diubah ditolak (ia masuk MAC); token
yang cacat bentuknya tidak panik.

### 3.11 Ringkasan: yang bersama dan yang tidak

| Logika | Pemakai | Lokasi |
|---|---|---|
| Skema kunci Redis | 8 service | `pkg/rediskeys` (3.1) |
| Token bucket | queue, sse-server, payment-api, nik-verify | `pkg/ratelimit` (3.2) |
| Circuit breaker | nik-verify, payment-api, admission-ctl | `pkg/breaker` (3.3) |
| `advance_now_serving.lua` | queue (FR-05), admission-ctl | `pkg/redisscript` (3.4) |
| `adjust_if_unchanged.lua` | admission-ctl | `pkg/redisscript` — ditaruh bersama demi loader yang sama |
| `hold_booking.lua` | booking | `pkg/redisscript` — idem |
| Middleware grant + device | booking ×3, payment ×4, nik-verify ×1, events ×1 | `pkg/grantauth` (3.5) |
| Record + writer outbox, lease | booking, payment-api, admission-ctl, user | `pkg/outbox` (3.6) |
| Status booking + transisi + DECR | booking, payment-api, admission-ctl, queue, user | `pkg/bookingstate` (3.7) |
| Event dan payload SSE | sse-server, admission-ctl, payment-api | `pkg/sseevent` (3.8) |
| Token e-tiket (mint + verify) | notification (mint), user (verify) | `pkg/eticket` (3.10) |
| `server_time` di envelope | semua | `pkg/apinizer` (3.9) |
| Tipe event outbox | user, booking, payment-api, admission-ctl, notification | `pkg/events` — **tambah** `payment.confirmed` (§13.1) |
| JWT issuer/validator | user, queue (`/me`) | `pkg/token` — ada |
| ULID | semua | `pkg/idgen` — ada |
| Error types | semua | `pkg/apperror` — **tambah** kode baru (§13.5) |
| HTTP server + graceful shutdown | semua (mode dev lokal) | `pkg/httpserver` — ada |
| Pool Postgres | semua yang pakai DB | `pkg/pgutil` — ada |
| Klien Redis | semua yang pakai Redis | `pkg/redisutil` — ada |

**Yang sengaja TIDAK di `/pkg`**: klien Dukcapil (satu pemakai), klien Midtrans (satu
pemakai), klien CAPTCHA (satu pemakai), connection manager SSE (satu pemakai). Memindahkan
yang berpemakai tunggal ke `pkg` hanya menambah lapisan.

---

## 4. Database Migrations

Migration `0004` menulis tentang dirinya sendiri:

> `-- OWNERSHIP: queue-service owns queue_entries, booking-service owns bookings.`
> `-- Neither exists in this repository yet, so the shape here is transcribed from`
> `-- ADR-005 §4 and is the MINIMUM user-service reads. When those services land,`
> `-- their own migrations become authoritative`

Itu terjadi di sini. `0005`–`0007` menambah, tidak memindahkan: `0004` tetap pemilik bentuk
dasar `bookings` dan `queue_entries` karena menurunkannya akan memecah user-service tanpa
alasan.

### 4.1 Migration 0005 — Katalog Event

File: `migrations/0005_event_catalog.up.sql` / `.down.sql`

Tabel `events` dari `0002` punya tujuh kolom (`id`, `name`, `event_date`, `event_time`,
`venue`, `image_url`, `created_at`) — cukup untuk merender baris tiket, tidak cukup untuk
`GET /events` dan `GET /events/{id}`.

```sql
-- 0005_event_catalog.up.sql
--
-- Katalog event untuk GET /events dan GET /events/{event_id} (kontrak §2.1, §2.2).
-- events dibuat di 0002 sebagai read-model tiket; di sini ia menjadi katalog.
--
-- Satu keputusan yang DDL ini tegakkan: kapasitas kategori hidup di SATU tabel.
-- Seluruh perhitungan drift stok diukur terhadap ticket_categories.capacity dan
-- ADR-005 §6 menuntutnya imutabel selama sale — dua tabel kapasitas berarti dua
-- garis dasar untuk satu angka yang justru tidak boleh bergerak.

BEGIN;

ALTER TABLE events
    ADD COLUMN IF NOT EXISTS description      TEXT        NOT NULL DEFAULT '',
    ADD COLUMN IF NOT EXISTS category         TEXT        NOT NULL DEFAULT '',
    ADD COLUMN IF NOT EXISTS location_city    TEXT        NOT NULL DEFAULT '',
    ADD COLUMN IF NOT EXISTS queue_capacity   BIGINT      NOT NULL DEFAULT 300000,
    -- NULL = jadwal belum ditetapkan. sale-ops (§11) menolak membuka event
    -- dengan sale_starts_at NULL, jadi kolom ini adalah gerbangnya.
    ADD COLUMN IF NOT EXISTS sale_starts_at   TIMESTAMPTZ,
    ADD COLUMN IF NOT EXISTS captcha_provider TEXT        NOT NULL DEFAULT 'turnstile',
    ADD COLUMN IF NOT EXISTS captcha_site_key TEXT        NOT NULL DEFAULT '',
    ADD COLUMN IF NOT EXISTS updated_at       TIMESTAMPTZ NOT NULL DEFAULT now();

-- Kategori tiket per event.
--
-- capacity : garis dasar imutabel. Rekonsiliasi stok (§9.8) mengukur drift
--            terhadapnya; mengubahnya di tengah sale menggeser garis dasar dan
--            membuat koreksi berikutnya salah arah (ADR-005 §6). Perubahan
--            hanya di luar jendela sale, dan last_drift harus di-reset.
-- available : SNAPSHOT untuk GET /events saat Redis dingin dan untuk rebuild.
--            Sumber kebenaran adalah stock:{ev}:{cat} di Redis.
CREATE TABLE IF NOT EXISTS ticket_categories (
    id            TEXT        PRIMARY KEY,
    event_id      TEXT        NOT NULL REFERENCES events (id) ON DELETE CASCADE,
    name          TEXT        NOT NULL,
    description   TEXT        NOT NULL DEFAULT '',
    price         BIGINT      NOT NULL,
    capacity      BIGINT      NOT NULL,
    available     BIGINT      NOT NULL,
    max_per_order INT         NOT NULL DEFAULT 3,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT ticket_categories_capacity_positive CHECK (capacity > 0),
    CONSTRAINT ticket_categories_available_range   CHECK (available BETWEEN 0 AND capacity)
);

CREATE INDEX IF NOT EXISTS ticket_categories_event_idx
    ON ticket_categories (event_id);

-- State antrian per event. DITULIS HANYA SAAT TRANSISI (ADR-005 §4).
--
-- now_serving TIDAK disimpan di sini. Ia diturunkan saat rebuild dari nomor
-- antrian tertinggi yang punya baris bookings, dan itu LEBIH BENAR daripada
-- nilai tersimpan — yang bisa tertinggal satu tick dan menyebabkan token yang
-- sudah di-grant di-grant ulang (ADR-005 §12).
CREATE TABLE IF NOT EXISTS event_queue_state (
    event_id    TEXT        PRIMARY KEY REFERENCES events (id) ON DELETE CASCADE,
    state       TEXT        NOT NULL DEFAULT 'SCHEDULED',
    sold_out_at TIMESTAMPTZ,
    opened_at   TIMESTAMPTZ,
    closed_at   TIMESTAMPTZ,
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),

    CONSTRAINT event_queue_state_state CHECK (
        state IN ('SCHEDULED', 'OPEN', 'SOLD_OUT', 'CLOSED')
    ),
    -- SOLD_OUT tanpa sold_out_at membuat transisi ke CLOSED tidak dapat
    -- dihitung (sold_out_at + SOLD_OUT_HOLD_SECONDS, ADR-005 §6).
    CONSTRAINT event_queue_state_sold_out_at CHECK (
        state <> 'SOLD_OUT' OR sold_out_at IS NOT NULL
    )
);

-- Event aktif = yang perlu di-tick admission-ctl (§9.1) dan di-scrape
-- metrics-scraper (§12). Keduanya memulai tiap siklus dengan query ini.
CREATE INDEX IF NOT EXISTS event_queue_state_active_idx
    ON event_queue_state (state) WHERE state IN ('OPEN', 'SOLD_OUT');

COMMIT;
```

### 4.2 Migration 0006 — Domain Booking

File: `migrations/0006_booking_domain.up.sql` / `.down.sql`

```sql
-- 0006_booking_domain.up.sql
--
-- Tabel yang dimiliki booking-service dan payment-api, plus kolom bookings yang
-- 0004 tidak punya karena user-service tidak membacanya.

BEGIN;

-- grant_token: kredensial yang berhak atas booking ini (keputusan §2.1).
--
-- BERBEDA dari queue_token, yang sudah ada di 0004. queue_token menjawab
-- "keterlibatan mana yang melahirkan baris ini" dan dipakai rebuild
-- (ADR-005 §12 langkah 3). grant_token menjawab "kredensial mana yang berhak
-- atas baris ini" dan dipakai setiap klausa kepemilikan §4/§5 kontrak.
-- Grant dicabut dengan satu DEL saat terminal; queue_token harus tetap dapat
-- dicocokkan ke queue_entries sesudahnya. Satu kolom tidak dapat keduanya.
ALTER TABLE bookings
    ADD COLUMN IF NOT EXISTS grant_token TEXT;

-- Partial: baris terminal boleh kehilangan grant-nya tanpa bertabrakan.
CREATE UNIQUE INDEX IF NOT EXISTS bookings_grant_token_key
    ON bookings (grant_token) WHERE grant_token IS NOT NULL;

-- Jalur idempotency POST /bookings: SELECT … WHERE grant_token = $1 FOR UPDATE
-- (ADR-005 §8). Unique index di atas sudah melayaninya.

-- Satu baris per kursi (BR-02: 1 NIK = 1 seat). Nomor bangku tidak
-- diimplementasikan di POC.
CREATE TABLE IF NOT EXISTS booking_items (
    id            TEXT        PRIMARY KEY,
    booking_id    TEXT        NOT NULL REFERENCES bookings (id) ON DELETE CASCADE,
    event_id      TEXT        NOT NULL,
    category_id   TEXT        NOT NULL,
    -- Disalin, bukan di-join: e-tiket harus menampilkan kategori dan harga
    -- sebagaimana berlaku SAAT PEMBELIAN, sekalipun katalog berubah kemudian.
    category_name TEXT        NOT NULL,
    unit_price    BIGINT      NOT NULL,
    nik_hmac      TEXT        NOT NULL,
    nik_masked    TEXT        NOT NULL,
    -- FR-26: tiket menuju tiap pemegang NIK, bukan hanya pembeli. Default
    -- mengikuti bookings.contact_email, dapat diubah per kursi.
    holder_email  TEXT        NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- Cegah NIK ganda DALAM satu booking (ADR-005 §4).
    CONSTRAINT booking_items_booking_nik_key UNIQUE (booking_id, nik_hmac)
);

CREATE INDEX IF NOT EXISTS booking_items_booking_idx
    ON booking_items (booking_id);

-- Rekonsiliasi stok (§9.8) menghitung per kategori per event, status HELD/
-- AWAITING_PAYMENT/CONFIRMED. Tanpa indeks ini query itu memindai tabel tiap
-- 60 detik per kategori.
CREATE INDEX IF NOT EXISTS booking_items_event_category_idx
    ON booking_items (event_id, category_id);

-- PENEGAK BR-02 LINTAS BOOKING: satu NIK = satu kursi per event, berlaku
-- lintas transaksi berbeda. Ini invariant basis data, bukan janji Redis.
--
-- nik:used di Redis adalah jalur cepat yang menolak duplikat sebelum transaksi
-- dimulai. Tabel ini yang membuat pelanggaran BR-02 MUSTAHIL TERSIMPAN, dan
-- yang memberi nik:used sesuatu untuk direkonsiliasi terhadapnya (ADR-005 §6).
CREATE TABLE IF NOT EXISTS nik_claims (
    event_id   TEXT        NOT NULL,
    nik_hmac   TEXT        NOT NULL,
    booking_id TEXT        NOT NULL REFERENCES bookings (id) ON DELETE CASCADE,
    claimed_at TIMESTAMPTZ NOT NULL DEFAULT now(),

    PRIMARY KEY (event_id, nik_hmac)
);

-- Sweep menghapus klaim per booking saat expiry (§9.3) — tanpa itu NIK
-- terkunci selamanya meski booking-nya batal (ADR-005 §8, siklus hidup NIK).
CREATE INDEX IF NOT EXISTS nik_claims_booking_idx
    ON nik_claims (booking_id);

-- Log idempotency pembayaran.
--
-- gateway_event_id adalah ID EVENT MILIK GATEWAY, bukan booking_id. Memakai
-- booking_id di sini akan menelan setiap event SAH berikutnya untuk booking
-- yang sama: pending → settlement → refund (ADR-005 §9, FR-25).
CREATE TABLE IF NOT EXISTS payment_events (
    id                 TEXT        PRIMARY KEY,
    booking_id         TEXT        NOT NULL REFERENCES bookings (id),
    gateway_event_id   TEXT        NOT NULL UNIQUE,
    transaction_status TEXT        NOT NULL,
    fraud_status       TEXT,
    -- payment_type webhook BOLEH berbeda dari yang dikirim saat charge:
    -- pembayaran QR GoPay lewat aplikasi QRIS lain datang sebagai 'qris'.
    -- Dicatat, TIDAK dicocokkan (ADR-005 §9).
    payment_type       TEXT,
    gross_amount       BIGINT,
    raw                JSONB       NOT NULL,
    received_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS payment_events_booking_idx
    ON payment_events (booking_id, received_at DESC);

-- tickets.booking_item_id — guard QR ganda.
--
-- ADR-005 §16 mendaftarkan "UNIQUE (booking_item_id) pada tickets" sebagai
-- mitigasi untuk "Outbox retry menerbitkan QR ganda". 0002 membuat tickets
-- dengan booking_id saja, sehingga mitigasi itu tidak dapat dibuat: dua
-- percobaan ticket.issue untuk booking yang sama tidak bertabrakan.
--
-- Nullable + partial unique, karena baris tickets yang sudah ada (bila ada di
-- lingkungan mana pun) tidak punya nilainya.
ALTER TABLE tickets
    ADD COLUMN IF NOT EXISTS booking_item_id TEXT REFERENCES booking_items (id);

CREATE UNIQUE INDEX IF NOT EXISTS tickets_booking_item_key
    ON tickets (booking_item_id) WHERE booking_item_id IS NOT NULL;

-- CATATAN: qr_nonce TIDAK di-rename. ADR-005 §4 menyebut kolom ini qr_payload;
-- nama yang sudah ada menang karena ia sudah punya penulis dan pembaca
-- (user-service GET /tickets/{id}/eticket). Pemetaan nama di layer mapper.
-- Keputusan §2.3.

COMMIT;
```

### 4.3 Migration 0007 — Indeks BR-07

File: `migrations/0007_br07_index.up.sql` / `.down.sql`

```sql
-- 0007_br07_index.up.sql
--
-- Backstop Aurora untuk BR-07, karena acct:{ev}:{account_id} adalah kunci Redis
-- dan dapat hilang pada flush/failover (ADR-005 §6).
--
-- 0004 sudah punya bookings_active_account_idx (account_id, created_at DESC)
-- untuk lookup BR-09. Itu indeks PENCARIAN; ini indeks PENEGAK. Keduanya
-- diperlukan dan tidak bertabrakan.

BEGIN;

-- Partial: mengecualikan account_id NULL, sehingga CUSTOMER TAMU TIDAK
-- TERBATAS — sesuai keputusan, konsekuensinya di ADR-005 §16. Mengecualikan
-- status terminal, sehingga pembeli yang sudah CONFIRMED bebas memulai
-- pembelian baru.
CREATE UNIQUE INDEX IF NOT EXISTS br07_one_active_per_event
    ON bookings (event_id, account_id)
    WHERE account_id IS NOT NULL
      AND status IN ('GRANTED', 'SELECTING', 'HELD',
                     'AWAITING_PAYMENT', 'PENDING_CANCELLATION');

-- Sweep (§9.3) membaca ZSET lalu memverifikasi ke writer per booking_id — itu
-- lookup primary key, tidak butuh indeks. Rekonsiliasi slots:open (§9.6)
-- TIDAK: ia menghitung COUNT(*) per event atas empat status, tiap 10 detik.
CREATE INDEX IF NOT EXISTS bookings_event_status_idx
    ON bookings (event_id, status);

-- Charge reconciler (§9.9) tiap 5 menit: HELD / AWAITING_PAYMENT dengan
-- payment_ref NULL dan payment_attempts > 0 — charge yang timeout dan
-- statusnya tidak pernah diketahui (ADR-005 §8).
CREATE INDEX IF NOT EXISTS bookings_orphan_charge_idx
    ON bookings (event_id)
    WHERE payment_ref IS NULL
      AND payment_attempts > 0
      AND status IN ('HELD', 'AWAITING_PAYMENT');

-- GET /admin/reconciliation/disputed (§10.7).
CREATE INDEX IF NOT EXISTS bookings_disputed_idx
    ON bookings (event_id) WHERE payment_disputed;

COMMIT;
```

> **`event_categories` tidak dibuat.** v1.0 membuatnya di `0007` untuk rekonsiliasi stok;
> keputusan §2.2 memakai `ticket_categories.capacity`. Tidak ada tabel kedua.

---

## 5. `services/booking` — Completion

Branch referensi punya skeleton: repo in-memory (`infrastructure/memory/init.go`), lookup
kategori hardcoded (`domain/valueobject/category.go`), tanpa DB, Redis, atau NIK guard.
Completion mengganti seluruh layer infrastruktur dan menambah tiga endpoint.

**Endpoint yang dimiliki service ini**: `GET /events/{event_id}/tickets` (§4.1 kontrak),
`POST /bookings` (§4.3), `GET /bookings/{booking_id}` (§4.4, termasuk `eticket_url` bertoken saat
`CONFIRMED` — FR-27, NFR-10). ~~`GET /bookings/{booking_id}/tickets`~~ dihapus (ADR-006 §8.8).

> ⚠️ **Tidak berlaku lagi (ADR-006 §8.8): endpoint di bawah dihapus; catatan dipertahankan sebagai riwayat.**
>
> **`GET /bookings/{id}/tickets` ditugaskan ke sini, bukan ke user-service.** ADR §12
> mendaftarkannya pada tabel kepemilikan objek sebagai endpoint ber-grant, dan
> user-service tidak punya middleware grant (ia bekerja dengan JWT akun). ADR §9 langkah 5
> menjelaskan mengapa ia wajib ada: "Email adalah kenyamanan; endpoint ini adalah sumbernya.
> Tanpa endpoint ini, 'kegagalan email tidak menggagalkan proses' benar secara teknis tetapi
> salah bagi customer." v1.0 tidak menugaskannya ke siapa pun.

### 5.1 Struktur direktori

```
services/booking/internal/booking/
  application/usecase/
    get_ticket_selection.go  ← BARU: GET /events/{id}/tickets + claim GRANTED→SELECTING
    create_booking.go        ← tulis ulang penuh
    get_booking.go           ← ganti repo in-memory dengan Postgres
    abandon_booking.go       ← BARU: transisi ABANDONED (§5.6)
    init.go                  ← tambah port baru
  domain/
    entity/
      booking.go             ← status dari pkg/bookingstate, bukan enum lokal
      booking_item.go        ← BARU
      ticket.go              ← BARU
    dto/
      request.go             ← tickets[] multi-kategori, contact_email, terms_version
      response.go            ← payment_ttl, session, rules, summary
    mapper/
      booking.go             ← + booking_item, ticket; qr_nonce → qr_payload (§2.3)
    valueobject/
      category.go            ← HAPUS lookup hardcoded; sisakan validasi NIK/qty
  infrastructure/
    postgres/
      db.go                  ← BARU
      init.go                ← BARU
      booking.go             ← BARU: IBookingRepository
      booking_item.go        ← BARU: booking_items + nik_claims dalam satu transaksi
      category.go            ← BARU: ticket_categories
      ticket.go              ← BARU: baca tickets untuk GET /bookings/{id} (CONFIRMED)
    redis/
      init.go                ← BARU
      stock.go               ← BARU: hold + release via pkg/redisscript
      nikcache.go            ← BARU: baca nik:valid:{hmac}
      expiry.go              ← BARU: ZADD expiries (rescore ke payment_expires_at)
    nikverify/
      client.go              ← BARU: HTTP ke nik-verify
    http/
      booking.go             ← handler
      ticket_selection.go    ← BARU
      router.go              ← + 1 rute, + pkg/grantauth.Middleware
      init.go
  memory/                    ← HAPUS seluruh direktori
  port/
    inbound/booking.go       ← + GetTicketSelection, Abandon
    outbound/
      booking.go             ← IBookingRepository
      booking_item.go        ← BARU: IBookingItemRepository
      category.go            ← BARU: ICategoryRepository
      ticket.go              ← BARU: ITicketRepository
      stock.go               ← BARU: IStockStore
      nikverify.go           ← BARU: INIKVerifier
```

> **Tidak ada `grant.go` di `port/outbound`.** Validasi grant ada di `pkg/grantauth`
> (§3.5) sebagai middleware, sehingga usecase menerima `grantauth.Grant` dari context
> alih-alih memanggil port. v1.0 memberi tiap service port grant sendiri.

### 5.2 Outbound ports

```go
// port/outbound/booking.go
type IBookingRepository interface {
    // ClaimForSelection menjalankan GRANTED → SELECTING secara idempoten
    // (keputusan §2.4). Tenggat TIDAK diubah: booking_expires_at sudah
    // ditetapkan saat GRANT.
    //
    //   UPDATE bookings SET status='SELECTING', slot_claimed_at=now()
    //    WHERE grant_token=$1 AND status='GRANTED'
    //
    // 0 baris bukan error: status mungkin sudah SELECTING (refresh halaman)
    // atau sudah lebih jauh (kembali dari halaman pembayaran). Mengembalikan
    // baris apa adanya supaya handler dapat memutuskan.
    ClaimForSelection(ctx context.Context, grantToken string) (entity.Booking, error)

    // GetForHold mengambil baris dengan FOR UPDATE di dalam tx. Kunci ditahan
    // SELAMA Lua dan seluruh UPDATE — itu guard idempotency-nya (ADR-005 §8).
    GetForHold(ctx context.Context, tx pgx.Tx, grantToken string) (entity.Booking, error)

    // MarkHeld menulis status, held_at, payment_expires_at, contact_email,
    // terms_version dan terms_accepted_at DALAM SATU UPDATE di dalam tx.
    //
    //   WHERE id = $1 AND status = 'SELECTING'
    //
    // Semua di satu tempat karena ADR-005 §7 menuntutnya: "Deadline pembayaran
    // ditulis dalam transaksi hold yang sama, jadi tidak ada jendela di mana
    // stok sudah terkunci tetapi deadline-nya belum ada."
    MarkHeld(ctx context.Context, tx pgx.Tx, in HeldUpdate) (rows int64, err error)

    // GetOwned mengikat booking_id ke pemegang grant — klausa kepemilikan
    // ADR-005 §12: WHERE id = $1 AND grant_token = $2.
    // Tidak cocok → apperror 404, BUKAN 403, dan diperiksa SEBELUM validasi
    // status apa pun agar kode respons tidak membocorkan keadaan booking
    // orang lain.
    GetOwned(ctx context.Context, bookingID, grantToken string) (entity.Booking, error)

    // Abandon: apa pun → ABANDONED (§5.6).
    Abandon(ctx context.Context, tx pgx.Tx, bookingID, grantToken string) (rows int64, err error)
}

// port/outbound/booking_item.go
type IBookingItemRepository interface {
    // Insert menulis booking_items DAN nik_claims dalam tx yang sama.
    //
    // Konflik pada nik_claims (event_id, nik_hmac) berarti Redis melewatkan
    // sesuatu — flush, failover, atau TTL habis. Transaksi batal, kompensasi
    // jalan, customer menerima NIK_TAKEN. Pelanggaran BR-02 mustahil
    // tersimpan (ADR-005 §8).
    Insert(ctx context.Context, tx pgx.Tx, bookingID string, items []entity.BookingItem) error

    ListByBooking(ctx context.Context, bookingID string) ([]entity.BookingItem, error)
}

// port/outbound/category.go
type ICategoryRepository interface {
    // ListByEvent membaca ticket_categories. price, name, description dan
    // max_per_order dari sini; `available` dari Redis (IStockStore.Levels).
    ListByEvent(ctx context.Context, eventID string) ([]entity.Category, error)
}

// port/outbound/stock.go
type IStockStore interface {
    // Hold menjalankan hold_booking.lua (§14.2) via pkg/redisscript.
    // Satu skrip mengurus stok seluruh kategori DAN kunci NIK per kursi.
    //
    // Kode yang mungkin: HELD, QTY_INVALID, QTY_EXCEEDED, NIK_COUNT_MISMATCH,
    // STOCK_UNINITIALIZED, NO_STOCK, NIK_TAKEN, STOCK_NEGATIVE.
    //
    // STOCK_UNINITIALIZED dan STOCK_NEGATIVE dijawab 503, BUKAN 409: stok
    // sebenarnya tidak diketahui, dan keduanya menuntut rekonsiliasi SEGERA
    // (ADR-005 §8).
    Hold(ctx context.Context, req HoldRequest) (HoldResult, error)

    // Release adalah KOMPENSASI WAJIB bila transaksi Aurora rollback setelah
    // Hold berhasil. Redis tidak ikut transaksi Aurora: tanpa langkah ini
    // stok dan NIK bocor, sweep meng-expire baris itu lewat jalur SELECTING
    // yang TIDAK mengembalikan stok, dan tiket hilang dari peredaran
    // selamanya (ADR-005 §6, §8). Emit HoldCompensated.
    Release(ctx context.Context, req HoldRequest) error

    // Levels membaca stock:{ev}:{cat} per kategori untuk §4.1.
    Levels(ctx context.Context, eventID string, categoryIDs []string) (map[string]int64, error)
}

type HoldRequest struct {
    EventID    string
    Quantities []CategoryQty // urutan menentukan urutan KEYS[1..m]
    NIKHmacs   []string      // satu per KURSI, total == sum(Quantities)
}

type HoldResult struct {
    OK    bool
    Code  string // "HELD" | "NO_STOCK" | "NIK_TAKEN" | ...
    Index int    // indeks kategori atau NIK yang bentrok; -1 bila tidak berlaku
}

// port/outbound/nikverify.go
type INIKVerifier interface {
    // Verify memanggil POST /nik/validate pada nik-verify, meneruskan
    // X-Grant-Token dan X-Device-Id pemanggil.
    //
    // source: "registered" | "saved" | "verified" (kontrak §4.2).
    // valid == false berarti BLOKIR. err non-nil hanya pada kegagalan
    // transport, dan itu juga BLOKIR — fail closed (BR-13).
    Verify(ctx context.Context, grantToken, deviceID, nik string) (valid bool, source string, err error)
}
```

### 5.3 Usecase: `GetTicketSelection` — `GET /events/{event_id}/tickets`

Ini juga claim-nya (keputusan §2.4).

```
1. grant := grantauth.FromContext(ctx)            -- middleware sudah memverifikasi
2. booking := repo.ClaimForSelection(grant.Token)
     status GRANTED   → di-UPDATE ke SELECTING, slot_claimed_at = now()
     status SELECTING → no-op (refresh halaman)
     status HELD / AWAITING_PAYMENT → no-op; handler TETAP merender halaman,
         karena customer yang menekan Back dari halaman pembayaran harus
         melihat pilihannya, bukan error
     status terminal  → 409 BOOKING_EXPIRED
     0 baris          → 404 (grant tidak punya booking; seharusnya mustahil
                        karena grant hanya diterbitkan SETELAH INSERT, §9.5)
3. categories := categoryRepo.ListByEvent(grant.EventID)
4. levels := stock.Levels(grant.EventID, ids(categories))
     available := levels[cat.ID]
     max_per_order := min(cat.MaxPerOrder, available, MAX_TICKETS_PER_TRANSACTION)
       -- kontrak §4.1 memberi max_per_order 0 untuk kategori habis
5. session := { expires_at: booking.BookingExpiresAt,
                ttl_seconds: booking.BookingExpiresAt - now(),
                server_time: now() }
     -- Tenggat dibaca dari Aurora, bukan dihitung ulang: ia dapat BERGESER
     -- bila admission di-pause (PAUSE_TTL_GRACE_SECONDS, §9.4), dan kontrak
     -- §3.3 memerintahkan klien membaca ulang alih-alih menghitung turun.
6. rules := { max_tickets_per_order: MAX_TICKETS_PER_TRANSACTION,
              one_nik_per_ticket: true }
```

### 5.4 Usecase: `CreateBooking` — `POST /bookings`

Urutannya adalah ADR §8 "Idempotency pada `/bookings`" apa adanya. Yang v1.0 ubah dan
di sini dikembalikan: **transaksi dibuka SEBELUM `FOR UPDATE` dan ditutup SETELAH
seluruh UPDATE/INSERT**, dan langkahnya **UPDATE** baris yang ada, bukan INSERT baris baru.

```
1. Validasi input:
     len(tickets) antara 1 dan MAX_TICKETS_PER_TRANSACTION
     tiap NIK: normalisasi (buang spasi/pemisah) lalu tepat 16 digit ASCII
       -- normalisasi WAJIB sebelum HMAC: "3171 1234 5678 9012" dan
       -- "3171123456789012" harus menghasilkan hmac identik, bukan dua klaim
       -- berbeda (ADR-005 §8)
     NIK duplikat dalam permintaan → 400 VALIDATION_ERROR (tanpa memanggil Redis)
     contact_email format; terms_version == TERMS_VERSION aktif
     tiap category_id ada di event ini
     → gagal: 400, tidak ada efek samping

2. grant := grantauth.FromContext(ctx)
     -- Validasi grant DAN device binding sudah dilakukan middleware (§3.5),
     -- termasuk penghitung rebind. Usecase tidak mengulanginya.

3. Verifikasi setiap NIK — FAIL CLOSED (BR-13):
     untuk tiap tiket: nikVerifier.Verify(grant.Token, deviceID, nik)
       valid == false        → 400 NIK_UNVERIFIED (atau NIK_NOT_FOUND / NIK_INACTIVE)
       err != nil            → 503 DUKCAPIL_UNAVAILABLE
     -- Dijalankan di sini, BUKAN hanya di on-blur frontend: §4.3 kontrak
     -- menuntut POST /bookings menolak kecuali SETIAP NIK punya bukti
     -- verifikasi. Panggilan ini umumnya cache/bukti-tersimpan, bukan
     -- Dukcapil (ADR-005 §8, "NIK tersimpan: memotong panggilan Dukcapil").

4. BEGIN
5.   booking := repo.GetForHold(tx, grant.Token)        -- FOR UPDATE
       status HELD atau lebih jauh → COMMIT; kembalikan booking yang ADA, 200
         -- Tanpa guard ini retry gagal dengan NIK_TAKEN atas NIK customer
         -- sendiri: SET nik:used NX bentrok dengan baris yang ditulis
         -- panggilan PERTAMA (ADR-005 §8).
       status != SELECTING          → COMMIT; 409
         -- Termasuk GRANTED: customer belum memuat Detail Ticket Page, jadi
         -- ia belum punya pilihan untuk disubmit (keputusan §2.4).

6.   held := stock.Hold(HoldRequest{...})                -- EVALSHA, DI DALAM tx
       !held.OK → COMMIT (tanpa perubahan); petakan kode:
         NO_STOCK            → 409 INSUFFICIENT_STOCK  { available, requested }
         NIK_TAKEN           → 409 NIK_ALREADY_USED
         QTY_EXCEEDED        → 400 QTY_EXCEEDED
         QTY_INVALID         → 400 QTY_INVALID
         NIK_COUNT_MISMATCH  → 400 VALIDATION_ERROR
         STOCK_UNINITIALIZED → 503 + alarm StockKeyMissing
         STOCK_NEGATIVE      → 503 + alarm StockNegativeGuard
       -- GAGAL HOLD TIDAK MENGHILANGKAN SLOT. Customer tetap SELECTING
       -- dengan sisa TTL; UI menampilkan pesan spesifik; pemilihan ulang
       -- kategori atau perbaikan NIK diizinkan (NFR-02, ADR-005 §8).
       -- Itu sebabnya cabang ini COMMIT, bukan ROLLBACK: tidak ada yang
       -- perlu dibatalkan, dan status SELECTING harus tetap tersimpan.

7.   rows := repo.MarkHeld(tx, HeldUpdate{
         ID: booking.ID, Status: HELD, HeldAt: now(),
         PaymentExpiresAt: now() + PAYMENT_TTL_SECONDS,    -- ANCHOR BR-05
         ContactEmail: req.ContactEmail,
         TermsVersion: req.TermsVersion, TermsAcceptedAt: now(),
       })
       rows != 1 → ROLLBACK + KOMPENSASI (langkah 10)

8.   itemRepo.Insert(tx, booking.ID, items)   -- booking_items + nik_claims
       konflik nik_claims → ROLLBACK + KOMPENSASI
       -- BR-02 ditegakkan basis data di sini, bukan di Redis.

9.   outbox.Enqueue(tx)  -- TIDAK ADA. Tidak ada email saat booking dibuat;
                         -- outbox pertama terbit saat CONFIRMED atau EXPIRED.
   COMMIT

10. ON ROLLBACK setelah Hold berhasil — KOMPENSASI WAJIB:
      stock.Release(sama persis dengan HoldRequest)
        -- INCRBY stock per kategori + DEL nik:used per NIK
      emit HoldCompensated
      -- nik_claims ikut di-rollback Aurora; hanya Redis yang perlu kompensasi.
      -- Bila kompensasi sendiri gagal → rekonsiliasi stok (§9.8) menangkapnya
      -- dalam ~60 detik.
      → 500

11. SETELAH COMMIT: expiry.Rescore(eventID, bookingID, payment_expires_at)
      -- ZADD expiries:{ev} (payment_expires_at) booking_id — rescore dari TTL
      -- booking ke TTL pembayaran (ADR-005 §8).
      -- Crash sebelum langkah ini TIDAK meng-expire lebih awal: sweep membaca
      -- deadline dari Aurora dan me-rescore entri basi (§9.3). Kegagalan di
      -- sini karena itu TIDAK menggagalkan request.

12. Respons 201 (kontrak §4.3): booking_id, status "PENDING_PAYMENT", event,
    tickets[] (category, price, nik_masked), summary, payment_ttl
    { expires_at, ttl_seconds: 600, server_time }, redirect_url
      -- ttl_seconds SELALU 600: setiap customer mendapat 10 menit penuh,
      -- berapa pun lama ia memilih kursi (kontrak §4.3, BR-05).
      -- TIDAK ADA holder_name: Dukcapil hanya memvalidasi NIK, dan respons
      -- §4.2 tidak pernah memuat nama (FR-15a). Contoh di kontrak §4.3 yang
      -- memuat holder_name adalah sisa versi lama — lihat §21.
```

### 5.5 Usecase: `GetBooking`

```
GET /bookings/{booking_id}            -- kontrak §4.4
  1. booking := repo.GetOwned(id, grant.Token)   -- 0 baris → 404
  2. items   := itemRepo.ListByBooking(id)
  3. status  := bookingstate.ToAPI(booking.Status)    -- pkg/bookingstate
  4. bila CONFIRMED: tickets := ticketRepo.ListByBooking(id)
       sertakan ticket_id + eticket_url per kursi
       eticket_url = "/tickets/{ticket_id}/eticket?t=" + pkg/eticket.Mint(ticket_id, event_date + 1 hari)
       (token sama dengan email; NFR-10 tanpa endpoint terpisah, ADR-006 §8.8)
  5. payment_ttl dari booking.PaymentExpiresAt (bukan dihitung ulang)
```

### 5.6 Usecase: `Abandon` — transisi `ABANDONED`

ADR §6 mendaftarkan `apa pun → ABANDONED` sebagai milik `booking-service`, dengan
`DECR slots:open` + `DEL nik:used` + `DEL acct:{ev}:{id}`. v1.0 tidak punya jalur ini
sama sekali, sehingga status itu ada di enum dan di CHECK constraint `0004` tanpa cara
mencapainya.

```
POST /bookings/{booking_id}/abandon   -- tidak ada di kontrak v2.5; lihat §21
  BEGIN
    rows := repo.Abandon(tx, id, grant.Token)   -- WHERE status IN (non-terminal)
    rows != 1 → ROLLBACK; 409
    DELETE FROM nik_claims WHERE booking_id = $1
  COMMIT
  SETELAH COMMIT:
    DECRBY slots:open:{ev} 1          -- terminal (pkg/bookingstate.DecrementsSlot)
    INCRBY stock:{ev}:{cat} qty       -- bila statusnya HELD atau lebih jauh
    DEL nik:used:{ev}:{hmac} per item
    DEL acct:{ev}:{account_id}
    DEL grant:{token} ; DEL token:{uuid}
    ZREM expiries:{ev} booking_id
    EVALSHA advance_now_serving.lua ; PUBLISH sse-broadcast
```

> **Diputuskan: endpoint ini dibangun tetapi TIDAK diekspos di API Gateway pada POC.**
> Kontrak v2.5 tidak memuatnya, jadi mengeksposnya akan menambah permukaan yang belum
> disepakati. Yang dibangun adalah usecase-nya, dipanggil hari ini hanya dari test —
> karena transisi ini ada di tabel kepemilikan `slots:open` ADR §6, dan tabel itu
> diimplementasikan sebagai kode di `pkg/bookingstate` (§3.7) yang testnya harus
> dapat melewati setiap barisnya. Masuk ke §21 sebagai item terbuka.

### 5.7 Config

```go
type Config struct {
    App       App
    Log       Log
    DB        DB           // writer endpoint — service ini MENULIS
    Redis     Redis
    NIKVerify NIKVerifyCfg // {BaseURL, TimeoutMs}
    Booking   BookingCfg   // {MaxTicketsPerTransaction, NIKUsedTTLSeconds,
                           //  PaymentTTLSeconds, TermsVersion}
    Grant     GrantCfg     // {DevicePepper, MaxDeviceRebinds} → pkg/grantauth
}
```

---

## 6. `services/events`

**Tipe compute**: Lambda + API Gateway
**Cache**: CloudFront 30 detik — handler menyetel `Cache-Control: public, max-age=30`;
distribusinya milik infra, bukan repo ini
**Endpoint**: `GET /events` (FR-01–03), `GET /events/{event_id}` (FR-11, FR-12)

Hanya baca. Satu-satunya service yang memakai **reader endpoint** Aurora.

### 6.1 Struktur direktori

```
services/events/
  cmd/lambda/main.go
  cmd/server/main.go
  internal/
    config/config.go, load.go
    delivery/server.go, lambda_proxy_test.go, integration_test.go
    event/
      application/usecase/
        list_events.go
        get_event.go
        init.go
      domain/
        dto/request.go, response.go
        entity/event.go, category.go, queue_summary.go
        valueobject/display.go        ← ResolveDisplay (pure function)
        mapper/event.go
      infrastructure/
        postgres/db.go, init.go, event.go, category.go
        redis/init.go, queue_state.go, stock.go
        http/event.go, router.go, init.go
      port/
        inbound/event.go
        outbound/event.go, queue_state.go
```

### 6.2 Outbound ports

```go
type IEventRepository interface {
    List(ctx context.Context, f dto.ListFilter) (items []entity.Event, total int, err error)
    GetByID(ctx context.Context, id string) (entity.Event, error)
}

type ICategoryRepository interface {
    // ListByEvent mengembalikan kategori + capacity. capacity dibutuhkan untuk
    // menghitung threshold SELLING_OUT_FAST, yang persentase dari kapasitas.
    ListByEvent(ctx context.Context, eventID string) ([]entity.Category, error)
    // PriceRangeByEvents mengisi min_price / max_price pada §2.1 untuk SEMUA
    // event dalam satu halaman dengan satu query — bukan N+1.
    // NULL untuk event yang seluruh kategorinya habis (kontrak §2.1).
    PriceRangeByEvents(ctx context.Context, eventIDs []string) (map[string]entity.PriceRange, error)
}

type IQueueStateStore interface {
    // Summary membaca, dalam satu pipeline per event:
    //   queue:{ev}:issued, queue:{ev} now_serving, queue:{ev}:state,
    //   queue:{ev}:capacity, slots:open:{ev}
    // depth = issued − now_serving (ADR-005 §4).
    //
    // Redis tak terjangkau → BUKAN 500. Jatuh ke snapshot Aurora:
    // event_queue_state.state dan ticket_categories.available, dengan depth
    // tidak diketahui → queue_state diturunkan tanpa FULL. Halaman utama yang
    // di-cache 30 detik tidak boleh mati karena Redis hiccup; penegaknya tetap
    // pemeriksaan atomik saat join (kontrak §2.1).
    Summary(ctx context.Context, eventIDs []string) (map[string]entity.QueueSummary, error)

    // Levels membaca stock:{ev}:{cat}.
    Levels(ctx context.Context, eventID string, categoryIDs []string) (map[string]int64, error)
}
```

### 6.3 `ResolveDisplay` — tiga input, bukan dua

v1.0 memberi signature `ResolveDisplay(status, queueState)`. Itu tidak dapat menghitung
aturan pertamanya: kontrak §2.1 membedakan `sold_out` dari `sold_out_may_return`
berdasarkan apakah **state internal** sudah `CLOSED` — dan `CLOSED` **bukan** nilai dari
enum `queue_state` yang dikirim ke klien (kontrak menghapusnya dari sana). Juga `badge_key`
hilang dari daftar aturan v1.0.

```go
// domain/valueobject/display.go
package valueobject

// Status adalah sumbu STOK saja (kontrak §2.1).
type Status string
const (
    StatusAvailable      Status = "AVAILABLE"       // available > threshold
    StatusSellingOutFast Status = "SELLING_OUT_FAST" // 0 < available <= threshold
    StatusSoldOut        Status = "SOLD_OUT"         // available == 0 — REVERSIBEL
)

// QueueState adalah sumbu IZIN MASUK saja, dan ini yang dikirim ke klien.
// Tidak ada CLOSED di sini: ia dulu berarti "event sold out", sehingga ia
// melaporkan urusan Status (kontrak §2.1).
type QueueState string
const (
    QueueNotOpen QueueState = "NOT_OPEN" // internal SCHEDULED
    QueueFull    QueueState = "FULL"     // depth >= capacity
    QueueDirect  QueueState = "DIRECT"   // depth == 0 dan masih ada slot — FR-05
    QueueOpen    QueueState = "OPEN"     // sisanya
)

// InternalState adalah queue:{ev}:state di Redis. Ia punya CLOSED; QueueState
// tidak. Input ketiga ada justru untuk perbedaan itu.
type InternalState string
const (
    InternalScheduled InternalState = "SCHEDULED"
    InternalOpen      InternalState = "OPEN"
    InternalSoldOut   InternalState = "SOLD_OUT"
    InternalClosed    InternalState = "CLOSED"
)

type Display struct {
    Enabled  bool    `json:"enabled"`
    Action   string  `json:"action"`    // JOIN_QUEUE | GO_TO_TICKETS | NONE
    LabelKey string  `json:"label_key"`
    BadgeKey *string `json:"badge_key"` // selling_out_fast | null
}

// ResolveDisplay adalah fungsi murni — tanpa I/O, mudah diuji sebagai tabel
// kebenaran. Kecocokan PERTAMA menang; urutan ini milik SERVER, bukan frontend
// (kontrak §2.1).
//
//   status == SOLD_OUT && internal == CLOSED → false NONE          sold_out
//   status == SOLD_OUT && internal != CLOSED → false NONE          sold_out_may_return
//   queueState == NOT_OPEN                   → false NONE          coming_soon
//   queueState == FULL                       → false NONE          queue_full
//   queueState == DIRECT                     → true  GO_TO_TICKETS buy_now
//   queueState == OPEN                       → true  JOIN_QUEUE    join_queue
//
// badge_key = selling_out_fast bila status == SELLING_OUT_FAST DAN Enabled.
//   Satu syarat itu menggantikan aturan khusus "sembunyikan saat FULL":
//   lencana urgensi hanya tampil ketika tombolnya memang dapat ditekan,
//   sehingga FULL, NOT_OPEN dan SOLD_OUT tercakup sekaligus. Menampilkan
//   "hampir habis" kepada 300.000 orang yang tidak dapat masuk hanya menambah
//   tekanan pada perilaku yang ingin diserap ruang tunggu (kontrak §2.1).
func ResolveDisplay(status Status, queueState QueueState, internal InternalState) Display

// DeriveStatus: threshold = capacity × SELLING_FAST_THRESHOLD_PCT / 100 (20).
func DeriveStatus(available, capacity int64, thresholdPct int) Status

// DeriveQueueState memetakan state internal + depth ke enum klien.
// SOLD_OUT dan CLOSED keduanya jatuh ke OPEN/FULL di sumbu ini — sold-out
// hanya dikatakan oleh Status.
func DeriveQueueState(internal InternalState, depth, capacity, slotsOpen int64) QueueState
```

> **Label dan badge adalah KUNCI, bukan teks** (FR-32). Server yang melokalkan harus
> memasukkan `Accept-Language` ke cache key CDN, memecah satu respons yang di-cache 30
> detik untuk 300.000 orang menjadi banyak varian. Kunci + tabel terjemahan di frontend
> adalah pencarian, bukan logika.

> **Halaman utama menampilkan waktu absolut, bukan hitung mundur.** `sale_starts_at`
> dikirim apa adanya; label pra-buka berbunyi "Buka 10 Nov 09:00". Hitung mundur hidup
> hanya ada di ruang tunggu, tempat `serverTime` datang pada setiap payload SSE.

### 6.4 Config

```go
type Config struct {
    App    App
    Log    Log
    DB     DB        // READER endpoint — service ini hanya membaca
    Redis  Redis
    Events EventsCfg // {SellingFastThresholdPct: 20, CacheMaxAgeSeconds: 30,
                     //  DefaultLimit: 10, MaxLimit: 50}
}
```

---

## 7. `services/queue`

**Tipe compute**: Lambda + API Gateway, dengan Provisioned Concurrency (T-30 menit)
**Endpoint**: `POST /events/{event_id}/queue/join` (FR-04, FR-05),
`GET /events/{event_id}/queue/me` (BR-09), `GET /events/{event_id}/queue/serving` (fallback)

> `GET /events/{event_id}/queue/stream` **bukan** milik service ini — ia di ALB → ECS
> (§8). API Gateway hanya punya tiga rute di atas.

### 7.1 Struktur direktori

```
services/queue/
  cmd/lambda/main.go
  cmd/server/main.go
  internal/
    config/config.go, load.go
    delivery/server.go, lambda_proxy_test.go, integration_test.go
    queue/
      application/usecase/
        join.go
        get_me.go
        get_serving.go
        init.go
      domain/
        dto/request.go, response.go
        entity/engagement.go, queue_state.go
        valueobject/poll_interval.go   ← profil pollAfterMs
        mapper/queue.go
      infrastructure/
        redis/
          init.go
          engagement.go    # HSET token:{uuid}, INCR issued, ZADD by_ticket
          queue_state.go   # GET now_serving, state, issued, capacity
          grant.go         # HGETALL grant:{token} (BACA saja; penulisnya §9.5)
          acct.go          # SET acct:{ev}:{id} NX  (BR-07)
          persist.go       # RPUSH qpersist:{ev}
          settled.go       # GET settled:{uuid}   (nisan, §10.5)
          admission.go     # EVALSHA advance_now_serving.lua (jalur bypass FR-05)
        postgres/
          db.go, init.go
          booking.go       # baca status booking untuk pemetaan state §3.3
        captcha/
          turnstile.go     # verify, fail-open
        http/queue.go, router.go, init.go
      port/
        inbound/queue.go
        outbound/
          queue_store.go
          grant_store.go
          booking.go
          captcha.go
```

> **Service ini butuh Postgres (reader).** v1.0 memberinya hanya Redis. Pemetaan state
> §3.3 kontrak membedakan `SELECTING`, `AWAITING_PAYMENT` dan `CONFIRMED`, dan ketiganya
> adalah **status baris `bookings`** — tidak ada di Redis. Tanpa pembacaan itu `/queue/me`
> tidak dapat merutekan customer yang kembali, yang merupakan seluruh gunanya (BR-09).

### 7.2 Usecase: `Join`

Urutan ADR §6 apa adanya. Yang v1.0 lewatkan dan di sini ditambahkan: penolakan
`SCHEDULED`, `DEVICE_ID_REQUIRED`, dan batas kedua per-customer.

```
 1. ip := client IP (X-Forwarded-For paling kanan yang dipercaya ALB/APIGW)
    accountID := optional Bearer → pkg/token.Validate
      -- Authorization di SINI, dan hanya di sini di antara jalur antrian
      -- (kontrak §3.1). Ia mengaktifkan BR-07 dan menanam account_id pada
      -- keterlibatan, yang kemudian menjadi bookings.account_id saat GRANT
      -- tanpa pernah menanyakan klien lagi.
    deviceID := X-Device-Id
      accountID == "" && deviceID == "" → 400 DEVICE_ID_REQUIRED
      -- Ditolak DI SINI, bukan nanti: grant tanpa pengikatan tidak dapat
      -- diikat belakangan (kontrak Appendix A).

 2. RATE LIMIT dulu:
      ratelimit.Allow(rediskeys.JoinRateLimit(ip), JOIN_RATE_LIMIT, 20/(5*60))
        → 429 TOO_MANY_REQUESTS { retry_after }
      bila login: batas kedua 3 join/menit per akun (kontrak §3.3)
        ratelimit.Allow("rl:joinacct:"+accountID, 3, 3.0/60)
    -- URUTAN 1 SEBELUM 2 ITU WAJIB. Rate limit adalah panggilan Redis lokal;
    -- verifikasi CAPTCHA adalah round trip ke pihak ketiga. Verifikasi dulu
    -- lalu membatasi berarti membayar satu panggilan eksternal untuk setiap
    -- percobaan yang akhirnya ditolak — penyerang memilih biaya kita
    -- (ADR-005 §6).

 3. CAPTCHA:
      captcha_token kosong → 400 CAPTCHA_REQUIRED
      captcha.Verify(token):
        invalid        → 403 CAPTCHA_FAILED   -- TANPA memberi ticket_no,
                         -- sehingga percobaan gagal tidak memakan posisi
        provider down  → LANJUT (FAIL-OPEN) + emit CaptchaFailOpenSeconds
      -- Fail-closed bila yang dipertaruhkan kebenaran; fail-open bila hanya
      -- keadilan. Fail-closed di sini berarti TIDAK SEORANG PUN dapat
      -- bergabung saat provider terganggu — sale mati sebelum dimulai
      -- (ADR-005 §6).
      -- Berlaku untuk tamu DAN user login: akun hanya berbiaya 5 per IP per
      -- jam, jadi mengecualikan login menjadikannya jalur ramah bot.

 4. state := GET queue:{ev}:state
      SCHEDULED → 409  { sale_starts_at }       -- BARU di v2.0
        -- Nilai AWAL. Sale belum dibuka (ADR-005 §6). 300.000 orang berkumpul
        -- SEBELUM gerbang dibuka; tanpa cabang ini mereka semua mendapat
        -- ticket_no untuk sale yang belum ada.
      CLOSED    → 410  { queue_state: "CLOSED" }
      kunci tidak ada → 503  -- event tidak di-seed (§11.3); JANGAN perlakukan
                             -- sebagai SCHEDULED, itu menyembunyikan salah
                             -- konfigurasi di belakang layar pra-buka

 5. QUEUE_FULL, fail fast:
      issued := GET queue:{ev}:issued ; nowServing := HGET queue:{ev} now_serving
      issued − nowServing >= QUEUE_CAPACITY
        → 503 QUEUE_FULL { queue_state: "FULL", queue_capacity }
      -- Tanpa buffering dan tanpa memberi nomor antrian yang tidak akan
      -- pernah terlayani (ADR-005 §6).

 6. BR-07, HANYA bila login:
      SET acct:{ev}:{accountID} {uuid} NX EX SALE_WINDOW_SECONDS
      gagal → BUKAN error. Ambil {uuid} yang ada, HGETALL token:{uuid},
              terbitkan ULANG cookie untuk token itu, kembalikan keadaannya
              dengan { rejoined: true, state: <§3.3> }, 200
              emit Br07JoinRecovered
      -- Menjawab 409 akan menghukum customer yang hanya berpindah perangkat —
      -- login di ponsel setelah antre di laptop. BR-07 tetap ditegakkan: ia
      -- tidak mendapat keterlibatan kedua (ADR-005 §6).
      -- Bila token:{uuid} TIDAK ADA padahal acct ada (TTL token habis lebih
      -- dulu, atau keterlibatan terminal menghapusnya): DEL acct dan lanjut
      -- ke langkah 7 sebagai join baru.

 7. ticket_no := INCR queue:{ev}:issued
 8. uuid := idgen.New()
    HSET token:{uuid} ticket_no, event_id, account_id?, device_hash?, joined_at
      EX SALE_WINDOW_SECONDS
      -- device_hash = HMAC(DEVICE_PEPPER, X-Device-Id) hanya untuk pemanggil
      -- ANONIM. Pemanggil login tidak: account_id sudah mengikat lebih kuat
      -- (ADR-005 §6, §12).
    ZADD queue:{ev}:by_ticket {ticket_no} {uuid}
 9. RPUSH qpersist:{ev} "{uuid}|{ticket_no}|{account_id}|{joined_at}"
      -- Satu RPUSH, tetap di jalur cepat Redis. Aurora TIDAK disentuh di
      -- sini: 300.000 join dalam ~60 detik = ~5.000 insert/detik tepat pada
      -- momen beban tertinggi, dan itu akan menjadikan Aurora SPOF baru di
      -- satu-satunya jalur yang sengaja dibuat Redis-only (ADR-005 §6, §12).
      -- Penulisannya di-batch leader admission-ctl (§9.7).

10. BYPASS FR-05 — antrian kosong:
      bila (issued − nowServing) == 1 setelah INCR, DAN slots:open < MAX,
      DAN remaining_stock > 0:
        res := redisscript.Run("advance_now_serving",
                 keys(now_serving, slots:open, state, regrant, by_ticket,
                      sold_out_at, resume_flag),
                 MAX_CONCURRENT_SLOTS, remaining_stock, now_unix)
        -- SKRIP YANG SAMA dengan admission-ctl (pkg/redisscript, §3.4),
        -- sehingga tidak ada jalur admission kedua yang bisa menyimpang dari
        -- yang utama (ADR-005 §5). Bukan "deficit=1": deficit dihitung di
        -- dalam skrip dari slots:open yang dibaca ulang.
        bila uuid kita ada di res.tokens:
          -- GRANT HARUS DIINSERT SEBELUM DIBERITAHUKAN (NFR-02).
          INSERT INTO bookings (id, event_id, queue_token, grant_token,
                                account_id, status='GRANTED', granted_at=now(),
                                booking_expires_at=now()+BOOKING_TTL_SECONDS)
            gagal setelah GRANT_INSERT_MAX_RETRIES:
              DECRBY slots:open:{ev} 1 ; SADD regrant:{ev} {uuid}
              emit GrantInsertFailures
              → lanjut sebagai QUEUED biasa (langkah 11)
          SET grant:{token} <fields> EX GRANT_TOKEN_TTL_SECONDS
          ZADD expiries:{ev} booking_expires_at booking_id
          → respons { granted: true, grant_token, booking_expires_at }
             klien langsung ke Detail Ticket Page tanpa pernah membuka stream
      -- Service ini karena itu MENULIS ke Aurora, meski hanya di jalur ini.

11. Set-Cookie: queue_token={uuid}; HttpOnly; Secure; SameSite=Strict;
                Max-Age={SALE_WINDOW_SECONDS}
      -- Max-Age DITURUNKAN dari nilai yang sama dengan TTL token:{uuid},
      -- bukan dipatok, supaya cookie dan kunci yang dirujuknya tidak pernah
      -- kedaluwarsa pada waktu yang berbeda (ADR-005 §12).
      -- SameSite=Strict per kontrak §9; stream memakai cookie dari origin
      -- yang sama (events.my-rusuh.my.id), jadi Strict cukup.

12. Respons 200: ticket_no, now_serving, position (ticket_no − now_serving),
    estimated_wait_minutes (position / rate_per_min), sse_url, event{id,name}
```

### 7.3 Usecase: `GetMe` — `GET /events/{event_id}/queue/me`

Resolusi auth — **salah satu** dari tiga (kontrak §3.3):

```
1. Cookie queue_token      → HGETALL token:{uuid}
2. X-Queue-Token           → HGETALL token:{uuid}
3. Authorization: Bearer   → pkg/token.Validate → accountID
                           → GET acct:{ev}:{accountID} → uuid
                           → HGETALL token:{uuid}
                           → terbitkan ULANG cookie pada respons
```

> **Pemulihan adalah READ, dan itu disengaja.** `POST …/queue/join` juga memulihkan
> keterlibatan yang ada (BR-07), tetapi ia jalur tulis yang membawa CAPTCHA dan dua rate
> limit. Customer yang sekadar kembali tidak boleh diminta menyelesaikan CAPTCHA untuk
> mengambil posisi yang sudah dimilikinya — dan di belakang NAT kantor atau operator
> seluler, batas 20 join per 5 menit habis justru karena orang-orang yang reconnect.
> Endpoint ini **tidak** menuntut `captcha_token` dan **tidak** dihitung terhadap batas
> join (ADR-005 §12).

Pemetaan ke `data.state`:

```
token:{uuid} tidak ada:
    GET settled:{uuid}  ada → { state: CONFIRMED, booking_id }
                              + Set-Cookie: queue_token=; Max-Age=0
      -- NISAN (§10.5). Tanpanya tamu yang me-refresh di celah antara
      -- pembayaran lunas dan email e-tiket tiba memanggil endpoint ini dengan
      -- token mati, dan pemetaan frontend untuk token tak terselesaikan
      -- adalah LAYAR KEDALUWARSA — memberi tahu orang yang baru saja membayar
      -- bahwa sesinya habis. Itu "jangan PHP" dalam bentuk paling murni, dan
      -- ia salah secara faktual (ADR-005 §12).
                        tidak → { state: EXPIRED }

token:{uuid} ada:
    booking := SELECT status, booking_expires_at, payment_expires_at,
                      payment_ref, id
                 FROM bookings WHERE queue_token = $uuid     -- READER cukup
      tidak ada baris     → { state: QUEUED, ticket_no, now_serving }
      GRANTED             → { state: GRANTED, grant_token, booking_expires_at,
                              server_time }
      SELECTING           → { state: SELECTING, grant_token, booking_id,
                              booking_expires_at, server_time }
      HELD                → { state: SELECTING, ... }   -- kontrak §3.3
                              -- menggabungkan SELECTING/HELD ke satu tujuan
                              -- halaman (Detail Ticket); HELD tanpa charge
                              -- belum punya instruksi pembayaran untuk
                              -- dirender
      AWAITING_PAYMENT    → { state: AWAITING_PAYMENT, grant_token, booking_id,
                              payment_expires_at, payment_ref, server_time }
      PENDING_CANCELLATION→ { state: AWAITING_PAYMENT, ... }
                              -- grace 30 detik berjalan; tampilkan
                              -- "memverifikasi pembayaran…", JANGAN layar
                              -- kedaluwarsa (kontrak §5.4)
      CONFIRMED           → { state: CONFIRMED, booking_id }
      EXPIRED/LAPSED/
        ABANDONED         → { state: EXPIRED }

grant_token diambil dari grant:{token} — DIVERIFIKASI masih hidup sebelum
dikembalikan. Pada LAPSED, admission-ctl sudah men-DEL-nya (§9.3): EX-nya 1080
sementara TTL booking 300, jadi tanpa DEL itu grant mati masih lolos verifikasi
~780 detik lagi (ADR-005 §6).
```

> **Tenggat dapat BERGESER.** Saat admission di-pause, server menambahkan
> `PAUSE_TTL_GRACE_SECONDS` ke `booking_expires_at` dan `payment_expires_at` setiap
> booking yang masih berjalan (§9.4). Endpoint ini karena itu selalu membaca tenggat dari
> Aurora, tidak pernah menghitungnya dari `granted_at` + konstanta.

### 7.4 Usecase: `GetServing` — polling fallback

```go
func (u *QueueUsecase) GetServing(ctx context.Context, eventID string) (dto.ServingResponse, error)
//   now_serving   := HGET queue:{ev} now_serving
//   rate_per_min  := HGET queue:{ev} rate_per_min
//   paused        := EXISTS pause:{ev}
//   server_time   := now()
//   poll_after_ms := valueobject.PollInterval(position)
```

```go
// domain/valueobject/poll_interval.go
//
// Profil requirements.md §5, ditentukan SERVER supaya klien tidak menanam
// interval sendiri (kontrak §3.4):
//   position > 1000  → 10000 ms
//   101–1000         →  5000 ms
//   11–100           →  2000 ms
//   1–10             →  1000 ms
func PollInterval(position int64) int
```

> Endpoint ini **bukan** transport utama. Ia untuk hidrasi awal sebelum stream terbuka,
> jaringan yang memblokir `text/event-stream`, dan degraded mode saat tier SSE tidak sehat
> (ADR-005 §13). Posisi dihitung klien dari `ticket_no` yang sudah dipegangnya; endpoint
> ini tidak menerima token, sehingga `position` untuk `poll_after_ms` diambil dari query
> `?position=` bila dikirim, dan default 1000 ms bila tidak.

### 7.5 Config

```go
type Config struct {
    App     App
    Log     Log
    DB      DB         // READER — kecuali jalur bypass FR-05, yang butuh WRITER
    Redis   Redis
    Captcha CaptchaCfg // {Provider, SecretKey, VerifyTimeoutMs: 800, FailOpen: true}
    Auth    AuthCfg    // {JWTSecret} — verifikasi Bearer di /me dan /join
    Queue   QueueCfg   // {Capacity: 300000, SaleWindowSeconds, GrantTokenTTLSeconds: 1080,
                       //  BookingTTLSeconds: 300, MaxConcurrentSlots,
                       //  JoinRateLimit: 20, JoinRateWindowSeconds: 300,
                       //  GrantInsertMaxRetries: 3, DevicePepper}
}
```

> **Dua endpoint DB**: `/me` dan `/serving` memakai reader; jalur bypass FR-05 di `/join`
> memakai writer karena ia meng-INSERT baris GRANTED. Config membawa keduanya
> (`DB.ReaderURL`, `DB.WriterURL`) dan `init.go` membuka dua pool.

---

## 8. `services/sse-server`

**Tipe compute**: ECS Fargate — long-lived, **bukan** Lambda
**Endpoint**: `GET /events/{event_id}/queue/stream`, `GET /health`
**Target**: 300.000 koneksi concurrent, max 60 task, 2 vCPU / 4 GB per task
**Deploy**: ECR + ECS service update — **bukan** SAM (§17.3)

| Kandidat | Constraint | Dampak |
|---|---|---|
| API Gateway HTTP | timeout 29 detik (hard) | SSE putus tiap 29 detik |
| Lambda | maks 15 menit | tidak cukup untuk antrian 30–60 menit |
| App Runner | `request_timeout` maks 120 detik | didiskualifikasi |

### 8.1 Struktur direktori

```
services/sse-server/
  cmd/server/main.go          # TIDAK ADA cmd/lambda
  internal/
    config/config.go, load.go
    delivery/
      server.go               # http.Server + SIGTERM handler
      server_test.go
    sse/
      application/
        manager.go            # connection registry + fan-out ter-chunk
        manager_test.go
        stale_detector.go     # keheningan sse-broadcast → status{stale}
        heartbeat_metric.go   # CtlHeartbeatAge → EMF
      infrastructure/
        http/
          stream.go           # handler GET …/stream
          health.go
          router.go
        redis/
          init.go
          token.go            # HGETALL token:{uuid}
          queue_state.go      # GET state, now_serving, rate_per_min, pause
          pubsub.go           # Subscribe 3 channel + reconnect backoff
        metrics/
          emf.go              # CloudWatch EMF ke stdout
      domain/
        conn.go               # satu koneksi + slowCount
```

> Payload dan nama event ada di `pkg/sseevent` (§3.8), bukan di
> `domain/event/sse_event.go` seperti v1.0 — admission-ctl dan payment-api mem-publish
> bentuk yang sama.

### 8.2 Connection Manager

```go
// application/manager.go
type Manager struct {
    mu      sync.RWMutex
    byEvent map[string]map[*domain.Conn]struct{} // eventID → set koneksi
    byToken map[string]*domain.Conn              // queue_token → koneksi
    lastBroadcastAt map[string]time.Time         // eventID → waktu; untuk §8.5, §8.6
}

func (m *Manager) Register(eventID, token string, c *domain.Conn)
func (m *Manager) Deregister(eventID, token string, c *domain.Conn) // WAJIB kedua map
func (m *Manager) FanOut(eventID string, frame []byte)              // sse-broadcast
func (m *Manager) SendToToken(token, event, data string) bool       // sse-grant
func (m *Manager) CloseEvent(eventID string)                        // sse-sold-out
func (m *Manager) BroadcastAge(eventID string) time.Duration        // §8.6
```

**Fan-out ter-chunk dengan backpressure** (ADR §11):

```go
// 500 koneksi per chunk, lalu yield. Tanpa chunking satu fan-out ke 7.000
// koneksi memblokir goroutine itu melewati tick berikutnya.
//
// Penulisan yang gagal (buffer socket penuh) dihitung: tiga kali berturut-turut
// dan koneksi itu ditutup. Satu klien lambat tidak boleh menahan 6.999 lainnya.
func (m *Manager) FanOut(eventID string, frame []byte) {
    // ... batch 500 → runtime.Gosched()
    //     !conn.Write(frame) → metrics.Inc("SSEWriteBackpressure")
    //                          conn.slowCount++ > 3 → conn.Close()
    //     berhasil           → conn.slowCount = 0
}
```

### 8.3 Stream handler

```
Step -1. RATE LIMIT koneksi:                               -- BARU di v2.0
    ratelimit.Allow(rediskeys.SSERateLimit(ip), SSE_CONN_RATE_LIMIT, 5.0)
      → 429
    -- ADR-005 §4 menaruh rl:sse:{ip} (5 conn/detik) di gerbang ALB sebagai
    -- pengganti WAF. v1.0 mengatribusikannya ke queue-service, yang tidak
    -- melayani endpoint ini — sehingga tidak ada yang menegakkannya.

Step 0. state := GET queue:{ev}:state
    CLOSED    → 410 Gone, JANGAN buka stream
                emit SoldOutStreamRejections
    SCHEDULED → 409, sale belum dibuka
    OPEN      → lanjut
    SOLD_OUT  → LANJUT, buka stream seperti biasa
                -- stok dapat KEMBALI dari booking yang kedaluwarsa; klien
                -- sengaja dipertahankan (ADR-005 §4, §6)
    -- Status non-200 membuat EventSource berhenti PERMANEN (HTML Living
    -- Standard), mencegah badai reconnect setelah sale benar-benar habis.
    -- Itu sebabnya 410 dipakai di CLOSED dan BUKAN di SOLD_OUT.

Step 1. token := Cookie queue_token (web) | X-Queue-Token (mobile)
    HGETALL token:{uuid} kosong → 401
    -- Authorization: Bearer TIDAK dipakai di sini. EventSource tidak dapat
    -- menyetel header, sehingga web harus memakai cookie; slot Bearer milik
    -- token sesi akun saja (kontrak §3.2, §9).
    -- Token tidak pernah lewat query string: ia akan tercatat di access log
    -- ALB, CloudWatch, dan history browser.

Step 2. Headers:
    Content-Type: text/event-stream
    Cache-Control: no-cache
    Connection: keep-alive
    X-Accel-Buffering: no
    (http.Flusher di-assert di sini; nil → 500)

Step 3. SNAPSHOT AWAL — dikirim SEBELUM Register:
    id: {now_serving}
    data: {"nowServing":…,"state":…,"ratePerMin":…,"paused":…,"serverTime":…}
    Flush()
    -- FR-10: posisi utuh segera setelah refresh. Tanpa ini posisi kosong
    -- sampai tick berikutnya.
    -- Sebelum Register, supaya broadcast yang datang di antaranya tidak
    -- mendahului snapshot dan membuat klien melihat nowServing turun.

Step 4. manager.Register(eventID, token, conn)

Step 5. Keepalive: goroutine menulis ": keepalive\n\n" tiap 15 detik
    -- Komentar SSE: tidak memicu handler klien, 13 byte. Event JSON tiap 30
    -- detik berukuran ~6× lebih besar dan tetap berisiko kena idle timeout
    -- ALB (default 60s, dikonfigurasi 3600s) (kontrak §3.2).

Step 6. Blokir sampai r.Context().Done() — klien disconnect atau shutdown

Step 7. manager.Deregister(eventID, token, conn)   -- WAJIB kedua map
```

### 8.4 Subscriber Redis Pub/Sub

Satu goroutine per channel, reconnect dengan exponential backoff 100ms → 1s:

```
sse-broadcast → manager.lastBroadcastAt[eventId] = now()
                frame := "id: {nowServing}\ndata: {...}\n\n"
                manager.FanOut(eventId, frame)
sse-grant     → manager.SendToToken(token, "grant", "{}")
                -- Payload KOSONG: token tidak dikirim lewat SSE. Klien
                -- merespons dengan GET …/queue/me untuk mengambil grant_token
                -- (kontrak §3.2).
sse-sold-out  → manager.FanOut(eventId, "event: sold-out\ndata: {...}\n\n")
                lalu manager.CloseEvent(eventId)
                -- Klien WAJIB es.close(); klien lama yang tetap reconnect
                -- menerima 410 pada Step 0 dan berhenti.
```

**Redis terputus** → kirim `event: status` `{"stale":true,"reason":"redis_disconnected"}`
ke semua koneksi aktif.
**Redis kembali** → `GET now_serving` terbaru → re-broadcast + `{"stale":false}`.

### 8.5 Stale detector — keheningan `sse-broadcast`

Tiap **1 detik** (ADR §11; v1.0 memakai 5):

```
age      := now − manager.lastBroadcastAt[ev]
           -- inisialisasi: waktu task mulai, BUKAN nol
expected := GET queue:{ev}:state ∈ {OPEN, SOLD_OUT}    -- cache ~1 detik

expected && age > SSE_STALE_AFTER_SECONDS (30)
    → status {stale:true, reason:"no_admission_updates"}
sebelumnya stale, sekarang tidak
    → status {stale:false}
```

Dikirim **hanya pada transisi**, tidak diulang. Nol trafik Redis tambahan —
`lastBroadcastAt` adalah efek samping dari subscription yang sudah dipegang.

`expected` ada supaya keheningan yang **sah** tidak dibaca sebagai kegagalan: pada
`SCHEDULED` dan `CLOSED` tidak ada broadcast yang diharapkan. Bila Redis hilang sehingga
`expected` tak terbaca, pertahankan nilai terakhir dan biarkan jalur kehilangan-Redis
(§8.4) yang memiliki `stale` — keduanya tidak pernah bersaing.

> **Yang basi adalah DATA, bukan koneksinya.** Pada `no_admission_updates` koneksinya
> justru sehat — `: keepalive` tetap datang tiap 15 detik — tetapi `nowServing` berhenti
> maju. Itu sebabnya keheningan tidak dapat dideteksi dari keepalive (kontrak §3.2).

### 8.6 `CtlHeartbeatAge` — metrik yang v1.0 lewatkan

ADR §11 dan §17 menyatakan ini **diputuskan dan ditutup**. Diemisi **sse-server**, bukan
`admission-ctl`: proses yang mati tidak dapat melaporkan umurnya sendiri, dan sse-server
sudah memegang angkanya.

```
tiap 10 detik, per event yang punya subscriber:
    EMF: namespace RusuhBooking/Queue
         metric  CtlHeartbeatAge  = now − lastBroadcastAt[ev]  (detik)
         dimensi EventId, TaskId
```

Statistiknya yang menentukan arti, dan itu **dua insiden berbeda**:

| Statistik lintas task | > 30 detik berarti | Tindakan |
|---|---|---|
| **Minimum** | tidak ada task yang menerima → `admission-ctl` memang mati | **page** |
| **Maximum** | subscription satu task putus → hanya klien task itu | ticket |

Ambang 30 detik = tiga heartbeat terlewat pada `BROADCAST_HEARTBEAT_SECONDS=10`. Ia juga
menyerap handover leader (≤ 5,5 detik) dengan nyaman, sehingga handover normal tidak
pernah mencapai indikator customer.

> **Satu kasus tidak punya pemulihan otomatis sama sekali** — leader yang masih
> memperpanjang lease-nya di satu thread timer sementara tick loopnya macet. Untuk itu
> alarm ini adalah **keseluruhan** mitigasinya (ADR §11). Alarm-nya didefinisikan di infra,
> tetapi **emisinya ada di repo ini** dan itu yang dibangun di sini.

### 8.7 SIGTERM

```go
// On SIGTERM:
// 1. Server berhenti menerima koneksi baru (ALB deregistration delay 60 detik
//    memberi waktu target di-drain dari rotasi).
// 2. Untuk setiap koneksi:
//      write "retry: {jitter(1000,15000)}\n"
//      write "event: reconnect\ndata: {}\n\n"
//      close
//    -- Di SINI reconnect otomatis EventSource JUSTRU DIINGINKAN; jitter
//    -- hanya menyebarkan waktunya. Bandingkan dengan sold-out, yang harus
//    -- menghentikan reconnect sama sekali (ADR-005 §4, §11).
// 3. time.Sleep(5 * time.Second)   -- biarkan write ter-flush
// 4. os.Exit(0)
```

> v1.0 menulis "tutup semua koneksi setelah slow-start 30s". ALB **slow start** mengatur
> seberapa cepat target **baru** menerima trafik; ia tidak berlaku pada shutdown. Angka
> yang relevan saat shutdown adalah deregistration delay (60 detik) untuk berhenti
> menerima, dan jeda flush 5 detik sebelum exit (ADR §11).

### 8.8 Config

```go
type Config struct {
    App   App       // {Port, Env, TaskID}  -- TaskID dari ECS metadata, untuk dimensi EMF
    Log   Log
    Redis Redis
    SSE   SSECfg    // {StaleAfterSeconds: 30, KeepaliveSeconds: 15,
                    //  HeartbeatMetricSeconds: 10, ShutdownFlushSeconds: 5,
                    //  FanOutChunkSize: 500, SlowConnThreshold: 3,
                    //  ConnRateLimit: 5}
}
```

---

## 9. `services/admission-ctl`

**Tipe compute**: ECS Fargate — loop long-lived, bukan HTTP
**Binary**: background process; HTTP hanya `/health` minimal untuk ALB/ECS
**Desired count**: **2** — 1 leader + 1 standby, dengan leader lease
**Tick**: 500 ms
**Deploy**: ECR + ECS service update (§17.3)

> **Jumlah instance bukan jawabannya.** Task ketiga menambah biaya tanpa menambah
> ketersediaan: hanya satu yang pernah bertindak. Yang dibeli oleh task kedua adalah
> failover ≤ TTL lease + satu tick ≈ **5,5 detik**, turun dari 30–90 detik (deteksi ECS +
> start task + health check) (ADR §11).

### 9.1 Struktur direktori

```
services/admission-ctl/
  cmd/server/main.go          # TIDAK ADA cmd/lambda
  internal/
    config/config.go, load.go
    delivery/
      runner.go               # tick loop + SIGTERM
      health.go
      runner_test.go
    admission/
      application/
        tick.go               # satu tick: langkah -1 … 9
        leader.go             # ambil / perpanjang / lepas + ctl:epoch
        pause.go              # langkah 0 + TTL grace
        sweep.go              # langkah 1 — lima status, dengan rescore
        advance.go            # langkah 2
        broadcast.go          # langkah 3
        reconcile_slots.go    # langkah 4
        close_sale.go         # langkah 5
        qpersist_drain.go     # langkah 6
        reconcile_stock.go    # langkah 7 + nik_claims
        charge_reconciler.go  # langkah 8  ← BARU di v2.0
      infrastructure/
        postgres/
          db.go, init.go
          booking.go          # verifikasi status, INSERT GRANTED, UPDATE terminal
          booking_item.go     # baca item untuk DEL nik:used + DELETE nik_claims
          queue_entry.go      # batch INSERT queue_entries
          category.go         # ticket_categories.capacity (§2.2)
          queue_state.go      # event_queue_state: daftar event aktif + transisi
          nik_claim.go        # baca untuk rekonsiliasi; hapus saat expiry
          outbox.go           # pkg/outbox.Enqueue
        redis/
          init.go
          leader.go           # SET NX + leader_renew.lua + leader_release.lua
          admission.go        # EVALSHA advance_now_serving.lua
          expiry.go           # ZRANGEBYSCORE / ZADD / ZREM
          stock.go            # slots:open, stock, nik:used, adjust_if_unchanged
          credential.go       # DEL grant / token / acct ; SET settled
          pubsub.go           # PUBLISH 3 channel (pkg/sseevent)
          qpersist.go         # LRANGE / LTRIM
          breaker.go          # pkg/breaker ×2, BACA saja
        midtrans/
          client.go           # expire + status query read-only
        metrics/emf.go
      domain/
        entity/tick_result.go, active_event.go
      port/outbound/
        booking.go, queue_entry.go, category.go, queue_state.go, nik_claim.go
        admission.go, expiry.go, stock.go, credential.go, publisher.go
        qpersist.go, gateway.go
```

### 9.2 Leader lease dengan fencing token

v1.0 memberi `SET ctl:leader {task_id} NX EX 5` dan tidak lebih. ADR §11 menyebut
perpanjangan dan pelepasan **tanpa syarat** sebagai "bug, bukan penyederhanaan":

> `EXPIRE` buta memperpanjang lease yang mungkin sudah bukan milik kita; `DEL` buta
> menghapus lease task lain dan **mengundang pemimpin kedua secara aktif**.

```lua
-- pkg/redisscript/scripts/leader_renew.lua
-- KEYS[1]=ctl:leader  ARGV[1]=task_id  ARGV[2]=lease_ms
if redis.call('GET', KEYS[1]) == ARGV[1] then
  redis.call('PEXPIRE', KEYS[1], ARGV[2])
  return 1
end
return 0                                    -- kepemimpinan hilang
```

```lua
-- pkg/redisscript/scripts/leader_release.lua
-- KEYS[1]=ctl:leader  ARGV[1]=task_id
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('DEL', KEYS[1])
end
return 0
```

```
AMBIL (bila belum punya):
  SET ctl:leader {task_id} NX EX CTL_LEADER_LEASE_SECONDS (5)
  berhasil → myEpoch = INCR ctl:epoch       -- fencing token, monoton
             emit CtlLeaderHandovers

PERPANJANG (tiap tick): EVALSHA leader_renew  → 0 berarti kehilangan kepemimpinan
LEPAS (SIGTERM):        EVALSHA leader_release

bila BUKAN leader → lewati seluruh tick (standby diam, tetap hangat)
```

**Lease ganda tidak dicegah — ia dibuat tidak berbahaya.** Dua jalan menuju dua pemegang,
dan Redis tidak dapat menutup satu pun: holder yang tertahan (GC, query sweep lambat, CPU
throttle) dan failover ElastiCache yang replikasinya asinkron. Karena itu:

```
sebelum tiap langkah NON-IDEMPOTEN:
    masih pemilik?   GET ctl:leader == {task_id}
    sisa TTL cukup?  PTTL ctl:leader > anggaran langkah ini
    bawa myEpoch ke dalam write yang dijaga; epoch basi → tolak,
      emit StaleLeaderRejected
```

Hanya **dua** operasi yang benar-benar rapuh bila dijalankan ganda, dan keduanya punya
pengaman mandiri di sumber dayanya:

| Operasi | Aman ganda? | Pengaman |
|---|---|---|
| SWEEP | ✅ sudah | `UPDATE … WHERE id=$1 AND status=$row.status` adalah CAS; yang kalah mendapat 0 baris dan berhenti sebelum efek sampingnya |
| ADVANCE | ✅ sudah | Lua atomik; `deficit` dihitung dari `slots:open` yang dibaca ulang |
| `SOLD_OUT → CLOSED` | ✅ praktis | `SET state CLOSED` idempoten; `sse-sold-out` ganda tidak berbahaya |
| Drain qpersist | ✅ | `ON CONFLICT DO NOTHING` |
| **Rekonsiliasi stok** | ❌ | `adjust_if_unchanged.lua` — compare-and-adjust (§9.8) |
| **Pause TTL grace** | ❌ | `SET pause:{ev}:grace_applied NX` (§9.4) |

> **Urutan klaim yang benar**: lease menjawab *siapa yang sebaiknya bekerja* —
> ketersediaan. Pengaman di sumber daya menjawab *apa yang terjadi bila dua tetap bekerja*
> — korektifitas. Bila lease dihapus besok, sistem tetap benar (hanya boros). Bila
> pengamannya dihapus dan hanya lease yang tinggal, sistem menjadi **mungkin** benar — dan
> "mungkin" bukan sifat yang pantas dipegang oleh stok tiket (ADR §11).

### 9.3 Satu tick

```
-1. LEADER LEASE (§9.2). Bukan leader → return.

 0. PAUSE CHECK + TTL GRACE (§9.4)

 1. SWEEP — per event aktif (§9.5). TIDAK di-pause.

 2. ADVANCE — per event aktif (§9.6). DI-PAUSE: bila paused, LEWATI SELURUHNYA.

 3. BROADCAST (§9.7). TIDAK di-pause.

 4. REKONSILIASI slots:open — tiap tick ke-20 (≈10 detik) (§9.7). TIDAK di-pause.

 5. SOLD_OUT → CLOSED (§9.7).

 6. DRAIN qpersist — tiap tick, TERAKHIR di antara langkah per-event, beranggaran
    QPERSIST_DRAIN_BUDGET_MS (§9.7).

 7. REKONSILIASI stok + nik_claims — tiap tick ke-120 (≈60 detik) (§9.8).

 8. CHARGE RECONCILER — tiap tick ke-600 (≈5 menit) (§9.9).

 9. emit tick metrics
```

Daftar event aktif: `SELECT event_id FROM event_queue_state WHERE state IN ('OPEN','SOLD_OUT')`
(indeks `event_queue_state_active_idx`, §4.1), di-cache 10 detik.

**Apa yang `paused` hentikan dan apa yang tetap jalan** (ADR §6):

| Saat `paused = true` | Jalan? | Alasan |
|---|---|---|
| ADVANCE — `INCRBY now_serving` + grant baru | ❌ | tidak ada yang dapat menyelesaikan pembelian |
| Pengurasan `regrant` | ❌ | ia juga memasukkan orang ke funnel yang sama |
| SWEEP — expiry booking dan pembayaran | ✅ | stok yang tertahan harus tetap dilepas |
| Broadcast + heartbeat | ✅ | customer harus melihat bannernya dan posisinya |
| Rekonsiliasi `slots:open` | ✅ | drift tidak berhenti karena admission berhenti |

Karena pengurasan `regrant` ada **di dalam** `advance_now_serving.lua`, gerbangnya
diletakkan di tick — EVALSHA dilewatkan sepenuhnya — bukan sebagai ARGV baru.

### 9.4 Langkah 0 — pause dan `PAUSE_TTL_GRACE_SECONDS`

v1.0 hanya memeriksa breaker dan mem-publish. Seluruh mekanisme grace hilang, dan ia salah
satu dari dua operasi non-idempoten dalam desain ini.

```
paused := breaker.New(redis, rediskeys.ServicePaymentGateway, …).State() == OPEN
       || breaker.New(redis, rediskeys.ServiceDukcapil, …).State() == OPEN
   -- Kedua dependensi diperlakukan SAMA: pause penuh. Antrian berhenti maju
   -- untuk semua orang tanpa pengecualian, dan tidak ada yang kehilangan
   -- posisinya (ADR-005 §8).
   -- "breaker OPEN" berarti dependensinya SEDANG TIDAK DAPAT DIPAKAI.

bila paused dan sebelumnya tidak (TRANSISI MASUK), per event aktif:
    SET pause:{ev}:grace_applied 1 NX EX SALE_WINDOW_SECONDS
      gagal → grace sudah diberikan untuk pause ini; LEWATI
      -- Guard ini BERDIRI SENDIRI, terlepas dari leader lease: dua task yang
      -- sama-sama mendeteksi transisi akan memberi +120 detik, bukan +60
      -- (ADR-005 §6, §11).
    UPDATE bookings
       SET booking_expires_at = booking_expires_at + PAUSE_TTL_GRACE_SECONDS,
           payment_expires_at = payment_expires_at + PAUSE_TTL_GRACE_SECONDS
     WHERE event_id = $ev
       AND status IN ('GRANTED','SELECTING','HELD',
                      'AWAITING_PAYMENT','PENDING_CANCELLATION')
     RETURNING id, status, booking_expires_at, payment_expires_at
    untuk tiap baris: ZADD expiries:{ev} <tenggat baru sesuai status> id
    emit PauseTtlGrantedSeconds, PauseTransitions

bila paused berubah (masuk ATAU keluar): SET / DEL pause:{ev}
```

> **Mengapa saat MULAI, bukan saat selesai.** Sweep tetap berjalan selama pause. Customer
> yang tenggatnya lewat **di tengah** pause sudah di-expire sebelum resume tiba, sehingga
> perpanjangan pada saat resume datang terlambat bagi justru orang-orang yang paling
> membutuhkannya. Menggeser di muka membuat sweep berjalan apa adanya tanpa perlakuan
> khusus (ADR §6).

> **Ruang TTL token cukup**: perjalanan maksimum 930 + 60 grace = 990, masih di bawah
> `GRANT_TOKEN_TTL_SECONDS` (1080). Margin menyusut dari 150 ke 90 detik.

> **Failover Redis/Aurora BUKAN peristiwa breaker**, sehingga customer yang `SELECTING`
> saat failover 45 detik kehilangan 45 detik tanpa penggantian. ADR §12 mencatat mitigasi
> termurahnya (perlakukan N tick gagal berturut-turut sebagai transisi `paused`) sebagai
> **belum diputuskan** — tidak dibangun di v2.0, masuk §21.

### 9.5 Langkah 1 — SWEEP: lima status, dengan rescore

Ini bagian yang paling berubah dari v1.0, dan perubahannya mengembalikan BR-05.

v1.0 menangani hanya `GRANTED`/`SELECTING` dan **meng-`ZREM`** entri yang tidak cocok.
Akibatnya: setiap booking `HELD` dan `AWAITING_PAYMENT` dibuang dari penjadwal dan
**tidak pernah di-expire**, sehingga TTL pembayaran 10 menit tidak ditegakkan sama sekali;
dan entri yang skornya basi hilang permanen alih-alih dikoreksi.

ADR §7 memperlakukan ZSET sebagai **petunjuk** dan Aurora sebagai **kebenaran**:

```
untuk booking_id dalam ZRANGEBYSCORE expiries:{ev} 0 now:

  row = SELECT status, granted_at, booking_expires_at, payment_expires_at
          FROM bookings WHERE id = $1                      -- WRITER
        -- Lag replika terbaca sebagai tenggat yang belum lewat → rescore
        -- palsu → booking hidup lebih lama dari seharusnya. Karena itu WRITER.

  deadline = CASE row.status
               WHEN 'GRANTED'              THEN row.booking_expires_at
               WHEN 'SELECTING'            THEN row.booking_expires_at
               WHEN 'HELD'                 THEN row.payment_expires_at
               WHEN 'AWAITING_PAYMENT'     THEN row.payment_expires_at
               WHEN 'PENDING_CANCELLATION' THEN row.payment_expires_at
                                              + PAYMENT_GRACE_SECONDS
               ELSE NULL                                   -- sudah terminal
             END

  deadline IS NULL → ZREM ; continue                       -- sudah terminal

  deadline > now()  → ZADD expiries:{ev} deadline booking_id   -- RESCORE
                      emit ExpiryRescored ; continue
                      -- JANGAN ZREM dan JANGAN expire. ZSET berada di luar
                      -- transaksi Aurora, sehingga selalu ada jendela crash:
                      -- booking yang sudah AWAITING_PAYMENT masih membawa
                      -- skor TTL booking, lalu akan di-expire beberapa puluh
                      -- detik setelah customer menerima nomor VA — tepat
                      -- ketika ia sedang membayar di ATM (ADR-005 §7).
                      -- v1.0 meng-ZREM di sini: booking itu lalu tidak pernah
                      -- di-expire sama sekali, dan stok + slot bocor permanen.

  -- deadline benar-benar terlewat:
  UPDATE bookings SET status = <terminal>
    WHERE id = $1 AND status = row.status                  -- CAS, guarded
  0 baris → ZREM ; continue                                -- pihak lain mendahului

  switch row.status:

    GRANTED → LAPSED                                       -- TERMINAL, FINAL
        DECRBY slots:open:{ev} 1
        DEL grant:{token}
          -- Kredensialnya dicabut DI SINI: EX-nya 1080 sementara TTL booking
          -- 300, jadi tanpa DEL ini grant mati masih lolos verifikasi ~780
          -- detik lagi (ADR-005 §6).
        DEL acct:{ev}:{account_id}                          -- boleh antre lagi
        DEL token:{uuid}                                    -- terminal-kegagalan
        ZREM queue:{ev}:by_ticket {uuid}
        -- TIDAK ADA SADD regrant. LAPSED sudah mendapat gilirannya dan
        -- membakar seluruh 300 detiknya. regrant hanya untuk kegagalan
        -- SERVER (ADR-005 §6).
        INSERT INTO outbox 'booking.expired'   -- hanya bila contact_email ada;
                                               -- GRANTED umumnya belum punya
        emit ExpiryRate{stage=lapsed}

    SELECTING → EXPIRED                                     -- TTL booking
        DECRBY slots:open:{ev} 1
        DEL grant:{token} ; DEL acct ; DEL token:{uuid}
        ZREM by_ticket
        -- TIDAK ada INCR stock: pada jalur normal belum ada stok yang diambil
        -- (hold terjadi saat tekan Pay). Bila Lua sempat berhasil lalu Aurora
        -- rollback, kompensasi §5.4 langkah 10 yang mengembalikannya, dan
        -- rekonsiliasi stok (§9.8) adalah jaring pengamannya.
        INSERT INTO outbox 'booking.expired'
        emit ExpiryRate{stage=selecting}

    HELD → EXPIRED                                          -- TTL pembayaran
        DECRBY slots:open:{ev} 1
        INCRBY stock:{ev}:{cat} qty   per kategori           -- lepas stok
        DELETE FROM nik_claims WHERE booking_id = $1         -- ← v1.0 LEWATKAN
          -- Tanpa ini NIK terkunci SELAMANYA meski booking-nya batal:
          -- UNIQUE (event_id, nik_hmac) adalah penegak BR-02, dan ia tidak
          -- membedakan klaim hidup dari klaim mati (ADR-005 §7, §8).
        DEL nik:used:{ev}:{hmac} per item
        DEL grant:{token} ; DEL acct ; DEL token:{uuid} ; ZREM by_ticket
        INSERT INTO outbox 'booking.expired'
        emit ExpiryRate{stage=held}

    AWAITING_PAYMENT → PENDING_CANCELLATION                 -- ← v1.0 LEWATKAN
        -- TIDAK melepas stok. Grace PAYMENT_GRACE_SECONDS (30) untuk callback
        -- yang sedang di jalan: customer yang mentransfer pada menit 9:59
        -- menghasilkan callback yang mungkin baru tiba 10:00:15 — pembayaran
        -- itu SAH dan tidak boleh hangus karena jaringan Midtrans
        -- (ADR-005 §7).
        -- Melepas stok lalu menerima callback sukses 10 detik kemudian berarti
        -- MENJUAL ULANG KURSI YANG SUDAH DIBAYAR.
        ZADD expiries:{ev} (payment_expires_at + PAYMENT_GRACE_SECONDS) id
        TIDAK DECR slots:open
        emit ExpiryRate{stage=grace_entered}

    PENDING_CANCELLATION → EXPIRED                          -- ← v1.0 LEWATKAN
        idem HELD (DECR slots, INCR stock, DELETE nik_claims, DEL nik:used,
                   DEL kredensial, outbox 'booking.expired')   -- FR-28
        POST /v2/{order_id}/expire → Midtrans
          -- VA/QR berhenti dapat dibayar. custom_expiry saat charge sudah
          -- menutupnya; ini menutup selisih jam antara Midtrans dan kita.
          -- Gagal → best-effort, TIDAK membatalkan expiry;
          -- emit MidtransExpireFailed (ADR-005 §7).
        emit ExpiryRate{stage=cancelled}

  ZREM expiries:{ev} booking_id
```

Setelah seluruh sweep satu event: `EVALSHA advance_now_serving.lua` + PUBLISH, supaya slot
yang baru dibebaskan langsung terpakai.

> **TTL booking berhenti mengatur begitu status menjadi `HELD`.** Keduanya tersimpan di
> baris yang sama, tetapi sweep memilih berdasarkan status — tidak pernah keduanya —
> sehingga tidak ada expiry ganda dan tidak ada tumpang tindih (ADR §7).

> **Biaya**: satu `SELECT` per entri yang jatuh tempo. Pada laju expiry puncak (~500/menit)
> itu ~8 query/detik terhadap writer — dapat diabaikan, dan hanya untuk entri yang
> benar-benar jatuh tempo, bukan seluruh ZSET.

### 9.6 Langkah 2 — ADVANCE

Dilewati seluruhnya bila `paused`.

```
per event aktif:
  remaining_stock := SUM(GET stock:{ev}:{cat}) atas seluruh kategori
    -- TOTAL, bukan per kategori: admission-ctl tidak tahu kategori apa yang
    -- akan dipilih customer yang belum masuk (ADR-005 §6).
  res := redisscript.Run("advance_now_serving",
           [now_serving, slots:open, state, regrant, by_ticket,
            sold_out_at, resume_flag],
           MAX_CONCURRENT_SLOTS, remaining_stock, now_unix)
  -- res = {from, to, state, tokens}

  untuk tiap uuid dalam res.tokens:
    grantToken := "gt_" + random
    INSERT INTO bookings (id, event_id, queue_token, grant_token, account_id,
                          status='GRANTED', granted_at=now(),
                          booking_expires_at=now()+BOOKING_TTL_SECONDS)
      -- account_id DARI token:{uuid}, bukan dari klien. Keterlibatan tahu
      -- identitasnya sejak join, ketika sesi masih pasti segar; mengambilnya
      -- dari sana membuat atribusi tidak bergantung pada jam klien sama
      -- sekali (ADR-005 §6).
      -- Baris ini juga PRASYARAT REKONSILIASI: tanpanya COUNT(*) tidak
      -- melihat slot granted-unclaimed, sehingga rekonsiliasi justru
      -- menyebabkan over-admission.
    batch INSERT, retry hingga GRANT_INSERT_MAX_RETRIES (3)
      tetap gagal:
        DECRBY slots:open:{ev} 1
        SADD regrant:{ev} {uuid}
        JANGAN publish sse-grant
        emit GrantInsertFailures
        -- Token ini BELUM pernah dilayani; ia tetap di antrian.
        continue

    HSET grant:{token} token, event_id, queue_token, account_id, device_hash
      EX GRANT_TOKEN_TTL_SECONDS (1080)
      -- device_hash disalin dari token:{uuid}; pkg/grantauth (§3.5)
      -- membacanya. v1.0 tidak pernah menyatakan siapa yang mengisi field
      -- ini, sehingga Grant.DeviceHash-nya tidak punya penulis.
    ZADD expiries:{ev} booking_expires_at booking_id

  SETELAH SELURUH INSERT BERHASIL: PUBLISH sse-grant per token
    -- URUTAN WAJIB: INSERT dulu, PUBLISH kemudian. Mem-publish lebih dahulu
    -- berarti memberi tahu customer gilirannya tiba untuk slot yang kemudian
    -- dikompensasi — pelanggaran NFR-02 "Jangan PHP" paling langsung
    -- (ADR-005 §6).

  bila res.state == 'SOLD_OUT' dan sebelumnya OPEN:
    UPDATE event_queue_state SET state='SOLD_OUT', sold_out_at=now()
  bila res.resume_flag disetel (SOLD_OUT → OPEN, stok kembali):
    UPDATE event_queue_state SET state='OPEN', sold_out_at=NULL
    DEL queue:{ev}:resume ; emit SoldOutResumed
```

> **Slot yang terisi dari `regrant` tidak memajukan `now_serving`.** Lua menguras `regrant`
> lebih dulu dan mengurangi `deficit` sebesar jumlah itu. Ini benar — token di `regrant`
> sudah dihitung "dilayani" ketika watermark melewatinya; INSERT-nya yang gagal, bukan
> gilirannya. Konsekuensinya customer yang menunggu kadang melihat slot terbebas tanpa
> posisinya bergerak. **Perlu diketahui tim dukungan sebelum hari-H** (ADR §6).

### 9.7 Langkah 3–6

**Langkah 3 — BROADCAST**

```
PUBLISH sse-broadcast {eventId, nowServing, state, ratePerMin, paused, serverTime}
  bila nowServing berubah, ATAU paused berubah, ATAU BROADCAST_HEARTBEAT_SECONDS
  (10) berlalu
  -- Heartbeat TIDAK BERSYARAT: terbit juga saat paused dan saat SOLD_OUT.
  -- Diam berarti loop mati, bukan "tidak ada kabar" — itu yang membuat
  -- detektor stale (§8.5) dapat dipercaya (ADR-005 §4).

ratePerMin := (nowServing sekarang − nowServing 60 detik lalu), ring buffer
  di memori leader; HSET queue:{ev} rate_per_min <nilai> supaya queue-service
  (§7.4) dan sse-server (§8.3) dapat membacanya.
```

**Langkah 4 — REKONSILIASI `slots:open`**, tiap tick ke-20 (`RECONCILE_EVERY_TICKS=20`)

```
per event aktif:
  C := SELECT COUNT(*) FROM bookings WHERE event_id=$1
        AND status IN ('GRANTED','SELECTING','HELD','AWAITING_PAYMENT')
       -- WRITER. Lag replika terbaca sebagai drift semu.
  R := GET slots:open:{ev}
  current_drift := R − C ; emit SlotsOpenDrift

  bila |current_drift| >= DRIFT_THRESHOLD (3)
   DAN |last_drift|    >= DRIFT_THRESHOLD:
      INCRBY slots:open:{ev} (−current_drift)
      -- current_drift, BUKAN last_drift (berumur ~10 detik, sudah basi;
      --   last_drift hanya konfirmasi bahwa drift persisten)
      -- INCRBY, BUKAN SET (SET menimpa grant antara pembacaan dan koreksi)
  last_drift := current_drift
```

**Langkah 5 — `SOLD_OUT → CLOSED`**

```
bila state == SOLD_OUT dan now() > sold_out_at + SOLD_OUT_HOLD_SECONDS (930):
  SET queue:{ev}:state CLOSED
  UPDATE event_queue_state SET state='CLOSED', closed_at=now() WHERE event_id=$1
  PUBLISH sse-sold-out {eventId}
  emit SoldOutHoldSeconds
  -- 930 = BOOKING_TTL + PAYMENT_TTL + PAYMENT_GRACE. DITURUNKAN, bukan
  -- ditebak: tidak ada hold yang dapat kedaluwarsa lebih lambat dari satu
  -- perjalanan penuh setelah hold terakhir dibuat (ADR-005 §6).
```

**Langkah 6 — DRAIN `qpersist`**

```
anggaran := QPERSIST_DRAIN_BUDGET_MS (200)
loop sampai anggaran habis atau list kosong:
  batch := LRANGE qpersist:{ev} 0 QPERSIST_BATCH_SIZE-1     -- BACA, JANGAN POP
  bila kosong → break
  BEGIN
    INSERT INTO queue_entries (event_id, queue_token, ticket_no, account_id, joined_at)
      VALUES (…), (…), …
      ON CONFLICT (event_id, queue_token) DO NOTHING
  COMMIT
  LTRIM qpersist:{ev} len(batch) -1        -- BUANG HANYA SETELAH COMMIT
  emit QPersistLag = LLEN, QPersistAgeSeconds = now − joined_at tertua
```

> **Baca-lalu-trim, bukan pop-lalu-tulis.** v1.0 memakai `LPOP`. Pop lebih dulu akan
> **kehilangan batch itu** bila insert gagal. Dengan trim sesudah commit, kegagalan berarti
> batch yang sama dibaca ulang pada tick berikutnya, dan `ON CONFLICT DO NOTHING` membuat
> pengulangan tidak berbahaya — at-least-once dengan sink idempoten, pola yang sama dengan
> outbox (ADR §12).

> **Drain adalah langkah TERAKHIR dan beranggaran waktu.** Tick yang tertahan lebih lama
> dari TTL lease kehilangan kepemimpinan sementara masih bertindak, jadi drain diletakkan
> paling akhir supaya ia dapat dipotong tanpa memengaruhi admission atau sweep. Tertinggal
> sebentar jauh lebih murah daripada kehilangan lease.

> **Panjang buffer ADALAH RPO**, dan ia paling buruk tepat saat badai join: ~5.000
> join/detik masuk vs kapasitas drain ~2.000/detik memuncak di ~180.000 entri, lalu
> terkuras ~90 detik sesudahnya. Jendela ~90 detik **diterima** (keputusan 30 Oktober
> 2026); `COPY` ke tabel temp adalah jalur eskalasi bila benchmark T-14 menunjukkan lag
> jauh melampaui perkiraan.

> **Lag drain tidak punya efek fungsional apa pun.** `queue_entries` bersifat TULIS-SAJA
> pada operasi normal — ia tidak pernah dibaca kecuali saat rebuild. Customer yang barisnya
> belum mendarat tetap punya posisi yang berfungsi penuh: terlihat, dapat dipulihkan, dapat
> di-grant (ADR §12).

### 9.8 Langkah 7 — rekonsiliasi stok dan `nik_claims`

Tiap tick ke-120 (`STOCK_RECONCILE_EVERY_TICKS=120`, ≈60 detik).

```
per kategori per event aktif:
  E := SELECT capacity FROM ticket_categories WHERE id = $cat        -- §2.2
         − SELECT COUNT(*) FROM booking_items bi
             JOIN bookings b ON b.id = bi.booking_id
            WHERE bi.category_id = $cat
              AND b.status IN ('HELD','AWAITING_PAYMENT','CONFIRMED')  -- WRITER
  R := GET stock:{ev}:{cat}

  bila R HILANG (bukan nol):
      SET stock:{ev}:{cat} E              -- SEGERA, tanpa konfirmasi kedua
      emit StockKeyRestored ; continue
      -- Aturan dua-tick ada untuk mencegah koreksi atas drift semu dari lag
      -- replika. Kunci yang HILANG bukan drift — ia menolak setiap hold
      -- sampai ditanam kembali, jadi menunggu 60 detik lagi berarti 60 detik
      -- penjualan mati demi kehati-hatian yang tidak berlaku di sini
      -- (ADR-005 §6).

  current_drift := R − E ; emit StockDrift
  bila |current_drift| >= STOCK_DRIFT_THRESHOLD (2)
   DAN |last_drift|    >= STOCK_DRIFT_THRESHOLD:
      ok := redisscript.Run("adjust_if_unchanged",
              [stock:{ev}:{cat}], R, −current_drift)
      !ok → emit StockAdjustRejected
      -- COMPARE-AND-ADJUST, bukan INCRBY buta: dua eksekusi bersamaan tidak
      -- mengoreksi 2×, bahkan bila leader lease kalah balapan (ADR-005 §11).
  last_drift := current_drift
```

**Rekonsiliasi `nik:used`** — pada tick yang sama, dan v1.0 tidak punya ini sama sekali:

```
A := SELECT nik_hmac FROM nik_claims WHERE event_id = $ev         -- WRITER
untuk hmac dalam A yang TIDAK ada di Redis:
    SET nik:used:{ev}:{hmac} 1 EX NIK_USED_TTL_SECONDS            -- pulihkan
    emit NikClaimRestored
-- Arah sebaliknya (ada di Redis, tidak di nik_claims) TIDAK dihapus otomatis:
-- itu bisa jadi hold yang sedang berjalan dan belum COMMIT. Kunci itu
-- dibatasi TTL-nya sendiri, dan NikOrphanRate dialarmkan bila jumlahnya
-- tidak wajar (ADR-005 §6).
```

> **Inilah alasan `nik_claims` ada.** Tanpanya `nik:used` tidak punya pembanding dan
> kebocoran NIK tidak terlihat di mana pun. ADR §16 mendaftarkannya sebagai mitigasi
> tunggal untuk "Redis flush / failover menghapus `nik:used`".

> **Interval 60 detik, bukan 10.** Query-nya lebih berat (join ke `booking_items` per
> kategori) dan drift stok tidak menyebabkan over-admission seperti drift `slots:open` —
> ia hanya menahan tiket dari peredaran, sehingga koreksi lebih lambat dapat diterima.

### 9.9 Langkah 8 — charge reconciler

Tiap tick ke-600 (≈5 menit). ADR §8 menetapkannya; v1.0 tidak punya.

```
rows := SELECT id, payment_attempts FROM bookings
         WHERE event_id = $ev
           AND payment_ref IS NULL
           AND payment_attempts > 0
           AND status IN ('HELD','AWAITING_PAYMENT')
        -- yaitu charge yang TIMEOUT dan statusnya tidak pernah diketahui
        -- (indeks bookings_orphan_charge_idx, §4.3)

untuk tiap baris:
  order_id := id || ':' || payment_attempts
  res := midtrans.Status(order_id)            -- QUERY READ-ONLY
    settlement / capture+accept → serahkan ke jalur webhook: INSERT
                                   payment_events + proses seperti callback
    deny / cancel / expire / not_found → payment_ref tetap NULL;
                                   customer boleh mencoba lagi
  emit ChargeReconcilerHits

-- RECONCILER TIDAK PERNAH MENGIRIM CHARGE BARU. Payment gateway adalah
-- dependensi WRITE: retry buta bisa double-charge, dan timeout tidak berarti
-- gagal — charge mungkin sudah berhasil (ADR-005 §8).
```

### 9.10 Config

```go
type Config struct {
    App       App            // {Env, TaskID}
    Log       Log
    DB        DB             // WRITER — rekonsiliasi dan sweep wajib writer
    Redis     Redis
    Admission AdmissionCfg   // {TickIntervalMs: 500, MaxConcurrentSlots,
                             //  BookingTTLSeconds: 300, PaymentTTLSeconds: 600,
                             //  PaymentGraceSeconds: 30, SoldOutHoldSeconds: 930,
                             //  GrantTokenTTLSeconds: 1080, NIKUsedTTLSeconds,
                             //  SaleWindowSeconds, PauseTtlGraceSeconds: 60,
                             //  BroadcastHeartbeatSeconds: 10,
                             //  DriftThreshold: 3, ReconcileEveryTicks: 20,
                             //  StockDriftThreshold: 2, StockReconcileEveryTicks: 120,
                             //  ChargeReconcileEveryTicks: 600,
                             //  GrantInsertMaxRetries: 3,
                             //  QPersistBatchSize: 1000, QPersistDrainBudgetMs: 200}
    Leader    LeaderCfg      // {LeaseSeconds: 5}
    Breaker   BreakerCfg     // {FailThreshold, OpenForSeconds} — dipakai BACA saja
    Midtrans  MidtransCfg    // {ServerKey, IsProduction} — expire + status query
}
```

---

## 10. `services/payment-api`

**Tipe compute**: Lambda + API Gateway, Provisioned Concurrency (T-30 menit)
**Endpoint**: `GET /payments/{booking_id}/methods` (FR-20),
`POST /payments/{booking_id}` (FR-20), `POST /payments/{booking_id}/retry`,
`GET /payments/{booking_id}/status` (FR-21), `POST /payments/webhook` (FR-25),
`GET /admin/reconciliation/disputed`
**Gateway**: Midtrans **Core API**; dua metode — `bank_transfer` (BCA/BNI/BRI/CIMB) dan
`gopay`

> **`/payments/webhook` punya resource policy sendiri**: allowlist IP gateway + verifikasi
> HMAC, tanpa CAPTCHA dan tanpa rate limit per-IP. Ia juga satu-satunya endpoint di service
> ini yang **tidak** memakai `pkg/grantauth`.

### 10.1 Struktur direktori

```
services/payment-api/
  cmd/lambda/main.go, cmd/server/main.go
  internal/
    config/config.go, load.go
    delivery/server.go, lambda_proxy_test.go, integration_test.go
    payment/
      application/usecase/
        get_methods.go
        initiate.go
        retry.go
        get_status.go
        webhook.go
        inline_publish.go     # blok inline pasca-COMMIT (§10.5 langkah 3)
        list_disputed.go      # GET /admin/reconciliation/disputed
        init.go
      domain/
        dto/request.go, response.go, webhook.go
        entity/payment.go, method.go
        valueobject/
          signature.go        # SHA512(order_id+status_code+gross_amount+ServerKey)
          order_id.go         # {booking_id}:{payment_attempts}
          transition.go       # transaction_status + fraud_status → transisi
        mapper/payment.go
      infrastructure/
        postgres/
          db.go, init.go
          booking.go          # FOR UPDATE, UPDATE status, payment_disputed
          payment_event.go    # INSERT payment_events
          booking_item.go     # baca item untuk IssueTickets + DEL nik:used
          ticket.go           # INSERT tickets per booking_item
          nik_claim.go        # DELETE pada jalur expire
          outbox.go           # pkg/outbox.Enqueue
        redis/
          init.go
          stock.go            # slots:open, stock
          credential.go       # DEL nik:used / acct / token ; SET settled
          expiry.go           # ZREM expiries
          admission.go        # EVALSHA advance_now_serving.lua (inline)
          pubsub.go           # PUBLISH (pkg/sseevent)
          breaker.go          # pkg/breaker — payment gateway, TULIS
          lease.go            # pkg/outbox.Lease
        midtrans/
          client.go           # charge, cancel, expire, status
          client_test.go
        http/payment.go, webhook.go, admin.go, router.go, init.go
      port/
        inbound/payment.go
        outbound/
          booking.go, payment_event.go, ticket.go, nik_claim.go
          midtrans.go, stock.go, credential.go, publisher.go, outbox.go
```

### 10.2 Outbound ports

```go
type IMidtransClient interface {
    // Charge: POST /v2/charge dengan transaction_details.order_id = key
    // turunan dan custom_expiry.
    Charge(ctx context.Context, req ChargeRequest) (ChargeResult, error)
    // Cancel: POST /v2/{order_id}/cancel — merchant membatalkan transaksi
    // pending. BUKAN expire: tidak ada yang kedaluwarsa saat customer
    // berganti metode, dan dashboard Midtrans melaporkan keduanya terpisah
    // (ADR-005 §9).
    Cancel(ctx context.Context, orderID string) error
    // Expire: POST /v2/{order_id}/expire — dipanggil sweep admission-ctl,
    // bukan service ini. Ada di interface karena klien-nya sama.
    Expire(ctx context.Context, orderID string) error
    // Status: GET /v2/{order_id}/status — READ-ONLY. Satu-satunya cara
    // reconciler mencari tahu nasib charge yang timeout.
    Status(ctx context.Context, orderID string) (StatusResult, error)
}

type IBookingRepository interface {
    // LockOwned: SELECT … WHERE id=$1 AND grant_token=$2 FOR UPDATE.
    //
    // KEPEMILIKAN, bukan hanya token yang valid. grantToken yang sah
    // membuktikan pemegangnya punya slot, BUKAN bahwa slot itu milik
    // booking_id di URL. booking_id berbentuk RB-00912 — berurutan dan mudah
    // ditebak, dan pada 300.000 customer bersamaan enumerasinya sepele
    // (ADR-005 §12).
    //
    // 0 baris → apperror 404, BUKAN 403. 403 mengonfirmasi bahwa booking itu
    // ada, yang sudah cukup untuk memetakan rentang id aktif. Dijalankan
    // SEBELUM validasi status apa pun.
    LockOwned(ctx context.Context, tx pgx.Tx, bookingID, grantToken string) (entity.Booking, error)

    // LockByPaymentRef untuk webhook, yang tidak punya grant token.
    LockByPaymentRef(ctx context.Context, tx pgx.Tx, orderID string) (entity.Booking, error)

    UpdateStatus(ctx context.Context, tx pgx.Tx, u StatusUpdate) (rows int64, err error)
    MarkDisputed(ctx context.Context, tx pgx.Tx, bookingID string) error
    ListDisputed(ctx context.Context, eventID string) ([]entity.DisputedRow, error)
}

type ITicketRepository interface {
    // Issue menerbitkan satu baris tickets per booking_item.
    // ON CONFLICT (booking_item_id) DO NOTHING — guard QR ganda saat outbox
    // retry (ADR-005 §16; kolomnya ditambahkan di 0006, §4.2).
    Issue(ctx context.Context, tx pgx.Tx, bookingID string) ([]entity.Ticket, error)
}
```

### 10.3 `GET /payments/{booking_id}/methods`

```
1. booking := repo.LockOwned(id, grant.Token)   -- tanpa FOR UPDATE; read saja
     0 baris → 404
2. pgOpen := breaker.New(redis, ServicePaymentGateway, …).State() == OPEN
3. methods := [
     { id: "bank_transfer", banks: [bca, bni, bri, cimb], enabled: !pgOpen },
     { id: "gopay",                                       enabled: !pgOpen },
   ]
   -- enabled = false saat breaker terbuka; UI menonaktifkan pilihan
   -- (kontrak §5.1).
   -- TIDAK ADA requires_tokenization maupun client_key: keduanya hanya
   -- dibutuhkan kartu, dan kartu di luar lingkup POC.
```

> **Empat bank, dan itu keputusan lingkup.** Midtrans mendokumentasikan `settlement → deny`
> (reversal, 1–5 menit, dana kembali ke customer) **hanya** untuk Permata, Mandiri Bill
> Payment, dan Indomaret. Ketiganya tidak dibuka, sehingga kelas kegagalan itu tidak dapat
> terjadi dan **handler reversal tidak diperlukan** (ADR §8, kontrak §5.1).

> **Kartu kredit dan OTC keluar karena alasan yang sama**: keduanya tidak dapat menghormati
> TTL pembayaran di sisi gateway. `custom_expiry` tidak berlaku untuk kartu (dinyatakan
> eksplisit oleh Midtrans), dan OTC menuntut customer datang ke gerai fisik. Keluarnya
> kartu juga mengeluarkan cakupan PCI-DSS dan alur tokenisasi di browser (NFR-07).

### 10.4 `POST /payments/{booking_id}` — Initiate

ADR §9 langkah 1 apa adanya. Perbaikan atas v1.0 ditandai.

```
 1. grant := grantauth.FromContext(ctx)      -- grant + device sudah divalidasi
 2. ratelimit.Allow(rediskeys.PayRateLimit(id), 3, 3.0/600) → 429

 3. BEGIN
 4.   b := repo.LockOwned(tx, id, grant.Token)        -- FOR UPDATE
        0 baris → COMMIT ; 404

        status == 'AWAITING_PAYMENT' → COMMIT ; 200 { payment_ref,
                                                      payment_expires_at,
                                                      instructions tersimpan }
          -- PEMUTARAN ULANG. Tidak memanggil gateway, tidak menulis apa pun,
          -- respons IDENTIK dengan yang pertama. Tombol "Bayar" yang terklik
          -- dua kali tidak boleh terlihat seperti kegagalan (kontrak §5.2).
          -- instructions disimpan di payment_events.raw charge pertama,
          -- sehingga va_number / qr_url dapat diputar ulang tanpa gateway.
        status == 'CONFIRMED'        → COMMIT ; 200 { status: 'CONFIRMED' }
        status != 'HELD'             → COMMIT ; 409 { reason }

 5.   pgState := breaker.State()
        OPEN → COMMIT ; 503 ADMISSION_PAUSED
 6.   orderID := b.ID + ":" + strconv.Itoa(b.PaymentAttempts)
        -- DITURUNKAN dari state Aurora, TIDAK PERNAH dikirim klien. Core API
        -- membawa idempotency pada order_id, bukan header tersendiri, dan
        -- MENOLAK order_id yang sudah dipakai. booking_id telanjang akan
        -- ditolak mentah pada percobaan kedua dan customer terkunci dari
        -- membayar booking-nya sendiri (ADR-005 §8).
        -- payment_attempts hanya naik pada retry EKSPLISIT setelah kegagalan,
        -- bukan pada duplikat bersamaan — sehingga dua klik Pay selalu
        -- menurunkan order_id yang identik dan Midtrans men-dedup-nya.

 7.   res := midtrans.Charge({
          order_id: orderID, gross_amount: b.Total, payment_type: req.Method,
          bank: req.Bank,
          custom_expiry: { order_time: b.HeldAt (WIB),
                           expiry_duration: "10", unit: "minute" },
        })
        -- order_time = held_at, BUKAN waktu request, BUKAN sisa jendela.
        -- Anchor 10:15 + 10 menit tetap berakhir 10:25 walau charge kedua
        -- terjadi pukul 10:23. Seandainya diisi now(), setiap pergantian
        -- metode memberi 10 menit baru dan TTL pembayaran tidak pernah habis
        -- (ADR-005 §9).
        -- ◀ v1.0 menulis "custom_expiry = held_at + 600s (payment_expires_at
        --   − now)" — dua rumus yang saling bertentangan di satu baris.
        -- WAJIB: expiry VA bawaan Midtrans 24 JAM vs TTL kita 10 menit.
        -- Tanpa ini VA tetap dapat dibayar sepanjang hari setelah stok
        -- dijual ulang.

        timeout → ROLLBACK ; 503
          -- JANGAN kirim ulang charge. Biarkan payment_ref NULL dan serahkan
          -- ke charge reconciler (§9.9). Payment gateway adalah dependensi
          -- WRITE: timeout tidak berarti gagal (ADR-005 §8, §9).
          breaker.RecordFailure()

        deny sinkron →
          UPDATE bookings SET status='HELD', payment_ref=NULL,
                 payment_method=NULL,
                 payment_attempts = payment_attempts + 1    -- WAJIB
            WHERE id=$1 AND status='HELD'
          COMMIT ; 200 { status: 'FAILED', can_retry: true, attempt }
          -- Kenaikan attempt DI SINI yang membuat charge berikutnya memakai
          -- order_id baru. Tanpanya Midtrans menolak percobaan kedua sebagai
          -- duplikat (ADR-005 §9).
          -- ◀ v1.0 tidak punya cabang ini sama sekali.

 8.   UPDATE bookings
         SET status='AWAITING_PAYMENT', payment_ref=$orderID,
             payment_method=$method
       WHERE id=$1 AND status='HELD'
      -- held_at TIDAK ditulis. payment_expires_at TIDAK ditulis — sudah ada
      -- sejak HELD (ADR-005 §7, §9).
      -- ◀ v1.0 menulis held_at=now() di sini. Karena custom_expiry.order_time
      --   diisi held_at, itu memberi 10 menit baru pada SETIAP charge dan
      --   BR-05 tidak pernah ditegakkan.
 9.   outbox.Enqueue(tx, Record{Type: events.TypePaymentInitiated, …})
    COMMIT

10. -- TIDAK perlu ZADD: skor penjadwal sudah payment_expires_at sejak HELD.
11. breaker.RecordSuccess()
12. 200 { payment_id: orderID, method, status: "PENDING",
          instructions: {bank, va_number} | {qr_url, deeplink_url},
          ttl: { expires_at: b.PaymentExpiresAt, ttl_seconds, server_time } }
```

### 10.5 `POST /payments/webhook`

```
 1. Resource policy API Gateway (allowlist IP) sudah menyaring di depan.
    signature_key == SHA512(order_id + status_code + gross_amount + ServerKey)
      -- Konkatenasi POLOS, tanpa pemisah.
      -- Tidak cocok → ABAIKAN, jangan proses, jangan ubah status apa pun.
      -- return 200 (jangan beri tahu pengirim bahwa tanda tangannya salah).
      -- Dokumentasi Midtrans memuat dua skema; SHA256withRSA pada halaman
      -- Security Specification berlaku untuk lini API lain, BUKAN webhook
      -- Core API (ADR-005 §9).

 2. BEGIN
 3.   INSERT INTO payment_events (id, booking_id, gateway_event_id,
                                  transaction_status, fraud_status,
                                  payment_type, gross_amount, raw)
        -- gateway_event_id UNIQUE = ID EVENT DARI GATEWAY, bukan booking_id.
        -- Duplikat memicu unique violation → SELURUH transaksi batal → 200.
      unique violation → ROLLBACK ; return 200

 4.   b := repo.LockByPaymentRef(tx, order_id)       -- FOR UPDATE
      transisi := valueobject.Resolve(transaction_status, fraud_status)

      SETTLED  (settlement, ATAU capture DENGAN fraud_status='accept'):
        rows := UPDATE bookings SET status='CONFIRMED'
                 WHERE id=$1 AND status IN ('AWAITING_PAYMENT',
                                            'PENDING_CANCELLATION')
          -- PENDING_CANCELLATION ikut diterima: itu seluruh gunanya grace.
        rows == 0 DAN status saat ini == 'EXPIRED':
          -- UANG DITERIMA SETELAH BOOKING KEDALUWARSA. Stok sudah dijual ulang.
          repo.MarkDisputed(tx, b.ID)        -- payment_disputed = true
            -- ◀ v1.0 melewatkan ini. Kolomnya ada di 0004 dan
            --   GET /admin/reconciliation/disputed (§10.7) membacanya;
            --   tanpa tulisan ini endpoint itu selalu kosong.
          outbox.Enqueue(tx, payment.failed_notice{reason: "disputed"})
            -- Isinya menyatakan dana diterima, booking sudah batal, dan
            -- refund sedang diproses — WAJIB email, bukan hanya log
            -- (ADR-005 §9).
          emit LatePaymentRefund             -- dialarmkan, bukan ditelan
          -- REFUND OTOMATIS TIDAK DIIMPLEMENTASIKAN DI POC (ADR-005 §9).
        rows == 1:
          tickets := ticketRepo.Issue(tx, b.ID)   -- satu per booking_item
          outbox.Enqueue(tx,
            Record{Type: events.TypePaymentConfirmed, AggregateID: b.ID, …},
            Record{Type: events.TypeTicketIssue, …} per ticket)   -- FR-26
        -- payment_type webhook BOLEH berbeda dari yang dikirim: pembayaran QR
        -- GoPay lewat aplikasi QRIS lain datang sebagai 'qris'. JANGAN
        -- cocokkan (ADR-005 §9).

      DECLINED (deny / cancel / failure, ATAU capture dengan fraud_status
                != 'accept'):
        UPDATE bookings SET status='HELD', payment_ref=NULL,
               payment_method=NULL,
               payment_attempts = payment_attempts + 1       -- WAJIB
         WHERE id=$1 AND status='AWAITING_PAYMENT'
        -- Kembali ke HELD, BUKAN EXPIRED: stok TETAP milik customer sampai
        -- TTL. Tidak ada DECR dan tidak ada INCR stock.
        -- capture + fraud_status != accept BUKAN lunas: dana belum diterima.
        -- Memperlakukan capture sebagai lunas akan menerbitkan tiket untuk
        -- transaksi yang tidak pernah didanai (ADR-005 §9).
        bila payment_attempts >= MAX_PAYMENT_ATTEMPTS:
          outbox.Enqueue(tx, payment.failed_notice{reason: "gateway_declined"})

      EXPIRED (expire dari sisi gateway):
        rows := UPDATE bookings SET status='EXPIRED'
                 WHERE id=$1 AND status='AWAITING_PAYMENT'
        rows == 1:
          DELETE FROM nik_claims WHERE booking_id = $1
          outbox.Enqueue(tx, booking.expired)                  -- FR-28

      LAINNYA (pending, refund, partial_refund, chargeback):
        catat saja; tidak ada transisi status
    COMMIT

 5. INLINE BEST-EFFORT — DI LUAR transaksi, SETELAH COMMIT:
    -- ◀ Seluruh langkah ini tidak ada di v1.0. Ia yang membuat NFR-06 ≈157 ms
    --   alih-alih ≈1107 ms, dan tanpanya tidak ada sse-grant pada pembayaran.

    untuk baris outbox bertipe payment.confirmed yang baru ditulis:
      if lease.Claim(outboxID):                 -- SET processed:{id} NX EX 30
          ZREM expiries:{ev} bookingID
          DECRBY slots:open:{ev} 1              -- TERMINAL (bookingstate)
          DEL nik:used tidak dilakukan          -- kursi final, NIK terpakai
          DEL acct:{ev}:{account_id}            -- CONFIRMED → boleh antre lagi
          DEL token:{uuid}
          SET settled:{uuid} {booking_id} EX SETTLED_TOMBSTONE_SECONDS (3600)
            -- NISAN. §7.3 membacanya supaya tamu yang me-refresh di celah
            -- antara pembayaran lunas dan email e-tiket tiba TIDAK melihat
            -- layar kedaluwarsa (ADR-005 §12).
            -- ◀ v1.0 membaca nisan ini di queue-service tetapi tidak pernah
            --   menulisnya di mana pun.
          EVALSHA advance_now_serving.lua       -- slot yang baru bebas terpakai
          PUBLISH sse-broadcast ; PUBLISH sse-grant per token
          UPDATE outbox SET published_at = now()     -- (i)
          lease.Complete(outboxID)                   -- (ii)
          -- (i) SEBELUM (ii) WAJIB:
          --   crash antara (i)-(ii) → lease kedaluwarsa 30 detik dan
          --     SELECT … WHERE published_at IS NULL tidak mengambilnya. Aman.
          --   dibalik → lease "done" 86400 detik sementara published_at tetap
          --     NULL → publisher mengambil baris tetapi Claim selalu gagal →
          --     tidak pernah tuntas dan OutboxLag naik palsu selamanya.
      Kegagalan di langkah ini TIDAK menggagalkan request.
    -- ticket.issue TIDAK diterbitkan inline: email tidak boleh menambah
    -- latensi webhook (ADR-005 §9).

 6. return 200   ← SELALU, termasuk saat unique violation dan tanda tangan salah
```

> **Lease memberi at-least-once, bukan exactly-once.** Bila inline crash setelah
> `DECRBY slots:open` tetapi sebelum `published_at`, lease kedaluwarsa dan publisher
> mengulang seluruh blok — termasuk `DECR`, satu-satunya operasi non-idempoten di dalamnya
> (`ZREM`, `EVALSHA` dan `PUBLISH` semuanya aman diulang). Yang membuat ini aman bukan
> lease, melainkan **rekonsiliasi `slots:open`** (§9.7), yang menangkap drift dalam ~10
> detik. Jendela itu diterima; menyembunyikannya tidak (ADR §9).

### 10.6 `POST /payments/{booking_id}/retry`

v1.0 hanya mendaftarkan nama file. Enam pemeriksaan ADR §9, **berurutan — yang paling
murah dan paling aman lebih dulu**:

```
BEGIN
  b := repo.LockOwned(tx, id, grant.Token)                   -- 1. KEPEMILIKAN
    0 baris → ROLLBACK ; 404
      -- 404, BUKAN 403: 403 memberi tahu penyerang bahwa booking itu ada.
      -- Pemeriksaan ini MENDAHULUI semua yang lain agar status booking orang
      -- lain tidak pernah bocor lewat kode respons.

  b.Status != 'AWAITING_PAYMENT' → ROLLBACK ; 409             -- 2. tidak ada
      -- Satu-satunya status dengan transaksi Midtrans HIDUP. HELD setelah
      -- penolakan TIDAK memenuhi syarat: tidak ada yang perlu dibatalkan,
      -- customer cukup memanggil POST /payments/{id} (§10.4).

  b.PaymentExpiresAt <= now() → ROLLBACK ; 409                -- 3. TTL habis
      PAYMENT_WINDOW_CLOSED
      -- Sweep belum sempat menjalankannya. Menerbitkan VA baru di sini
      -- memberi customer instrumen yang mati sebelum ia sampai ke ATM.

  (b.PaymentExpiresAt − now()) < MIN_RETRY_REMAINING_SECONDS (60)
                              → ROLLBACK ; 409                -- 4. terlalu mepet
      PAYMENT_WINDOW_CLOSED
      -- Sisa 15 detik tidak cukup untuk metode apa pun. Lebih jujur menolak
      -- daripada menyerahkan VA yang pasti kedaluwarsa.

  b.PaymentAttempts >= MAX_PAYMENT_ATTEMPTS (5)
                              → ROLLBACK ; 429                -- 5. batas
      MAX_ATTEMPTS_EXCEEDED

  breaker.State() == OPEN     → ROLLBACK ; 503                -- 6. percuma
      -- Tanpa Midtrans kita tidak dapat membatalkan transaksi lama, dan
      -- menaikkan attempt tanpa membatalkannya meninggalkan DUA INSTRUMEN
      -- HIDUP untuk satu booking.

  oldOrderID := b.ID + ":" + strconv.Itoa(b.PaymentAttempts)
  midtrans.Cancel(oldOrderID)                  -- MASIH memegang row lock
    sudah settlement → ROLLBACK ; 409 PAYMENT_ALREADY_SETTLED
                       -- webhook akan mengonfirmasi secara normal
    gagal lain       → ROLLBACK ; 503 — JANGAN naikkan attempt

  UPDATE bookings SET status='HELD', payment_ref=NULL, payment_method=NULL,
         payment_attempts = payment_attempts + 1
   WHERE id=$1 AND status='AWAITING_PAYMENT'
COMMIT

200 { booking_id, status: "HELD", attempt, cancelled_order_id: oldOrderID,
      payment_expires_at: b.PaymentExpiresAt, server_time }
```

> **Cancel SEBELUM COMMIT, bukan sesudah.** Kedua urutan bisa gagal, tetapi tidak
> simetris: commit berhasil lalu cancel gagal meninggalkan attempt yang sudah naik **dan**
> VA lama yang masih hidup — dua instrumen, dan customer dapat membayar keduanya. Cancel
> berhasil lalu commit gagal hanya meninggalkan VA yang tidak lagi berfungsi, yang terlihat
> pada polling status dan dibersihkan sweep pada TTL. Yang kedua dapat dipulihkan; yang
> pertama **mengambil uang dua kali** (ADR §9).

> **Row lock ditahan melewati panggilan HTTP.** Itu lazimnya dihindari, tetapi di sini
> kuncinya adalah baris booking milik **satu** customer — ia hanya memblokir permintaan
> bersamaan customer itu sendiri, yang memang harus diblokir. Timeout ke Midtrans dibuat
> pendek.

> **TTL tidak ikut mundur.** `payment_expires_at` tetap dihitung dari `held_at` (BR-05).
> Customer yang berganti metode pada menit ke-8 menerima VA baru dengan sisa 2 menit,
> bukan 10.

### 10.7 `GET /payments/{booking_id}/status` dan `GET /admin/reconciliation/disputed`

```
GET /payments/{booking_id}/status            -- polling 2 detik dari halaman bayar
  b := repo.LockOwned(id, grant.Token)       -- read, 0 baris → 404
  pkg/bookingstate.ToPaymentAPIStatus(b):
    AWAITING_PAYMENT                  → PENDING      -- tetap di layar instruksi
    CONFIRMED                         → PAID         -- ke halaman tiket
    HELD dengan payment_attempts > 0  → FAILED       -- kembali ke daftar metode
    HELD dengan payment_attempts == 0 → PENDING      -- belum pernah charge
    PENDING_CANCELLATION              → PENDING      -- "memverifikasi pembayaran…",
                                                     -- JANGAN layar kedaluwarsa
    EXPIRED / LAPSED / ABANDONED      → EXPIRED
  FAILED menyertakan reason, can_retry, attempt, payment_expires_at, server_time
    -- Inilah SATU-SATUNYA sinyal bahwa pembayaran gagal. Tanpa status ini
    -- customer menatap countdown yang terus berjalan tanpa tahu charge-nya
    -- ditolak. Frontend yang menerimanya kembali ke daftar metode TANPA
    -- memanggil /retry (kontrak §5.4).

GET /admin/reconciliation/disputed?event_id=…    -- kontrak §6.3; v1.0 tidak punya
  -- TIDAK memakai pkg/grantauth: diamankan sama seperti dashboard operasional,
  -- terpisah dari jalur customer. Di POC: resource policy API Gateway
  -- (allowlist IP kantor) + IAM, tanpa kredensial aplikasi.
  rows := repo.ListDisputed(eventID)   -- WHERE payment_disputed
  per baris: booking_id, order_id, gateway_event_id, gross_amount,
             payment_type, paid_at, cancelled_at, contact_email,
             refund_status: "PENDING_MANUAL"
  -- Setiap baris memuat order_id dan gateway_event_id, sehingga pencocokan
  -- di dashboard Midtrans tidak memerlukan pencarian manual (ADR-005 §9).
```

### 10.8 Config

```go
type Config struct {
    App      App
    Log      Log
    DB       DB             // WRITER
    Redis    Redis
    Midtrans MidtransCfg    // {ServerKey, ClientKey, IsProduction, TimeoutMs,
                            //  WebhookIPAllowlist}
    Grant    GrantCfg       // {DevicePepper, MaxDeviceRebinds} → pkg/grantauth
    Payment  PaymentCfg     // {MaxAttempts: 5, MinRetryRemainingSeconds: 60,
                            //  PaymentTTLSeconds: 600, PaymentGraceSeconds: 30,
                            //  SettledTombstoneSeconds: 3600,
                            //  OutboxLeaseSeconds: 30}
    Breaker  BreakerCfg     // {FailThreshold, OpenForSeconds} — service ini MENULIS
}
```

> **Server key dipakai untuk Basic Auth charge DAN verifikasi `signature_key` webhook.
> Client key dikirim ke browser untuk tokenisasi kartu; server key TIDAK PERNAH.** Dengan
> kartu tidak dibuka, client key sebenarnya tidak terpakai di POC — ia tetap di config
> supaya penambahan kartu kelak tidak menyentuh dua tempat.

---

## 11. `services/sale-ops`

**Tipe compute**: Lambda, dipicu EventBridge — **bukan** API Gateway
**Fungsi**: menyiapkan event untuk dijual, dan membuka sale-nya

Service ini tidak ada di v1.0, dan tanpanya **sale tidak pernah dibuka**: `queue:{ev}:state`
lahir `SCHEDULED` dan tidak ada komponen lain yang memindahkannya ke `OPEN`. Kunci
`stock:{ev}:{cat}` juga tidak pernah ditanam, sehingga setiap hold menjawab
`STOCK_UNINITIALIZED`.

ADR §6 menamainya: "job terjadwal pada `sale_starts_at` (EventBridge, satu kali per event)".

### 11.1 Struktur direktori

```
services/sale-ops/
  cmd/lambda/main.go          # satu handler, dispatch pada field `action`
  cmd/server/main.go          # CLI lokal: ./sale-ops seed|open|close --event-id=…
  internal/
    config/config.go, load.go
    delivery/handler.go
    saleops/
      application/usecase/
        seed.go               # tanam state Redis dari Aurora
        open.go               # SCHEDULED → OPEN
        close.go              # paksa CLOSED (operator)
        rebuild.go            # pemulihan setelah kehilangan Redis total
        init.go
      domain/dto/request.go, response.go
      infrastructure/
        postgres/db.go, init.go, event.go, category.go, queue_state.go,
                 queue_entry.go, booking.go
        redis/init.go, seed.go, state.go
        http/handler.go, router.go
      port/outbound/event.go, state.go
```

### 11.2 `seed` — dijalankan T-30 menit

```
1. ev := SELECT … FROM events WHERE id = $1
     sale_starts_at IS NULL → error: event tidak punya jadwal
2. cats := SELECT … FROM ticket_categories WHERE event_id = $1
     kosong → error
3. Redis, satu pipeline:
     SET  queue:{ev}:capacity  ev.queue_capacity
     SET  queue:{ev}:issued    0        NX   -- NX: seed ulang tidak me-reset
     HSET queue:{ev}            now_serving 0 rate_per_min 0   -- bila belum ada
     SET  slots:open:{ev}      0        NX
     SET  queue:{ev}:state     'SCHEDULED'  NX
     per kategori:
       SET stock:{ev}:{cat} cat.available NX
         -- `available`, bukan `capacity`: seed ulang setelah sale berjalan
         -- harus memakai snapshot, bukan mengembalikan stok yang sudah
         -- terjual. Pada event baru keduanya sama.
4. UPSERT event_queue_state (event_id, state='SCHEDULED')
5. Verifikasi: baca kembali seluruh kunci dan kembalikan ringkasannya
     -- Checklist T-10 menit memeriksa ringkasan ini, bukan menebak.
```

> **Seluruh `SET` memakai `NX`.** Seed adalah operasi yang akan dijalankan dua kali oleh
> operator yang tidak yakin apakah yang pertama berhasil, dan kedua kalinya tidak boleh
> me-reset `issued` menjadi 0 — itu akan menerbitkan ulang nomor antrian yang sudah
> dipegang 300.000 orang.

### 11.3 `open` — dipicu EventBridge pada `sale_starts_at`

```
1. BEGIN
     UPDATE event_queue_state SET state='OPEN', opened_at=now()
      WHERE event_id=$1 AND state='SCHEDULED'            -- CAS
     0 baris → COMMIT; sudah terbuka atau sudah selesai; return (idempoten)
   COMMIT
2. SET queue:{ev}:state 'OPEN'
3. Verifikasi stok ter-seed: tiap stock:{ev}:{cat} ADA
     ada yang hilang → ROLLBACK state ke SCHEDULED dan alarm
     -- Membuka sale yang stoknya tidak ter-seed berarti 300.000 orang
     -- bergabung ke antrian yang setiap hold-nya menjawab 503.
4. PUBLISH sse-broadcast {state: 'OPEN', nowServing: 0, …}
     -- Klien pra-buka yang sudah memegang koneksi? Tidak ada: stream menolak
     -- SCHEDULED dengan 409 (§8.3). Publish ini untuk admission-ctl dan
     -- metrics; klien menemukan state baru dari GET /events yang di-cache
     -- 30 detik, atau dari join pertamanya.
5. emit SaleOpened
```

> **EventBridge satu kali per event**, dijadwalkan saat `sale_starts_at` ditetapkan.
> Minimum granularitas EventBridge adalah 1 menit, dan itu cukup di sini — gerbangnya
> dibuka pada menit, bukan pada detik. Ini satu-satunya pekerjaan terjadwal yang **tidak**
> dapat dipindah ke loop admission-ctl, karena admission-ctl hanya mem-tick event yang
> sudah `OPEN`/`SOLD_OUT`.

### 11.4 `rebuild` — pemulihan setelah kehilangan Redis total

ADR §12 menyatakan prosedurnya "harus ada dan dilatih". Dijalankan manual.

```
1. state, sold_out_at ← event_queue_state
2. issued             ← MAX(ticket_no) FROM queue_entries WHERE event_id=$ev
3. now_serving        ← MAX(qe.ticket_no)
                          FROM bookings b
                          JOIN queue_entries qe ON qe.queue_token = b.queue_token
                         WHERE b.event_id = $ev
   -- now_serving sengaja TIDAK dipersistenkan. Diturunkan dari nomor antrian
   -- tertinggi yang punya baris bookings, dan itu LEBIH BENAR daripada nilai
   -- tersimpan — yang bisa tertinggal satu tick dan menyebabkan token yang
   -- sudah di-grant di-grant ULANG (ADR-005 §12).
4. by_ticket          ← ZADD dari queue_entries WHERE ticket_no > now_serving
5. token:{uuid}       ← HSET per baris queue_entries (EX SALE_WINDOW_SECONDS)
6. grant:{token}      ← HSET per baris bookings non-terminal yang punya grant_token
7. expiries:{ev}      ← ZADD per baris bookings non-terminal, skor sesuai status
8. slots:open, stock, nik:used ← biarkan rekonsiliasi admission-ctl (§9.7, §9.8)
                                 menanamnya; mereka sudah punya jalurnya
```

> Langkah 6 dan 7 tidak ada di daftar ADR §12 tetapi wajib: tanpa `grant:{token}` setiap
> customer yang sudah di-grant mendapat `401` di `/queue/me`, dan tanpa `expiries` tidak
> ada satu pun booking berjalan yang akan pernah di-expire. Ditambahkan di sini dan
> diusulkan kembali ke ADR (§21).

### 11.5 Config

```go
type Config struct {
    App   App
    Log   Log
    DB    DB       // WRITER
    Redis Redis
    Sale  SaleCfg  // {QueueCapacityDefault: 300000, SaleWindowSeconds}
}
```

---

## 12. `services/metrics-scraper`

**Tipe compute**: Lambda, EventBridge tiap 1 menit
**Fungsi**: membaca Redis dan Aurora, menerbitkan **CloudWatch EMF** pada namespace
`RusuhBooking/Queue`

FR-33, FR-34, FR-35 dan NFR-09 seluruhnya bergantung pada komponen ini, dan v1.0 tidak
menyebutnya sama sekali. ADR §13: "Tidak ada endpoint aplikasi untuk ini. `metrics-scraper`
(EventBridge, tiap 1 menit) membaca Redis dan Aurora dan menerbitkan CloudWatch EMF."

### 12.1 Struktur direktori

```
services/metrics-scraper/
  cmd/lambda/main.go
  cmd/server/main.go
  internal/
    config/config.go, load.go
    delivery/handler.go
    metrics/
      application/usecase/scrape.go, init.go
      domain/entity/sample.go
      infrastructure/
        postgres/db.go, init.go, query.go
        redis/init.go, query.go
        emf/writer.go           # struktur log EMF ke stdout
      port/outbound/source.go, sink.go
```

### 12.2 Metrik yang diterbitkan

Per event aktif, dimensi `EventId`:

| Metrik EMF | Sumber | Requirement |
|---|---|---|
| `QueueDepth` | Redis: `issued − now_serving` | FR-33 |
| `NowServing`, `RatePerMin` | Redis | FR-33 |
| `ConversionRate` | Aurora: `COUNT(CONFIRMED) / COUNT(DISTINCT queue_token)` | FR-34 |
| `ExpiryRate` per tahap | Aurora: `COUNT(*)` per status terminal, dimensi `Stage` | FR-35 |
| `StockLevel` per kategori | Redis, dimensi `CategoryId` | NFR-09 |
| `OpenSlots` | Redis: `slots:open:{ev}` | NFR-09 |
| `DukcapilCircuitState`, `PGCircuitState` | Redis: `breaker:{service}:state` → 0/1/2 | NFR-09 |
| `AdmissionPausedSeconds` | Redis: `pause:{ev}` umur | NFR-09 |
| `QPersistLag`, `QPersistAgeSeconds` | Redis: `LLEN`, entri tertua | §9.7 — angka RPO |
| `OutboxLag` | Aurora: `COUNT(*) WHERE published_at IS NULL` | ADR §4 |
| `RegrantSetSize` | Redis: `SCARD regrant:{ev}` | ADR §4 |
| `SlotsOpenDrift`, `StockDrift` | diemisi admission-ctl, bukan di sini | §9.7, §9.8 |
| `CtlHeartbeatAge` | diemisi sse-server, bukan di sini | §8.6 |

> **Dua metrik sengaja TIDAK di sini.** `CtlHeartbeatAge` diemisi sse-server karena proses
> yang mati tidak dapat melaporkan umurnya sendiri (§8.6), dan metrik drift diemisi
> admission-ctl karena hanya ia yang tahu nilai `last_drift`. Scraper menerbitkan apa yang
> dapat dibaca dari luar; sisanya milik proses yang memilikinya.

### 12.3 Dua keterbatasan yang diterima

**Granularitas EMF standar adalah 1 menit** ditambah jeda ingestion log, sehingga
"real-time" pada FR-33 dan NFR-09 berarti *dalam satu menit*, bukan sub-detik.

**Dashboard CloudWatch menuntut akses IAM AWS**: ia melayani tim SJT, **bukan** pemilik
acara atau stakeholder bisnis. Bila kelak metrik ini perlu dilihat di luar AWS, endpoint
aplikasi harus ditambahkan — service ini sudah memegang query-nya (ADR §13).

### 12.4 Config

```go
type Config struct {
    App     App
    Log     Log
    DB      DB          // READER — seluruhnya baca
    Redis   Redis
    Metrics MetricsCfg  // {Namespace: "RusuhBooking/Queue"}
}
```

---

## 13. Perubahan pada Service yang Sudah Ada

v1.0 menandai keempat service yang sudah ada sebagai selesai. Tiga di antaranya butuh
perubahan sebelum service baru dapat berfungsi.

### 13.1 `pkg/events` — tambah `payment.confirmed`

`pkg/events/events.go` punya lima tipe: `account.verify`, `auth.magic_link`,
`ticket.issue`, `booking.expired`, `payment.failed_notice`. ADR §9 menuntut satu lagi.

```go
// TypePaymentConfirmed memicu efek samping admission pada jalur fallback:
// ZREM expiries → DECR slots:open → EVALSHA advance → PUBLISH. Jalur UTAMA-nya
// inline di webhook (§10.5 langkah 5); baris outbox ini adalah JAMINANNYA —
// bila inline crash, publisher mengulang blok yang sama.
//
// Ia tidak mengirim email. payment.confirmed → notification tidak punya
// template, dan itu benar: yang dikirim ke customer adalah ticket.issue.
// Owned by payment-api.
TypePaymentConfirmed Type = "payment.confirmed"
```

Tambahkan juga ke `IsKnown()` dan ke `events_test.go`.

> `payment.refund` **tetap tidak dibuat**. `events_test.go` saat ini menegaskan
> `Type("payment.refund").IsKnown() == false` dengan komentar "not built in the POC", dan
> ADR §9 setuju: refund disputed dicatat dan direkonsiliasi manual (§10.5, §10.7). Jalur
> otomatisnya adalah tindak lanjut sebelum produksi.
> `payment.initiated` (ADR §9 langkah 1 nomor 5) **juga ditambahkan** dengan alasan yang
> sama seperti `payment.confirmed`: ia ditulis dalam transaksi charge dan tidak mengirim
> email, tetapi ia jejak audit yang dituntut ADR. Dua tipe baru, bukan satu.

### 13.2 `services/outbox-publisher` — efek samping per tipe

`internal/outbox/application/usecase/drain.go` hari ini meneruskan **setiap** baris ke SQS
tanpa melihat tipenya. ADR §9 langkah 4 menuntut dua tipe diproses di sini, bukan dikirim
ke SQS:

```
payment.confirmed → ZREM expiries → DECR slots:open → EVALSHA advance → PUBLISH
                    (TIDAK ke SQS: tidak ada email)
payment.initiated → (TIDAK ke SQS; audit saja — tandai published)
ticket.issue      → INSERT tickets per booking_item  → lalu SQS → SES
booking.expired   → SQS → SES ke contact_email + tiap holder_email      FR-28
payment.failed_notice → SQS → SES ke contact_email
account.verify    → SQS → SES link verifikasi                           FR-31
auth.magic_link   → SQS → SES
```

Yang berubah:

```
port/outbound/
  handler.go         ← BARU: Handler interface + registry per events.Type
  admission.go       ← BARU: IAdmissionEffects (Redis + EVALSHA + PUBLISH)
  ticket.go          ← BARU: ITicketIssuer (INSERT tickets)
application/usecase/
  drain.go           ← dispatch pada record.Type sebelum Publish
  handler_payment.go ← BARU
  handler_ticket.go  ← BARU
infrastructure/
  redis/admission.go ← BARU: pkg/redisscript + pkg/sseevent
  postgres/ticket.go ← BARU
config/
  config.go          ← + Redis (sudah ada untuk lease), + DB WRITER
```

> **`ticket.issue` menerbitkan tiket di sini, bukan di payment-api.** v1.0 menaruh
> `IssueTickets` di transaksi webhook (§10.5 langkah 4 memang memanggilnya di sana, dan itu
> benar — jalur cepat). ADR §9 langkah 4 menugaskannya **juga** ke publisher sebagai
> fallback. Keduanya aman berjalan karena `UNIQUE (booking_item_id)` pada `tickets`
> (`0006`, §4.2): percobaan kedua tidak menerbitkan QR kedua. Itu guard yang tanpa kolom
> baru di `0006` tidak akan ada.

> **`drain.go` tetap: Publish SEBELUM MarkPublished.** Komentarnya sudah menjelaskan
> alasannya, dan alasan itu tidak berubah. Yang ditambahkan hanya cabang per tipe sebelum
> `Publish`, dengan urutan lease yang sama: `Claim` → kerja → `MarkPublished` → `Complete`.

> **`publisher-id` pada lease.** `Claim` menulis `worker_id`; inline publisher payment-api
> dan outbox-publisher harus memakai nilai yang **berbeda** supaya metrik
> `OutboxInlineHitRate` dapat memisahkan keduanya. `pkg/outbox.Lease` menerima `workerID`
> pada konstruktornya.

### 13.3 `services/notification` — dua tipe baru di consumer

`payment.confirmed` dan `payment.initiated` **tidak** sampai ke SQS (§13.2), jadi
notification tidak butuh template untuknya. Yang berubah hanya guard:

```
Tipe yang tidak dikenal → biarkan di queue, JANGAN drop (sudah menjadi perilaku
hari ini: events.Type.IsKnown()). Dengan dua tipe baru di pkg/events yang tidak
pernah dikirim ke SQS, tambahkan keduanya ke daftar "dikenal tetapi tidak
di-handle di sini" agar tidak masuk DLQ bila suatu saat salah dikirim.
```

Satu perubahan nyata: **`booking.expired` dikirim ke `contact_email` DAN setiap
`holder_email`** (ADR §9 langkah 4, FR-28). Payload-nya karena itu harus memuat daftar
holder. Periksa `pkg/events.BookingExpiredPayload` — bila ia hanya membawa satu alamat,
perluas.

Dan `Accept-Language` → `events.Locale` pada payload menentukan bahasa email (FR-32).
Producer baru (payment-api, admission-ctl) wajib mengisinya; default `LocaleID`.

### 13.4 `services/nik-verify` — migrasi kunci Redis

```
dukcapil:breaker:state     → breaker:dukcapil:state
dukcapil:breaker:failures  → breaker:dukcapil:failures
dukcapil:ratelimit         → rl:dukcapil
nik:valid:{hmac}           → tidak berubah
```

Ganti `infrastructure/redis/init.go` konstanta dengan `pkg/rediskeys`, dan
`infrastructure/redis/breaker.go` + `limiter.go` dengan `pkg/breaker` + `pkg/ratelimit`.

> **Lakukan ini SEBELUM admission-ctl ada.** Hari ini tidak ada pembaca lain, sehingga
> migrasi ini adalah perubahan satu service. Setelah admission-ctl membaca
> `breaker:dukcapil:state`, ia menjadi perubahan lintas service yang harus dideploy
> serentak — dan bila terlewat, breaker yang dibaca admission-ctl akan selalu kosong
> (dibaca sebagai CLOSED) sehingga **pause Dukcapil tidak pernah terjadi**, secara senyap.
>
> Ini alasan `pkg/rediskeys` (§3.1) dikerjakan pertama di §19.

Satu perubahan fungsional: `POST /nik/validate` kini di belakang `pkg/grantauth` (kontrak
§4.2 menuntut `X-Grant-Token` + `X-Device-Id`). Periksa apakah handler hari ini sudah
memeriksanya; bila belum, tambahkan middleware — tanpa itu endpoint berbayar Dukcapil
terbuka untuk siapa pun yang tahu URL-nya.

### 13.5 `pkg/apperror` — kode baru

Kode dari Appendix A kontrak yang belum ada di `pkg/apperror`:

```
GRANT_TOKEN_MISSING, GRANT_TOKEN_INVALID, DEVICE_ID_REQUIRED, DEVICE_MISMATCH,
CAPTCHA_REQUIRED, CAPTCHA_FAILED, QUEUE_FULL, ADMISSION_PAUSED,
STOCK_UNINITIALIZED, STOCK_NEGATIVE, QTY_EXCEEDED, QTY_INVALID,
INSUFFICIENT_STOCK, NIK_ALREADY_USED, NIK_UNVERIFIED, TERMS_NOT_ACCEPTED,
PAYMENT_ALREADY_INITIATED, PAYMENT_ALREADY_SETTLED, PAYMENT_WINDOW_CLOSED,
MAX_ATTEMPTS_EXCEEDED, BANK_NOT_SUPPORTED, BOOKING_EXPIRED, PAYMENT_EXPIRED
```

Dengan pemetaan HTTP-nya, karena beberapa tidak intuitif: `STOCK_UNINITIALIZED` dan
`STOCK_NEGATIVE` adalah **503**, bukan 409 — stok sebenarnya tidak diketahui (ADR §8).
`QUEUE_FULL` adalah **503**. `DEVICE_ID_REQUIRED` adalah **400**, `DEVICE_MISMATCH`
adalah **401**.

### 13.6 `services/user` — migrasi ke `pkg/outbox`

`services/user/internal/user/infrastructure/postgres/outbox.go` adalah satu-satunya
implementasi `INSERT INTO outbox` hari ini, dan `pkg/outbox` (§3.6) menggantikannya.
Migrasi ini **opsional untuk fungsionalitas** tetapi wajib untuk menghindari dua penulis
tabel yang sama dengan bentuk query berbeda.

Dilakukan setelah `pkg/outbox` ada dan setelah booking/payment memakainya — bukan
sebelumnya, supaya user-service tidak menjadi kelinci percobaan untuk API yang masih
berubah.

### 13.7 Token e-tiket — dua konsumen `pkg/eticket`

Satu-satunya butir di §13 yang memperbaiki **requirement yang tidak terpenuhi**, bukan
merapikan yang sudah jalan. Keduanya harus rilis bersama: tautan bertoken yang diverifikasi
rute lama menjawab 401, dan rute baru tanpa tautan bertoken tidak menolong siapa pun.

**`services/notification` — terbitkan token ke dalam tautan.** Hari ini
`domain/template/render.go` menyusunnya tanpa token:

```go
"ETicketLink": r.links.AppBase + "/tickets/" + p.TicketID + "/eticket"
```

Menjadi — `exp` diturunkan dari `EventDate` yang **sudah** ada di `TicketIssuePayload`,
jadi payload tidak perlu diperluas:

```
exp   := parse(p.EventDate) + 24h            -- kontrak §6.2
tok,_ := eticket.Mint(p.TicketID, exp)
"ETicketLink": AppBase + "/tickets/" + p.TicketID + "/eticket?t=" + tok
```

> Bila `Mint` gagal atau `EventDate` tidak dapat di-parse, **jangan kirim tautan tanpa
> token** — itu mengirim email yang tombolnya pasti 401. Kembalikan error sehingga pesan
> itu di-retry lalu masuk DLQ dan terlihat, bukan terkirim dalam keadaan rusak.

**`services/user` — terima token sebagai alternatif sesi.** `GET /tickets/{id}/eticket`
menerima **salah satu dari dua**, dan tidak pernah keduanya sebagai syarat:

| Jalur | Resolusi kepemilikan |
|---|---|
| `Authorization: Bearer` | klausa kepemilikan yang ada: *pembeli **atau** pemegang NIK terdaftar* |
| `?t=` | `eticket.Verify(t, ticketID, now)` — tanpa identitas akun, tanpa pembacaan store |

Gagal → **401 `ETICKET_TOKEN_INVALID`** (§13.5; satu kode untuk tidak sah, kedaluwarsa dan
tidak cocok — membedakannya memberi tahu penebak bagian mana yang hampir benar). Endpoint
ini **tidak** menerima `X-Grant-Token`: grant membuktikan slot, bukan kepemilikan tiket, dan
ia sudah kedaluwarsa jauh sebelum tanggal event.

> `GET /users/me/tickets` (§6.1) **tidak** berubah: ia mendaftar tiket satu akun, dan token
> berlingkup satu tiket tidak dapat menjawabnya. Pemegang NIK tanpa akun memang tidak punya
> daftar — ia punya email per kursi, dan itulah yang FR-26 janjikan.

**Config baru di kedua service**: `ETICKET_SIGNING_KEY`, wajib, nilai yang **sama**. Nilai
yang berbeda membuat setiap tautan gagal verifikasi, dan gejalanya hanya muncul di sisi
customer — jadi startup kedua service memvalidasi keberadaannya (§3.10).

---

## 14. Lua Scripts

Semua di `pkg/redisscript/scripts/`, di-embed dengan `//go:embed`, di-`SCRIPT LOAD` sekali
saat startup, dijalankan via `EVALSHA` dengan fallback `EVAL` pada `NOSCRIPT` (§3.4).

| Skrip | Pemanggil |
|---|---|
| `advance_now_serving.lua` | admission-ctl (§9.6), queue-service (§7.2 langkah 10), payment-api (§10.5 langkah 5), outbox-publisher (§13.2) |
| `hold_booking.lua` | booking-service (§5.4 langkah 6) |
| `adjust_if_unchanged.lua` | admission-ctl (§9.8) |
| `token_bucket.lua` | `pkg/ratelimit` (§3.2) — di paket itu, bukan di sini |
| `leader_renew.lua`, `leader_release.lua` | admission-ctl (§9.2) |

> **Empat pemanggil untuk `advance_now_serving.lua`**, bukan satu. v1.0 menaruhnya di
> `services/admission-ctl/internal/.../scripts/` — di dalam `internal/` modul lain, jadi
> tiga pemanggil lainnya harus menyalinnya. ADR §5 menyatakan alasan skrip ini tunggal:
> "tidak ada jalur admission kedua yang bisa menyimpang dari yang utama."

### 14.1 `advance_now_serving.lua`

Dari ADR-005 §6, verbatim. v1.0 menyatakan skrip ini "terlampir di dokumen ini §12";
§12 v1.0 adalah bagian CI/CD dan skripnya tidak pernah ada di sana.

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
-- gagal, jadi ia belum pernah benar-benar dilayani. TIDAK ada no-show di sini --
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
`requirements.md` §6, dinyatakan sebagai *deficit* agar satu tick dapat memberi banyak
slot sekaligus secara atomik.

> **Kepemilikan `slots:open`**: Lua hanya `INCR`. `DECR` dilakukan pemanggil, **sebelum**
> `EVALSHA`, dan hanya bila `UPDATE` Aurora mengenai tepat 1 baris (ADR §6). Double-DECR →
> `slots:open` negatif → `deficit = target − (−n)` → **over-admission**. Aturan ini
> dikodekan di `pkg/bookingstate.DecrementsSlot` (§3.7).

### 14.2 `hold_booking.lua`

Dari ADR-005 §8, verbatim. v1.0 menuliskan versi sendiri yang kehilangan
`STOCK_UNINITIALIZED`, guard `STOCK_NEGATIVE`, kuota lintas kategori dan
`NIK_COUNT_MISMATCH`, dan yang header ARGV-nya tidak cocok dengan isinya.

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

> **`STOCK_UNINITIALIZED` dan `STOCK_NEGATIVE` adalah 503, bukan 409.** Keduanya BUKAN
> "habis": yang pertama berarti kunci stok hilang dari Redis, yang kedua berarti guard
> terakhir menyala. Stok sebenarnya tidak diketahui. Keduanya dialarmkan
> (`StockKeyMissing`, `StockNegativeGuard`) dan keduanya menuntut rekonsiliasi **segera**,
> bukan pada tick berikutnya (ADR §8).

> **Gagal hold TIDAK menghilangkan slot.** `NO_STOCK`, `NIK_TAKEN` atau `QTY_EXCEEDED` →
> customer tetap `SELECTING` dengan sisa TTL; UI menampilkan pesan spesifik; pemilihan
> ulang kategori atau perbaikan NIK diizinkan (NFR-02).

### 14.3 `adjust_if_unchanged.lua`

```lua
-- adjust_if_unchanged.lua
-- Compare-and-adjust: hanya terapkan delta bila nilai saat ini MASIH sama
-- dengan yang diamati. Dengan begitu dua eksekusi bersamaan tidak mengoreksi
-- 2x, bahkan bila leader lease kalah balapan (ADR-005 Bagian 11).
--
-- Ini salah satu dari DUA pengaman mandiri yang membuat lease ganda tidak
-- berbahaya, bukan hanya tidak mungkin. Yang kedua adalah
-- SET pause:{ev}:grace_applied NX (§9.4).
--
-- KEYS[1] = stock:{ev}:{cat}
-- ARGV[1] = expected current value (R yang diamati)
-- ARGV[2] = delta (boleh negatif)
local current = redis.call('GET', KEYS[1])
if current == false then return 0 end
if tonumber(current) ~= tonumber(ARGV[1]) then return 0 end
redis.call('INCRBY', KEYS[1], tonumber(ARGV[2]))
return 1
```

> Satu perbedaan dari ADR §6: cek `current == false` ditambahkan di depan. ADR menuliskan
> `tonumber(redis.call('GET', KEYS[1]))` langsung, yang menghasilkan `nil` pada kunci
> hilang dan `nil ~= tonumber(ARGV[1])` memang mengembalikan 0 — jadi perilakunya sama.
> Cek eksplisit membuat maksudnya terbaca, dan jalur kunci-hilang sudah ditangani
> terpisah di §9.8 (`SET stock E` segera, tanpa aturan dua-tick).

---

## 15. Scaffold per Service

Daftar direktori di §5–§12 memuat kode Go saja. Setiap service di repo ini juga punya
sepuluh file non-Go, dan CI **menegakkan** sebagian di antaranya (§17). v1.0 tidak
mendaftarkan satu pun, yang menyembunyikan ~70 file dari estimasi.

| File | Dibutuhkan oleh | Catatan |
|---|---|---|
| `go.mod`, `go.sum` | `isolation-check`, CI | Wajib ada `replace … => ../../pkg` (§16) |
| `Makefile` | `template.yaml` (`BuildMethod: makefile`) | Target `build-lambda` / `run`; untuk ECS: `docker-build` |
| `Dockerfile` | job `hadolint` | **Juga untuk Lambda**: booking/user/nik-verify/outbox-publisher punya satu. `notification` belum — bila `hadolint` di-set `fail-on-missing`, itu akan terlihat saat matrix diperluas |
| `.dockerignore` | build image | |
| `.golangcicd.yml` | job `golangci-lint` | Salin dari service terdekat; jangan tulis ulang aturan |
| `sonar-project.properties` | job `sonarqube` | `sonar.projectKey` unik per service |
| `example.env` | onboarding + `cmd/server` lokal | Daftar seluruh env var dari `internal/config` |
| `README.md` | — | Satu halaman: apa yang dimiliki service ini, kunci Redis dan tabel yang disentuhnya |
| `template.yaml` | job `deploy-*` (SAM) | **Hanya untuk service Lambda.** sse-server dan admission-ctl tidak punya (§17.3) |
| `_mock/…` | test | `go.uber.org/mock`; regenerasi tiap kali port berubah |
| `events/*.json` | `sam local invoke` | Opsional; `booking` punya `create-booking.json` |
| `.gitignore` | — | `coverage*.out` |

Service **ECS** (`sse-server`, `admission-ctl`) mengganti `template.yaml` dengan:

| File | Fungsi |
|---|---|
| `Dockerfile` | **wajib**, bukan opsional — image-nya yang dideploy |
| `taskdef.json` | ECS task definition, parameterised; `image` diisi CI dari tag ECR |
| `Makefile` target `docker-build`, `docker-push` | |

### 15.1 Per service, jumlah file non-Go

| Service | Lambda/ECS | File non-Go |
|---|---|---|
| `events` | Lambda | 11 |
| `queue` | Lambda | 11 |
| `payment-api` | Lambda | 11 |
| `sale-ops` | Lambda | 11 |
| `metrics-scraper` | Lambda | 11 |
| `sse-server` | **ECS** | 11 (taskdef menggantikan template) |
| `admission-ctl` | **ECS** | 11 |

Ditambah perubahan pada `booking` (hapus `memory/`, regenerasi `_mock/`, perluas
`example.env`, perbarui `template.yaml` dengan dua rute baru).

---

## 16. `go.work` dan `go.mod`

### 16.1 `go.work`

```
go 1.26.10

use (
	./pkg
	./services/admission-ctl    // baru
	./services/booking
	./services/events           // baru
	./services/metrics-scraper  // baru
	./services/nik-verify
	./services/notification
	./services/outbox-publisher
	./services/payment-api      // baru
	./services/queue            // baru
	./services/sale-ops         // baru
	./services/sse-server       // baru
	./services/user
)
```

### 16.2 `go.mod` per service — `replace` itu WAJIB

v1.0 memberi `require …/pkg v0.0.0` tanpa `replace`. CI menjalankan
`GOWORK=off go build ./...` pada setiap modul (job `isolation-check`, dan root `Makefile`
punya target yang sama), dan tanpa `replace` build itu **gagal**: tidak ada
`…/be-rusuh-war-ticketing/pkg` yang dipublikasikan ke proxy mana pun.

Setiap service yang ada sudah punya barisnya:

```
// Shared library lives in this monorepo; resolved locally (no proxy publish).
replace github.com/PTServerJanganTumbang-BookingTicket/be-rusuh-war-ticketing/pkg => ../../pkg
```

Salin verbatim ke ketujuh `go.mod` baru.

### 16.3 Dependensi per service

```
Semua service baru:
  …/be-rusuh-war-ticketing/pkg v0.0.0
  github.com/redis/go-redis/v9
  github.com/stretchr/testify
  go.uber.org/mock

Service yang menyentuh Postgres (events, queue, payment-api, sale-ops,
metrics-scraper, admission-ctl):
  github.com/jackc/pgx/v5

Service Lambda (events, queue, payment-api, sale-ops, metrics-scraper):
  github.com/aws/aws-lambda-go
  github.com/awslabs/aws-lambda-go-api-proxy   -- hanya yang di balik API Gateway;
                                               -- sale-ops dan metrics-scraper
                                               -- dipicu EventBridge, jadi TIDAK
  github.com/go-playground/validator/v10       -- yang punya request body

Service ECS (sse-server, admission-ctl):
  TIDAK ada aws-lambda-go
  admission-ctl: github.com/jackc/pgx/v5
  sse-server:    tidak ada klien DB sama sekali
```

> **`aws-sdk-go-v2/service/sqs` tidak dibutuhkan service baru mana pun.** Publikasi SQS
> adalah pekerjaan `outbox-publisher`, yang sudah punya dependensinya. v1.0 meninggalkan
> pertanyaan ini sebagai komentar yang menjawab dirinya sendiri di tengah spesifikasi
> dependensi; jawabannya ditetapkan di sini.

> **`sse-server` tidak punya klien Postgres.** Ia membaca state antrian dari Redis dan
> memverifikasi token dari Redis; tidak satu pun jalurnya menyentuh Aurora. Menambahkan
> pool DB "untuk nanti" pada service yang memegang 7.000 koneksi per task adalah biaya
> memori tanpa pemakai.

---

## 17. CI/CD

Bagian ini menggantikan §12 v1.0 seluruhnya. v1.0 memerintahkan penambahan 5 service ke
"matrix `test-services`" pada `.github/workflows/cicd.yml` — tidak ada job bernama itu dan
tidak ada matrix berbentuk itu. Yang berikut ditulis terhadap pipeline yang sebenarnya ada.

### 17.1 Bentuk pipeline yang sebenarnya

```
detect-changes    → dorny/paths-filter per service, lalu skrip bash menyusun
                    matrix `modules` berisi PATH MODUL (pkg, services/booking, …).
                    Perubahan pkg memaksa SELURUH service masuk matrix.
golangci-lint     → matrix.module
test              → matrix.module, + coverage gate
govulncheck       → matrix.module
hadolint          → Dockerfile
sonarqube         → per service
deploy-booking    → sam build && sam deploy --stack-name rusuh-booking
deploy-user       → idem
deploy-nik-verify → idem
deploy-notification    → idem
deploy-outbox-publisher → idem
```

Deploy jobs memakai AWS OIDC (`role-to-assume: secrets.AWS_DEPLOY_ROLE_ARN`), berjalan
hanya pada `push` dan hanya bila filter service itu `true`, dan `needs: [detect-changes,
sonarqube]`.

### 17.2 Perubahan pada `detect-changes`

Untuk tiap service baru, **empat** tempat:

```yaml
# 1. outputs
outputs:
  modules: ${{ steps.set.outputs.modules }}
  booking: ${{ steps.filter.outputs.booking }}
  # … yang sudah ada …
  events: ${{ steps.filter.outputs.events }}
  queue: ${{ steps.filter.outputs.queue }}
  sse_server: ${{ steps.filter.outputs.sse_server }}
  admission_ctl: ${{ steps.filter.outputs.admission_ctl }}
  payment_api: ${{ steps.filter.outputs.payment_api }}
  sale_ops: ${{ steps.filter.outputs.sale_ops }}
  metrics_scraper: ${{ steps.filter.outputs.metrics_scraper }}

# 2. filters — perhatikan setiap filter JUGA memantau pkg/** dan go.work
filters: |
  events:
    - 'services/events/**'
    - 'pkg/**'
    - 'go.work'
  queue:
    - 'services/queue/**'
    - 'pkg/**'
    - 'go.work'
  sse_server:
    - 'services/sse-server/**'
    - 'pkg/**'
    - 'go.work'
  admission_ctl:
    - 'services/admission-ctl/**'
    - 'pkg/**'
    - 'go.work'
  payment_api:
    - 'services/payment-api/**'
    - 'pkg/**'
    - 'go.work'
  sale_ops:
    - 'services/sale-ops/**'
    - 'pkg/**'
    - 'go.work'
  metrics_scraper:
    - 'services/metrics-scraper/**'
    - 'pkg/**'
    - 'go.work'

# 3. skrip penyusun matrix
if [ "${{ steps.filter.outputs.events }}" = "true" ]; then mods+=("services/events"); fi
if [ "${{ steps.filter.outputs.queue }}" = "true" ]; then mods+=("services/queue"); fi
if [ "${{ steps.filter.outputs.sse_server }}" = "true" ]; then mods+=("services/sse-server"); fi
if [ "${{ steps.filter.outputs.admission_ctl }}" = "true" ]; then mods+=("services/admission-ctl"); fi
if [ "${{ steps.filter.outputs.payment_api }}" = "true" ]; then mods+=("services/payment-api"); fi
if [ "${{ steps.filter.outputs.sale_ops }}" = "true" ]; then mods+=("services/sale-ops"); fi
if [ "${{ steps.filter.outputs.metrics_scraper }}" = "true" ]; then mods+=("services/metrics-scraper"); fi

# 4. fallback "run all" — daftar ini dipakai bila tidak ada filter yang cocok;
#    service yang terlewat di sini TIDAK PERNAH diuji pada push ke branch baru
if [ ${#mods[@]} -eq 0 ]; then
  mods+=("pkg" "services/booking" "services/user" "services/nik-verify" \
         "services/notification" "services/outbox-publisher" \
         "services/events" "services/queue" "services/sse-server" \
         "services/admission-ctl" "services/payment-api" \
         "services/sale-ops" "services/metrics-scraper")
fi
```

> **Perubahan `pkg` memaksa 13 modul masuk matrix.** Dengan delapan paket baru (§3),
> hampir setiap PR fase awal akan menjalankan seluruh matrix: 13 lint + 13 test + 13
> govulncheck. Itu konsekuensi yang diterima dari memakai `pkg` dengan benar, tetapi ia
> mengubah waktu CI dari ~6 job menjadi ~40. Pertimbangkan `fail-fast: false` (sudah
> disetel) dan `concurrency` per-ref (sudah disetel).

### 17.3 Job deploy — dan dua di antaranya BUKAN SAM

Ini pekerjaan terbesar yang v1.0 tidak sebut sama sekali.

**Lima service Lambda** mendapat job yang disalin dari `deploy-booking`, dengan tiga
penggantian: `working-directory`, `--stack-name`, dan kondisi `if`:

| Job | `working-directory` | `--stack-name` |
|---|---|---|
| `deploy-events` | `services/events` | `rusuh-events` |
| `deploy-queue` | `services/queue` | `rusuh-queue` |
| `deploy-payment-api` | `services/payment-api` | `rusuh-payment-api` |
| `deploy-sale-ops` | `services/sale-ops` | `rusuh-sale-ops` |
| `deploy-metrics-scraper` | `services/metrics-scraper` | `rusuh-metrics-scraper` |

**Dua service ECS tidak dapat memakai SAM.** `sse-server` adalah proses HTTP long-lived di
balik ALB dan `admission-ctl` adalah loop tanpa HTTP; keduanya ECS Fargate (ADR §4), dan
`AWS::Serverless::Function` tidak dapat mendeskripsikan satu pun. Pipeline ini **belum
pernah** mendeploy container ke ECS, jadi jalurnya dibangun dari nol:

```yaml
deploy-sse-server:
  name: Deploy sse-server → ECS Fargate
  needs: [detect-changes, sonarqube]
  runs-on: ubuntu-latest
  if: github.event_name == 'push' && needs.detect-changes.outputs.sse_server == 'true'
  permissions:
    id-token: write
    contents: read
  defaults:
    run:
      working-directory: services/sse-server
  steps:
    - uses: actions/checkout@…
    - name: Configure AWS credentials (OIDC)
      uses: aws-actions/configure-aws-credentials@…
      with:
        role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
        aws-region: ${{ vars.AWS_REGION }}
    - name: Login to ECR
      uses: aws-actions/amazon-ecr-login@…
      id: ecr
    - name: Build and push image
      run: |
        IMAGE="${{ steps.ecr.outputs.registry }}/rusuh-sse-server:${{ github.sha }}"
        docker build -t "$IMAGE" .
        docker push "$IMAGE"
        echo "IMAGE=$IMAGE" >> "$GITHUB_ENV"
    - name: Render task definition
      uses: aws-actions/amazon-ecs-render-task-definition@…
      id: taskdef
      with:
        task-definition: services/sse-server/taskdef.json
        container-name: sse-server
        image: ${{ env.IMAGE }}
    - name: Deploy ECS service
      uses: aws-actions/amazon-ecs-deploy-task-definition@…
      with:
        task-definition: ${{ steps.taskdef.outputs.task-definition }}
        service: rusuh-sse-server
        cluster: ${{ vars.ECS_CLUSTER }}
        wait-for-service-stability: true
```

`deploy-admission-ctl` identik, dengan `rusuh-admission-ctl` dan **tanpa** target group —
ia tidak melayani trafik.

**Tiga hal yang ikut dibutuhkan dan ada di luar repo ini**:

| Kebutuhan | Pemilik |
|---|---|
| Repository ECR `rusuh-sse-server`, `rusuh-admission-ctl` | infra |
| Cluster ECS, ALB + target group + listener (`events.my-rusuh.my.id`, HTTP/2, idle 3600s, deregistration delay 60s, slow start 30s, sticky OFF, health `/health`) | infra |
| IAM: role deploy diperluas untuk `ecr:*` dan `ecs:UpdateService`, `iam:PassRole` task role | infra |

> **`wait-for-service-stability: true` pada `sse-server` bisa memakan belasan menit** saat
> `desired_count` tinggi. Itu diinginkan: deploy yang "berhasil" sementara task baru
> crash-loop adalah deploy yang gagal. Tetapi **jangan** mendeploy selama sale aktif —
> rolling deploy memutus koneksi SSE dan memicu reconnect berjitter pada seluruh armada
> (§8.7). Tambahkan guard manual di checklist pra-sale, bukan di CI.

### 17.4 Coverage gate 80% — per modul, dan ini mengikat

`cicd.yml` job `test`:

```bash
grep -vE '/(cmd|_mock)/' coverage.out > coverage.gate.out || cp coverage.out coverage.gate.out
coverage=$(go tool cover -func=coverage.gate.out | awk '/^total:/ {sub(/%/,"",$3); print $3}')
awk -v c="$coverage" 'BEGIN { if (c+0 < 80) { print "Code coverage below threshold!"; exit 1 } }'
```

**80% atas seluruh kode kecuali `cmd/` dan `_mock/`.** v1.0 tidak menyebut gate ini, dan ia
pendorong effort terbesar pada dua service:

| Service | Yang sulit diuji | Pendekatan |
|---|---|---|
| `sse-server` | `Manager` fan-out dengan 500-chunk + backpressure; stale detector berbasis waktu; SIGTERM | Clock yang dapat di-inject; `Conn` sebagai interface dengan fake yang dapat dibuat gagal menulis; `httptest.NewRecorder` tidak cukup — butuh `ResponseRecorder` yang mengimplementasikan `http.Flusher` |
| `admission-ctl` | Tick dengan 9 langkah, masing-masing bersyarat | Setiap langkah adalah fungsi terpisah dengan port-nya sendiri (§9.1 sudah memecahnya demikian, justru untuk ini); `tick.go` hanya orkestrasi dan ia yang paling tipis |

Dua teknik yang membuat gate ini tercapai tanpa test teater:

1. **Clock dan ID generator di-inject** (`func() time.Time`, `pkg/idgen` di-wrap interface).
   Tanpa itu setiap langkah berbasis tenggat tidak dapat diuji secara deterministik.
2. **`miniredis` untuk Lua.** `advance_now_serving.lua` dan `hold_booking.lua` adalah
   logika bisnis, dan mereka harus punya test tabel sendiri — bukan hanya diuji lewat
   usecase. `miniredis` mendukung `EVAL`; `SCRIPT LOAD`/`EVALSHA` di-fallback ke `EVAL`
   oleh `pkg/redisscript` (§3.4), yang membuat ini bekerja.

### 17.5 `hadolint` dan `sonarqube`

`hadolint` memeriksa Dockerfile. Tujuh service baru butuh satu masing-masing — termasuk
lima service Lambda, yang mengikuti pola `booking/Dockerfile`.

`sonarqube` butuh `sonar-project.properties` per service dengan `sonar.projectKey` unik.
Salin dari service terdekat dan ganti key; jangan gabungkan tujuh service ke satu project,
karena `deploy-*` bergantung pada job ini dan satu project gabungan membuat kegagalan
quality gate satu service memblokir deploy semua.

### 17.6 Root `Makefile`

**Tidak ada perubahan yang diperlukan.**

```makefile
MODULES := pkg services/*
```

Glob itu menangkap setiap service baru secara otomatis pada `build`, `test`, `tidy`,
`isolation-check` dan `lint`.

v1.0 mengusulkan target `test-events`, `test-queue`, … Itu menduplikasi `make test` dan
tidak mengikuti pola file ini (yang hanya punya lima target workspace-wide plus dua
passthrough `booking-*`). Bila passthrough per service diinginkan, polanya adalah:

```makefile
sse-server-run:
	$(MAKE) -C services/sse-server run

sse-server-docker-build:
	$(MAKE) -C services/sse-server docker-build
```

— dan hanya untuk service yang benar-benar sering dijalankan lokal.

---

## 18. Peta Kepemilikan

Satu tabel per sumber daya, karena pertanyaan "siapa yang menulis ini" adalah yang paling
sering salah dijawab ketika tujuh service berbagi satu Redis dan satu Aurora.

### 18.1 Kunci Redis

| Kunci | Penulis | Pembaca |
|---|---|---|
| `queue:{ev}:issued` | queue (INCR) | queue, events, metrics-scraper |
| `queue:{ev}` (`now_serving`, `rate_per_min`) | admission-ctl (Lua + HSET) | queue, sse-server, events, metrics-scraper |
| `queue:{ev}:state` | sale-ops (seed, open), admission-ctl (Lua, CLOSED) | semua |
| `queue:{ev}:capacity` | sale-ops | queue, events |
| `queue:{ev}:by_ticket` | queue (ZADD), admission-ctl (Lua ZREMRANGEBYSCORE, sweep ZREM) | — |
| `queue:{ev}:sold_out_at`, `:resume` | admission-ctl (Lua) | admission-ctl |
| `slots:open:{ev}` | **Lua INCR saja**; DECR oleh admission-ctl, payment-api, booking (terminal) | admission-ctl, queue, events, metrics-scraper |
| `regrant:{ev}` | admission-ctl (SADD), Lua (SPOP) | — |
| `expiries:{ev}` | admission-ctl, booking (rescore), payment-api (ZREM inline) | admission-ctl |
| `qpersist:{ev}` | queue (RPUSH), admission-ctl (LTRIM) | admission-ctl (LRANGE) |
| `token:{uuid}` | queue (HSET), admission-ctl (DEL terminal), payment-api (DEL CONFIRMED) | queue, sse-server, grantauth |
| `settled:{uuid}` | payment-api (inline) | queue |
| `grant:{token}` | admission-ctl, queue (bypass FR-05); DEL oleh admission-ctl, payment-api, booking | `pkg/grantauth` (booking, payment-api, nik-verify), queue |
| `acct:{ev}:{id}` | queue (SET NX); DEL oleh admission-ctl, payment-api, booking | queue |
| `stock:{ev}:{cat}` | sale-ops (seed), booking (Lua), admission-ctl (sweep INCR + rekonsiliasi) | booking, events, metrics-scraper |
| `nik:used:{ev}:{hmac}` | booking (Lua SET NX), admission-ctl (DEL + restore), payment-api (DEL expire) | booking (Lua) |
| `nik:valid:{hmac}` | nik-verify | nik-verify |
| `ctl:leader`, `ctl:epoch` | admission-ctl | admission-ctl |
| `pause:{ev}`, `pause:{ev}:grace_applied` | admission-ctl | queue, sse-server, metrics-scraper |
| `breaker:dukcapil:*` | nik-verify | admission-ctl, metrics-scraper |
| `breaker:payment-gateway:*` | payment-api | admission-ctl, metrics-scraper |
| `rl:*` | pemanggil masing-masing | — |
| `processed:{outbox_id}` | payment-api (inline), outbox-publisher | keduanya |

### 18.2 Transisi status `bookings`

| Transisi | Pemilik | DECR `slots:open`? |
|---|---|---|
| — → GRANTED | admission-ctl (§9.6), queue (bypass FR-05) | **INCR +1**, oleh Lua |
| GRANTED → SELECTING | **booking** (§5.3, keputusan §2.4) | ❌ slot yang sama |
| SELECTING → HELD | booking (§5.4) | ❌ slot yang sama |
| HELD → AWAITING_PAYMENT | payment-api (§10.4) | ❌ slot yang sama |
| AWAITING_PAYMENT → HELD (deny) | payment-api (§10.4, §10.5, §10.6) | ❌ |
| AWAITING_PAYMENT → PENDING_CANCELLATION | admission-ctl (§9.5) | ❌ stok masih di-hold selama grace |
| PENDING_CANCELLATION → CONFIRMED | payment-api webhook (§10.5) | ✅ terminal |
| AWAITING_PAYMENT → CONFIRMED | payment-api webhook inline, fallback outbox-publisher | ✅ terminal |
| GRANTED → LAPSED | admission-ctl (§9.5) | ✅ terminal, **final** |
| SELECTING → EXPIRED | admission-ctl (§9.5) | ✅ terminal |
| HELD → EXPIRED | admission-ctl (§9.5) | ✅ terminal + lepas stok + `DEL nik:used` + `DELETE nik_claims` |
| PENDING_CANCELLATION → EXPIRED | admission-ctl (§9.5) | ✅ idem + Midtrans expire |
| AWAITING_PAYMENT → EXPIRED (gateway) | payment-api webhook (§10.5) | ✅ idem |
| apa pun → ABANDONED | booking (§5.6) | ✅ terminal |

Tabel ini **adalah** `pkg/bookingstate` (§3.7). Bila keduanya berbeda, kode yang menang
dan dokumen yang diperbaiki.

### 18.3 Tabel Aurora

| Tabel | Penulis | Pembaca |
|---|---|---|
| `accounts`, `saved_niks`, `consent_records`, `magic_tokens`, `refresh_tokens` | user | user, nik-verify (bukti) |
| `events`, `ticket_categories` | back-office (di luar lingkup), sale-ops (snapshot `available`) | events, booking, admission-ctl |
| `event_queue_state` | sale-ops, admission-ctl | events, admission-ctl, metrics-scraper |
| `bookings` | admission-ctl, booking, payment-api, queue (bypass) | semua di atas + user, metrics-scraper |
| `booking_items` | booking | payment-api, admission-ctl |
| `nik_claims` | booking (INSERT), admission-ctl + payment-api (DELETE) | admission-ctl (rekonsiliasi) |
| `payment_events` | payment-api | payment-api |
| `tickets` | payment-api, outbox-publisher (fallback) | user, booking |
| `queue_entries` | admission-ctl (drain) | sale-ops (rebuild), user, metrics-scraper |
| `outbox` | user, booking, payment-api, admission-ctl (via `pkg/outbox`) | outbox-publisher |

### 18.4 Endpoint DB: writer vs reader

| Service | Endpoint | Alasan |
|---|---|---|
| `events` | **reader** | seluruhnya baca; halaman utama di-cache 30 detik |
| `metrics-scraper` | **reader** | seluruhnya baca; toleran lag |
| `queue` | reader + **writer** | `/me` dan `/serving` baca; bypass FR-05 meng-INSERT GRANTED |
| `booking`, `payment-api`, `sale-ops` | **writer** | menulis |
| `admission-ctl` | **writer** | sweep dan rekonsiliasi **wajib** writer — lag replika terbaca sebagai drift semu (ADR §6, §7) |
| `user` | writer (sudah) | |

---

## 19. Urutan Pengerjaan

Urutan v1.0 mengklaim "setiap langkah hanya bergantung pada yang sebelumnya" dan itu tidak
berlaku: booking (langkah 7) butuh baris GRANTED yang hanya admission-ctl (langkah 10)
tulis, dan queue (langkah 8) butuh Lua yang baru datang di langkah 10. §15 v1.0 juga
mengatakan sse-server bergantung pada "—" sementara §14 mengatakan ia butuh queue.

Urutan di bawah disusun supaya **setiap fase dapat diuji ujung-ke-ujung ketika selesai**.

### Fase A — Fondasi (tanpa blocker, dapat diparalelkan)

| # | Artefak | Catatan |
|---|---|---|
| A1 | `pkg/rediskeys` (§3.1) | **PERTAMA.** §13.4 bergantung padanya, dan migrasi kunci nik-verify harus terjadi sebelum ada pembaca kedua |
| A2 | `pkg/bookingstate` (§3.7) | Tabel transisi sebagai kode; test-nya tabel kebenaran |
| A3 | `pkg/ratelimit`, `pkg/breaker` (§3.2, §3.3) | |
| A4 | `pkg/redisscript` + tiga skrip (§3.4, §14) | Test tabel dengan `miniredis` |
| A5 | `pkg/outbox` (§3.6) | |
| A6 | `pkg/grantauth` (§3.5) | Butuh A1 |
| A7 | `pkg/sseevent` (§3.8) | |
| A8 | `pkg/events` + `payment.confirmed`, `payment.initiated` (§13.1) | |
| A9 | `pkg/apinizer` + `server_time` (§3.9) | Menyentuh respons SELURUH service yang ada — jalankan test mereka |
| A10 | `pkg/apperror` + kode baru (§13.5) | |
| A11 | Migration 0005, 0006, 0007 (§4) | |
| A12 | Migrasi kunci nik-verify (§13.4) | Butuh A1, A3 |
| A13 | `pkg/eticket` (§3.10) | Tanpa blocker — stdlib saja. Konsumennya di E6 |

**Dapat diuji di akhir Fase A**: `make test` seluruh workspace hijau; migration up/down
bersih; nik-verify masih lulus test-nya dengan kunci baru.

### Fase B — Jalur yang dapat dilihat customer sebelum sale dibuka

| # | Artefak | Butuh |
|---|---|---|
| B1 | `services/sale-ops` (§11) | A11 |
| B2 | `services/events` (§6) | A11, B1 (untuk data yang dapat dibaca) |

**Dapat diuji**: seed event → `GET /events` menampilkan `queue_state: NOT_OPEN` +
`sale_starts_at`; `GET /events/{id}` menampilkan kategori, harga, stok, `captcha.site_key`;
`sale-ops open` memindahkan state ke OPEN dan `GET /events` berubah menjadi `OPEN`/`DIRECT`.

> B2 sebelum queue: halaman utama adalah satu-satunya layar yang dilihat 300.000 orang
> sebelum gerbang dibuka, dan ia tidak bergantung pada apa pun kecuali katalog.

### Fase C — Antrian dan admission (satu unit, tidak dapat dipisah)

| # | Artefak | Butuh |
|---|---|---|
| C1 | `services/queue` — `/join`, `/me`, `/serving` **tanpa** bypass FR-05 | A1–A6, B1 |
| C2 | `services/admission-ctl` — leader, pause, advance, broadcast, drain | A1–A5, A7, A11 |
| C3 | `services/sse-server` | A1, A3, A7 |
| C4 | `services/queue` — bypass FR-05 (§7.2 langkah 10) | C2 (skrip + pola INSERT GRANTED) |

**Dapat diuji**: join → cookie + `ticket_no`; stream membuka dan mengirim snapshot;
admission-ctl memajukan `now_serving` dan klien melihatnya bergerak; `grant` sampai ke
token yang tepat; `/me` mengembalikan `GRANTED` + `grant_token`; kill leader → standby
mengambil alih ≤ 5,5 detik; kill **kedua** task → `status {stale:true}` sampai ke klien
≤ 30 detik dan `CtlHeartbeatAge` Minimum naik.

> **C1–C3 harus selesai bersama.** Masing-masing sendirian tidak dapat diuji: queue tanpa
> admission-ctl menerbitkan nomor yang tidak pernah maju; admission-ctl tanpa queue tidak
> punya token untuk di-grant; sse-server tanpa keduanya tidak punya apa pun untuk
> disiarkan. v1.0 memberinya tiga langkah berurutan.
>
> C4 dipisah **justru karena** ia bergantung pada C2: ia menyalin pola INSERT GRANTED, dan
> menulisnya sebelum C2 ada berarti menulisnya dua kali.

### Fase D — Booking

| # | Artefak | Butuh |
|---|---|---|
| D1 | `services/booking` — `GET /events/{id}/tickets` + claim (§5.3) | C2 (baris GRANTED) |
| D2 | `services/booking` — `POST /bookings` (§5.4) | D1, A4 |
| D3 | `services/booking` — `GET /bookings/{id}`, `/tickets`, abandon (§5.5, §5.6) | D2 |
| D4 | Sweep lengkap di admission-ctl (§9.5) — lima status | D2 |

**Dapat diuji**: grant → buka Detail Ticket Page → status menjadi SELECTING; submit →
HELD + `booking_items` + `nik_claims`; submit ulang → booking yang sama, 200, bukan
`NIK_TAKEN`; NIK yang sama di dua booking → ditolak; rollback Aurora setelah Lua berhasil →
stok dan NIK kembali; TTL booking habis → `SELECTING → EXPIRED` dan slot dilepas.

> **D4 setelah D2**, bukan bersama C2: sweep `HELD → EXPIRED` tidak dapat diuji sebelum
> ada yang dapat membuat baris `HELD`. Yang dibangun di C2 adalah sweep `GRANTED`/
> `SELECTING` saja; itu cukup untuk Fase C dan jujur tentang apa yang sudah ada.

### Fase E — Pembayaran

| # | Artefak | Butuh |
|---|---|---|
| E1 | `services/payment-api` — `/methods`, `/{id}` initiate, `/status` | D2, A3, A5 |
| E2 | `services/payment-api` — webhook + blok inline (§10.5) | E1, A4, A7 |
| E3 | `services/payment-api` — `/retry` (§10.6) | E1 |
| E4 | `services/outbox-publisher` — efek samping per tipe (§13.2) | E2, A8 |
| E5 | `services/notification` — `booking.expired` multi-holder (§13.3) | E4 |
| E6 | **Token e-tiket (§13.7)** — `notification` menerbitkan `?t=`, `user` memverifikasinya | A13, E4 |
| E6 | `services/payment-api` — `/admin/reconciliation/disputed` (§10.7) | E2 |
| E7 | Rekonsiliasi stok + `nik_claims` + charge reconciler (§9.8, §9.9) | E2 |

**Dapat diuji**: charge sandbox → VA; charge ganda bersamaan → satu charge, dua respons
identik; webhook diputar ulang → 200 tanpa efek; webhook `settled` setelah EXPIRED →
`payment_disputed` + email + muncul di `/admin/reconciliation/disputed`; deny → HELD dengan
attempt naik; ganti metode saat pending → VA lama di-cancel, TTL tidak mundur; TTL
pembayaran habis → `PENDING_CANCELLATION` lalu EXPIRED + Midtrans expire.

> **E6 rilis sebagai satu unit.** Tautan bertoken yang diverifikasi rute lama menjawab 401,
> dan rute baru tanpa tautan bertoken tidak menolong siapa pun. Ia ditaruh di Fase E karena
> `ticket.issue` baru benar-benar terbit setelah webhook pembayaran bekerja (E4) — sebelum
> itu tidak ada e-tiket untuk dibuka. Ini satu-satunya butir §13 yang menutup **requirement
> yang tidak terpenuhi** (FR-26), bukan merapikan yang sudah jalan.

### Fase F — Observability dan penutup

| # | Artefak | Butuh |
|---|---|---|
| F1 | `services/metrics-scraper` (§12) | E2 (agar `ConversionRate` punya data) |
| F2 | `CtlHeartbeatAge` di sse-server (§8.6) | C3 |
| F3 | `sale-ops rebuild` (§11.4) | C2, D2 |
| F4 | Migrasi `services/user` ke `pkg/outbox` (§13.6) | A5, E4 |
| F5 | CI: 7 job deploy + jalur ECS (§17.3) | semua |

> **F2 dapat dikerjakan di Fase C** dan sebaiknya demikian — ia bagian dari pengujian
> "kill kedua task ctl". Ditulis di sini supaya tidak hilang bila C3 dianggap selesai
> tanpanya.

> **F5 terakhir, tetapi `detect-changes` (§17.2) harus diperbarui di setiap fase.** Service
> yang modulnya ada di `go.work` tetapi tidak ada di `cicd.yml` tidak pernah di-lint, di-test,
> atau di-scan — dan karena fallback "run all" hanya menyala bila **tidak ada** filter yang
> cocok, PR yang menyentuh service lain akan tetap hijau sambil melewatkannya sepenuhnya.

---

## 20. Ringkasan Tabel

| # | Artefak | Tipe | Bergantung pada | Endpoint / fungsi |
|---|---|---|---|---|
| 1 | `pkg/rediskeys` | Paket | — | Skema kunci Redis terpusat |
| 2 | `pkg/bookingstate` | Paket | — | Status + transisi + kepemilikan `slots:open` |
| 3 | `pkg/ratelimit` | Paket | 1 | Token bucket Redis generik |
| 4 | `pkg/breaker` | Paket | 1 | Circuit breaker Redis generik |
| 5 | `pkg/redisscript` | Paket | — | Loader EVALSHA + 5 skrip Lua |
| 6 | `pkg/outbox` | Paket | `pkg/events` | Record + writer in-transaction + lease |
| 7 | `pkg/grantauth` | Paket | 1 | Middleware grant + device binding |
| 8 | `pkg/sseevent` | Paket | — | Nama event + payload SSE |
| 9 | `pkg/events` (+2 tipe) | Paket | — | `payment.confirmed`, `payment.initiated` |
| 10 | `pkg/apinizer` (+`server_time`) | Paket | — | BR-10 pada setiap respons REST |
| 11 | `pkg/apperror` (+23 kode) | Paket | — | Appendix A kontrak |
| 12 | Migration 0005 | SQL | — | `events` extend, `ticket_categories`, `event_queue_state` |
| 13 | Migration 0006 | SQL | 0004 | `bookings.grant_token`, `booking_items`, `nik_claims`, `payment_events`, `tickets.booking_item_id` |
| 14 | Migration 0007 | SQL | 0006 | Indeks BR-07, event+status, orphan charge, disputed |
| 15 | `services/sale-ops` | Lambda (EventBridge) | 12 | `seed`, `open`, `close`, `rebuild` |
| 16 | `services/events` | Lambda | 12, 15 | `GET /events`, `GET /events/{id}` |
| 17 | `services/queue` | Lambda | 1–7, 15 | `POST …/join`, `GET …/me`, `GET …/serving` |
| 18 | `services/admission-ctl` | **ECS Fargate** | 1–6, 8, 12–14 | Loop 500 ms: lease, pause+grace, sweep ×5 status, advance, broadcast, 2 rekonsiliasi, drain, charge reconciler |
| 19 | `services/sse-server` | **ECS Fargate** | 1, 3, 8 | `GET …/queue/stream`, `CtlHeartbeatAge` |
| 20 | `services/booking` (completion) | Lambda | 13, 5, 7, 18 | `GET /events/{id}/tickets`, `POST /bookings`, `GET /bookings/{id}` |
| 21 | `services/payment-api` | Lambda | 13, 4–7, 20 | `/payments/*`, webhook, `/admin/reconciliation/disputed` |
| 22 | `services/outbox-publisher` (perubahan) | Lambda | 6, 9, 21 | Efek samping per tipe, bukan hanya SQS |
| 23 | `services/notification` (perubahan) | Lambda | 9, 22 | `booking.expired` multi-holder, locale |
| 24 | `services/nik-verify` (migrasi) | Lambda | 1, 3, 4, 7 | Kunci Redis + `pkg/grantauth` |
| 25 | `services/metrics-scraper` | Lambda (EventBridge) | 12–14 | EMF `RusuhBooking/Queue` — FR-33/34/35, NFR-09 |
| 26 | `go.work` + 7 `go.mod` | Config | — | `replace … => ../../pkg` wajib |
| 27 | Scaffold 7 × 11 file | Config | — | Dockerfile, template/taskdef, sonar, golangci, Makefile, env |
| 28 | CI/CD | YAML | semua | 7 filter + 7 job deploy, 2 di antaranya jalur ECS baru |

---

## 21. Yang Masih Terbuka

### 21.1 Keputusan yang dibutuhkan sebelum coding fase tertentu

| # | Item | Diblokir | Pemilik |
|---|---|---|---|
| 1 | `SALE_WINDOW_SECONDS` — belum pernah punya nilai, dan kini memikul TTL `acct`, TTL `token:{uuid}` **dan** `ACCESS_TOKEN_TTL_SECONDS` (ADR §17 #10). Fallback 21600 | Fase C | Tim + pemilik tender |
| 2 | `MAX_CONCURRENT_SLOTS` — tuas utama untuk okupansi slot (ADR §17 #5) | Fase C | Tim |
| 3 | `MAX_DEVICE_REBINDS`, dan apakah pengikatan device ditegakkan di web (ADR §17 #12) | `pkg/grantauth` (A6) | Tim + QA |
| 4 | `NIK_USED_TTL_SECONDS` — "durasi sale + margin"; harus **melampaui akhir sale** | Fase D | Tim |
| 5 | Kapasitas koneksi per task ECS — menentukan `desired_count` (ADR §17 #4) | Fase F deploy | Tim, T-14 hari |
| 6 | Provider CAPTCHA + kuota untuk 300.000 verifikasi dalam ~60 detik (ADR §17 #8) | Fase C | Tim, T-7 hari |
| 7 | IP webhook Midtrans untuk allowlist resource policy (ADR §17 #9) | Fase E | Tim, T-7 hari |
| 8 | `DUKCAPIL_RATE_LIMIT_RPS` dan SLA (ADR §17 #1, #2) | — (sudah terimplementasi, nilainya saja) | Pemilik tender |

### 21.2 Yang sengaja tidak dibangun di POC

| Item | Alasan |
|---|---|
| Refund otomatis | Dicatat `payment_disputed`, direkonsiliasi manual via §10.7. Refund memindahkan uang keluar; jalur otomatis berdasarkan callback adalah jalur yang paling mahal bila salah (ADR §9) |
| Reversal handler (`settlement → deny`) | Hanya terjadi pada Permata, Mandiri Bill, Indomaret — ketiganya tidak dibuka (§10.3) |
| Kartu kredit, OTC | Tidak dapat menghormati TTL di sisi gateway; mengeluarkan cakupan PCI-DSS (§10.3) |
| Push notification | ADR §1 |
| Nomor bangku | ADR §4 — tiket berisi QR, kategori, NIK pemegang |
| Endpoint self-service hapus PII | Operasional manual; lantai "tidak sebelum event berlangsung" karena `nik_claims` menegakkan BR-02 (ADR §8) |
| Purge berjadwal `queue_entries` | Tumbuh 300.000 baris/event; tindak lanjut sebelum produksi (ADR §8) |
| `payment.refund` sebagai tipe outbox | §13.1 |
| `POST /bookings/{id}/abandon` diekspos | Usecase dibangun, rute tidak didaftarkan — kontrak v2.5 tidak memuatnya (§5.6) |
| Pengawas liveness di klien | Ditunda; setelah leader lease yang tersisa hanya sse-server macet sendiri atau TCP half-open, dan nol kode klien lebih berharga (ADR §17) |
| WAF | Digantikan rate limit Redis + CAPTCHA; degradasi NFR-05 yang diterima, harus dipulihkan sebelum produksi (ADR §16) |

### 21.3 Ketidaksesuaian dokumen yang ditemukan saat menyusun rencana ini

Diputuskan di §2 agar implementasi tidak terhenti, tetapi **dokumen sumbernya sebaiknya
diperbaiki** supaya keputusan ini tidak perlu ditemukan ulang.

**Butir 7–9 sudah diperbaiki** di api-contracts v2.5 dan ditandai selesai di bawah — jangan
dibuka ulang. Butir 1–6 dan 10 menyangkut **ADR-005**, yang sengaja tidak disentuh, sehingga
semuanya masih berlaku.

| # | Dokumen | Isu | Diselesaikan di |
|---|---|---|---|
| 1 | ADR-005 | §4 menulis `bookings(… queue_token …)`; §6, §9, §12 menulis `grant_token`. Keduanya dibutuhkan, dan keduanya berbeda | §2.1 |
| 2 | ADR-005 | §6 menulis `SELECT capacity FROM event_categories WHERE id = $cat` — nama tabel yang tidak ada di skema §4, dan kolom `id` yang tidak cocok dengan kunci (`event_id`, `category_id`) | §2.2 |
| 3 | ADR-005 | §4 menulis `tickets(… qr_payload …)`; `0002` sudah memakai `qr_nonce` | §2.3 |
| 4 | ADR-005 | §6 tabel kepemilikan menugaskan `GRANTED → SELECTING (claim)` ke booking-service, tetapi tidak ada endpoint yang dinyatakan menjalankannya | §2.4 |
| 5 | ADR-005 | §9 jalur "settled atas booking EXPIRED" menyebut `payment.refund` di satu tempat dan `payment.failed_notice` di tempat lain | §10.5 memakai `payment.failed_notice`, sesuai `pkg/events` yang sudah ada |
| 6 | ADR-005 | §12 prosedur rebuild tidak memuat `grant:{token}` maupun `expiries:{ev}` | §11.4 menambahkannya |
| ~~7~~ | api-contracts | ~~§4.3 memuat `holder_name` dan error "NIK Name Mismatch" melawan FR-15a~~ | **SELESAI di v2.5** — keduanya dihapus (changelog butir 53). §5.4 tetap tidak mengembalikan `holder_name` |
| ~~8~~ | api-contracts | ~~§5.4 contoh `FAILED` memuat `reason: "Kartu ditolak penerbit"` padahal kartu tidak dibuka~~ | **SELESAI di v2.5** — diganti "Pembayaran ditolak penyedia", dengan catatan bahwa `reason` adalah passthrough `status_message` untuk support, bukan teks tampilan |
| ~~9~~ | api-contracts | ~~§5.5 contoh request webhook memakai bentuk generik (`external_id`, `status: "PAID"`)~~ | **SELESAI di v2.5** — diganti body Midtrans Core API sebenarnya, header `X-Callback-Signature` dihapus (Midtrans tidak mengirimnya), dan `gateway_event_id` diturunkan dari `order_id + ':' + transaction_status + ':' + status_code` (changelog butir 49) |
| 10 | ADR-005 | §12 mencatat mitigasi "N tick gagal berturut-turut diperlakukan sebagai transisi `paused`" sebagai belum diputuskan | Tidak dibangun; §9.4 |

### 21.4 Ditambahkan di luar ADR, perlu konfirmasi

| # | Item | Alasan | §  |
|---|---|---|---|
| 1 | `services/sale-ops` sebagai service tersendiri | ADR §6 menyebut "job terjadwal (EventBridge)" tanpa menugaskannya; seeding Redis tidak disebut sama sekali | 11 |
| 2 | `ticket_categories.available` sebagai snapshot | Dibutuhkan `GET /events` saat Redis dingin dan oleh `sale-ops seed` | 4.1 |
| 3 | `event_queue_state.opened_at`, `closed_at` | ADR hanya menyebut `state` dan `sold_out_at`; keduanya dibutuhkan untuk audit dan untuk `SoldOutHoldSeconds` | 4.1 |
| 4 | `payment.initiated` sebagai tipe outbox | ADR §9 langkah 1 menulis `INSERT INTO outbox (type='payment.initiated')` tetapi `pkg/events` tidak punya | 13.1 |
| 5 | Batas kedua "3 join/menit per akun" | Kontrak §3.3 menyebutnya saat menjelaskan mengapa pemulihan harus READ, tetapi §3.1 tidak mendaftarkannya sebagai respons 429 | 7.2 |
| 6 | `token_bucket.lua` mengembalikan `retry_after_ms` | Kontrak §4.2 dan §3.1 memuat `retry_after` pada 429; skrip nik-verify hari ini hanya mengembalikan 0/1 | 3.2 |

---

## Referensi

- `documentations/adr/ADR-005-final-architecture-decision.md` — arsitektur of record
- `documentations/api/api-contracts.md` **v2.5** — kontrak per endpoint
- `documentations/requirement/requirements.md` v1.0 — FR-01..35, NFR-01..11, BR-01..13
- `documentations/payment-method/` — dua metode Midtrans Core API
- `be-rusuh-war-ticketing/migrations/README.md` — konvensi migration
- `be-rusuh-war-ticketing/.github/workflows/cicd.yml` — pipeline yang §17 ditulis terhadapnya
- ADR-001 (model counter antrian), ADR-002 (`INCR` + `now_serving`) — keputusan berlaku;
  skema key dan ilustrasi polling di dalamnya digantikan ADR-005
- ADR-003, ADR-004 — superseded by ADR-005
