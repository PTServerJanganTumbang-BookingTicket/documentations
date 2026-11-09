# Rusuh Booking Tiket — API Contracts

**Versi**: 2.12.0  
**Tanggal**: 9 November 2026  
**Status**: Draft  
**Base URL**: `https://api.my-rusuh.my.id` (tanpa versi di path untuk saat ini; v2.9). Stream: `https://events.my-rusuh.my.id`  
**Selaras dengan**: ADR-005 (Keputusan Arsitektur Final — SSE + ECS Fargate), diamandemen ADR-006 (antrian)  
**Pasangan mesin**: `rusuh-booking-apidog.json` (Apidog) — **sumber kebenaran bentuk respons**. Dokumen ini menjelaskan perilaku dan alasannya; keduanya wajib sinkron. `rusuh-booking-openapi.yaml` diarsipkan (`archive/`) dan tidak dipelihara lagi

> **v2.12 (bentuk event stream seragam, ADR-006 §4.2):** setiap event bernama membawa `state` dan `serverTime`; `stale.reason` diganti `message` (teks siap tampil; **breaking** bagi klien yang membaca `reason`, field itu hanya diagnostik); `sold-out` dan `reconnect` menambah `state`, `message`, `serverTime` (`reconnect` tidak lagi `data: {}`). Lihat Appendix C butir 92.
>
> **v2.12** (ADR-006 §8.14, **tidak breaking** bagi klien yang merutekan berdasarkan respons): **tanpa langsung-masuk
> (straight-in).** Join **tidak lagi menunggu grant** dan tidak pernah menjawab `granted: true`; jawabannya selalu
> `queued` atau `rejoined`, seketika. Bila ada tempat kosong, tick admission-ctl berikutnya (≤ 500 ms) memasukkan
> customer dan klien tahu lewat stream + `GET /queue/me`. Status `DIRECT` ("Beli Tiket") tetap sebagai label
> tombol dan tetap memanggil join. v2.11 di bawah **dicabut**. Lihat Appendix C, v2.11 → v2.12.
>
> ~~**v2.11** (ADR-006 §8.11)~~ *(dicabut oleh v2.12)*: **semua yang muat langsung masuk.** Join kini menunggu grant
> (≤ 1 detik) bagi **setiap** pendatang yang nomornya muat di tempat kosong, bukan hanya nomor pertama di antrian
> kosong. Saat penjualan dibuka, batch pertama (hingga `MAX_CONCURRENT_SLOTS` orang) menerima `granted: true`
> langsung dari join dan tidak melihat ruang tunggu. Bentuk respons tidak berubah, tetapi **`granted: true` kini
> bisa datang dari tombol "Masuk Antrian" (`JOIN_QUEUE`) juga**: klien merutekan berdasarkan respons, bukan
> tombol. Lihat Appendix C, v2.10 → v2.11.
>
> **v2.10** (ADR-006 §8.10, **breaking**): status antrian **`SOLD_OUT` diganti `FULL_BOOKED`** (semua kursi sedang di-hold,
> **reversibel**). Berlaku untuk `queue:{ev}.state`, payload dan event SSE `status`, `/queue/serving`, error join
> 409 (`error_code` dan `data.queue_state`), `sold_out_at` → `full_booked_at`, `soldOutAt` → `fullBookedAt`,
> `SOLD_OUT_HOLD_SECONDS` → `FULL_BOOKED_HOLD_SECONDS`. **`CLOSED`, event `sold-out` dan `SALE_CLOSED` (410) tidak
> berubah**: `sold-out` tetap berarti sale selesai dan klien wajib `close()`. `status: SOLD_OUT` pada halaman utama
> (§2.1, sumbu stok) **tidak berubah** — lihat catatan di §2.1. Lihat Appendix C, v2.9 → v2.10.
>
> **v2.9** (ADR-006 §8.9): **`queue_token` hanya berlaku bagi pemiliknya, tanpa rebind.** Token tamu dari
> perangkat lain → `401 DEVICE_MISMATCH`; token akun dipakai akun lain atau tamu → `401 UNAUTHORIZED`. Klien
> **menghapus token antriannya** pada kedua error itu lalu join lagi (tempat baru). Cookie `queue_token` kini
> `Path=/events/{event_id}` (satu cookie per event). Format `X-Device-Id` tidak lagi divalidasi. Lihat Appendix C,
> v2.8 → v2.9.
>
> **v2.8** (join diselaraskan dengan implementasi `services/queue`): kode baru **`SOLD_OUT`** (409,
> pendatang baru saat stok habis), **`QUEUE_NOT_READY`** (503) dan **`INTERNAL_ERROR`** (5xx tanpa kode
> sendiri); cookie `queue_token` kini `Domain=rusuhtiket.com; Path=/`; `sse_url` menjadi **URL absolut**
> host stream; §3.1 mendaftar semua respons join. Lihat Appendix C, v2.7 → v2.8.
>
> **v2.7** (ADR-006 §4.2, §4.3, §6.2): **pause admission dihapus** — field `paused` hilang dari
> payload SSE dan `/queue/serving`, error `ADMISSION_PAUSED` diganti `PAYMENT_GATEWAY_UNAVAILABLE`.
> Event SSE **`status` kini membawa `state`** dan dikirim tiap transisi (`OPEN ↔ SOLD_OUT`,
> `→ CLOSED`); indikator basi yang dulu bernama `status` menjadi event **`stale`** (BREAKING).
> Respons join menambah **`queue_token`** (untuk mobile) dan **tidak lagi** membawa
> `estimated_wait_minutes`: frontend menghitung estimasi dari `ratePerMin` (§3.5).
> **Envelope distandarkan:** `data` selalu ada (`{}` bila kosong) dan setiap error membawa
> **`meta.error_code`** (Appendix A). OpenAPI YAML diarsipkan; Apidog menjadi sumber kebenaran.

> **v2.6** (ADR-006): event SSE `grant` dihapus — klien memanggil `/queue/me` saat
> `nowServing ≥ ticket_no`. Join tidak lagi memberi grant sendiri: pada antrian kosong ia
> menunggu ≤ 1 detik untuk tick admission-ctl berikutnya, sehingga `granted: true` tetap mungkin.
> **Dicabut di v2.12:** join tidak menunggu sama sekali dan tidak pernah menjawab `granted`.
> `/queue/serving` menambah `poll_tiers` dan tidak menerima query string.

> **v2.5** memperbaiki kontrak terhadap API Midtrans yang sebenarnya, dan menurunkan
> `server_time` ke envelope. Webhook §5.5 sebelumnya mendeklarasikan header
> `X-Callback-Signature` yang **tidak pernah dikirim Midtrans** dan memetakan idempotency ke
> field `event_id` yang **tidak ada** di notifikasinya; keduanya diganti dengan field yang
> benar-benar dikirim. `server_time` kini bagian dari `meta` pada **setiap** respons (BR-10,
> §7), bukan diulang per endpoint. Empat contoh yang bertentangan dengan keputusan POC
> dihapus: `holder_name` dan error "nama tidak cocok" (§4.3) melawan FR-15a yang menyatakan
> nama tidak pernah dikumpulkan, dan contoh penolakan kartu (§5.4) melawan §5.1 yang
> mengeluarkan kartu dari lingkup. `GET /auth/verify-email` (§1.7) didokumentasikan — ia
> sudah berjalan di `user-service` tetapi tidak pernah masuk kontrak. Rinciannya di
> Appendix C.
>
> **v2.4** menutup lima hal yang v2.3 tinggalkan terbuka. `expires_in` **1080 → 22740**
> detik: sesi akun dimulai saat login, *sebelum* antrian, jadi 1080 — yang diturunkan dari
> perjalanan — mati di ruang tunggu untuk hampir seluruh antrian; ia sekarang diturunkan dari
> `SALE_WINDOW_SECONDS` (§1.3). `bookings.account_id` diambil dari **keterlibatan** saat
> GRANT, bukan dari header saat checkout, sehingga `Authorization` hilang dari §4 dan §5
> seluruhnya. Kredensial antrian **dihapus** pada keadaan terminal, dengan nisan pada
> `CONFIRMED` supaya customer yang baru membayar tidak melihat layar kedaluwarsa (§3.3).
> §6.2 menerima token dari tautan email, karena FR-26 mengirim tiket ke pemegang NIK yang
> umumnya tidak punya akun dan tombol di email mereka sebelumnya menjawab 401. Dan pengikatan
> device untuk grant anonim akhirnya **didefinisikan** — ADR-005 mengklaimnya sejak awal tanpa
> menyebut header, algoritma, atau apa pun. Rinciannya di Appendix C.
>
> **v2.3** memisahkan tiga kredensial yang sebelumnya berbagi satu header.
> `Authorization: Bearer` kini **selalu dan hanya** token sesi akun. Grant slot pindah ke
> header `X-Grant-Token`, dan token ruang tunggu untuk klien mobile pindah ke
> `X-Queue-Token`. Sebelumnya ketiganya dikirim sebagai `Authorization: Bearer
> {access_token}` dan tidak satu pun endpoint menyatakan mana yang ia terima — §9
> mengikatnya ke `queue_token` sementara §6.1 hanya mungkin berarti JWT akun.
>
> Akibatnya bukan soal nama. Dengan satu slot saja, `POST /bookings` tidak punya tempat
> untuk JWT akun, sehingga booking customer login **tidak dapat diatribusikan ke akunnya
> pada saat dibuat** — padahal §6.1 menyelesaikan kepemilikan lewat `account_id`. Dua
> header menutup lubang itu: grant membuktikan slot, JWT menyatakan milik siapa.
> Rinciannya di §7, §9 dan Appendix C.
>
> **v2.2** menambahkan `POST /auth/refresh` dan menaikkan `expires_in` dari 300 ke
> **1080** detik. Nilai 300 tidak pernah diturunkan dari apa pun; 1080 adalah angka
> yang sama dengan `GRANT_TOKEN_TTL_SECONDS` dan diturunkan dari perjalanan customer
> (ADR-005 §7), sehingga sesi hidup lebih lama daripada journey dan tidak ada refresh
> yang jatuh di tengah pembayaran. Rinciannya di §1.3, §1.4 dan Appendix C.
>
> **v2.0** menyelaraskan kontrak ini dengan ADR-005. Perubahan yang memutus kompatibilitas:
> `POST /queue/claim-token` **dihapus** (grant kini didorong lewat SSE setelah slot aman),
> payload dan nama event SSE berubah, webhook mendapat switch per tipe event, dan validasi
> NIK menjadi *fail closed*. Lihat Appendix C.

---

## Daftar Isi

1. [Authentication](#1-authentication)
2. [Events](#2-events)
3. [Queue](#3-queue)
4. [Booking](#4-booking)
5. [Payment](#5-payment)
6. [Tickets (Accounts › Tickets)](#6-tickets-accounts--tickets)
7. [Common Response Structure](#7-common-response-structure)

---

## Response Structure

Semua endpoint menggunakan format response yang konsisten:

```json
{
  "meta": {
    "code": 200,
    "message": "Success"
  },
  "data": { ... }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `meta.code` | int | HTTP status code |
| `meta.message` | string | Human-readable message (single message) — untuk ditampilkan, **jangan** di-parse |
| `meta.error_code` | string | **Hanya pada error (4xx/5xx).** Kode mesin dari Appendix A; frontend merutekan berdasarkan field ini |
| `meta.server_time` | string | Jam server ISO-8601, pada **setiap** respons (BR-10, §7) |
| `meta.pagination` | object | Pagination info (only on list endpoints) |
| `data` | object/array | Response payload. **Selalu ada**: `{}` bila tidak ada isi, tidak pernah `null` atau dihilangkan |

> **Contoh per endpoint menghilangkan `server_time` dan `error_code` demi keringkasan.** Keduanya
> tetap selalu ada; kode error tertulis pada judul respons, mis. "Response (409 - `NIK_ALREADY_SAVED`)".
> Bentuk lengkapnya ada di Apidog. Pengecualian: `204 No Content` (tanpa body, sesuai HTTP) dan
> stream SSE §3.2.

---

## 1. Authentication

### 1.1 Request Magic Link (Login)

Mengirim link login ke email. UI: "Send Login Link" button.

**Endpoint**: `POST /auth/magic-link`

**Request**:
```json
{
  "email": "you@example.com"
}
```

**Response** (200):
```json
{
  "meta": {
    "code": 200,
    "message": "Login link sent to your email"
  },
  "data": {}
}
```

**Response** (429 - Rate Limited):
```json
{
  "meta": {
    "code": 429,
    "message": "Please wait before requesting another link"
  },
  "data": {
    "retry_after": 60
  }
}
```

**Requirement Coverage**: FR-31 (verifikasi email via link)

---

### 1.2 Resend Magic Link

UI: "Resend Login Link" button di "Check your inbox" screen.

**Endpoint**: `POST /auth/magic-link/resend`

**Request**:
```json
{
  "email": "you@example.com"
}
```

**Response** (200):
```json
{
  "meta": {
    "code": 200,
    "message": "New login link sent"
  }
}
```

---

### 1.3 Verify Magic Link

Verifikasi token dari link email. Redirect target dari email link.

**Endpoint**: `GET /auth/verify`

**Cookies**:
```
magic_token={token}
```

**Cookies**:
```
access_token={access_token}
```

**Response** (200):
```json
{
  "meta": {
    "code": 200,
    "message": "Login successful"
  },
  "data": {
    "access_token": "eyJhbG...",
    "refresh_token": "8Kq2mN9vL4wX1yB6cF3dH5jG0aZpR7tS",
    "expires_in": 22740,
    "user": {
      "id": "usr_123",
      "email": "you@example.com",
      "nik": "3171xxxxxxxxxx22"
    },
    "redirect_context": {
      "type": "booking",
      "event_id": "evt_konser_a",
      "booking_id": "RB-00912",
      "message": "Booking context restored"
    }
  }
}
```

| Field | Catatan |
|---|---|
| `access_token` | JWT HS256 **sesi akun**. `expires_in` **22740** detik (6 jam 19 menit) — diturunkan dari `SALE_WINDOW_SECONDS + 930 + 210` (ADR-005 §7; grace pause dihapus ADR-006 §4.3, total tetap). **Bukan** sama dengan `GRANT_TOKEN_TTL_SECONDS`: keduanya mengukur interval berbeda. Perpanjangannya lewat §1.4. Dikirim di `Authorization: Bearer` dan **tidak pernah diterima sebagai grant slot** — §4 dan §5 dijaga `X-Grant-Token` (§7) |
| `refresh_token` | **Opaque, bukan JWT.** Nilai acak; server menyimpan hash-nya saja, sehingga membaca basis data tidak cukup untuk menerbitkan sesi |
| `redirect_context` | **Dihilangkan** (bukan dikirim kosong) bila akun tidak punya keterlibatan aktif. Pemeriksaan frontend adalah "apakah field-nya ada", bukan "apakah isinya kosong" |

> **Dari mana 22740 datang, dan mengapa bukan 1080.** Sesi akun dimulai saat
> **login**, yang terjadi *sebelum* antrian. Jadi ia harus mencakup penantian
> **dan** perjalanan:
>
> ```
> SALE_WINDOW_SECONDS   21600   login → giliran tiba  (env; fallback 6 jam)
> journey                  930   (BR-22 pause grace dihapus, ADR-006 §4.3)
> margin                   210
>                       ─────── = 22740 detik
> ```
>
> Versi 2.3 memakai 1080, yang diturunkan dari perjalanan — benar untuk **grant**,
> dan hanya kebetulan untuk sesi. Pada `ratePerMin` 318 dan antrian 300.000,
> customer di posisi terakhir menanti sekitar **15,7 jam**. Sesi 18 menit mati di
> ruang tunggu untuk hampir seluruh antrian.
>
> Yang rusak bukan ruang tunggunya — stream memakai cookie dan §3.3 memulihkan
> lewat cookie. Yang rusak adalah **§1.6 pada Detail Ticket Page**: Daftar NIK
> Teman menjawab 401 tepat ketika customer membukanya, memaksa NIK diketik
> manual, memaksa panggilan Dukcapil, dan Dukcapil *fail closed* (BR-13). Satu
> gangguan Dukcapil kemudian memblokir booking yang seharusnya lolos dari bukti
> tersimpan — di dalam TTL booking 300 detik.
>
> **Harganya tertulis di §9**: angka ini sekaligus jendela pencabutan, karena
> access token adalah JWT stateless dan tidak ada store yang dibaca saat
> verifikasi. Pada 22740 detik claim `nik_verified` adalah **otorisasi** yang
> ter-cache enam jam, bukan hanya identitas. Mengeluarkan claim itu adalah
> pasangan yang semestinya menyertai perubahan ini, dan masih terbuka
> (ADR-005 Bagian 17 #11).

> **`redirect_context` diturunkan dari keterlibatan aktif akun** (BR-07: satu posisi
> antrian **atau** satu booking yang belum terminal). Booking mengalahkan antrian —
> begitu slot diberikan, booking adalah tempat customer sebenarnya berada.
> `type` bernilai `booking` atau `queue`; `booking_id` hanya ada pada yang pertama.
>
> Pencariannya *best-effort*: kegagalannya menghilangkan field, bukan menggagalkan
> login. Mendarat di halaman utama adalah degradasi; login yang gagal adalah kerusakan.

**Requirement Coverage**: FR-31, BR-09 (re-entrant user kembali ke halaman yang sama)

---

### 1.4 Refresh Session

Menukar refresh token dengan pasangan token baru. Dipanggil klien saat access
token mendekati kedaluwarsa, atau setelah menerima `401 TOKEN_EXPIRED`.

Access token berlaku 22740 detik (§1.3) — mencakup penantian antrian **dan**
perjalanan 930 detik — sehingga endpoint ini **tidak** dipanggil di tengah
journey, dan juga tidak di ruang tunggu. Ia ada untuk sesi yang berumur hari,
bukan untuk menambal token yang terlalu pendek.

**Endpoint**: `POST /auth/refresh`

**Request**:
```json
{
  "refresh_token": "8Kq2mN9vL4wX1yB6cF3dH5jG0aZpR7tS"
}
```

> Token dikirim di **body**, bukan header `Authorization`. Pada endpoint ini ia
> bukan kredensial pembawa melainkan objek permintaannya sendiri, dan menjauhkannya
> dari `Authorization` mencegahnya tercatat oleh proxy yang me-log header itu.

**Response** (200):
```json
{
  "meta": {
    "code": 200,
    "message": "Session refreshed"
  },
  "data": {
    "access_token": "eyJhbG...",
    "refresh_token": "wB3nT8kM2pQ7vF1xJ9cL5gH4dZ6yA0sR",
    "expires_in": 22740
  }
}
```

**Cookies**: `access_token` ikut di-set ulang (HttpOnly, Secure, SameSite=Strict)
untuk klien web. Klien mobile membaca dari body.

#### Rotasi: token lama hangus setiap kali dipakai

Setiap panggilan mengembalikan refresh token **baru** dan menghanguskan yang
dikirim. Klien **wajib** mengganti salinan simpanannya — bila tidak, panggilan
berikutnya akan terbaca sebagai pemutaran ulang dan sesi berakhir.

| Keadaan token yang dikirim | Respons | Kode |
|---|---|---|
| Valid, belum pernah dipakai | 200 + pasangan baru | — |
| **Sudah pernah dipakai** (replay) | 401, **seluruh rantai dicabut** | `TOKEN_INVALID` |
| Kedaluwarsa, dicabut, atau tidak dikenal | 401 | `TOKEN_EXPIRED` |
| Akun sudah tidak ada | 401 | `TOKEN_INVALID` |
| Field `refresh_token` kosong/absen | 400 | `VALIDATION_ERROR` |

> **Mengapa replay mencabut seluruh rantai, bukan hanya mata rantai itu.**
> Token sekali-pakai yang muncul dua kali berarti ada dua pihak memegangnya —
> tanda kredensial bocor. Yang menggenggam mata rantai **terbaru** bisa jadi
> penyerang, bukan customer, sehingga mencabut satu token saja justru
> mempertahankan akses penyerang. Mengakhiri sesi dan meminta login ulang adalah
> respons termurah yang benar (OAuth 2.0 BCP §4.13).
>
> **Replay dan token kedaluwarsa dibedakan di kode, tetapi tidak di pesan.**
> Keduanya 401; `TOKEN_INVALID` tidak memberi tahu pemanggil bahwa ia baru
> memicu deteksi pencurian.

#### Keluarga token (rotation family)

Setiap login membuka **keluarga** tersendiri. Pencabutan karena replay hanya
mengenai keluarga itu, sehingga customer tidak ikut keluar dari perangkat lain
yang sesinya sehat.

| Properti | Nilai |
|---|---|
| Masa berlaku refresh token | **30 hari** |
| Masa berlaku access token | **22740 detik** (§1.3) — diturunkan dari `SALE_WINDOW_SECONDS` + perjalanan, bukan konstanta lepas |
| Bentuk | Opaque acak 256-bit; server menyimpan **hash** saja |
| Pemakaian | **Sekali**; redeem menerbitkan penggantinya dalam satu transaksi |

> **`nik_verified` dibaca ulang dari akun, bukan disalin dari token lama.** NIK yang
> baru terverifikasi sejak refresh terakhir harus langsung berlaku, dan NIK yang
> dicabut tim operasional harus berhenti diklaim.

**Response** (401):
```json
{
  "meta": {
    "code": 401,
    "message": "Session ended; please sign in again"
  },
  "data": {}
}
```

**Requirement Coverage**: FR-31, BR-09, NFR-02 (customer tidak diputus sepihak),
NFR-07

---

### 1.5 Register Account

Buat akun baru dengan email dan NIK.

**Endpoint**: `POST /auth/register`

**Request**:
```json
{
  "email": "alex@example.com",
  "nik": "0000000000000000",
  "name": "ALEX"
}
```

**Response** (200):
```json
{
  "meta": {
    "code": 200,
    "message": "Verification link sent to your email"
  },
  "data": {
    "user_id": "usr_456"
  }
}
```

**Request** (lengkap):
```json
{
  "email": "alex@example.com",
  "nik": "3171xxxxxxxxxx22",
  "consent": { "accepted": true, "policy_version": "2026-10-01" }
}
```

| Field | Wajib | Catatan |
|---|---|---|
| `nik` | ✅ | **NIK_Terdaftar** — 16 digit, diverifikasi Dukcapil saat registrasi, terikat permanen ke akun |
| `consent.accepted` | ✅ | Persetujuan pemrosesan NIK pihak ketiga. Checkbox **tidak boleh pre-checked** |
| `consent.policy_version` | ✅ | Versi kebijakan privasi yang ditampilkan saat itu |

> **NIK diverifikasi sekali, di sini.** Setelah itu pembelian oleh akun ini tidak memanggil
> Dukcapil lagi untuk NIK pembeli. Itulah tujuan utama `NIK_Terdaftar`: Dukcapil berbayar
> per panggilan, dan pola pemakaian tiket adalah NIK yang sama berulang kali.
>
> **Consent diperlukan karena akun boleh menyimpan NIK orang lain** (§1.6). Rekamnya
> immutable — timestamp, versi kebijakan, identitas akun — dan pencabutan hanya lewat tim
> operasional (UU PDP No. 27/2022).

**Validation Errors** (400):
```json
{
  "meta": { "code": 400, "message": "NIK must be exactly 16 digits" },
  "data": {}
}
```

**Response** (409 - `NIK_ALREADY_REGISTERED`): NIK sudah terikat ke akun lain. Satu NIK
hanya dapat didaftarkan pada satu akun secara global.

**Response** (422 - `CONSENT_REQUIRED`): registrasi diblokir sampai persetujuan dicentang.

**Response** (429): melewati batas 5 akun baru per IP per jam.

**Requirement Coverage**: FR-29, FR-30, FR-31

---

### 1.6 Daftar NIK Teman

NIK yang sering diajak hadir bersama, disimpan sekali dan dipakai berulang **tanpa
memanggil Dukcapil lagi**. Maksimal 5 per akun. Tidak tersedia untuk guest.

**Endpoint**: `GET /accounts/me/saved-niks`

```json
{
  "meta": { "code": 200, "message": "Success" },
  "data": {
    "max_entries": 5,
    "saved_niks": [
      { "id": "snk_1", "nik_masked": "3171********22", "label": "Budi",
        "verified_at": "2026-10-20T08:11:00Z", "status": "VERIFIED" }
    ]
  }
}
```

**Endpoint**: `POST /accounts/me/saved-niks`

```json
{ "nik": "3171xxxxxxxxxx33", "label": "Budi" }
```

> **Dukcapil dipanggil di sini, dan barisnya hanya dibuat jika jawabannya valid.** Urutannya
> Dukcapil dulu, INSERT kemudian — tidak ada status `PENDING`. Entri yang belum terverifikasi
> akan dipakai checkout untuk *melewati* Dukcapil, sehingga menyimpannya berarti memindahkan
> lubang fail-closed satu langkah ke belakang.
>
> Basis data menegakkannya lewat `CHECK (verified_at IS NOT NULL)`, bukan hanya kode.
>
> NIK lengkap tidak pernah dikembalikan — hanya bentuk ter-mask.

**Response** (201): entri dibuat; `verified_at` terisi dari jawaban Dukcapil saat itu.

**Response** (409 - `MAX_SAVED_NIKS_REACHED`): sudah 5 entri; hapus salah satu dulu.

**Response** (409 - `NIK_ALREADY_SAVED`): NIK sudah ada di daftar akun ini.

**Response** (422 - `CONSENT_REQUIRED`): akun belum memberi persetujuan §1.5.

**Response** (422 - NIK ditolak Dukcapil): tidak disimpan.

**Response** (503 - `DUKCAPIL_UNAVAILABLE`): tidak disimpan, tidak ada entri parsial.

> **Menolak di sini tidak merugikan customer.** Menyimpan NIK tidak berada di bawah tekanan
> TTL — tidak ada stok yang di-hold dan tidak ada countdown berjalan — sehingga mencoba lagi
> beberapa menit kemudian tidak kehilangan apa pun. Itu yang membedakannya dari checkout,
> di mana penolakan menghentikan penjualan yang sedang berlangsung.

> **Tidak ada revalidasi berkala.** Hasil verifikasi dianggap permanen: NIK yang sudah
> dinyatakan valid tidak diverifikasi ulang. Aman di POC karena gate check-in berada di luar
> lingkup, sehingga tidak ada komponen hilir yang memeriksa ulang NIK sebuah tiket.

**Endpoint**: `DELETE /accounts/me/saved-niks/{id}`

**Response** (204). Penghapusan **tidak** memengaruhi tiket yang sudah dibeli dengan NIK itu.

**Requirement Coverage**: FR-15a (mengurangi panggilan Dukcapil), NFR-14 (UU PDP)

---

### 1.7 Verify Email Address

Mengonfirmasi kepemilikan alamat email setelah registrasi (FR-31). **Berbeda dari §1.3**:
§1.3 adalah login tanpa kata sandi dan menerbitkan sesi; endpoint ini hanya mengisi
`accounts.email_verified_at` dan tidak menerbitkan token apa pun.

**Endpoint**: `GET /auth/verify-email`

**Query Parameters**:
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `token` | string | Yes | Token dari tautan email `account.verify`; berlaku **15 menit**, sekali pakai. Sama dengan TTL tautan login — `magic_tokens` memakai satu konstanta untuk kedua `purpose` |

**Response** (200):
```json
{
  "meta": {
    "code": 200,
    "message": "Email verified",
    "server_time": "2026-11-03T09:12:00Z"
  },
  "data": { "email_verified": true }
}
```

**Response** (400 - `TOKEN_INVALID`): token tidak dikenal, sudah dipakai, atau kedaluwarsa.

> **Akun yang belum terverifikasi tetap dapat membeli.** Verifikasi email hanya membuka
> riwayat pesanan (FR-31, ADR-005 Bagian 12); ia tidak pernah menghalangi pembelian. Yang
> memblokir pembelian adalah `nik_verified_at IS NULL`, dan itu urusan §1.5.

> **Tautan ini kedaluwarsa dalam 15 menit, dan itu bukan jalan buntu — jangan bangun
> endpoint resend untuknya.** §1.3 **juga** mengisi `email_verified_at`: menebus tautan login
> membuktikan kendali atas alamat itu, yaitu hal yang sama persis yang dibuktikan tautan
> verifikasi. Jadi customer yang tautannya kedaluwarsa cukup meminta magic link (§1.1) dan
> login — akunnya terverifikasi sebagai efek sampingnya. Penulisannya idempoten
> (`COALESCE(email_verified_at, now())`), sehingga timestamp pertama yang menang.
>
> Konsekuensinya endpoint ini adalah **kenyamanan, bukan satu-satunya jalan**: ia melayani
> customer yang ingin mengonfirmasi tanpa login. `POST /auth/magic-link/resend` menerbitkan
> `purpose = LOGIN`, dan itu memang benar — ia menyelesaikan kedua kebutuhan sekaligus.
>
> Dinyatakan di sini karena TTL 15 menit tanpa catatan ini mengundang kesimpulan yang salah:
> bahwa tautan yang terlewat mengunci akun dalam keadaan belum terverifikasi selamanya, lalu
> seseorang membangun jalur resend kedua yang menulis kolom yang sama dengan prasyarat yang
> sedikit berbeda.

> **Mengapa diberi nomor §1.7 dan bukan §1.4.** Endpoint ini sudah berjalan di
> `user-service` sejak branch `user-notif-nikverify-outboxpublisher` tetapi tidak pernah
> masuk kontrak. Menempatkannya di urutan logis berarti menggeser §1.4–§1.6, dan ketiganya
> dirujuk lintas dokumen. Nomor di akhir adalah pilihan yang tidak memecahkan rujukan apa pun.

**Requirement Coverage**: FR-31

---

## 2. Events

### 2.1 List Events

Menampilkan daftar event. UI: Home Page "Daftar Event".

**Endpoint**: `GET /events`

**Query Parameters**:
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `filter` | string | No | `all`, `available` (`status != SOLD_OUT`), `upcoming` (`queue_state == NOT_OPEN`) |
| `limit` | int | No | Default: 10, Max: 50 |
| `offset` | int | No | Default: 0 |

**Response** (200):
```json
{
  "meta": {
    "code": 200,
    "message": "Success",
    "pagination": {
      "totalData": 2,
      "totalPage": 1,
      "limit": 10,
      "offset": 0
    }
  },
  "data": [
    {
      "id": "evt_konser_a",
      "name": "Konser A — Jakarta",
      "category": "LIVE MUSIC",
      "location": {
        "venue": "GBK",
        "city": "Jakarta"
      },
      "date": "2026-12-25",
      "image_url": "https://cdn.rusuh.id/events/konser-a.jpg",
      "status": "SELLING_OUT_FAST",
      "queue_state": "FULL",
      "queue_depth": 300000,
      "queue_capacity": 300000,
      "sale_starts_at": "2026-11-10T02:00:00Z",
      "display": {
        "enabled": false,
        "action": "NONE",
        "label_key": "queue_full",
        "badge_key": null
      },
      "min_price": 500000,
      "max_price": 1500000
    },
    {
      "id": "evt_konser_b",
      "name": "Konser B — Bandung",
      "category": "LIVE MUSIC",
      "location": {
        "venue": "GOR Saparua",
        "city": "Bandung"
      },
      "date": "2027-01-10",
      "image_url": "https://cdn.rusuh.id/events/konser-b.jpg",
      "status": "SOLD_OUT",
      "queue_state": "OPEN",
      "queue_depth": 0,
      "queue_capacity": 300000,
      "sale_starts_at": "2026-10-20T02:00:00Z",
      "display": {
        "enabled": false,
        "action": "NONE",
        "label_key": "sold_out",
        "badge_key": null
      },
      "min_price": null,
      "max_price": null
    }
  ]
}
```

**`status`** — sumbu stok saja:

| Nilai | Kondisi |
|---|---|
| `AVAILABLE` | `available > threshold` |
| `SELLING_OUT_FAST` | `0 < available <= threshold` |
| `SOLD_OUT` | `available == 0` — **reversibel**: kursi dari booking yang kedaluwarsa dapat membuatnya `AVAILABLE` lagi |

> **Catatan v2.10:** nilai ini adalah sumbu stok halaman utama dan **belum** diganti nama, padahal artinya sama dengan state antrian `FULL_BOOKED` (reversibel). Rekomendasi: ganti menjadi `FULL_BOOKED` bersama FR-03c; sementara itu klien menganggap `status: SOLD_OUT` pada daftar event sama dengan `FULL_BOOKED` pada antrian.

`threshold = capacity × SELLING_FAST_THRESHOLD_PCT / 100` (bawaan 20).

**`queue_state`** — sumbu izin masuk saja:

| Nilai | Kondisi | Tombol |
|---|---|---|
| `NOT_OPEN` | sale belum dibuka (`sale_starts_at` di depan) | Nonaktif |
| `FULL` | `queue_depth >= queue_capacity` | Nonaktif, "Antrian penuh" |
| `DIRECT` | `queue_depth == 0` dan masih ada slot bebas | "Beli Tiket" — FR-05 (label saja, lihat catatan di bawah) |
| `OPEN` | sisanya | "Masuk Antrian" |

> **`COMING_SOON` dihapus dari `status`**, dan `CLOSED` dihapus dari `queue_state`.
> `COMING_SOON` adalah keadaan siklus hidup sale, bukan keadaan stok — ia menjadi
> `queue_state: NOT_OPEN` + `sale_starts_at`. `CLOSED` dulu berarti "event sold out",
> sehingga ia melaporkan urusan `status`.
>
> Keduanya kini dapat bernilai bebas satu terhadap yang lain, dan pada puncak sale kombinasi
> `status: AVAILABLE` + `queue_state: FULL` (tiket masih ada, antrian sudah penuh) adalah hal
> normal — satu enum tidak dapat menyatakannya.

**`display`** — keputusan server, bukan frontend:

| Field | Nilai |
|---|---|
| `enabled` | bool — tombol dapat ditekan |
| `action` | `JOIN_QUEUE` \| `GO_TO_TICKETS` \| `NONE` |
| `label_key` | `sold_out` \| `sold_out_may_return` \| `coming_soon` \| `queue_full` \| `buy_now` \| `join_queue` |
| `badge_key` | `selling_out_fast` \| `null` |

Aturan resolusi, dijalankan **server-side**, kecocokan pertama menang:

```
status == SOLD_OUT
  dan queue state CLOSED   → enabled=false  action=NONE           label=sold_out
status == SOLD_OUT
  dan belum CLOSED         → enabled=false  action=NONE           label=sold_out_may_return
queue_state == NOT_OPEN    → enabled=false  action=NONE           label=coming_soon
queue_state == FULL        → enabled=false  action=NONE           label=queue_full
queue_state == DIRECT      → enabled=true   action=GO_TO_TICKETS  label=buy_now
queue_state == OPEN        → enabled=true   action=JOIN_QUEUE     label=join_queue
```

> **`GO_TO_TICKETS` tetap memanggil `POST …/queue/join` (§3.1).** Grant hanya lahir dari join: tanpa
> join tidak ada `X-Grant-Token`, tidak ada pengikatan device dan tidak ada penegakan BR-07. Sejak v2.12 join
> **tidak pernah** menjawab `granted: true`: untuk `GO_TO_TICKETS` maupun `JOIN_QUEUE` jawabannya `queued` atau
> `rejoined`. Bedanya hanya layar yang ditampilkan: untuk `GO_TO_TICKETS` klien membuka stream dan menampilkan
> "Memproses…" (bukan layar antrian); giliran tiba dalam satu tick (≤ 500 ms) bila ada tempat kosong, lalu klien
> memanggil `/queue/me` dan pindah ke Detail Ticket.

```
badge_key = selling_out_fast  bila status == SELLING_OUT_FAST DAN display.enabled
          = null              selainnya
```

> **Label adalah kunci, bukan teks.** FR-32 menuntut dua bahasa; teks jadi dari server akan
> memasukkan `Accept-Language` ke cache key CDN dan memecah satu respons yang di-cache 30
> detik untuk 300.000 orang. Kunci + tabel terjemahan di frontend adalah pencarian, bukan
> logika.
>
> **Halaman utama menampilkan waktu absolut, bukan hitung mundur.** Hitung mundur pra-buka
> menuntut frontend membandingkan `sale_starts_at` dengan jam — logika di frontend, dan
> keliru juga karena respons ini di-cache 30 detik. Label pra-buka berbunyi
> "Buka 10 Nov 09:00". Hitung mundur hidup hanya ada di ruang tunggu lewat SSE `serverTime`.

> `queue_depth` adalah perkiraan — response ini di-cache CDN 30 detik. Penegak sebenarnya
> adalah pemeriksaan atomik saat `POST /events/{id}/queue/join`. Gunanya menampilkan di sini
> adalah agar customer melihat antrian penuh **sebelum** mengklik, bukan sesudah.

**Requirement Coverage**: FR-01, FR-02, FR-03

---

### 2.2 Get Event Detail

Detail event untuk sidebar info dan breadcrumb.

**Endpoint**: `GET /events/{event_id}`

**Response** (200):
```json
{
  "meta": {
    "code": 200,
    "message": "Success"
  },
  "data": {
    "id": "evt_konser_a",
    "name": "Konser A — Jakarta",
    "category": "LIVE MUSIC",
    "location": {
      "venue": "GBK",
      "city": "Jakarta"
    },
    "date": "2026-12-25",
    "time": "19:00",
    "image_url": "https://cdn.rusuh.id/events/konser-a.jpg",
    "description": "Energi panggung. Cerita yang kamu bawa pulang.",
    "status": "AVAILABLE",
    "queue_state": "OPEN",
    "queue_depth": 1248,
    "queue_capacity": 300000,
    "sale_starts_at": "2026-11-10T02:00:00Z",
    "display": {
      "enabled": true,
      "action": "JOIN_QUEUE",
      "label_key": "join_queue",
      "badge_key": null
    },
    "captcha": {
      "provider": "turnstile",
      "site_key": "0x4AAAAAAA..."
    },
    "ticket_categories": [
      {
        "id": "cat_vip",
        "name": "VIP",
        "description": "Lebih dekat dengan energi panggung.",
        "price": 1500000,
        "available": 12,
        "status": "AVAILABLE"
      },
      {
        "id": "cat_festival",
        "name": "Festival",
        "description": "Nikmati konser dari area festival.",
        "price": 850000,
        "available": 240,
        "status": "AVAILABLE"
      },
      {
        "id": "cat_tribune",
        "name": "Tribune",
        "description": "Area tribun dengan panorama panggung.",
        "price": 500000,
        "available": 0,
        "status": "SOLD_OUT"
      }
    ]
  }
}
```

> **`captcha.site_key` ada di sini supaya token dapat diambil lebih awal.** Klien memuat
> widget CAPTCHA pada halaman ini — termasuk saat `queue_state: NOT_OPEN` — dan menyegarkan
> tokennya tiap ~60 detik, sehingga pada `sale_starts_at` ia sudah memegang token segar untuk
> dibawa ke `POST …/queue/join` (§3.1). Site key bersifat publik; secret key tidak pernah
> dikirim ke klien.

**Requirement Coverage**: FR-11, FR-12

---

## 3. Queue

### 3.1 Join Queue

User bergabung ke antrian. Dipanggil saat klik tombol event dengan `action` `JOIN_QUEUE` ("Masuk
Antrian") **atau** `GO_TO_TICKETS` ("Beli Tiket", antrian kosong) — §2.1. Kembali setelah refresh atau
dari aplikasi bank memakai §3.3, bukan join.

**Endpoint**: `POST /events/{event_id}/queue/join`

**Headers**:
```
X-Device-Id: {device_id}               # wajib bila customer TIDAK login; acak 128 bit, format tidak divalidasi
Authorization: Bearer {access_token}   # bila login — mengaktifkan BR-07 dan atribusi
X-Queue-Token: {queue_token}           # opsional (mobile): rejoin BR-09; web memakai cookie
```

**Request**:
```json
{ "captcha_token": "0.AbCdEf..." }
```

| Field | Wajib | Catatan |
|---|---|---|
| `captcha_token` | ✅ | Token CAPTCHA **tak terlihat**, diperoleh klien sebelum gerbang buka — lihat di bawah. Berlaku untuk tamu **dan** user login |

> **`X-Device-Id` adalah tempat pengikatan grant dimulai.** Nilai acak 128 bit yang dibuat
> klien sekali (UUID, hex 32 karakter atau base64url) dan disimpan di `localStorage` / secure store. Server
> **tidak** memvalidasi formatnya (v2.9) — hanya wajib ada untuk tamu — jadi kekuatan pengikatan bergantung pada
> klien membuat nilai acak 128 bit dan memakainya terus. Server menyimpan
> `HMAC(DEVICE_PEPPER, device_id)` pada keterlibatan, lalu mencocokkannya pada setiap
> panggilan ber-grant (§4, §5). Tamu yang tidak mengirimnya ditolak di sini, bukan nanti —
> grant tanpa pengikatan tidak dapat diikat belakangan.
>
> **`Authorization` di sini, dan hanya di sini di antara jalur antrian.** Ia mengaktifkan dua
> hal yang keduanya butuh identitas akun pada saat join: penegakan BR-07 lewat
> `acct:{ev}:{account_id}`, dan `account_id` pada keterlibatan — yang kemudian menjadi
> `bookings.account_id` saat GRANT, tanpa pernah menanyakan klien lagi (§4).

> **Token diperoleh SEBELUM `sale_starts_at`, bukan saat klik.** `ticket_no` berasal dari
> `INCR issued` pada saat join; bila klien harus menyelesaikan tantangan lebih dulu, manusia
> yang butuh tiga detik akan mendapat nomor antrian **lebih besar** daripada bot yang
> menembus layanan pemecah dalam 200 ms. Jadi:
>
> 1. Halaman pra-buka (`queue_state: NOT_OPEN`, §2.1) mengambil token saat dimuat
> 2. Token disegarkan tiap ~60 detik selama menunggu — **wajib**, karena token CAPTCHA
>    berumur pendek dan sekali pakai
> 3. Pada `sale_starts_at` klien sudah memegang token segar; join hanya membawanya
>
> Verifikasi murni server-side, sehingga tidak ada komponen berkecepatan-manusia di dalam
> perlombaan. Mode interaktif (teka-teki visual) **tidak boleh** dipakai di endpoint ini.

> **Urutan pemeriksaan server: rate limit dulu, CAPTCHA kemudian.** Rate limit adalah
> panggilan Redis lokal; verifikasi CAPTCHA adalah round trip eksternal. Membaliknya berarti
> membayar satu panggilan pihak ketiga untuk tiap percobaan yang akhirnya ditolak.

**Response** (200):
```json
{
  "meta": {
    "code": 200,
    "message": "Successfully joined queue"
  },
  "data": {
    "ticket_no": 32748,
    "now_serving": 31500,
    "position": 1248,
    "queue_token": "qt_xyz789",
    "sse_url": "https://events.my-rusuh.my.id/events/evt_konser_a/queue/stream",
    "event": {
      "id": "evt_konser_a",
      "name": "Konser A — Jakarta"
    }
  }
}
```

| Field | Tipe | Arti |
|---|---|---|
| `ticket_no` | int | Nomor antrian customer, tetap selama sale |
| `now_serving` | int | Nomor global yang sedang dilayani saat respons dibuat |
| `position` | int | `ticket_no − now_serving`. `≤ 0` berarti giliran sudah tiba: panggil §3.3 |
| `queue_token` | string | Token ruang tunggu (`qt_…`). **Mobile** menyimpannya di secure store dan mengirimnya sebagai `X-Queue-Token` ke §3.2–§3.3. **Web mengabaikannya** dan memakai cookie HttpOnly (jangan simpan di `localStorage`) |
| `sse_url` | string | **URL absolut** stream §3.2 (`https://events.my-rusuh.my.id/events/{event_id}/queue/stream`). Absolut karena stream berada di host lain (`events.*`) dari join (`api.*`) |

> Estimasi waktu tunggu **tidak** dikirim di sini: frontend menghitungnya dari `ratePerMin` pada
> snapshot SSE pertama (§3.5).

> Server sets cookie on join:
> ```
> Set-Cookie: queue_token=qt_xyz789; Domain=my-rusuh.my.id; Path=/events/{event_id}; HttpOnly; Secure;
>             SameSite=Strict; Max-Age={SALE_WINDOW_SECONDS}
> ```
>
> **`Path=/events/{event_id}`** (v2.9): satu cookie per event. Dengan `Path=/`, join event lain menimpa token
> event ini, sehingga tamu yang kembali mendapat tempat kedua. Path ini mencakup join, `/queue/me` dan stream
> (`/events/{event_id}/queue/...` di kedua host), dan tidak cocok untuk event lain (`/events/E10` bukan
> `/events/E1`).
>
> **`Domain=my-rusuh.my.id`**: join dilayani `api.my-rusuh.my.id`, stream `events.my-rusuh.my.id`; cookie
> host-only dari `api.*` tidak terkirim ke `events.*`. Keduanya satu *site*, jadi `SameSite=Strict` tetap
> berlaku. Token yang sama juga ada di body (`queue_token`) untuk mobile.
>
> `Max-Age` adalah **batas atas, bukan umur yang diharapkan.** Sejak v2.4 keterlibatan yang
> mencapai keadaan terminal menghapus kredensialnya lebih cepat daripada TTL — lihat §3.3.

**Response** (200 - Already in Queue / Rejoined):
```json
{
  "meta": {
    "code": 200,
    "message": "Rejoined existing queue position"
  },
  "data": {
    "ticket_no": 32748,
    "now_serving": 31520,
    "position": 1228,
    "queue_token": "qt_xyz789",
    "sse_url": "https://events.my-rusuh.my.id/events/evt_konser_a/queue/stream",
    "rejoined": true,
    "state": "QUEUED"
  }
}
```

> **Dua jalan menuju response "Rejoined".** Pertama, `queue_token` yang sama kembali — re-entrancy
> BR-09. Kedua, **BR-07**: pemanggil sudah login dan akunnya sudah punya keterlibatan aktif
> untuk event ini, mungkin dari perangkat lain. Keduanya memulihkan keterlibatan yang ada
> dan menerbitkan ulang cookie untuknya; tidak ada keterlibatan kedua yang dibuat.
>
> `state` mengikuti nilai §3.3 (`QUEUED`, `GRANTED`, `SELECTING`, `AWAITING_PAYMENT`, …).
> Bila bukan `QUEUED`, field posisi diabaikan dan frontend merutekan sesuai `state` —
> customer sudah melewati antrian.
>
> **Menolak dengan 409 ditolak sebagai desain**: itu akan menghukum customer yang hanya
> berpindah perangkat. BR-07 tetap tegak — ia tidak mendapat slot kedua — dan user login
> mendapat pemulihan lintas perangkat sebagai efek samping.
>
> **Customer tamu tidak terkena BR-07.** Tanpa identitas stabil tidak ada yang dapat
> ditegakkan; batas bagi tamu adalah CAPTCHA + rate limit per IP.
>
> **`queue_token` hanya berlaku bagi pemiliknya** (v2.9, ADR-006 §8.9). Token **tamu** hanya dari perangkat yang
> menerimanya (`HMAC(DEVICE_PEPPER, X-Device-Id)` = `device_hash`), **tanpa rebind**: perangkat lain →
> `401 DEVICE_MISMATCH`. Token **akun** hanya untuk akun yang sama, dari perangkat mana pun: akun lain atau
> tamu → `401 UNAUTHORIZED`. Pada keduanya tidak ada nomor yang terpakai, tempat pemilik tidak berubah, dan
> respons **menghapus cookie** `queue_token`. Pemanggil yang **login** dengan token **tamu** mendapat tempatnya
> sendiri (tempat tamu tidak diambil alih), karena mengambil alih tempat tamu melewati klaim
> `acct:{ev}:{account_id}` dan membuka jalan ke dua tempat untuk satu akun (BR-07). Token dari event
> lain, atau yang sudah tidak ada, diabaikan: join berjalan sebagai join baru.
>
> **Urutan pemeriksaan:** pemulihan (BR-09, lalu BR-07) dijalankan **sebelum** gerbang `FULL_BOOKED` dan
> `QUEUE_FULL`, jadi customer yang sudah punya tempat tidak pernah ditolak oleh keduanya.

**Tidak ada respons "langsung masuk"** (dicabut di v2.12, ADR-006 §8.14). Join tidak menunggu dan tidak
menjawab `granted`; bahkan bila tempat masih kosong, jawabannya respons antre di atas dan tick admission-ctl
berikutnya (≤ 500 ms) memasukkan customer. Klien merutekan berdasarkan `state` / `rejoined`, lalu mengikuti stream
(§3.2) dan `GET /queue/me` (§3.3).

**Response** (503 - `QUEUE_FULL`): antrian sudah mencapai `QUEUE_CAPACITY` (300.000).
```json
{
  "meta": { "code": 503, "message": "Antrian sedang penuh, silakan coba beberapa saat lagi" },
  "data": { "queue_state": "FULL", "queue_capacity": 300000 }
}
```

> **Fail fast, tanpa buffering.** Customer tidak diberi nomor antrian yang tidak akan pernah
> terlayani. Halaman utama sudah menampilkan `queue_state` (§2.1), jadi penolakan di sini
> adalah jaring pengaman untuk perlombaan di detik yang sama, bukan jalur normal.

**Response** (400 - `CAPTCHA_REQUIRED`): `captcha_token` tidak disertakan.

**Response** (403 - `CAPTCHA_FAILED`): token tidak valid, sudah dipakai, atau kedaluwarsa.
Klien mengambil token baru dan mencoba lagi — **tanpa** mendapat `ticket_no`, sehingga
percobaan yang gagal tidak memakan posisi antrian.

**Response** (429): melewati `rl:join:{ip}` (20 / 5 menit), atau — bila pemanggil login —
melewati batas 3 join per akun per menit.

**Response** (404 - `NOT_FOUND`): `event_id` tidak dikenal (tidak ada `queue:{ev}.state`) atau tidak dapat menjadi hash tag Redis.

**Response** (503 - `SERVICE_UNAVAILABLE`): Redis tidak dapat dihubungi, atau server sedang shutdown (drain, SIGTERM). Disertai `Retry-After`; `EventSource` mencoba lagi. Gagal tertutup: stream tidak pernah dibuka tanpa snapshot. Bila state bukan salah satu dari empat nilai yang dikenal: `503 QUEUE_NOT_READY`.

**Response** (409 - `SALE_NOT_OPEN`): `queue:{ev}.state` masih `SCHEDULED`.
```json
{
  "meta": { "code": 409, "message": "Penjualan belum dibuka" },
  "data": { "queue_state": "NOT_OPEN", "sale_starts_at": "2026-11-10T02:00:00Z" }
}
```

> **Ini keadaan AWAL, bukan tepi.** 300.000 orang berkumpul **sebelum** gerbang dibuka, bukan
> sesudah. Tanpa penolakan di sini mereka semua mendapat `ticket_no` untuk sale yang belum
> ada. Halaman utama sudah menampilkan `queue_state: NOT_OPEN` + `sale_starts_at` (§2.1);
> penolakan di sini adalah penegaknya.

**Response** (409 - `FULL_BOOKED`): `queue:{ev}.state` = `FULL_BOOKED`, untuk **pendatang baru** saja.
```json
{
  "meta": { "code": 409, "message": "Tiket sedang habis", "error_code": "FULL_BOOKED" },
  "data": { "queue_state": "FULL_BOOKED", "full_booked_at": "2026-11-10T02:40:12Z" }
}
```

> **Mengapa ditolak, padahal stok bisa kembali.** Kursi yang kembali dari booking kedaluwarsa jatuh ke
> orang yang **sudah** di antrian lebih dulu; pendatang baru saat `FULL_BOOKED` hampir tidak mungkin
> terlayani, dan halaman utama sudah menonaktifkan tombolnya (`sold_out_may_return`, §2.1). Sama dengan
> alasan `QUEUE_FULL`: tidak menerbitkan nomor yang tidak akan terlayani. Yang sudah mengantri tetap
> tersambung dan dipulihkan (§3.2 tidak menjawab 410 untuk `FULL_BOOKED`). Saat state kembali `OPEN`, join
> diterima lagi.

**Response** (410 - `SALE_CLOSED`): `queue:{ev}.state` = `CLOSED`; `data`: `{ "queue_state": "CLOSED" }`.
Berlaku juga bagi yang rejoin.

**Response** (503 - `QUEUE_NOT_READY`): `queue:{ev}.state` tidak ada atau bernilai tak dikenal (event
belum di-seed). Bukan "belum dibuka": tidak ada yang diketahui tentang sale ini.

**Response** (503 - `INTERNAL_ERROR`): Redis tidak dapat dihubungi atau penulisan gagal. Gagal tertutup;
klien mencoba lagi. Tidak ada `ticket_no` yang terpakai tanpa tempat.

**Response** (401 - `DEVICE_MISMATCH`): `queue_token` tamu event ini dikirim dari perangkat lain (atau token
tidak punya `device_hash`). Respons membawa `Set-Cookie: queue_token=; Path=/events/{event_id}; Max-Age=0`.

**Response** (401 - `UNAUTHORIZED`): `queue_token` milik sebuah akun dikirim oleh akun lain atau oleh tamu. Cookie
dihapus dengan cara yang sama.

> **Langkah klien pada `DEVICE_MISMATCH` / `UNAUTHORIZED` dari join** (v2.9):
> 1. Hapus token antrian yang tersimpan. **Web:** server sudah menghapus cookie `queue_token`; jangan kirim ulang
>    nilai lama. **Mobile:** hapus `queue_token` dari secure store, berhenti mengirim `X-Queue-Token`.
> 2. Tampilkan "Sesi antrian ini terdaftar di perangkat/akun lain".
> 3. Bila customer ingin mengantri, panggil join lagi **tanpa** token (CAPTCHA baru) → tempat baru di belakang.
>
> Tanpa langkah 1, setiap join berikutnya mengirim token yang sama dan selalu dijawab 401.

**Response** (401 - `TOKEN_EXPIRED` / `TOKEN_INVALID`): `Authorization` dikirim tetapi tidak sah
(termasuk `Bearer` tanpa token). **Tidak** diperlakukan sebagai tamu: itu akan membiarkan pemanggil
menghindari BR-07 dengan mengirim token rusak.

**Response** (400 - `VALIDATION_ERROR`): body bukan JSON, lebih dari 8 KB, atau `captcha_token` bukan string.

**Response** (404 - `NOT_FOUND`): `event_id` tidak dapat menjadi hash tag Redis (memuat `{`, `}` atau
spasi).

**Response** (429): `data`: `{ "retry_after": 42 }` (detik) dan header `Retry-After` bernilai sama.
Limiter yang tidak tersedia **fail-open** (alasan sama dengan CAPTCHA di bawah).

**Response** (400 - `DEVICE_ID_REQUIRED`): pemanggil anonim tidak mengirim `X-Device-Id`.

> Ditolak **di sini**, bukan nanti: grant tanpa pengikatan tidak dapat diikat belakangan
> (Appendix C).

> **Provider CAPTCHA tidak tersedia → join tetap diterima (fail-open).** Aturannya:
> *fail-closed bila yang dipertaruhkan kebenaran, fail-open bila hanya keadilan.* Dukcapil
> fail-open akan menerbitkan tiket atas NIK yang belum terverifikasi; CAPTCHA fail-open hanya
> kehilangan gesekan terhadap bot — tidak ada oversell, tidak ada NIK yang lolos, tidak ada
> uang yang bergerak. Fail-closed di sini akan membuat **tidak seorang pun** dapat bergabung
> saat provider terganggu, mematikan sale sebelum dimulai. `CaptchaFailOpenSeconds`
> dialarmkan agar jendela tanpa proteksi terlihat.

**Requirement Coverage**: FR-04, FR-05 (antrian kosong = ruang tunggu sekitar satu tick, ADR-006 §8.14), FR-10 (posisi konsisten saat
refresh), FR-18 (`FULL_BOOKED`), NFR-05 (FCFS fairness), BR-07 (satu keterlibatan aktif per event untuk user
login), BR-09 (pemulihan dengan `queue_token`)

---

### 3.2 Queue Status Stream (SSE)

Server-Sent Events stream untuk update posisi real-time. Client connect setelah join queue.

**Endpoint**: `GET /events/{event_id}/queue/stream`

**Cookies** (web): `queue_token=qt_xyz789`
**Headers** (mobile): `X-Queue-Token: qt_xyz789`

> **Web memakai cookie, dan itu bukan pilihan gaya.** `EventSource` tidak dapat menyetel
> header request, sehingga stream ini satu-satunya endpoint yang *harus* diautentikasi lewat
> cookie. Klien mobile menyetel header, jadi mereka memakai `X-Queue-Token`. Yang **tidak**
> dipakai di sini adalah `Authorization: Bearer` — slot itu milik token sesi akun saja (§7).

> Token **tidak pernah** dikirim lewat query string — ia akan tercatat di access log ALB,
> CloudWatch, dan history browser.

**Response** (200): `text/event-stream`

**Snapshot awal** — dikirim segera saat koneksi terbuka, sebelum broadcast berikutnya. Tanpa
ini posisi kosong sampai tick berikutnya, dan FR-10 tidak terpenuhi setelah refresh.

```
id: 31600
data: {"nowServing":31600,"state":"OPEN","ratePerMin":318,"serverTime":"2026-10-24T13:10:00.412Z"}
```

**Event default (tanpa nama)** — dikirim saat `nowServing` berubah, atau tiap 10 detik sebagai
heartbeat isi. **Byte yang sama untuk semua koneksi event.**

```
data: {"nowServing":31604,"state":"OPEN","ratePerMin":318,"serverTime":"2026-10-24T13:10:01.004Z"}
```

| Field | Tipe | Arti |
|---|---|---|
| `nowServing` | int | Nomor antrian global yang sedang dilayani |
| `state` | string | `SCHEDULED` \| `OPEN` \| `FULL_BOOKED` \| `CLOSED` |
| `ratePerMin` | int | Laju antrian: nomor yang dilewati `nowServing` per menit. Dipakai frontend untuk estimasi (§3.5) |
| `serverTime` | string | Waktu server ISO-8601 — klien menyinkronkan jamnya (BR-10) |

> Frontend menghitung `position = ticket_no − nowServing` dan estimasi menit (§3.5).

> **Heartbeat 10 detik TIDAK bersyarat** — ia terbit juga saat `state: FULL_BOOKED`, ketika
> `nowServing` tidak bergerak. Keheningan karena itu berarti satu hal saja: sisi server berhenti
> bekerja. Itulah yang membuat `stale {stale:true, message:"Data antrian belum diperbarui"}` di bawah
> dapat dipercaya.

**Giliran tiba — tanpa event khusus (v2.6, ADR-006 §4.2).** Event `grant` dihapus. Pada setiap
payload (termasuk snapshot awal setelah reconnect), bila `nowServing ≥ ticket_no`, klien memanggil `GET /events/{event_id}/queue/me` untuk mengambil
`grant_token`.

> admission-ctl mem-publish broadcast **setelah** baris booking tick itu tertulis, sehingga
> `/queue/me` biasanya langsung menjawab `GRANTED`. Bila baris belum terbaca (grant pending atau
> lag replika), `/queue/me` menjawab `QUEUED` dengan posisi 0 dan klien mengulang setelah 1 detik —
> tidak pernah menolak customer yang gilirannya sudah tiba (NFR-02).

**Event: `status`** — **state sale berubah** (v2.7). Dikirim sekali pada setiap transisi
`queue:{ev}.state`, ke semua koneksi event.

```
event: status
data: {"state":"FULL_BOOKED","fullBookedAt":"2026-10-24T13:40:12.000Z","serverTime":"2026-10-24T13:40:12.120Z"}
```

| Field | Tipe | Arti |
|---|---|---|
| `state` | string | State baru: `OPEN` \| `FULL_BOOKED` \| `CLOSED` |
| `fullBookedAt` | string \| null | Hanya pada `FULL_BOOKED`: kapan stok mencapai nol. `CLOSED` menyusul paling lambat `FULL_BOOKED_HOLD_SECONDS` (930 detik) setelahnya |
| `serverTime` | string | Waktu server (BR-10) |

| Transisi | Klien |
|---|---|
| `OPEN → FULL_BOOKED` | tampilkan "Tiket sedang habis — kursi dapat kembali dari booking yang dibatalkan", sembunyikan estimasi, **tetap tersambung** |
| `FULL_BOOKED → OPEN` | stok kembali; lanjutkan posisi dan estimasi seperti biasa |
| `→ CLOSED` | diikuti event `sold-out`; klien menutup stream |

> `state` tetap ada di setiap payload default, jadi klien yang baru reconnect tahu state dari
> snapshot. Event `status` adalah sinyal "baru saja berubah", sehingga klien tidak perlu
> membandingkan payload sendiri.

**Event: `stale`** — indikator **kesegaran data** antrian, bukan state sale dan bukan status
koneksi. Dua pemicu, satu payload, satu indikator. (Sampai v2.6 bernama `status`.)

```
event: stale
data: {"stale":true,"state":"OPEN","message":"Data antrian belum diperbarui","serverTime":"2026-10-24T13:10:31.000Z"}
```

| Field | Tipe | Arti |
|---|---|---|
| `stale` | bool | Data antrian tidak lagi segar |
| `state` | string | Keadaan sale saat ini (sama seperti event lain) |
| `message` | string | Teks siap tampil. **Hanya ada saat `stale: true`**; tidak ada saat pulih. Bukan kode: klien tidak bercabang pada isinya |
| `serverTime` | string | Waktu server ISO-8601 |

Kedua pemicunya (isi `message`):

| `message` | Pemicu |
|---|---|
| "Koneksi server terputus, mencoba menyambung kembali" | sse-server kehilangan Redis |
| "Data antrian belum diperbarui" | tidak ada broadcast > 30 detik padahal `state ∈ {OPEN, FULL_BOOKED}` — `admission-ctl` diam, atau Pub/Sub ke task ini putus |

> **Yang basi adalah DATA, bukan koneksinya.** Pada keheningan 30 detik koneksinya justru
> sehat — `: keepalive` tetap datang tiap 15 detik — tetapi `nowServing` berhenti maju. Itu
> sebabnya keheningan tidak dapat dideteksi dari keepalive, dan sebabnya indikator ini ada.

> **Klien TIDAK BOLEH** menutup stream, mereset posisi, atau reconnect sendiri karena
> `stale`. 300.000 klien yang reconnect pada satu sinyal bersama adalah badai yang dihindari
> seluruh desain ini. Yang benar: pertahankan angka posisi di layar, redupkan, tampilkan
> "Memperbarui…". Satu indikator melayani kedua penyebab; tidak ada cabang baru di klien.

> Dikirim **hanya pada transisi**, tidak diulang selama tetap basi. `{"stale":false,"state":…,"serverTime":…}` (tanpa `message`)
> menyusul saat data mengalir lagi, dan posisinya berlanjut tanpa kode transisi khusus.

**Event: `sold-out`** — sale ditutup permanen. Dikirim tepat setelah `status {state: CLOSED}`,
bukan saat stok pertama kali mencapai nol (itu `status {state: FULL_BOOKED}`).

```
event: sold-out
data: {"state":"CLOSED","message":"Tiket sudah habis","serverTime":"2026-10-24T13:55:42.000Z"}
```

> **Klien WAJIB memanggil `eventSource.close()`** pada event ini. Tanpa itu `EventSource`
> reconnect otomatis dan seluruh crowd membanjiri ALB setelah sold out. Server menolak
> koneksi baru dengan `410 Gone` sebagai jaring pengaman.

> **`state: FULL_BOOKED` BUKAN event ini, dan klien tidak boleh menutup stream karenanya.**
> `FULL_BOOKED` berarti "tidak ada stok saat ini" dan **reversibel**: booking yang kedaluwarsa
> melepaskan kursinya kembali, dan admission melanjutkan dari posisi antrian yang sama. Klien
> menampilkan "Tiket sedang habis — kursi dapat kembali dari booking yang dibatalkan" dan
> **tetap tersambung**; bila `state` kembali ke `OPEN`, hitung mundur dan posisinya berlanjut
> tanpa kode transisi khusus.
>
> `FULL_BOOKED` berumur paling lama `FULL_BOOKED_HOLD_SECONDS` (930 detik — satu perjalanan penuh,
> setelah itu tidak ada hold yang mungkin kedaluwarsa lagi), lalu server menyetel `CLOSED`
> dan mengirim `sold-out`. Jadi satu-satunya pemutusan terjadi sekali, di akhir.

> **Bentuk seragam (v2.12):** setiap event bernama (`status`, `stale`, `sold-out`, `reconnect`) dan pesan default membawa `state` dan `serverTime`. `message` (teks siap tampil, Indonesia) ada pada `stale` true, `sold-out` dan `reconnect`. Klien bercabang pada nama event, `stale` dan `state`, **bukan** pada `message`.

**Event: `reconnect`** — task sedang shutdown; `retry:` berisi jeda berjitter 1–15 detik.

```
event: reconnect
retry: 7300
data: {"state":"OPEN","message":"Server sedang diperbarui, menyambung ulang otomatis","serverTime":"2026-10-24T13:10:00.000Z"}
```

> `state` tidak pernah `CLOSED` di sini (stream yang ditutup memakai `sold-out`). Apidog: `QueueStreamReconnect`.

**Keepalive** — komentar SSE tiap **15 detik**, bukan event bernama:

```
: keepalive
```

> Komentar tidak memicu handler di klien dan berukuran 13 byte; sebuah event JSON tiap 30
> detik berukuran ~6× lebih besar dan tetap berisiko kena idle timeout ALB (default 60s,
> dikonfigurasi 3600s).

**Response** (401 - `UNAUTHORIZED`): queue token tidak ada atau tidak dikenal.

**Response** (429 - `TOO_MANY_REQUESTS`): melewati `rl:sse:{ip}` (5 koneksi / detik).

> Rate limit ini duduk di **sse-server**, bukan di API Gateway — endpoint ini satu-satunya
> yang dilayani ALB. Ia pengganti WAF di gerbang itu (ADR-005 Bagian 4).

**Response** (409 - `SALE_NOT_OPEN`): `queue:{ev}.state` masih `SCHEDULED`.

**Response** (410 - `SALE_CLOSED`): `queue:{ev}.state` sudah `CLOSED`. `meta.message` selalu **"Tiket sudah habis"** (juga pada join, §3.1), karena `CLOSED` berarti sale benar-benar selesai. `EventSource` berhenti
permanen pada status non-200 — ini disengaja.

> **`FULL_BOOKED` TIDAK menjawab 410.** Stream tetap dibuka seperti biasa: stok dapat kembali
> dari booking yang kedaluwarsa, dan klien sengaja dipertahankan. Hanya `CLOSED` — yang
> disetel `FULL_BOOKED_HOLD_SECONDS` (930 detik) setelah `full_booked_at` — yang memutus.

**Requirement Coverage**: FR-07, FR-08, FR-09, FR-10, BR-10, NFR-06

---

### 3.3 Get Queue State (Re-entrancy)

Satu-satunya endpoint yang dipanggil klien saat kembali — setelah refresh, ganti tab, atau
kembali dari aplikasi bank. Mengembalikan **token yang sama** selama TTL belum habis.

**Endpoint**: `GET /events/{event_id}/queue/me`

**Auth — salah satu dari dua**, dan itu yang membuat endpoint ini cukup untuk semua
skenario kembali:

| | Dipakai saat | Resolusi server |
|---|---|---|
| `Cookie: queue_token=qt_xyz789` | perangkat sama — tab ditutup, refresh, kembali dari aplikasi bank | `HGET token:{uuid}` |
| `X-Queue-Token: qt_xyz789` | perangkat sama, klien mobile — tidak ada cookie store | `HGET token:{uuid}` |
| `Authorization: Bearer <token sesi akun>` | perangkat berbeda, cookie dibersihkan, incognito, app di-install ulang | `acct:{ev}:{account_id}` → uuid token → `HGET token:{uuid}`; cookie diterbitkan ulang pada respons |

> **Pemulihan adalah READ, dan itu disengaja.** `POST …/queue/join` juga memulihkan
> keterlibatan yang ada (BR-07), tetapi ia jalur tulis yang membawa **CAPTCHA** dan dua rate
> limit (`rl:join:{ip}` 20 per 5 menit, plus 3 join per customer per menit). Customer yang
> sekadar kembali tidak boleh diminta menyelesaikan CAPTCHA untuk mengambil posisi yang sudah
> dimilikinya — dan di belakang NAT kantor atau operator seluler, batas 20 join per 5 menit
> habis justru karena orang-orang yang reconnect. Endpoint ini **tidak** menuntut
> `captcha_token` dan **tidak** dihitung terhadap batas join.
>
> **Hanya untuk customer login.** Tamu tidak punya token sesi akun, sehingga pemulihan mereka
> berhenti pada cookie — lintas perangkat memang di luar cakupan untuk tamu.

**Response** (200): bentuk `data` mengikuti keadaan customer.

| Keadaan | `data` | Halaman tujuan |
|---|---|---|
| Masih mengantre | `{"state":"QUEUED","ticket_no":31842,"now_serving":31600}` | Waiting Room |
| Giliran tiba | `{"state":"GRANTED","grant_token":"gt_abc123","booking_expires_at":"…","server_time":"…"}` | Detail Ticket, countdown **sudah berjalan** |
| Sedang memilih | `{"state":"SELECTING","grant_token":"gt_abc123","booking_id":"RB-00912","booking_expires_at":"…","server_time":"…"}` | Detail Ticket, countdown dilanjutkan |
| Menunggu bayar | `{"state":"AWAITING_PAYMENT","grant_token":"gt_abc123","booking_id":"RB-00912","payment_expires_at":"…","payment_ref":"pay_123","server_time":"…"}` | Payment, countdown dilanjutkan |
| Selesai | `{"state":"CONFIRMED","booking_id":"RB-00912"}` | Halaman tiket |
| Kedaluwarsa | `{"state":"EXPIRED"}` | Layar expired |

> Grant pending atau lag replika menjawab `QUEUED` dengan `ticket_no ≤ now_serving` ("sebentar
> lagi"); klien mengulang setelah 1 detik.
>
> **Tamu wajib mengirim `X-Device-Id`** (ADR-006 §8.9): token tamu hanya berlaku dari perangkat yang menerimanya, jadi perangkat lain
> mendapat `401 DEVICE_MISMATCH` dan cookie `queue_token` dihapus; tanpa header itu `400 DEVICE_ID_REQUIRED`. Token akun hanya untuk akun
> yang sama (`401 UNAUTHORIZED`). Pemanggil yang membawa `Authorization: Bearer` selalu dijawab dengan tempat **akunnya sendiri**; cookie
> tamu diabaikan, tidak diambil alih.
>
> **`EXPIRED` tidak menghapus cookie.** `LAPSED` bersifat final, tetapi `token:{uuid}` dan `acct` dipertahankan sampai jendela penjualan habis
> supaya `POST …/queue/join` menjawab `rejoined: true` dengan `state: LAPSED`, bukan memberi tempat kedua. Hanya nisan (`CONFIRMED`) dan token
> milik orang lain yang menghapus cookie. `booking_id` berbentuk `RB-<ULID>`.

#### Kapan kredensial antrian berhenti ada

Cookie dan `token:{uuid}` punya TTL (`SALE_WINDOW_SECONDS`), tetapi TTL saja berarti
kredensial tetap hidup lama setelah ia berhenti berarti apa pun. Sejak v2.4 keadaan terminal
menghapusnya aktif:

| Keadaan | Kredensial antrian | Akibat pada §3.3 |
|---|---|---|
| grant diterbitkan | **tetap** | pemulihan setelah grant justru bergantung padanya — `GRANTED`, `SELECTING`, `AWAITING_PAYMENT`, termasuk "kembali dari aplikasi bank" |
| `EXPIRED`, `LAPSED`, `ABANDONED` | dihapus | `{"state":"EXPIRED"}` — tidak ada apa pun untuk kembali ke sana. `LAPSED` bersifat final: tidak ada kesempatan kedua (ADR-005 Bagian 7) |
| `CONFIRMED` | dihapus, diganti **nisan** | tetap `{"state":"CONFIRMED","booking_id":…}` selama `SETTLED_TOMBSTONE_SECONDS` |

> **Nisan, bukan perpanjangan.** Pada `CONFIRMED`, `token:{uuid}` dihapus dan `settled:{uuid}`
> ditulis memuat `booking_id` dan tidak lebih — bukan `ticket_no`, bukan `account_id`, bukan
> `device_hash`. §3.3 tetap dapat menjawab `CONFIRMED`, tetapi tidak ada kredensial yang
> tersisa untuk memulihkan apa pun.
>
> **Yang dicegahnya adalah kebohongan, bukan kebocoran.** Tanpa nisan, tamu yang me-refresh di
> celah antara pembayaran lunas dan email e-tiket tiba memanggil endpoint ini dengan token
> mati, dan pemetaan frontend untuk token tak terselesaikan adalah **layar kedaluwarsa** —
> memberi tahu orang yang baru membayar bahwa sesinya habis. Itu "jangan PHP" dalam bentuk
> paling murni, dan ia salah secara faktual.
>
> Respons yang dijawab dari nisan mengirim `Set-Cookie: queue_token=; Max-Age=0` bersamaan.
> Transisi terminal terjadi di `admission-ctl` atau webhook, bukan di dalam permintaan
> customer, jadi tidak ada respons lain untuk menumpangi.
>
> **Setelah nisan kedaluwarsa, email adalah rutenya** — ditambah mendaftar dengan NIK yang
> sama, yang mewarisi tiket lewat klausa kepemilikan §6.1. `contact_email` dan `holder_email`
> wajib di §4.3, jadi kanal itu selalu ada. Itulah yang membuat penghapusan ini dapat
> diterima — dan mengapa tautan e-tiket **wajib** dapat dibuka tanpa akun (§6.2).

> `POST /queue/claim-token` dari v1.1 **dihapus**. Pada model itu klien meminta slot setelah
> `your_turn`, dan bisa menerima `429 Slot penuh` — memberi tahu customer bahwa gilirannya
> tiba lalu menolaknya, pelanggaran langsung NFR-02 "Jangan PHP". Pada v2.0 grant hanya
> diterbitkan setelah slot diamankan, jadi tidak ada jalur penolakan setelah `grant`.

> **Hitung mundur selalu dari tenggat server** (`booking_expires_at`, `payment_expires_at`) dan
> `server_time`, bukan dikurangi sendiri. Sejak v2.7 tenggat **tidak pernah bergeser** (pause
> dan BR-22 dihapus, ADR-006 §4.3).

**Requirement Coverage**: FR-08, FR-10, BR-09, NFR-02, NFR-07

---

### 3.4 Get Now Serving (Polling Fallback)

Dipakai untuk hidrasi awal, jaringan yang memblokir `text/event-stream`, dan degraded mode
saat tier SSE tidak sehat. **Bukan** transport utama.

**Endpoint**: `GET /events/{event_id}/queue/serving`

**Response** (200):
```json
{
  "meta": { "code": 200, "message": "Success" },
  "data": {
    "now_serving": 31600,
    "rate_per_min": 318,
    "state": "OPEN",
    "server_time": "2026-10-24T13:10:00.412Z",
    "poll_after_ms": 1000,
    "poll_tiers": [
      { "max_position": 10,   "poll_after_ms": 1000 },
      { "max_position": 100,  "poll_after_ms": 2000 },
      { "max_position": 1000, "poll_after_ms": 5000 },
      { "max_position": null, "poll_after_ms": 10000 }
    ]
  }
}
```

> Tabel interval ditentukan **server** (`poll_tiers`), mengikuti profil di `requirements.md` §5:
> 10s (>1000) → 5s (101–1000) → 2s (11–100) → 1s (1–10). Endpoint **tidak menerima query
> string** — satu entri CDN per event, `Cache-Control: public, max-age=1` — jadi server tidak
> tahu posisi pemanggil; klien memilih baris `poll_tiers` untuk posisinya. `poll_after_ms`
> adalah default bila posisi tidak diketahui. Klien tidak menanam interval sendiri (ADR-006 §8.5).

> Dalam mode polling, frontend menghitung posisi dan estimasi dengan rumus §3.5 memakai
> `now_serving` dan `rate_per_min` dari respons ini, dan mendeteksi perubahan state dari `state`.

**Requirement Coverage**: FR-07 (fallback), NFR-08

---

### 3.5 Estimasi Waktu Tunggu (FR-09)

Bukan endpoint: **frontend menghitung** estimasi dari data yang sudah ia punya. Server hanya
mengirim `nowServing` + `ratePerMin` (SSE §3.2) atau `now_serving` + `rate_per_min` (§3.4);
tidak ada field estimasi di respons mana pun. Rumus ini ditulis di sini supaya web dan mobile
sama. Rinciannya di ADR-006 §6.

`ratePerMin` = berapa nomor yang dilewati `nowServing` per menit, diukur admission-ctl dalam 60
detik terakhir selama `state = OPEN` (nilai awal `MAX_CONCURRENT_SLOTS / 5` pada menit pertama).

**Rumus frontend**:

```js
const position = ticketNo - nowServing;

if (state === 'FULL_BOOKED')   show('Tiket sedang habis. Mungkin tersedia lagi, tetap di antrian');
else if (position <= 0)     show('Giliran Anda sebentar lagi');        // dan panggil §3.3
else if (ratePerMin <= 0)   show('Menghitung estimasi…');
else {
  const raw = Math.ceil(position / ratePerMin);
  const estimatedWaitMinutes =
      raw <= 10 ? raw
    : raw <= 30 ? Math.ceil(raw / 5) * 5
    : raw <= 60 ? Math.ceil(raw / 10) * 10
    :             Math.ceil(raw / 30) * 30;
  show(label(estimatedWaitMinutes));
}
```

| `estimatedWaitMinutes` | Teks |
|---|---|
| 1–59 | "Estimasi waktu tunggu ± {m} menit" |
| 60–180 | "Estimasi waktu tunggu ± {m/60} jam" (mis. 90 → "± 1,5 jam") |
| > 180 | "Estimasi waktu tunggu lebih dari 3 jam" |

> Pembulatan selalu **ke atas**: estimasi condong pesimis, dan angka berubah dalam langkah besar
> sehingga tidak berkedip tiap 500 ms.

**Contoh** (`ratePerMin` = 200): posisi 150 → "± 1 menit" · 1.000 → "± 5 menit" · 2.400 →
"± 15 menit" · 6.000 → "± 30 menit" · 145.000 → "lebih dari 3 jam".

**Requirement Coverage**: FR-09, NFR-02 ("Jangan PHP")

---

## 4. Booking

Seluruh endpoint §4 dan §5 dijaga oleh **grant**, dan hanya oleh grant. `Authorization` tidak
dikirim ke endpoint mana pun di sini.

| Header | Wajib | Membuktikan | Bila salah/hilang |
|---|---|---|---|
| `X-Grant-Token` | ✅ | pemegang **punya slot** | `401 GRANT_TOKEN_MISSING` / `GRANT_TOKEN_INVALID` |
| `X-Device-Id` | ✅ bila **tidak login** | grant masih di perangkat yang menerimanya | `401 DEVICE_MISMATCH` |

> **Mengapa grant yang menjaga, bukan sesi.** Tamu boleh membeli (FR-02): ia tidak punya akun,
> jadi tidak ada sesi untuk diperiksa. Yang dimilikinya adalah slot.
>
> **Dan mengapa sesi tidak lagi dikirim sama sekali.** v2.3 menerimanya sebagai header
> opsional untuk mengisi `bookings.account_id`. Itu gagal secara diam-diam: bila sesi akun
> mati di ruang tunggu, klien tidak mengirim apa pun, baris tertulis `account_id = NULL`, dan
> pembelian customer login tercatat sebagai pembelian tamu — tanpa error, tanpa 401, tanpa
> jejak. v2.4 mengambil `account_id` dari **keterlibatan** (`token:{uuid}`, yang mengetahuinya
> sejak join ketika sesi masih pasti segar) dan menuliskannya pada INSERT saat GRANT. Atribusi
> karena itu tidak bergantung pada jam klien, dan header itu tidak punya pekerjaan lagi.
>
> **`X-Device-Id` mengikat grant ke perangkat yang menerimanya** — satu-satunya hal yang
> berdiri antara customer dan menjual slot antriannya, dan di antrian 300.000 untuk 10.000
> tiket slot punya harga pasar. Nilai acak 128 bit yang dibuat klien, bukan fingerprint pasif;
> server menyimpan HMAC-nya saja. **Tanpa rebind** (ADR-006 §8.9): perangkat lain → `401 DEVICE_MISMATCH`;
> tamu yang menghapus data situs kehilangan slotnya. Rinciannya di ADR-005 Bagian 12.
> Customer login tidak mengirimnya: `account_id` sudah mengikat lebih kuat.
>
> Tamu yang kelak mendaftar dengan NIK yang sama tetap mewarisi tiketnya — klausa kepemilikan
> §6.1 menyelesaikan *pembeli **atau** pemegang NIK terdaftar*, jadi tidak ada aturan tambahan
> yang diperlukan.

### 4.1 Get Ticket Selection Page

Menampilkan kategori tiket dengan stok. UI: "Pilih tempatmu di konser."

**Endpoint**: `GET /events/{event_id}/tickets`

**Headers**:
```
X-Grant-Token: {grant_token}
X-Device-Id: {device_id}               # wajib bila customer TIDAK login
```

> `Authorization` tidak dikirim ke endpoint ini — lihat tabel di awal §4.

**Response** (200):
```json
{
  "meta": {
    "code": 200,
    "message": "Success"
  },
  "data": {
    "event": {
      "id": "evt_konser_a",
      "name": "Konser A — Jakarta",
      "date": "2026-12-25",
      "venue": "GBK, Jakarta",
      "image_url": "https://cdn.rusuh.id/events/konser-a.jpg"
    },
    "session": {
      "expires_at": "2026-10-23T13:15:00Z",
      "ttl_seconds": 287,
      "server_time": "2026-10-23T13:10:13Z"
    },
    "categories": [
      {
        "id": "cat_vip",
        "name": "VIP",
        "description": "Lebih dekat dengan energi panggung.",
        "price": 1500000,
        "available": 12,
        "max_per_order": 3
      },
      {
        "id": "cat_festival",
        "name": "Festival",
        "description": "Nikmati konser dari area festival.",
        "price": 850000,
        "available": 240,
        "max_per_order": 3
      },
      {
        "id": "cat_tribune",
        "name": "Tribune",
        "description": "Area tribun dengan panorama panggung.",
        "price": 500000,
        "available": 0,
        "max_per_order": 0
      }
    ],
    "rules": {
      "max_tickets_per_order": 3,
      "one_nik_per_ticket": true
    }
  }
}
```

> **Endpoint ini MENGUBAH STATE, meskipun ia `GET`.** Pemuatannya adalah *claim*:
> `bookings.status` berpindah `GRANTED → SELECTING` dan `slot_claimed_at` dicatat. Tenggat
> **tidak** diubah — `booking_expires_at` sudah ditetapkan saat GRANT, dan tidak ada grace
> antrian (ADR-005 Bagian 7).
>
> Transisi ini idempoten: memuat ulang halaman saat status sudah `SELECTING` tidak
> melakukan apa pun, dan status `HELD` atau `AWAITING_PAYMENT` tetap merender halaman —
> customer yang menekan Back dari halaman pembayaran harus melihat pilihannya, bukan error.
>
> Konsekuensi operasional: **respons ini tidak boleh di-cache**, dan tidak boleh di-prefetch
> oleh klien. `Cache-Control: no-store`. ADR-005 Bagian 6 menugaskan transisi ini ke
> `booking-service` tanpa menyebut endpoint mana yang menjalankannya; ini endpointnya.

**Requirement Coverage**: FR-11, FR-12, FR-13, FR-17, BR-01, BR-02

---

### 4.2 Validate NIK

Validasi NIK ke Dukcapil sebelum submit booking. UI: dipanggil saat input NIK blur atau sebelum "Lanjut ke Pembayaran".

**Endpoint**: `POST /nik/validate`

**Headers**:
```
X-Grant-Token: {grant_token}
X-Device-Id: {device_id}               # wajib bila customer TIDAK login
```

> `Authorization` tidak dikirim ke endpoint ini — lihat tabel di awal §4.

**Request**:
```json
{
  "nik": "3171xxxxxxxxxx22"
}
```

**Response** (200 - Valid):
```json
{
  "meta": {
    "code": 200,
    "message": "NIK is valid"
  },
  "data": {
    "nik": "3171xxxxxxxxxx22",
    "valid": true,
    "name": "JOHN DOE",
    "status": "ACTIVE"
  }
}
```

**Response** (200 - Invalid):
```json
{
  "meta": {
    "code": 200,
    "message": "NIK not found"
  },
  "data": {
    "nik": "0000000000000000",
    "valid": false,
    "reason": "NIK_NOT_FOUND"
  }
}
```

**Response** (200 - Invalid):
```json
{
  "meta": {
    "code": 200,
    "message": "NIK is not valid"
  },
  "data": {
    "valid": false,
    "reason": "NIK_INACTIVE"
  }
}
```

> **Hanya NIK yang dikirim ke Dukcapil.** Nama dan tanggal lahir tidak dikumpulkan maupun
> dikirim, dan respons tidak pernah memuat nama (FR-15a, `requirements.md` §12).

**Response** (200 - NIK sudah dipakai untuk event ini):
```json
{
  "meta": { "code": 200, "message": "NIK already used for this event" },
  "data": {
    "valid": true,
    "already_used": true,
    "reason": "NIK_ALREADY_USED"
  }
}
```

> `already_used` bersifat **indikatif, bukan reservasi** — ia hanya memberi tahu customer lebih
> awal bahwa NIK ini sudah terpakai, alih-alih membiarkannya mengisi tiga NIK lalu ditolak
> setelah menekan Pay. Penegak sebenarnya adalah `SET nik:used NX` dan `UNIQUE (event_id,
> nik_hmac)` pada saat hold, yang berjalan atomik. Dua customer yang mengetik NIK yang sama
> keduanya bisa melihat `already_used: false`; yang menekan Pay lebih dulu menang, yang kalah
> menerima `NIK_TAKEN`.
>
> NIK dinormalisasi sebelum dicocokkan: spasi dan pemisah dibuang, wajib tepat 16 digit.

**Validation Errors** (400):
```json
{
  "meta": {
    "code": 400,
    "message": "NIK must be exactly 16 digits"
  },
  "data": {}
}
```

**Rate Limited** (429):
```json
{
  "meta": {
    "code": 429,
    "message": "Please wait before validating another NIK"
  },
  "data": {
    "retry_after": 5
  }
}
```

**Reason Values**:
| Reason | Description |
|--------|-------------|
| `NIK_NOT_FOUND` | NIK tidak terdaftar di Dukcapil |
| `NIK_INACTIVE` | NIK terdaftar tapi status tidak aktif |
| `DUKCAPIL_UNAVAILABLE` | Gagal menghubungi Dukcapil — **pembayaran diblokir**, bukan fallback |

> **NIK terdaftar atau tersimpan tidak memanggil Dukcapil.** Bila `nik` cocok dengan
> `accounts.nik_hmac` pemegang token atau salah satu entri `saved_niks` miliknya, endpoint
> menjawab `valid: true` dari bukti verifikasi yang tersimpan — tanpa panggilan berbayar.
> Response menyertakan `"source": "registered" | "saved" | "verified"`. Dua yang pertama
> adalah data milik pemanggil sendiri, sehingga frontend dapat menampilkan centang seketika
> tanpa menunggu jaringan.
>
> **`verified` sengaja menggabungkan cache hit dan panggilan Dukcapil baru.** Cache
> `nik:valid:{hmac}` dikunci oleh hash NIK saja, bukan oleh akun, sehingga membedakan
> keduanya akan memberi tahu pemanggil bahwa NIK itu baru saja diverifikasi **oleh orang
> lain** — sebuah probe untuk mengetahui NIK mana yang aktif membeli tiket di platform ini.
> Frontend tidak membutuhkan perbedaan itu.
>
> Verifikasi tidak dilewati, hanya tidak diulang. NIK di luar sumber-sumber itu tetap
> melalui Dukcapil, dan aturan di bawah berlaku penuh.
>
> **Customer tamu membayar penuh**: tanpa akun, `registered` dan `saved` tidak dapat berlaku,
> sehingga setiap NIK yang ia ketik memanggil Dukcapil kecuali kebetulan masih ada di cache
> 24 jam.

> **Fail closed.** Baik NIK yang ditolak Dukcapil maupun Dukcapil yang tidak dapat dihubungi
> sama-sama memblokir pembayaran. `POST /bookings` menolak permintaan kecuali **setiap** NIK
> di dalamnya sudah terverifikasi. Konsekuensinya Dukcapil adalah single point of failure yang
> diterima secara sadar: saat ia mati, validasi NIK gagal cepat dengan error yang dapat di-retry,
> sementara antrian tetap maju (pause dihapus, ADR-006 §4.3).

**Requirement Coverage**: FR-15a (validasi NIK ke Dukcapil)

---

### 4.3 Create Booking

Submit pemilihan tiket. UI: "Lanjut ke Pembayaran" button.

**Endpoint**: `POST /bookings`

**Headers**:
```
X-Grant-Token: {grant_token}
X-Device-Id: {device_id}               # wajib bila customer TIDAK login
```

> `Authorization` tidak dikirim ke endpoint ini — lihat tabel di awal §4.

**Request**:
```json
{
  "event_id": "evt_konser_a",
  "tickets": [
    {
      "category_id": "cat_vip",
      "nik": "3171xxxxxxxxxx22",
      "holder_email": "buyer@example.com"
    },
    {
      "category_id": "cat_vip",
      "nik": "3171xxxxxxxxxx33",
      "holder_email": "friend@example.com"
    }
  ],
  "contact_email": "buyer@example.com",
  "terms_version": "2026-10-01"
}
```

> **Multi-kategori diizinkan.** `tickets[]` boleh memuat `category_id` yang berbeda-beda —
> misalnya 1 Premium + 1 Festival + 1 VIP. Batas FR-13 adalah **3 tiket per transaksi secara
> total**, bukan 3 per kategori. Hold-nya all-or-nothing: stok seluruh kategori diperiksa
> sebelum satu pun dikurangi, dalam satu skrip atomik.
>
> **Satu baris per kursi** (BR-02: 1 NIK = 1 seat). `holder_email` adalah alamat tujuan
> e-tiket kursi itu — FR-26 menuntut tiket sampai ke tiap pemegang NIK, bukan hanya pembeli.
> Tidak ada field nama: Dukcapil hanya memvalidasi NIK.
>
> **Panggilan inilah yang memulai TTL pembayaran.** "Booking dibuat" pada BR-05 adalah saat
> ini: kursi dipilih, NIK diisi, stok di-hold. `payment_ttl.ttl_seconds` karena itu selalu
> **600** — setiap customer mendapat 10 menit penuh, berapa pun lama ia memilih kursi.
> TTL booking (5 menit, dihitung dari `granted_at` — **bukan** dari saat halaman dibuka)
> berhenti berlaku begitu booking ini dibuat; keduanya berurutan,
> tidak tumpang tindih.
>
> **Idempoten.** Submit yang diulang tidak menjalankan hold kedua. Server mengunci baris
> booking (`SELECT … FOR UPDATE` atas `grant_token`); bila status sudah `HELD` atau lebih
> jauh, booking yang ada dikembalikan dengan `200`. Tanpa guard ini retry akan gagal dengan
> `NIK_TAKEN` atas NIK customer sendiri.

**Response** (201):
```json
{
  "meta": {
    "code": 201,
    "message": "Booking created successfully"
  },
  "data": {
    "booking_id": "RB-00912",
    "status": "PENDING_PAYMENT",
    "event": {
      "id": "evt_konser_a",
      "name": "Konser A — Jakarta",
      "date": "2026-12-25",
      "venue": "GBK, Jakarta"
    },
    "tickets": [
      {
        "category": "VIP",
        "price": 1500000,
        "nik_masked": "3171********xx22"
      },
      {
        "category": "VIP",
        "price": 1500000,
        "nik_masked": "3171********xx33"
      }
    ],
    "summary": {
      "subtotal": 3000000,
      "service_fee": 0,
      "total": 3000000
    },
    "payment_ttl": {
      "expires_at": "2026-10-23T13:25:00Z",
      "ttl_seconds": 600,
      "server_time": "2026-10-23T13:15:00Z"
    },
    "redirect_url": "/bookings/RB-00912/payment"
  }
}
```

**Validation Errors** (400):
```json
{
  "meta": {
    "code": 400,
    "message": "Maximum 3 tickets per order"
  },
  "data": {}
}
```

**Response** (400 - NIK Invalid):
```json
{
  "meta": {
    "code": 400,
    "message": "NIK 3171********xx22 tidak terdaftar di Dukcapil"
  },
  "data": {}
}
```

> **Tidak ada error "nama tidak cocok", dan tidak ada field nama pada respons.** Dukcapil
> memvalidasi **NIK saja**; form tidak mengumpulkan nama dan §4.2 tidak pernah
> mengembalikannya (FR-15a). v2.4 masih memuat keduanya sebagai sisa versi lama — dihapus di
> v2.5.

**Response** (409 - NIK Already Used):
```json
{
  "meta": {
    "code": 409,
    "message": "NIK 3171xxxxxxxxxx22 sudah digunakan untuk event ini"
  },
  "data": {}
}
```

**Response** (409 - Insufficient Stock):
```json
{
  "meta": {
    "code": 409,
    "message": "Stok VIP tidak mencukupi"
  },
  "data": {
    "available": 1,
    "requested": 2
  }
}
```

**Requirement Coverage**: FR-13, FR-14, FR-15, FR-16, NFR-04 (atomic stock), BR-01, BR-02

---

### 4.4 Get Booking Detail

Detail booking untuk Payment Page dan Success Page.

**Endpoint**: `GET /bookings/{booking_id}`

**Headers**:
```
X-Grant-Token: {grant_token}
X-Device-Id: {device_id}               # wajib bila customer TIDAK login
```

> `Authorization` tidak dikirim ke endpoint ini — lihat tabel di awal §4.

**Response** (200 - Pending Payment):
```json
{
  "meta": {
    "code": 200,
    "message": "Success"
  },
  "data": {
    "booking_id": "RB-00912",
    "status": "PENDING_PAYMENT",
    "event": {
      "id": "evt_konser_a",
      "name": "Konser A — Jakarta",
      "date": "2026-12-25",
      "venue": "GBK, Jakarta",
      "image_url": "https://cdn.rusuh.id/events/konser-a.jpg"
    },
    "tickets": [
      {
        "category": "VIP",
        "price": 1500000,
        "nik_masked": "3171********xx22"
      }
    ],
    "summary": {
      "quantity": 1,
      "price_per_ticket": 1500000,
      "total": 1500000
    },
    "payment_ttl": {
      "expires_at": "2026-10-23T13:25:00Z",
      "ttl_seconds": 342,
      "server_time": "2026-10-23T13:19:18Z"
    }
  }
}
```

**Response** (200 - Paid):
```json
{
  "meta": {
    "code": 200,
    "message": "Success"
  },
  "data": {
    "booking_id": "RB-00912",
    "status": "PAID",
    "event": {
      "id": "evt_konser_a",
      "name": "Konser A — Jakarta",
      "date": "2026-12-25",
      "venue": "GBK, Jakarta",
      "image_url": "https://cdn.rusuh.id/events/konser-a.jpg"
    },
    "tickets": [
      {
        "ticket_id": "TKT-001",
        "category": "VIP",
        "price": 1500000,
        "nik_masked": "3171********xx22",
        "eticket_url": "/tickets/TKT-001/eticket?t=et_9f3a2c"
      }
    ],
    "summary": {
      "quantity": 1,
      "total": 1500000
    },
    "paid_at": "2026-10-23T13:20:00Z",
    "payment_method": "Virtual Account"
  }
}
```

> **`eticket_url` sudah membawa token e-tiket (`?t=`) saat `PAID`.** Token yang sama dengan
> tautan email (§6.2): HMAC atas satu `ticket_id`, berlaku sampai tanggal event + 1 hari,
> diterbitkan lewat `pkg/eticket`. Success Page menautkannya langsung, sehingga customer —
> termasuk tamu — dapat membuka e-tiket **tanpa akun dan tanpa menunggu email**. Inilah jalur
> NFR-10 ("email adalah kenyamanan"), menggantikan `GET /bookings/{id}/tickets` yang dihapus.
> Endpoint ini tetap dijaga `X-Grant-Token`; setelah grant kedaluwarsa, tautan yang sudah
> diterima tetap berlaku.

**Requirement Coverage**: FR-21, FR-22, FR-27, NFR-10

---

## 5. Payment

Header dan aturannya sama dengan §4: `X-Grant-Token` wajib, `X-Device-Id` wajib bila tidak login,
`Authorization` tidak dikirim.

### 5.1 Get Payment Methods

Metode pembayaran yang tersedia beserta **parameter yang dibutuhkan klien**. UI: Payment
Method selection.

**Endpoint**: `GET /payments/{booking_id}/methods`

**Headers**:
```
X-Grant-Token: {grant_token}
X-Device-Id: {device_id}               # wajib bila customer TIDAK login
```

> `Authorization` tidak dikirim ke endpoint ini — lihat tabel di awal §4.

**Response** (200):
```json
{
  "meta": { "code": 200, "message": "Success" },
  "data": {
    "methods": [
      {
        "id": "bank_transfer",
        "name": "Transfer Bank / Virtual Account",
        "description": "Verifikasi otomatis, tanpa unggah bukti.",
        "icon_url": "https://cdn.rusuh.id/icons/bank.svg",
        "enabled": true,
        "banks": [
          { "code": "bca",  "name": "BCA",  "icon_url": "https://cdn.rusuh.id/icons/bca.svg" },
          { "code": "bni",  "name": "BNI",  "icon_url": "https://cdn.rusuh.id/icons/bni.svg" },
          { "code": "bri",  "name": "BRI",  "icon_url": "https://cdn.rusuh.id/icons/bri.svg" },
          { "code": "cimb", "name": "CIMB Niaga", "icon_url": "https://cdn.rusuh.id/icons/cimb.svg" }
        ]
      },
      {
        "id": "gopay",
        "name": "E-Wallet (GoPay / QRIS)",
        "description": "Scan QR atau buka aplikasi GoPay.",
        "icon_url": "https://cdn.rusuh.id/icons/ewallet.svg",
        "enabled": true
      }
    ]
  }
}
```

| Field | Arti |
|---|---|
| `id` | Dikirim kembali sebagai `method` pada §5.2. Nilainya sama dengan `payment_type` Midtrans |
| `banks` | Hanya untuk `bank_transfer`. Daftar bank yang dibuka |
| `enabled` | `false` saat breaker payment gateway terbuka; UI menonaktifkan pilihan |

**Dua metode saja di POC: `bank_transfer` dan `gopay`.** Kartu kredit dan Over the Counter
dikeluarkan karena alasan yang sama — keduanya tidak dapat menghormati TTL pembayaran 10
menit di sisi gateway. `custom_expiry` **tidak berlaku** untuk kartu (dinyatakan eksplisit
oleh Midtrans), dan OTC menuntut customer datang ke gerai fisik. Dua metode yang tersisa
keduanya menerima `custom_expiry`, sehingga jendela Midtrans dan jendela kita berakhir pada
detik yang sama untuk **setiap** metode yang hidup.

> Tidak ada `requires_tokenization` maupun `client_key` pada respons ini: keduanya hanya
> dibutuhkan kartu. Dengan kartu di luar cakupan, tokenisasi Midtrans.js di browser dan
> seluruh permukaan PCI-DSS ikut keluar dari POC.

**Keputusan lingkup POC — hanya empat bank.** Midtrans mendokumentasikan kasus *reversal*
(`settlement` berubah menjadi `deny` dalam 1–5 menit, dana kembali ke customer) **hanya**
untuk Permata, Mandiri Bill Payment, dan Indomaret. Ketiganya tidak dibuka, sehingga kelas
kegagalan itu tidak dapat terjadi dan handler reversal tidak diperlukan. Lihat
`payment-method/01-bank-transfer-va.md`.

> "Virtual Account" dan "Transfer Bank" adalah **satu** metode di Core API, bukan dua.
> Customer memilih bank dan menerima nomor VA; tidak ada jalur transfer manual dengan
> unggah bukti.

**Requirement Coverage**: FR-20

---

### 5.2 Initiate Payment

Memulai pembayaran dengan metode terpilih. UI: tombol "Bayar Sekarang".

**Endpoint**: `POST /payments/{booking_id}`

**Headers**:
```
X-Grant-Token: {grant_token}
X-Device-Id: {device_id}               # wajib bila customer TIDAK login
```

> `Authorization` tidak dikirim ke endpoint ini — lihat tabel di awal §4.

Bentuk request berbeda per metode:

**Transfer Bank**:
```json
{ "method": "bank_transfer", "bank": "bca" }
```

**E-Wallet**:
```json
{ "method": "gopay" }
```

**Response** (200) — bentuk `data.instructions` mengikuti metode:

**Transfer Bank**:
```json
{
  "meta": { "code": 200, "message": "Payment initiated" },
  "data": {
    "payment_id": "RB-00912:1",
    "method": "bank_transfer",
    "status": "PENDING",
    "instructions": { "bank": "bca", "va_number": "8801234567890" },
    "ttl": { "expires_at": "2026-10-25T10:25:00Z", "ttl_seconds": 540, "server_time": "2026-10-25T10:16:00Z" }
  }
}
```

**E-Wallet**:
```json
{
  "data": {
    "payment_id": "RB-00912:1",
    "method": "gopay",
    "status": "PENDING",
    "instructions": {
      "qr_url": "https://api.midtrans.com/v2/gopay/.../qr-code",
      "deeplink_url": "https://gojek.link/..."
    },
    "ttl": { "expires_at": "2026-10-25T10:25:00Z", "ttl_seconds": 540, "server_time": "2026-10-25T10:16:00Z" }
  }
}
```

> Desktop menampilkan `qr_url`, mobile memakai `deeplink_url`. QR tersebut QRIS — customer
> boleh membayar dengan aplikasi apa pun, dan webhook kemudian datang sebagai
> `payment_type: qris`, bukan `gopay`.

**Response** (200 - sudah pernah diinisiasi): **identik dengan respons pertama**, termasuk
`payment_id` dan `instructions`. Tidak ada charge kedua ke gateway.

> **Idempoten tanpa kunci dari klien.** Baris `bookings` adalah catatan idempotency-nya.
> Server mengunci baris (`SELECT … FOR UPDATE`); bila status sudah `AWAITING_PAYMENT`,
> `payment_ref` yang tersimpan dikembalikan apa adanya. Kunci yang dikirim ke Midtrans adalah
> **`order_id` = `{booking_id}:{payment_attempts}`**, diturunkan server — Core API membawa
> idempotency pada `order_id`, bukan pada header tersendiri, dan menolak `order_id` yang
> sudah dipakai.
>
> Tombol "Bayar" yang terklik dua kali menghasilkan satu charge dan dua respons identik;
> ia tidak boleh terlihat seperti kegagalan.

**Response** (409 - `BOOKING_STATUS_INVALID`): booking belum `HELD` atau sudah terminal.

> **Setelah penolakan, panggil endpoint ini lagi — bukan `/retry`.** Jalur `deny` menaikkan
> `payment_attempts` sendiri, sehingga `order_id` yang diturunkan sudah baru dan Midtrans
> menerimanya. `/retry` hanya untuk membatalkan pembayaran yang masih `pending` (§5.3).

**Response** (503 - `PAYMENT_GATEWAY_UNAVAILABLE`): breaker payment gateway terbuka. Booking tetap `HELD`; klien boleh mencoba lagi selama tenggat pembayaran belum lewat.

### Expiry yang dikirim ke Midtrans

Untuk `bank_transfer` dan `gopay`, charge menyertakan `custom_expiry` yang **menyamai TTL
pembayaran**:

```json
"custom_expiry": { "order_time": "<held_at, WIB>", "expiry_duration": "10", "unit": "minute" }
```

Tanpa ini, expiry VA bawaan Midtrans adalah **24 jam** — VA tetap dapat dibayar sepanjang
hari setelah sweep melepas stoknya dan menjualnya ke orang lain. `order_time` diisi
`held_at`, bukan waktu request, agar jendela Midtrans berakhir bersama jendela kita (BR-05).

**Kedua metode POC menerima `custom_expiry`**, dan itu alasan kartu dikeluarkan: Midtrans
menyatakan eksplisit bahwa `custom_expiry` tidak berlaku untuk kartu, sehingga BR-05 hanya
akan ditegakkan sweep kita tanpa dukungan gateway. Untuk kedua metode, sweep juga memanggil
`POST /v2/{order_id}/expire` saat meng-expire booking, sebagai lapis kedua terhadap selisih
jam.

**Requirement Coverage**: FR-20, FR-21, FR-24 (API shared web/mobile), FR-25 (idempoten)

---

### 5.3 Cancel & Switch Payment Method

Dipanggil **hanya** ketika pembayaran sebelumnya masih `pending` dan customer ingin berganti
metode — misalnya sudah menerima VA BCA lalu memilih GoPay.

**Endpoint**: `POST /payments/{booking_id}/retry`

> **Tidak dipanggil setelah pembayaran ditolak.** Jalur `deny` sudah mengembalikan booking ke
> `HELD` dan menaikkan `payment_attempts` sendiri, dan tidak ada transaksi Midtrans yang
> hidup. Customer cukup memilih metode lain lalu memanggil `POST /payments/{booking_id}`.
>
> **Tidak dipanggil saat customer hanya mengganti pilihan di UI** sebelum menekan "Bayar
> Sekarang". Pada titik itu belum ada apa pun di Midtrans; pilihan metode adalah state
> frontend sampai charge dikirim.

**Headers**:
```
X-Grant-Token: {grant_token}
X-Device-Id: {device_id}               # wajib bila customer TIDAK login
```

> `Authorization` tidak dikirim ke endpoint ini — lihat tabel di awal §4.

**Response** (200):
```json
{
  "meta": { "code": 200, "message": "Previous payment cancelled" },
  "data": {
    "booking_id": "RB-00912",
    "status": "HELD",
    "attempt": 2,
    "cancelled_order_id": "RB-00912:1",
    "payment_expires_at": "2026-10-25T10:25:00Z",
    "server_time": "2026-10-25T10:23:10Z"
  }
}
```

Klien kemudian memanggil `POST /payments/{booking_id}` dengan metode baru.

> Server membatalkan transaksi lama di Midtrans (`POST /v2/{order_id}/cancel`) **sebelum**
> meng-commit perubahan. Bila urutannya dibalik dan cancel gagal, customer memiliki attempt
> yang sudah naik **dan** VA lama yang masih dapat dibayar — dua instrumen hidup untuk satu
> booking.
>
> `cancel`, bukan `expire`: tidak ada yang kedaluwarsa, merchant membatalkannya. Keduanya
> dilaporkan terpisah di dashboard Midtrans.

**Response** (409 - `PAYMENT_ALREADY_SETTLED`): pembayaran lama ternyata sudah lunas saat
cancel dicoba. Tidak ada perubahan; webhook akan mengonfirmasi booking secara normal.

**Response** (404 - `NOT_FOUND`): `booking_id` tidak ada **atau bukan milik pemegang token**. Keduanya
menjawab sama — `403` akan mengonfirmasi bahwa booking itu ada.

**Response** (409 - `BOOKING_STATUS_INVALID`): booking tidak dalam `AWAITING_PAYMENT`. Termasuk
`HELD` setelah penolakan: tidak ada transaksi Midtrans yang hidup, panggil §5.2.

**Response** (409 - `PAYMENT_WINDOW_CLOSED`): TTL sudah lewat, atau sisanya di bawah
`MIN_RETRY_REMAINING_SECONDS`. Menerbitkan instrumen baru dengan sisa 15 detik hanya
memindahkan kegagalan ke tempat yang lebih menyakitkan.

**Response** (429 - `MAX_ATTEMPTS_EXCEEDED`): melewati `MAX_PAYMENT_ATTEMPTS`. Tiap
percobaan menerbitkan VA/QR baru di Midtrans.

**Response** (503 - `PAYMENT_GATEWAY_UNAVAILABLE`): Midtrans tidak dapat dihubungi. `payment_attempts` **tidak** dinaikkan;
VA lama masih berlaku dan customer dapat membayarnya atau mencoba lagi.

> **TTL tidak di-reset.** `payment_expires_at` tetap dihitung dari `held_at` (BR-05). Customer
> yang berganti metode pada menit ke-8 menerima VA baru dengan sisa 2 menit, bukan 10.

**Requirement Coverage**: FR-20, NFR-02 (Jangan PHP)

---

### 5.4 Get Payment Status

Polling status pembayaran. UI: Payment Page polling.

**Endpoint**: `GET /payments/{booking_id}/status`

**Headers**:
```
X-Grant-Token: {grant_token}
X-Device-Id: {device_id}               # wajib bila customer TIDAK login
```

> `Authorization` tidak dikirim ke endpoint ini — lihat tabel di awal §4.

**Response** (200 - Pending):
```json
{
  "meta": {
    "code": 200,
    "message": "Payment pending"
  },
  "data": {
    "status": "PENDING",
    "server_time": "2026-10-23T13:22:00Z",
    "ttl_seconds": 180
  }
}
```

**Response** (200 - Paid):
```json
{
  "meta": {
    "code": 200,
    "message": "Payment successful"
  },
  "data": {
    "status": "PAID",
    "paid_at": "2026-10-23T13:23:00Z",
    "redirect_url": "/bookings/RB-00912/success"
  }
}
```

**Response** (200 - Expired):
```json
{
  "meta": {
    "code": 200,
    "message": "Payment expired"
  },
  "data": {
    "status": "EXPIRED",
    "expired_at": "2026-10-23T13:25:00Z"
  }
}
```

**Response** (200 - Failed): pembayaran ditolak, **stok masih milik customer**.
```json
{
  "meta": { "code": 200, "message": "Payment failed" },
  "data": {
    "status": "FAILED",
    "reason": "Pembayaran ditolak penyedia",
    "can_retry": true,
    "attempt": 2,
    "payment_expires_at": "2026-10-25T10:25:00Z",
    "server_time": "2026-10-25T10:19:40Z"
  }
}
```

> `reason` diteruskan apa adanya dari `status_message` gateway dan **tidak** diterjemahkan —
> ia untuk dukungan, bukan untuk dirender sebagai pesan utama. Frontend menampilkan teks
> lokalnya sendiri dan menaruh `reason` di detail. Dengan kartu di luar lingkup (§5.1),
> penolakan yang mungkin terjadi hanya berasal dari `bank_transfer` dan `gopay`.

> **Inilah satu-satunya sinyal bahwa pembayaran gagal.** Tanpa status ini customer menatap
> countdown yang terus berjalan tanpa tahu kartunya ditolak. Frontend yang menerimanya
> kembali ke daftar metode — **tanpa memanggil `/retry`**: jalur `deny` sudah menaikkan
> `payment_attempts`, dan tidak ada transaksi Midtrans yang perlu dibatalkan.
>
> `payment_expires_at` tetap dihitung dari `held_at`; countdown **tidak** di-reset.

### Pemetaan status booking → status API

| Status `bookings` | `data.status` | Arti bagi frontend |
|---|---|---|
| `AWAITING_PAYMENT` | `PENDING` | Tetap di layar instruksi, lanjut polling |
| `CONFIRMED` | `PAID` | Ke halaman tiket |
| `HELD` dengan `payment_attempts > 0` | **`FAILED`** | Kembali ke daftar metode |
| `HELD` dengan `payment_attempts = 0` | `PENDING` | Belum pernah charge; tetap di daftar metode |
| `PENDING_CANCELLATION` | `PENDING` | TTL lewat, grace 30 detik berjalan. Tampilkan "memverifikasi pembayaran…", **jangan** layar kedaluwarsa |
| `EXPIRED` / `LAPSED` | `EXPIRED` | Layar kedaluwarsa |

**Requirement Coverage**: FR-22, FR-23

---

### 5.5 Payment Gateway Callback (Webhook)

Callback dari payment gateway. Endpoint internal — tidak dipanggil klien.

**Endpoint**: `POST /payments/webhook`

**Headers**: tidak ada header kustom.

> **v2.4 mendeklarasikan `X-Callback-Signature: sha256=...` dan itu salah dua kali.** Midtrans
> **tidak mengirim header tanda tangan apa pun** pada HTTP notification Core API — tanda
> tangannya adalah field `signature_key` **di dalam body**. Dan skemanya SHA512, bukan SHA256;
> contoh header v2.4 bertentangan dengan formula yang ditulis lima baris di bawahnya.
> Implementasi yang mengikuti v2.4 secara literal akan menuntut header yang tidak pernah
> datang, lalu **menolak setiap notifikasi yang sah**.

> Dilindungi **resource policy API Gateway** (allowlist IP gateway) ditambah verifikasi
> `signature_key`. Tidak ada CAPTCHA dan tidak ada rate limit per-IP pada endpoint ini.

**Request** (dari gateway) — bentuk Core API yang sebenarnya:
```json
{
  "transaction_time": "2026-10-24 20:23:00",
  "transaction_status": "settlement",
  "transaction_id": "9f2a1c7b-4e1a-4c7d-9f3b-2a1c7b9f2a1c",
  "status_message": "midtrans payment notification",
  "status_code": "200",
  "signature_key": "fe5f725ea770c451…",
  "settlement_time": "2026-10-24 20:23:05",
  "payment_type": "bank_transfer",
  "order_id": "RB-00912:1",
  "merchant_id": "G123456789",
  "gross_amount": "3000000.00",
  "fraud_status": "accept",
  "currency": "IDR",
  "va_numbers": [{ "bank": "bca", "va_number": "8801234567890" }]
}
```

> **v2.4 memuat bentuk generik** (`external_id`, `status: "PAID"`, `paid_amount`) yang
> bertentangan dengan tabel transisi di bawahnya sendiri — tabel itu memakai
> `transaction_status` dan `fraud_status`. Tidak satu pun field dalam contoh v2.4 dikirim
> Midtrans. Diganti dengan bentuk yang sebenarnya.
>
> `gross_amount` datang sebagai **string berdesimal** (`"3000000.00"`), bukan integer. Ia
> masuk formula tanda tangan **apa adanya**: menormalkannya lebih dulu menghasilkan hash yang
> tidak pernah cocok.
>
> `transaction_time` dan `settlement_time` berformat `YYYY-MM-DD HH:mm:ss` **waktu WIB tanpa
> penanda zona** — bukan ISO-8601. Parse sebagai `Asia/Jakarta`.

**Verifikasi tanda tangan**:

```
signature_key == SHA512(order_id + status_code + gross_amount + ServerKey)
```

Konkatenasi polos tanpa pemisah. Tidak cocok → **abaikan notifikasi**, jangan proses, jangan
ubah status apa pun.

> Dokumentasi Midtrans memuat dua skema tanda tangan. SHA256withRSA pada halaman *Security
> Specification* berlaku untuk lini API lain, **bukan** webhook Core API.

> **`gross_amount` dipakai sebagai string yang diterima**, termasuk desimalnya. Formulanya
> konkatenasi mentah; `3000000` dan `"3000000.00"` menghasilkan dua hash berbeda dan hanya
> yang kedua cocok.

> **Pemeriksaan kedua yang direkomendasikan Midtrans**: panggil Get Transaction Status dan
> bandingkan hasilnya dengan notifikasi. Di sistem ini pemeriksaan itu **sudah ada** sebagai
> charge reconciler (`rusuh-admission-ctl`, tiap 5 menit, read-only) — jadi notifikasi tidak
> perlu memanggilnya inline dan menambah latensi ke jalur webhook.

**Idempotency**: kunci **diturunkan**, karena Midtrans tidak mengirim ID event.

```
gateway_event_id = order_id + ':' + transaction_status + ':' + status_code
```

Disimpan di `payment_events.gateway_event_id` dengan constraint `UNIQUE`. Pengiriman ulang
memicu unique violation, seluruh transaksi batal, dan endpoint tetap menjawab `200`.

> **v2.4 memetakan idempotency ke field `event_id`, dan field itu tidak ada.** Notifikasi
> Core API tidak memuat ID event dalam bentuk apa pun, sehingga pemetaan itu tidak dapat
> diimplementasikan: `gateway_event_id` akan selalu NULL dan constraint `UNIQUE` tidak
> menjaga apa pun.
>
> **Mengapa bukan `transaction_id` saja.** Ia stabil **sepanjang hidup satu transaksi**:
> notifikasi `pending` dan `settlement` untuk pembayaran yang sama membawa `transaction_id`
> yang identik. Memakainya sendirian karena itu mengulangi persis kesalahan yang dihindari
> dengan tidak memakai `booking_id` — ia **menelan setiap event sah berikutnya** untuk
> transaksi yang sama.
>
> **Mengapa bukan `transaction_id` + status.** Ia bekerja, tetapi ketiga komponen di atas
> adalah **field yang sama yang masuk formula tanda tangan** (`order_id`, `status_code`) plus
> `transaction_status` yang sudah dipakai tabel transisi. Tidak ada field tambahan yang perlu
> dipercaya, dan `order_id` sudah memuat `booking_id` dan nomor percobaan — sehingga kunci ini
> dapat dibaca manusia saat rekonsiliasi (§6.3).
>
> **Yang tetap**: satu baris per *event*, bukan per *booking*. `pending → settlement → refund`
> menghasilkan tiga baris, dan ketiganya diproses.

**Perlakuan per tipe event**:

| `transaction_status` | `fraud_status` | Transisi | Efek |
|---|---|---|---|
| `settlement` | `accept` | → `CONFIRMED` | Lepas slot, terbitkan tiket per `booking_item`, kirim e-tiket ke tiap `holder_email` |
| `capture` (kartu) | **`accept`** | → `CONFIRMED` | Idem. `capture` menjadi `settlement` keesokan hari tanpa tindakan |
| `capture` (kartu) | **bukan `accept`** | tidak ada | **Belum lunas.** Jangan terbitkan tiket; tetap `AWAITING_PAYMENT` |
| `settlement` / `capture` atas booking **`EXPIRED`** | — | tidak ada | **`payment.refund` diterbitkan** + alarm `LatePaymentRefund`. Uang masuk setelah stok dijual ulang |
| `deny` / `cancel` / `failure` | — | → `HELD`, `payment_ref` dikosongkan, **`payment_attempts` dinaikkan** | Stok **tetap** milik customer sampai TTL. Kenaikan attempt membuat charge berikutnya memakai `order_id` baru, sehingga customer cukup memilih metode lain dan memanggil §5.2 |
| `expire` | — | → `EXPIRED` | Lepas stok, bebaskan NIK, kirim email expired |
| `pending` | — | tidak ada | Normal bagi VA dan e-wallet yang belum dibayar |
| `refund` / `partial_refund` / `chargeback` | — | tidak ada | Dicatat di `payment_events` + alarm; di luar lingkup POC |

> **`transaction_status` saja tidak cukup untuk kartu.** `capture` dengan `fraud_status`
> selain `accept` berarti dana **belum** diterima. Memperlakukan `capture` sebagai lunas akan
> menerbitkan tiket untuk transaksi yang tidak pernah didanai.
>
> **`payment_type` pada webhook dapat berbeda dari yang dikirim.** Customer yang membayar QR
> GoPay memakai aplikasi QRIS lain menghasilkan notifikasi `payment_type: qris`. Handler
> tidak boleh mencocokkan `payment_type` webhook dengan yang dikirim saat charge.
>
> **Reversal tidak ditangani**, dan tidak perlu: `settlement → deny` hanya terjadi pada
> Permata, Mandiri Bill, dan Indomaret, yang tidak dibuka di POC (§5.1).

**Response** (200) — **selalu**, termasuk pada duplikat:
```json
{
  "meta": { "code": 200, "message": "Callback processed" },
  "data": {}
}
```

**Requirement Coverage**: FR-25 (idempoten callback), FR-22, FR-23, FR-26, NFR-07 (PCI-DSS via gateway)

---

## 6. Tickets (Accounts › Tickets)

> Sebelumnya bernama **User**. Di Apidog berada di dalam folder Accounts.

### 6.1 Get My Tickets

List tiket user. UI: "Tiket saya" menu.

**Endpoint**: `GET /users/me/tickets`

**Headers**:
```
Authorization: Bearer {access_token}
```

> Token **sesi akun**, bukan grant. Endpoint ini tidak menerima `X-Grant-Token`:
> kepemilikan di sini diselesaikan lewat `account_id`, dan grant tidak membawa
> identitas akun apa pun (§7).

**Query Parameters**:
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `status` | string | No | `upcoming`, `past`, `all` |
| `limit` | int | No | Default: 10 |
| `offset` | int | No | Default: 0 |

**Response** (200):
```json
{
  "meta": {
    "code": 200,
    "message": "Success",
    "pagination": {
      "totalData": 1,
      "totalPage": 1,
      "limit": 10,
      "offset": 0
    }
  },
  "data": [
    {
      "ticket_id": "TKT-001",
      "booking_id": "RB-00912",
      "event": {
        "id": "evt_konser_a",
        "name": "Konser A — Jakarta",
        "date": "2026-12-25",
        "venue": "GBK",
        "image_url": "https://cdn.rusuh.id/events/konser-a.jpg"
      },
      "category": "VIP",
      "price": 1500000,
      "status": "VALID",
      "eticket_url": "/tickets/TKT-001/eticket"
    }
  ]
}
```

---

### 6.2 Get E-Ticket

Download/view e-ticket. UI: "Lihat E-Tiket" button.

**Endpoint**: `GET /tickets/{ticket_id}/eticket[?t={eticket_token}]`

**Dua jalan masuk, dan keduanya wajib ada:**

| Kredensial | Dipakai oleh | Resolusi |
|---|---|---|
| `Authorization: Bearer {access_token}` | pemilik akun, dari menu "Tiket Saya" (§6.1) | klausa kepemilikan: *pembeli **atau** pemegang NIK terdaftar* |
| `?t={eticket_token}` | **pemegang NIK**, dari tautan di email e-tiket | HMAC terhadap `ticket_id`; tidak membawa identitas akun |

Endpoint ini tidak menerima `X-Grant-Token`: grant membuktikan slot, bukan kepemilikan tiket,
dan grant sudah kedaluwarsa jauh sebelum tanggal event.

> **Mengapa `?t=` wajib ada, bukan kemudahan tambahan.** FR-26 menuntut tiket dikirim ke
> **masing-masing pemegang NIK**, dan pemegang NIK kedua dan ketiga biasanya bukan pemilik
> akun — sering bukan pembeli sama sekali. Rute yang hanya menerima sesi akun berarti tombol
> "Buka e-tiket" di email mereka menjawab **401**: jalur pengiriman utama requirement itu
> tidak berfungsi untuk orang-orang yang requirement itu ditulis untuk mereka. Hal yang sama
> berlaku bagi pembeli tamu, yang setelah §3.3 menghapus kredensial antriannya hanya memiliki
> email.
>
> **Bentuk tokennya.** HMAC bertanda tangan atas `ticket_id` + kedaluwarsa, stateless sehingga
> verifikasinya tidak membaca store, dengan kunci khusus — bukan signing secret JWT dan bukan
> `DEVICE_PEPPER`. Lingkupnya **satu tiket**: ia tidak membuka tiket lain dalam booking yang
> sama, tidak membuka §6.1, dan tidak membawa `account_id`. Berlaku sampai tanggal event + 1
> hari, karena tiket memang perlu dibuka di gerbang.
>
> **Dicabut hanya dengan rotasi kunci**, dan rotasi membatalkan setiap tautan yang sudah
> terkirim. Karena itu kuncinya tidak dirotasi di dalam masa penjualan event aktif.
>
> **Residual, dinyatakan terang:** meneruskan email berarti meneruskan tiket. Itu benar untuk
> e-tiket mana pun, dan BR-02 (1 NIK = 1 kursi) tetap mengikat kursi pada NIK, bukan pada
> penerima email. Pemeriksaan identitas di gerbang venue di luar cakupan POC (ADR-005
> Bagian 8), jadi itulah batas yang kita punya.

**Headers** (jalur akun):
```
Authorization: Bearer {access_token}
```

**Response** (200):
```json
{
  "meta": {
    "code": 200,
    "message": "Success"
  },
  "data": {
    "ticket_id": "TKT-001",
    "event": {
      "name": "Konser A — Jakarta",
      "date": "2026-12-25",
      "time": "19:00",
      "venue": "GBK, Jakarta"
    },
    "category": "VIP",
    "holder": {
      "nik_masked": "3171********xx22"
    },
    "qr_code": "data:image/png;base64,iVBORw0KGgo...",
    "qr_value": "TKT-001|evt_konser_a|3171xxxxxxxxxx22",
    "instructions": "Tunjukkan QR code dan identitas pemegang saat masuk venue."
  }
}
```

**Requirement Coverage**: FR-27 (QR code, NIK pemegang)

---

### 6.3 Disputed Payment Reconciliation

Booking yang menerima callback sukses **setelah** pembatalan final. Refund **tidak**
dieksekusi otomatis di POC; operator merekonsiliasi lewat dashboard Midtrans.

**Endpoint**: `GET /admin/reconciliation/disputed?event_id={event_id}`

```json
{
  "meta": { "code": 200, "message": "Success" },
  "data": {
    "disputed": [
      {
        "booking_id": "RB-00912",
        "order_id": "RB-00912:2",
        "gateway_event_id": "evt_9f2a1c7b",
        "gross_amount": 1500000,
        "payment_type": "bank_transfer",
        "paid_at": "2026-10-26T10:25:48Z",
        "cancelled_at": "2026-10-26T10:25:30Z",
        "contact_email": "buyer@example.com",
        "refund_status": "PENDING_MANUAL"
      }
    ]
  }
}
```

> Setiap baris memuat `order_id` dan `gateway_event_id`, sehingga pencocokan di dashboard
> Midtrans tidak memerlukan pencarian manual. Customer sudah menerima email
> `payment.failed_notice` yang menyatakan dana diterima, booking batal, dan refund diproses —
> email itu wajib, bukan sekadar catatan log.
>
> **Sebelum produksi**: otomatiskan lewat `POST /v2/{order_id}/refund`, dengan alarm pada
> setiap eksekusi. Yang ditunda di POC adalah pemanggilannya, bukan desainnya.

Diamankan sama seperti dashboard operasional: terpisah dari jalur customer, tidak memakai
grant token antrian.

**Requirement Coverage**: FR-25, NFR-09

---

## 7. Common Response Structure

### Success Response

```json
{
  "meta": {
    "code": 200,
    "message": "Success",
    "server_time": "2026-11-03T09:12:00.412Z"
  },
  "data": { ... }
}
```

> **`server_time` ada pada SETIAP respons, sukses maupun error** (BR-10). Ia bukan field
> per-endpoint: ia bagian dari envelope, diisi satu kali di layer respons bersama dan tidak
> pernah ditulis oleh handler.
>
> **Tidak ada `GET /time`.** Klien menghitung `offset = server_time − jam lokal` sekali dari
> panggilan yang memang harus ia lakukan, lalu merender **semua** hitung mundur terhadap
> offset itu. Jam perangkat tidak pernah dipercaya langsung. Klien yang belum membuka stream
> karena itu sudah punya offsetnya; yang sudah membuka stream mendapat `serverTime` pada
> setiap payload SSE (§3.2).
>
> **Contoh per endpoint di dokumen ini menghilangkan `server_time` demi keringkasan.** Ia
> tetap selalu ada. Yang **tidak** dihilangkan adalah `server_time` di dalam `data` — pada
> §3.3, §4.1, §4.3, §4.4, §5.2, §5.3 dan §5.4 ia menyertai sebuah **tenggat tertentu**
> (`expires_at`, `payment_expires_at`), dan itu hal yang berbeda: envelope memberi jam
> server, field di dalam `data` memberi pasangan jam-dan-tenggat yang hitung mundurnya
> diturunkan darinya.

### Success Response with Pagination

```json
{
  "meta": {
    "code": 200,
    "message": "Success",
    "server_time": "2026-11-03T09:12:00.412Z",
    "pagination": {
      "totalData": 10,
      "totalPage": 1,
      "limit": 10,
      "offset": 0
    }
  },
  "data": [ ... ]
}
```

### Error Response

```json
{
  "meta": {
    "code": 400,
    "message": "NIK must be exactly 16 digits",
    "error_code": "VALIDATION_ERROR",
    "server_time": "2026-11-03T09:12:00.412Z"
  },
  "data": {}
}
```

> **`error_code` wajib pada setiap error, `message` tidak untuk di-parse.** `message` boleh
> berubah kata-katanya; `error_code` adalah kontrak. `data` tetap objek — berisi detail bila ada
> (mis. `retry_after` pada 429, `queue_state` pada 409/503 join), `{}` bila tidak.

### Error yang berlaku per endpoint (selain yang ditulis di bagiannya)

Bagian per endpoint menulis error yang khas. Error umum berikut juga berlaku dan ada di Apidog:

| Endpoint | Error |
|---|---|
| `GET /auth/verify` | 401 `TOKEN_INVALID` (tautan tidak dikenal, dipakai, atau kedaluwarsa) |
| `POST /auth/refresh` | 400 `VALIDATION_ERROR` (`refresh_token` kosong) |
| `POST /auth/register` | 400 `VALIDATION_ERROR` (format NIK / email) |
| `GET /events/{event_id}` | 404 `NOT_FOUND` |
| `GET /events/{event_id}/queue/me` | 401 `UNAUTHORIZED` (tidak ada kredensial, atau Bearer tanpa `acct` untuk event ini) |
| `GET /events/{event_id}/tickets` | 401 `GRANT_TOKEN_MISSING` / `GRANT_TOKEN_INVALID` |
| `POST /nik/validate` | 400 `VALIDATION_ERROR` · 429 `TOO_MANY_REQUESTS` · 503 `DUKCAPIL_UNAVAILABLE` |
| `GET /bookings/{booking_id}` | 404 `NOT_FOUND` |
| `GET /tickets/{ticket_id}/eticket` | 401 `ETICKET_TOKEN_INVALID` · 404 `NOT_FOUND` |
| semua endpoint ber-grant (§4, §5) | 401 `GRANT_TOKEN_MISSING` / `GRANT_TOKEN_INVALID` / `DEVICE_MISMATCH` |

> `GET /bookings/{booking_id}/tickets` **dihapus** (v2.7, ADR-006 §8.8): §4.4 sudah mengembalikan
> `tickets[]` dengan `eticket_url` bertoken saat `PAID`.

### HTTP Status Codes

| Code | Meaning | When |
|------|---------|------|
| 200 | OK | Request berhasil |
| 201 | Created | Resource baru dibuat (booking, payment) |
| 400 | Bad Request | Validation error (NIK invalid, melebihi batas) |
| 401 | Unauthorized | Token tidak ada atau expired |
| 403 | Forbidden | Token valid tapi tidak punya akses |
| 404 | Not Found | Resource tidak ditemukan |
| 409 | Conflict | NIK sudah digunakan, stok habis |
| 410 | Gone | Tiket sudah habis saat claim |
| 429 | Too Many Requests | Rate limited |
| 500 | Internal Error | Server error |

---

## 8. Review vs Requirements

### Coverage Matrix — Functional

| Req ID | Endpoint | Status |
|--------|----------|--------|
| FR-01 | `GET /events` | ✅ |
| FR-02 | `GET /events` (`status: AVAILABLE`) | ✅ |
| FR-03 | `GET /events` (`status: SOLD_OUT`) | ✅ |
| FR-04 | `POST /events/{id}/queue/join` | ✅ |
| FR-05 | `POST /events/{id}/queue/join` → antre; bila ada tempat kosong, tick admission-ctl berikutnya (≤ 500 ms) memasukkan customer dan klien tahu lewat stream + `/queue/me` (tanpa jalur langsung-masuk, ADR-006 §8.14) | ⚠️ Perlu persetujuan pemilik requirement (ADR-006 §10 #8) |
| FR-06 | Join response + SSE payload `nowServing` | ✅ |
| FR-07 | `GET /events/{id}/queue/stream`, fallback `…/queue/serving` | ✅ |
| FR-08 | broadcast `nowServing ≥ ticket_no` → `GET /events/{id}/queue/me` | ✅ |
| FR-09 | `ratePerMin` pada payload SSE / `rate_per_min` pada §3.4; frontend menghitung estimasi (§3.5) | ✅ |
| FR-10 | Snapshot awal pada stream + `GET …/queue/me` | ✅ |
| FR-11 | `GET /events/{id}` | ✅ |
| FR-12 | `GET /events/{id}/tickets` | ✅ |
| FR-13 | `POST /bookings` (maks 3, ditegakkan di Lua) | ✅ |
| FR-14 | `POST /bookings` → `NO_STOCK` / `QTY_EXCEEDED`, diperiksa lintas seluruh kategori sebelum hold | ✅ |
| FR-15 | Validasi format 16 digit, klien dan server | ✅ |
| FR-15a | `POST /nik/validate` — **NIK saja**, fail closed | ✅ |
| FR-16 | `POST /bookings` membuat hold | ✅ |
| FR-17 | `booking_expires_at` + `server_time` pada response | ✅ |
| FR-18 | Sweep server-side; tidak terlihat di API | ⚙️ Backend |
| FR-19 | `server_time` pada setiap response REST + `serverTime` pada tiap payload SSE | ✅ |
| FR-20 | `GET /payments/{id}/methods`, `POST /payments/{id}`, `POST /payments/{id}/retry` | ✅ |
| FR-21 | `payment_expires_at` + `ttl_seconds` | ✅ |
| FR-22 | Webhook `PAID` → e-tiket per `holder_email` | ✅ |
| FR-23 | Webhook `EXPIRE` / sweep → stok dilepas | ✅ |
| FR-24 | REST/JSON, identik untuk web dan mobile | ✅ |
| FR-25 | `payment_events.gateway_event_id UNIQUE`; charge diputar ulang dari state booking | ✅ |
| FR-26 | `holder_email` per kursi pada §4.3, **dan** tautan e-tiket yang dapat dibuka pemegang NIK tanpa akun (§6.2 `?t=`) | ✅ |
| FR-27 | `GET /tickets/{id}/eticket` — QR, kategori, NIK. **Nomor bangku di luar lingkup POC** | ✅ |
| FR-28 | Webhook `EXPIRE` / sweep → email expired | ⚙️ Backend |
| FR-29 | Semua endpoint pembelian dapat diakses tanpa akun; grant anonim diikat ke `X-Device-Id` (§9) | ✅ |
| FR-30 | `POST /bookings` hanya butuh NIK + email | ✅ |
| FR-31 | `POST /auth/magic-link`, `GET /auth/verify`, `POST /auth/refresh` | ✅ |
| FR-32 | Header `Accept-Language`; pesan error dikembalikan sebagai kode | ✅ |
| FR-33 | CloudWatch EMF `QueueDepth` — tidak ada endpoint aplikasi | ⚙️ Observability |
| FR-34 | CloudWatch EMF `ConversionRate` | ⚙️ Observability |
| FR-35 | CloudWatch EMF `ExpiryRate` per tahap | ⚙️ Observability |

### Coverage Matrix — Non-Functional & Business Rules

| Req ID | Bagaimana dipenuhi di API | Status |
|--------|---------------------------|--------|
| NFR-02 | Tidak ada jalur penolakan setelah `grant`; gagal hold tidak menghilangkan slot | ✅ |
| NFR-04 | Hold stok atomik dalam satu skrip Lua | ⚙️ Backend |
| NFR-05 | FCFS lewat `ticket_no`; **tanpa WAF di POC**, bot pada `/queue/join` hanya ditahan rate limit per-IP | ⚠️ Terdegradasi (POC) |
| NFR-06 | SSE push; p95 < 1 detik | ✅ |
| NFR-07 | `X-Grant-Token` wajib pada endpoint booking/payment, `Authorization` opsional di atasnya; tidak ada data kartu | ✅ |
| NFR-08 | REST/JSON, platform-agnostic | ✅ |
| NFR-09 | CloudWatch EMF, namespace `RusuhBooking/Queue` | ⚙️ Observability |
| NFR-10 | Kegagalan email tidak mengubah status booking; `GET /bookings/{id}` (§4.4) mengembalikan `eticket_url` bertoken `?t=` saat `PAID`, jadi e-tiket dapat dibuka tanpa email | ✅ |
| BR-01 | Maks 3 tiket per `POST /bookings` | ✅ |
| BR-02 | Satu entri per kursi, satu NIK masing-masing; `UNIQUE (event_id, nik_hmac)` menolak NIK yang sudah dipakai booking lain di event yang sama | ✅ |
| BR-03/04 | `booking_expires_at` = `granted_at` + 5 menit — tidak ada grace antrian, jam mulai saat slot diberikan | ✅ |
| BR-05/06 | `payment_expires_at` = `held_at` + 10 menit — berurutan setelah TTL booking, penuh bagi setiap customer | ✅ |
| BR-07 | `POST …/queue/join` memulihkan keterlibatan yang ada untuk akun yang sama — satu per event, user login saja | ✅ |
| BR-09 | `GET /events/{id}/queue/me` mengembalikan token yang sama; `redirect_context` pada §1.3 memulihkan halaman; `POST /auth/refresh` menjaga sesi tetap hidup sepanjang TTL | ✅ |
| BR-10 | `server_time` pada tiap respons; klien menghitung offset jam sekali dan merender semua countdown terhadapnya | ✅ |

Legenda: ✅ terpenuhi di kontrak · ⚙️ di luar permukaan API (backend atau observability) ·
⚠️ terpenuhi sebagian, keterbatasan dicatat

---

## 9. Security Considerations

| Aspect | Implementation |
|--------|----------------|
| Access Token (sesi akun) | JWT HS256, **22740 detik** (= `SALE_WINDOW_SECONDS` + 990 + 150 — ADR-005 §7; **bukan** `GRANT_TOKEN_TTL_SECONDS`, yang mengukur interval lain). Dikirim via `Authorization: Bearer`, dan **hanya kredensial inilah** yang pernah menempati slot itu. Diterbitkan `GET /auth/verify` (§1.3) dan `POST /auth/refresh` (§1.4); juga diterima `GET …/queue/me` sebagai alternatif cookie untuk pemulihan lintas perangkat (§3.3). Stateless: setiap service yang memegang signing secret dapat memverifikasinya tanpa memanggil `user`. Karena tidak ada store yang dibaca saat verifikasi, **umur token inilah jendela pencabutannya** — memperpanjangnya memperlebar jendela itu |
| Grant Token | Opaque acak (`gt_…`), **bukan JWT**, dikirim via header `X-Grant-Token`. Diperoleh dari `GET …/queue/me` setelah `nowServing ≥ ticket_no` (§3.3), tersimpan server di `grant:{token} EX GRANT_TOKEN_TTL_SECONDS`. Isi hash: `event_id`, `booking_id`, `queue_token`, dan `account_id` (login) atau `device_hash` (tamu); umurnya `GRANT_TOKEN_TTL_SECONDS` (1080) sejak `granted_at`, bukan 300 (ADR-006 §8.13). Membuktikan pemegangnya **punya slot**, bukan siapa dirinya — diterbitkan untuk tamu juga. Karena tiap verifikasi membacanya dari Redis, berbeda dari access token ia **dapat dicabut seketika** dengan satu `DEL` |
| Queue Token (ruang tunggu) | `queue_token` cookie (HttpOnly, Secure) untuk web, header `X-Queue-Token` untuk mobile; set via `Set-Cookie` saat queue join. `Max-Age` diturunkan dari `SALE_WINDOW_SECONDS`, sama dengan TTL `token:{uuid}` yang dirujuknya. Membuktikan **posisi dalam antrian**, bukan slot. `Max-Age` adalah batas atas: keadaan terminal menghapusnya lebih cepat, dan `CONFIRMED` menggantinya dengan nisan `settled:{uuid}` yang tidak memulihkan apa pun (§3.3) |
| E-ticket Token | HMAC bertanda tangan atas satu `ticket_id` + kedaluwarsa, dikirim sebagai query `?t=` pada tautan email (§6.2). Stateless, tanpa identitas akun, lingkup satu tiket, berlaku sampai tanggal event + 1 hari. Kunci khusus — bukan signing secret JWT, bukan `DEVICE_PEPPER`. Ada karena FR-26 mengirim tiket ke pemegang NIK yang umumnya tidak punya akun; tanpa ini tombol di email mereka menjawab 401 |
| Refresh Token | **Opaque acak 256-bit, bukan JWT**, 30 hari, disimpan server sebagai hash SHA-256. Sekali pakai dengan **rotasi**; pemutaran ulang mencabut seluruh keluarga (§1.4). Klien menyimpannya aman (Keychain/Keystore/memory) dan menggantinya setiap refresh |
| Signing secret | Terpisah dari pepper HMAC NIK, dan **berbeda sifatnya**: signing secret dapat dan sebaiknya dirotasi, pepper tidak pernah (ADR-005 §8) |
| NIK | Masked in responses (3171********xx22), stored encrypted |
| Payment | Via payment gateway (PCI-DSS compliant), no card data stored |
| Rate Limiting | Per IP on queue join, per user on auth |
| CSRF | `SameSite=Strict` on queue cookie; Origin validation on state-changing endpoints |
| **Object ownership** | Setiap endpoint yang menerima `booking_id` atau `ticket_id` dari path mengikatnya ke pemegang grant: `WHERE id = $1 AND grant_token = $grantToken`. Grant yang sah membuktikan pemegangnya punya slot, bukan bahwa slot itu miliknya. Tidak cocok → **404**, bukan 403. `account_id` baris itu **sudah** terisi sejak INSERT saat GRANT, diambil dari keterlibatannya, jadi tidak ada kredensial klien yang perlu membuktikan kepemilikan akun di sini (§4) |
| **Satu slot, satu kredensial** | Tidak ada endpoint yang menerima dua kredensial berbeda di slot yang sama. Middleware memutuskan per rute, bukan per token: ia tidak mencoba memverifikasi tanda tangan lalu jatuh ke pencarian Redis. Tanpa aturan ini, grant tamu dapat lolos di `GET /me/tickets`, atau JWT akun lolos pada `booking_id` yang bukan miliknya — tepat yang dicegah baris di atas |
| **Atribusi akun** | `bookings.account_id` ditulis pada INSERT saat GRANT dari `token:{uuid}`, **tidak** dari kredensial yang dibawa klien saat checkout. Keterlibatan mengetahui identitasnya sejak join, ketika sesi masih pasti segar; bergantung pada token saat checkout berarti sesi yang kedaluwarsa menurunkan pembelian customer login menjadi pembelian tamu tanpa satu pun error |
| **Pengikatan device (grant anonim)** | `X-Device-Id` — acak 128 bit yang dibuat klien (format tidak divalidasi), disimpan server sebagai `HMAC(DEVICE_PEPPER, device_id)`, dicocokkan pada join (rejoin), `/queue/me` dan setiap panggilan ber-grant. Tidak cocok → **401** `DEVICE_MISMATCH`, **tanpa rebind** (ADR-006 §8.9). Token akun dicocokkan ke `account_id`. **Bukan** fingerprint pasif: canvas/WebGL rapuh dan memanen karakteristik perangkat tanpa dasar persetujuan (NFR-11). **Non-goal**: ia menghentikan jual-beli slot, bukan penyerang yang menyalin `localStorage` korban |
| SSE Auth | `queue_token` cookie (web) atau `X-Queue-Token` (mobile) divalidasi saat koneksi stream. `Authorization` tidak dipakai di sini |

---

## Appendix A: Error Codes

| Code | Description |
|------|-------------|
| `VALIDATION_ERROR` | Input validation failed |
| `UNAUTHORIZED` | **401** — kredensial tidak ada atau tidak dikenal (queue token pada §3.2, §3.3), atau `queue_token` milik sebuah akun dipakai akun lain / tamu pada join (§3.1, v2.9; cookie dihapus) |
| `NOT_FOUND` | **404** — resource tidak ada, atau bukan milik pemanggil (tidak dibedakan) |
| `SALE_NOT_OPEN` | **409** — `queue:{ev}.state` masih `SCHEDULED` (§3.1, §3.2) |
| `SALE_CLOSED` | **410** — `queue:{ev}.state` sudah `CLOSED`; join (§3.1) dan stream (§3.2) ditolak |
| `FULL_BOOKED` | **409** — `queue:{ev}.state` = `FULL_BOOKED`; join pendatang baru ditolak, yang sudah mengantri tetap dipulihkan (§3.1) |
| `QUEUE_NOT_READY` | **503** — `queue:{ev}.state` tidak ada atau tak dikenal: event belum di-seed (§3.1) |
| `BOOKING_STATUS_INVALID` | **409** — booking tidak dalam status yang dibutuhkan endpoint (§5.2, §5.3) |
| `NIK_ALREADY_USED` | NIK sudah digunakan untuk event ini |
| `NIK_NOT_FOUND` | NIK tidak terdaftar di Dukcapil |
| `NIK_INACTIVE` | NIK terdaftar tapi status tidak aktif |
| `NIK_VALIDATION_FAILED` | Validasi NIK ke Dukcapil gagal |
| `NIK_UNVERIFIED` | Ada NIK dalam booking yang belum lolos `POST /nik/validate` |
| `DUKCAPIL_UNAVAILABLE` | Dukcapil tidak dapat dihubungi — pembayaran diblokir (fail closed) |
| `QTY_EXCEEDED` | Melebihi 3 tiket per transaksi, dijumlahkan lintas kategori (BR-01) |
| `QTY_INVALID` | Jumlah tiket sebuah kategori bukan bilangan ≥ 1 |
| `STOCK_UNINITIALIZED` | **503** — kunci stok hilang dari Redis (flush/failover). Bukan "habis": stok tidak diketahui |
| `STOCK_NEGATIVE` | **503** — guard terakhir menyala; seluruh pengurangan dikembalikan |
| `INSUFFICIENT_STOCK` | Stok tiket tidak mencukupi |
| `TICKETS_UNAVAILABLE` | Tiket sudah habis |
| `TERMS_NOT_ACCEPTED` | Syarat & ketentuan dan persetujuan PDP belum diterima |
| `PAYMENT_ALREADY_INITIATED` | Charge sudah berjalan; respons sebelumnya diputar ulang |
| `BANK_NOT_SUPPORTED` | Bank di luar bca/bni/bri/cimb |
| `MAX_ATTEMPTS_EXCEEDED` | Melewati batas percobaan pembayaran per booking |
| `PAYMENT_ALREADY_SETTLED` | Pembayaran lama sudah lunas; pembatalan tidak dilakukan |
| `PAYMENT_WINDOW_CLOSED` | TTL pembayaran habis atau sisanya terlalu pendek untuk ganti metode |
| `QUEUE_FULL` | **503** — antrian mencapai kapasitas maksimum (§3.1) |
| `INTERNAL_ERROR` | **5xx** — kegagalan server tanpa kode yang lebih spesifik (mis. Redis tidak dapat dihubungi). Pesan tidak pernah membawa detail internal |
| `CAPTCHA_REQUIRED` | `captcha_token` tidak disertakan pada `POST …/queue/join` |
| `CAPTCHA_FAILED` | Token CAPTCHA tidak valid, sudah dipakai, atau kedaluwarsa |
| `NIK_ALREADY_REGISTERED` | NIK sudah terikat ke akun lain (satu NIK = satu akun, global) |
| `NIK_ALREADY_SAVED` | NIK sudah ada di Daftar NIK Teman akun ini |
| `MAX_SAVED_NIKS_REACHED` | Daftar NIK Teman sudah mencapai 5 entri |
| `CONSENT_REQUIRED` | Persetujuan pemrosesan data pribadi pihak ketiga belum diberikan |
| `PAYMENT_GATEWAY_UNAVAILABLE` | **503** — breaker payment gateway terbuka; dapat di-retry. Menggantikan `ADMISSION_PAUSED` (v2.7, pause dihapus) |
| `BOOKING_EXPIRED` | TTL booking habis |
| `PAYMENT_EXPIRED` | TTL pembayaran habis |
| `TOO_MANY_REQUESTS` | Rate limited |
| `TOKEN_EXPIRED` | **Hanya tentang sesi akun.** Access token kedaluwarsa, atau refresh token kedaluwarsa/dicabut/tidak dikenal — klien me-refresh (§1.4) atau login ulang |
| `TOKEN_INVALID` | Token tidak sah, **atau refresh token diputar ulang** — pada kasus kedua seluruh keluarga token sudah dicabut dan customer harus login ulang. Satu kode untuk keduanya: memberi kode terpisah akan memberi tahu penyerang bahwa ia memicu deteksi |
| `GRANT_TOKEN_MISSING` | **401** — `X-Grant-Token` tidak disertakan pada endpoint §4/§5. Kode terpisah karena penyebabnya biasanya klien yang masih mengirim grant di `Authorization`, dan `TOKEN_INVALID` tidak mengarahkan siapa pun ke sana |
| `GRANT_TOKEN_INVALID` | **401** — grant tidak dikenal, kedaluwarsa, atau sudah dicabut. Beda dari `BOOKING_EXPIRED`: yang ini soal kredensial, bukan soal TTL booking. Keduanya dapat terjadi berurutan dan frontend merutekannya berbeda |
| `DEVICE_ID_REQUIRED` | **400** — `X-Device-Id` tidak disertakan oleh pemanggil anonim pada `POST …/queue/join` atau endpoint ber-grant. Ditolak di join, bukan nanti: grant tanpa pengikatan tidak dapat diikat belakangan |
| `DEVICE_MISMATCH` | **401** — `queue_token` tamu atau grant dipakai dari perangkat yang bukan penerimanya (v2.9: **tanpa rebind**, ADR-006 §8.9). Di join respons juga menghapus cookie `queue_token`; klien menghapus token antriannya (§3.1) |
| `ETICKET_TOKEN_INVALID` | **401** — token `?t=` pada tautan e-tiket tidak sah, kedaluwarsa, atau tidak cocok dengan `ticket_id` di path. Satu kode untuk ketiganya: membedakannya memberi tahu penebak mana bagian yang hampir benar |

---

## Appendix B: Real-time Strategy

### Queue Status: SSE (Server-Sent Events)

| Aspect | Value |
|--------|-------|
| Transport | SSE (`GET /events/{event_id}/queue/stream`) |
| Auth | Cookie `queue_token` (web — `EventSource` tidak bisa menyetel header) / header `X-Queue-Token` (mobile) — tidak pernah query string |
| Broadcast | Saat `nowServing` berubah, atau tiap 10 detik |
| Perubahan state | Event `status {state}` tiap transisi `OPEN ↔ FULL_BOOKED`, `→ CLOSED` |
| Data basi | Event `stale {stale, state, message?, serverTime}` |
| Tick admission | 500 ms |
| Keepalive | Komentar `: keepalive` tiap **15 detik** (bukan event bernama) |
| Snapshot awal | Ya — dikirim saat koneksi terbuka (FR-10) |
| Reconnect | Otomatis dengan `Last-Event-ID`; `retry:` berjitter saat shutdown |
| Sold out sementara | event `status {state: FULL_BOOKED}` (dan `state` pada payload) → klien **tetap tersambung**; stok dapat kembali |
| Sale ditutup | event `status {state: CLOSED}` + event `sold-out` → klien **wajib** `close()`; koneksi baru ditolak `410 Gone` |

### Payment Status: Polling

| Aspect | Value |
|--------|-------|
| Transport | REST polling (`GET /payments/{id}/status`) |
| Interval | 2–3 detik |
| Duration | Sampai `payment_expires_at` |

**Mengapa polling, padahal tier SSE sudah ada.** Pilihan transport mengikuti **bentuk
fan-out**, bukan ketersediaan infrastruktur.

| | Antrian (SSE) | Status pembayaran (polling) |
|---|---|---|
| Penonton | 300.000 | 1 |
| Nilai | satu `now_serving` bersama | satu booking milik pribadi |
| Pola | siarkan satu nilai ke banyak | antarkan satu nilai ke pemiliknya |
| Batas konkurensi | besar antrian | `MAX_CONCURRENT_SLOTS` |

Keunggulan SSE adalah menyiarkan satu nilai ke audiens besar tanpa satu request per penonton.
Status pembayaran punya audiens **satu orang**, sehingga keunggulan itu tidak berlaku. Pada
~500 slot terbuka, polling 2 detik hanya ~250 rps — sekitar 1,8 juta request untuk sale dua
jam, di bawah $2.

Yang menentukan: **customer sering tidak ada di halaman.** Untuk VA dan e-wallet ia pergi ke
ATM atau aplikasi bank, dan koneksi SSE tidak bertahan melewati backgrounding — keterbatasan
yang sudah tercatat untuk antrian. Yang dibutuhkan saat ia kembali adalah status **pada saat
itu**, yaitu satu permintaan. SSE tidak memberi apa pun di atas itu, sehingga keduanya tetap
harus diimplementasikan.

Biayanya pun nyata: status pembayaran adalah pengiriman **tertarget**, bukan siaran, sehingga
menuntut registry koneksi kedua — persis yang membuat WebSocket ditolak di Bagian 2 ADR-005.
Menumpangkannya pada stream antrian lebih buruk lagi: tier itu kini hanya menyiarkan satu
counter, dan ia adalah tier yang diskalakan ke 300.000 koneksi, tempat kesederhanaan paling
berharga.

> Status pembayaran tidak lewat SSE: ia tiba dari webhook gateway, bukan dari antrian, dan
> hanya relevan bagi satu customer.

---

## Appendix C: Perubahan v1.1 → v2.0

Penyelarasan dengan ADR-005. Yang memutus kompatibilitas ditandai **BREAKING**.

| # | Perubahan | Alasan |
|---|---|---|
| 1 | **BREAKING** `POST /queue/claim-token` dihapus | Bisa menjawab `429 Slot penuh` setelah memberi tahu customer gilirannya tiba — pelanggaran NFR-02. Grant kini diterbitkan hanya setelah slot aman |
| 2 | **BREAKING** `GET /queue/stream` → `GET /events/{event_id}/queue/stream` | Konsisten dengan `POST /events/{event_id}/queue/join` |
| 3 | **BREAKING** Payload SSE: `{now_serving, throughput, server_time}` → `{nowServing, state, ratePerMin, paused, serverTime}` | `paused` dan `state` tidak ada sebelumnya; klien tidak bisa membedakan antrian berhenti dari antrian lambat |
| 4 | **BREAKING** Event `your_turn` → `grant` (payload kosong) | Token tidak dikirim lewat SSE; klien mengambilnya dari `…/queue/me` |
| 5 | **BREAKING** Event `sold_out` → `sold-out`, klien wajib `close()` | Tanpa `close()` seluruh crowd reconnect setelah sold out |
| 6 | **BREAKING** `heartbeat` 30 detik → komentar `: keepalive` 15 detik | 30 detik tanpa margin terhadap idle timeout; event JSON ~6× lebih besar dan memicu handler |
| 7 | **BREAKING** `nik_name` dihapus dari `POST /bookings` | Dukcapil memvalidasi NIK saja; nama tidak dikumpulkan maupun dikirim (FR-15a) |
| 8 | **BREAKING** `DUKCAPIL_ERROR` fallback-ke-format → `DUKCAPIL_UNAVAILABLE` memblokir | Fail closed: tidak ada tiket terbit dengan NIK yang belum terverifikasi |
| 9 | Ditambahkan `GET /events/{event_id}/queue/me` | Re-entrancy BR-09 dan pengambilan token setelah `grant` |
| 10 | Ditambahkan `GET /events/{event_id}/queue/serving` | Jalur fallback polling yang sebelumnya tidak terdokumentasi |
| 11 | Ditambahkan `POST /payments/{booking_id}/retry` | Tanpanya customer yang charge-nya ditolak terkunci: gateway memutar ulang penolakan yang tersimpan |
| 12 | **Tidak ada** endpoint baru untuk BR-10 dan FR-33–35 | BR-10 dipenuhi oleh `server_time` yang sudah ada di tiap respons — endpoint waktu tersendiri akan redundan. FR-33–35 diterbitkan sebagai CloudWatch EMF, bukan lewat API |
| 12a | TTL pembayaran ditegaskan berjalan dari `POST /bookings` | BR-05 "sejak booking dibuat" = saat tombol bayar ditekan dan stok di-hold, sesuai wireframe tender. Kontrak sudah benar (`ttl_seconds: 600`); ADR-005 yang disesuaikan |
| 12b | §5.1 dan §5.2 ditulis ulang untuk tiga metode Midtrans Core API | VA dan "Transfer Bank" adalah satu metode, bukan dua; kartu absen dari daftar lama; tiap metode butuh parameter berbeda. `custom_expiry` ditambahkan karena default VA Midtrans 24 jam melawan TTL 10 menit |
| 12c | Akun dengan NIK_Terdaftar, Daftar NIK Teman, consent record, grace pembayaran 30 detik, kapasitas antrian | Penambahan fitur yang dikonfirmasi 26 Oktober. Pendorong utama Daftar NIK Teman adalah biaya Dukcapil per panggilan: NIK yang sudah diverifikasi dipakai ulang tanpa panggilan baru |
| 13 | Webhook: switch per tipe event + `gateway_event_id UNIQUE` | Sebelumnya hanya menangani `PAID`; "idempoten" dinyatakan tanpa mekanisme |
| 14 | Webhook: `PAID` atas booking `EXPIRED` → `payment.refund` | Uang diterima setelah stok dijual ulang sebelumnya tidak punya jalur |
| 15 | Snapshot awal pada stream | FR-10 sebelumnya ditandai terpenuhi tanpa mekanisme |
| 16 | Matriks coverage dikoreksi | FR-10, FR-19 dan FR-25 sebelumnya ditandai ✅ tanpa dasar |

### v2.0 → v2.1

| # | Perubahan | Alasan |
|---|---|---|
| 17 | `status` dan `queue_state` dipisah bersih; `COMING_SOON` dan `CLOSED` dihapus | Keduanya mencampur sumbu: `COMING_SOON` adalah siklus hidup sale, `CLOSED` melaporkan stok. Pada puncak sale `AVAILABLE` + `FULL` adalah keadaan normal yang satu enum tidak dapat menyatakan |
| 18 | `status` menambah `SELLING_OUT_FAST`; `queue_state` menambah `NOT_OPEN` dan `DIRECT` | `DIRECT` menutup FR-05 yang sebelumnya tersembunyi di dalam `OPEN`; `NOT_OPEN` menggantikan `COMING_SOON` bersama `sale_starts_at` |
| 19 | Ditambahkan blok `display` beserta aturan resolusi server-side | Presedensi lima cabang sebelumnya menjadi logika frontend. Label sebagai kunci menjaga cache CDN tetap satu varian meski FR-32 menuntut dua bahasa |
| 20 | `has_queue` dihapus; `queue_length` yang tertinggal diperbaiki menjadi `queue_depth` | `has_queue` menjadi mubazir setelah `DIRECT` ada. Kedua objek contoh pada respons yang sama sebelumnya memakai nama field berbeda |
| 21 | SSE `state`: `OPEN \| closed_sold_out` → `SCHEDULED \| OPEN \| SOLD_OUT` | Casing tidak konsisten dalam satu enum, dan tidak ada nilai awal — sale tidak punya permulaan yang terdefinisi |
| 22 | `booking_expires_at` dihitung dari `granted_at`, bukan `slot_claimed_at` | Grace antrian dihapus; tanpa anchor baru tidak ada apa pun yang merebut kembali slot yang di-grant tapi tidak pernah diklaim |
| 23 | `POST /bookings` menyatakan multi-kategori all-or-nothing + 4 kode error baru | `hold_booking.lua` sebelumnya hanya menerima satu kategori, padahal `tickets[]` sudah mengizinkan beberapa |
| 24 | **BREAKING** `POST …/queue/join` mewajibkan `captcha_token`; `CAPTCHA_REQUIRED` / `CAPTCHA_FAILED` ditambahkan | Vektor bot ada di `/join` (memborong posisi), bukan di Detail Ticket Page. Token diterbitkan pra-gerbang karena tantangan interaktif di titik `INCR issued` akan memberi bot nomor antrian lebih kecil daripada manusia |
| 25 | `GET /events/{id}` menambah `captcha.site_key` | Agar klien dapat memuat widget dan menyegarkan token selama `queue_state: NOT_OPEN`, sebelum gerbang buka |
| 26 | Response "Rejoined" menambah `state` dan kini juga melayani BR-07 | Akun yang sudah punya keterlibatan aktif memulihkannya alih-alih ditolak 409 — menolak akan menghukum perpindahan perangkat, sementara BR-07 tetap tegak |
| 27 | `credit_card` dihapus dari §5.1/§5.2; `requires_tokenization`, `client_key`, `token_id`, `CARD_TOKEN_INVALID` ikut hilang | `custom_expiry` tidak berlaku untuk kartu, sehingga kartu adalah satu-satunya metode yang tidak dapat menghormati TTL 10 menit di sisi gateway. Keluarnya kartu juga mengeluarkan cakupan PCI-DSS dari POC |
| 28 | SSE `state` menambah `CLOSED`; `SOLD_OUT` kini **reversibel** dan **tidak** menutup stream | Stok kembali dari booking yang kedaluwarsa, dan sold-out satu arah membuat kursi itu tidak pernah terjual. Pemutusan dipindahkan ke `CLOSED`, sekali, setelah 930 detik |
| 29 | `label_key` menambah `sold_out_may_return` | Customer perlu tahu bedanya "habis, mungkin kembali" dari "sale ditutup" — halaman utama di-refresh manual, jadi perbedaan itu menentukan apakah layak dicoba lagi |
| 30 | `GET …/queue/me` wajib dibaca ulang saat `paused` berubah | Server menambah `PAUSE_TTL_GRACE_SECONDS` ke tenggat yang sedang berjalan; klien yang menghitung turun dari nilai lama akan meleset 60 detik |
| 31 | `GET …/queue/me` menerima token sesi akun selain cookie | Pemulihan lintas perangkat sebelumnya hanya mungkin lewat `POST …/queue/join`, yang membawa CAPTCHA dan dua rate limit — memperlakukan customer yang kembali sebagai serangan |
| 32 | Cookie `Max-Age` diturunkan dari `SALE_WINDOW_SECONDS`, bukan 7200 | Cookie dan `token:{uuid}` yang dirujuknya harus kedaluwarsa bersamaan; 7200 adalah angka yang tidak terikat apa pun |
| 33 | Event `status` menambah `reason` (`redis_disconnected` \| `no_admission_updates`) dan pemicu kedua: keheningan `sse-broadcast` > 30 detik | `status` sebelumnya hanya menyala saat sse-server kehilangan Redis. Bila `admission-ctl` yang mati, Redis dan sse-server dua-duanya sehat — tidak ada sinyal apa pun sementara antrian membeku di balik koneksi yang tampak hidup dan keepalive yang tetap datang. Deteksinya di server, bukan timer di klien, sehingga nol kode klien bertambah |

### v2.1 → v2.2

| # | Perubahan | Alasan |
|---|---|---|
| 28 | Ditambahkan `POST /auth/refresh` (§1.4) | `expires_in` sudah ada di §1.3 sejak v2.0 tanpa cara memperpanjangnya, sehingga sesi berakhir diam-diam dan customer harus meminta magic link baru. Endpoint ini melengkapi angka yang sudah dinyatakan kontrak, bukan menambah fitur |
| 34 | `expires_in` **300 → 1080** detik (§1.3) | Nilai 300 tidak diturunkan dari apa pun dan kebetulan sama dengan `BOOKING_TTL_SECONDS` — tampaknya tersalin dari sana. ADR-005 §7 mewajibkan umur token diturunkan dari perjalanan, bukan ditulis tetap, dan sudah melakukannya untuk `GRANT_TOKEN_TTL_SECONDS`: 300 booking + 600 pembayaran + 30 grace + margin = 1080. Pada 300 detik customer me-refresh dua sampai tiga kali di tengah journey, termasuk di halaman pembayaran; refresh yang gagal di sana memutusnya — kegagalan "jangan PHP" yang NFR-02 ada untuk mencegah. Pada 300 detik juga, 300.000 concurrent menghasilkan sampai ~1.000 rps refresh tepat saat flash crowd |
| 35 | §1.3 dan §9 menegaskan umur access token **adalah** jendela pencabutannya | Access token adalah JWT stateless: tidak ada store yang dibaca saat verifikasi, sehingga tidak ada yang dapat menariknya lebih awal. Konsekuensinya perlu tertulis agar siapa pun yang kelak ingin memperpanjangnya tahu apa yang ia perdagangkan. Bagian sesi yang berumur panjang dengan sengaja adalah refresh token — tersimpan, ter-hash, dapat dicabut, dan berotasi |
| 29 | §1.3 ditegaskan: `refresh_token` **opaque, bukan JWT** | Contoh lama menampilkan `eyJhbG...` untuk kedua token, yang menyiratkan refresh token juga JWT. Ia disimpan server sebagai hash agar membaca basis data tidak cukup untuk menerbitkan sesi — sifat yang tidak dimiliki JWT yang dapat diverifikasi sendiri |
| 30 | Rotasi + pencabutan keluarga didokumentasikan; `TOKEN_INVALID` dan `TOKEN_EXPIRED` dipertegas | Token sekali-pakai yang muncul dua kali berarti kredensial bocor, dan yang memegang mata rantai terbaru bisa jadi penyerang. Mencabut satu token saja justru mempertahankan akses penyerang (OAuth 2.0 BCP §4.13). Satu kode untuk replay dan token tidak sah agar penyerang tidak diberi tahu bahwa ia memicu deteksi |
| 31 | `redirect_context` ditegaskan diturunkan dari keterlibatan aktif (BR-07), dan **dihilangkan** bila tidak ada | Field-nya ada di contoh sejak v2.0 tanpa menyatakan dari mana isinya berasal atau apa yang terjadi saat tidak ada apa pun yang sedang berjalan. Dihilangkan, bukan dikirim kosong, agar pemeriksaan frontend sederhana |
| 32 | §1.4 Register → §1.5, §1.5 Daftar NIK Teman → §1.6 | Refresh adalah kelanjutan siklus hidup sesi dan dibaca tepat setelah `GET /auth/verify` yang menerbitkan tokennya. Dua rujukan silang internal ikut disesuaikan |
| 33 | §9 menambah baris signing secret | Signing secret access token dapat dirotasi; pepper HMAC NIK tidak pernah (ADR-005 §8). Keduanya pernah tertukar dalam pembahasan, dan akibat menukarnya tidak simetris |

> **Mengapa bukan 24 jam, seperti kebanyakan aplikasi.** *(Ditulis untuk v2.2,
> ketika angkanya 1080. v2.4 menaikkannya ke 22740 — alasannya di §1.3 dan di
> butir 41 di bawah; prinsipnya tidak berubah, hanya intervalnya yang ternyata
> salah.)* Yang berumur sehari di aplikasi lain umumnya adalah *refresh* token
> atau session cookie sisi server — keduanya dapat dicabut. Di sini bagian itu
> sudah berumur 30 hari, sehingga dari sisi customer sesi memang sudah panjang;
> menaikkan *access* token tidak memperbaiki pengalaman, hanya memperlebar
> jendela token yang dicuri. Itulah mengapa v2.4 menaikkannya **hanya sejauh
> antrian memaksa** — 6 jam 19 menit diturunkan dari `SALE_WINDOW_SECONDS` —
> bukan ke angka bulat 24 jam yang tidak diturunkan dari apa pun.
>
> Bila kelak diinginkan 24 jam, pasangannya adalah **mengeluarkan `nik_verified`
> dari claim** sehingga token menjadi identitas murni dan otorisasi selalu
> diperiksa ke basis data. Claim itu ada justru untuk menghindari satu pembacaan
> basis data per permintaan di bawah beban flash crowd, jadi pertukarannya nyata —
> tetapi tanpa itu, otorisasi yang ter-cache sehari tidak dapat dibenarkan.

### v2.2 → v2.3

| # | Perubahan | Alasan |
|---|---|---|
| 36 | **BREAKING** Grant slot pindah dari `Authorization: Bearer` ke header `X-Grant-Token` (§4, §5) | `Authorization: Bearer` memikul tiga kredensial sekaligus: JWT sesi akun (§1.3, §6.1), grant slot (§4, §5) dan token ruang tunggu untuk mobile (§3.2). Tidak satu pun endpoint menyatakan mana yang ia terima, dan dua di antaranya tidak bisa sama: §9 mengikat Bearer ke `queue_token`, sementara `GET /me/tickets` hanya mungkin berarti JWT akun. Sejak v2.2 keduanya juga berumur sama (1080 detik), sehingga satu-satunya pembeda runtime yang tersisa pun hilang |
| 37 | **BREAKING** Token ruang tunggu mobile pindah ke header `X-Queue-Token` (§3.2, §3.3) | Konsekuensi dari #36: bila `Authorization` hanya untuk sesi akun, token ruang tunggu butuh slotnya sendiri. Web tetap memakai cookie karena `EventSource` tidak dapat menyetel header request — itu batasan transport, bukan pilihan desain, dan sekarang tertulis sebagai batasan |
| 38 | `POST /bookings` dan seluruh §4/§5 menerima `Authorization` **opsional** di samping grant | Ini yang memperbaiki cacat fungsional, bukan hanya penamaan. Dengan satu slot, grant mengisinya dan tidak ada tempat untuk JWT akun — sehingga booking customer login tidak dapat diatribusikan ke akunnya saat dibuat, padahal §6.1 menyelesaikan kepemilikan lewat `account_id`. Grant wajib karena tamu boleh membeli dan tidak punya sesi; sesi opsional karena hanya customer login punya satu |
| 39 | `access_token` di respons §3.3 → `grant_token`, contoh `at_abc123` → `gt_abc123` | Field dan nilainya menyandang nama kredensial lain. Grant bukan JWT, tidak dapat diverifikasi sendiri, dan dapat dicabut — tiga sifat yang berlawanan dengan access token, di balik satu nama yang sama |
| 40 | §9 menambah baris Grant Token dan "Satu slot, satu kredensial"; baris Identity Token dilipat ke Access Token | §9 sebelumnya mendefinisikan "Access Token" sebagai JWT saja, lalu dipakai di §4 untuk grant — pembaca mengambil definisi itu dan salah. "Identity Token" adalah nama ketiga untuk JWT yang sama dan sudah dihapus. Baris baru menyatakan larangan yang membuat klausa kepemilikan berarti: middleware memilih kredensial per rute, tidak mencoba keduanya |

> **Yang tidak berubah.** Nama field `access_token` di §1.3 dan §1.4 tetap, karena di sana ia
> memang access token akun. Umur, rotasi dan semantik refresh token tidak tersentuh. Klausa
> kepemilikan §9 secara fungsional sama — hanya kini nama kolom dan nama variabelnya merujuk
> kredensial yang sama.

> **Biaya menunda.** `queue-service` dan `booking-service` belum ditulis, dan `user-service`
> hanya pernah menerbitkan JWT akun; tidak ada kode terimplementasi yang membaca slot Bearer
> sebagai grant. Hari ini perubahan ini hanya menyentuh dokumen. Begitu klien mobile atau
> booking rilis, ia menjadi perubahan yang memutus kompatibilitas di tiga permukaan.

> **CORS.** `X-Grant-Token` dan `X-Queue-Token` keduanya memicu preflight — tetapi
> `Authorization` sudah memicunya di setiap panggilan §4/§5, jadi tidak ada round trip baru.
> Yang perlu ditambahkan hanya keduanya ke `Access-Control-Allow-Headers`.

---

### v2.4 → v2.5

| # | Perubahan | Alasan |
|---|---|---|
| 49 | **BREAKING** §5.5: header `X-Callback-Signature` **dihapus**; tanda tangan dibaca dari field `signature_key` **di body** | Midtrans tidak mengirim header tanda tangan apa pun pada HTTP notification Core API. Contoh v2.4 juga menulis `sha256=` sementara formula lima baris di bawahnya SHA512. Implementasi yang mengikuti v2.4 secara literal menuntut header yang tidak pernah datang dan **menolak setiap notifikasi yang sah** — endpoint pembayaran yang tidak pernah mengonfirmasi satu pun booking |
| 50 | **BREAKING** §5.5: contoh request diganti ke bentuk Core API yang sebenarnya | v2.4 memuat `event_id`, `external_id`, `status: "PAID"`, `paid_amount` — tidak satu pun dikirim Midtrans, dan semuanya bertentangan dengan tabel transisi di §5.5 itu sendiri, yang memakai `transaction_status` dan `fraud_status`. Ditambahkan dua hal yang mudah salah: `gross_amount` adalah **string berdesimal** yang masuk formula tanda tangan apa adanya, dan `transaction_time` adalah WIB tanpa penanda zona |
| 51 | **BREAKING** §5.5: `gateway_event_id` **diturunkan** dari `order_id + ':' + transaction_status + ':' + status_code` | v2.4 memetakannya ke `event_id`, yang tidak ada. Kolomnya akan selalu NULL dan `UNIQUE` tidak menjaga apa pun. `transaction_id` sendirian juga tidak cukup: ia stabil sepanjang hidup satu transaksi, jadi `pending` dan `settlement` membawa nilai identik dan yang kedua akan ditelan — persis kesalahan yang dihindari dengan tidak memakai `booking_id`. Ketiga komponen yang dipilih sudah dipercaya di tempat lain: dua di antaranya masuk formula tanda tangan |
| 52 | §7: `server_time` menjadi bagian **envelope** `meta`, pada setiap respons | BR-10 menuntut server sebagai sumber waktu, dan ADR-005 Bagian 6 menyatakan setiap respons REST membawanya supaya tidak ada `GET /time`. v2.4 hanya memuatnya di dalam `data` pada tujuh endpoint, sehingga klien yang memanggil endpoint lain tidak punya cara menghitung offset jamnya. Diisi di layer respons bersama, bukan oleh handler |
| 53 | §4.3: `holder_name` dan error "nama tidak cocok" **dihapus** | Keduanya melawan §4.2 dan FR-15a di dokumen yang sama: Dukcapil memvalidasi NIK saja, form tidak mengumpulkan nama, dan §4.2 tidak pernah mengembalikannya. Sisa versi sebelum FR-15a dipersempit |
| 54 | §5.4: contoh `reason` tidak lagi menyebut kartu | §5.1 mengeluarkan kartu dari lingkup POC karena `custom_expiry` tidak berlaku padanya. Contoh "Kartu ditolak penerbit" menjelaskan kegagalan dari metode yang tidak dibuka. Ditambahkan juga bahwa `reason` adalah passthrough `status_message` untuk dukungan, bukan teks yang dirender |
| 55 | §1.7 `GET /auth/verify-email` didokumentasikan | Endpoint ini sudah berjalan di `user-service` tetapi tidak pernah masuk kontrak — tidak di markdown, tidak di OpenAPI, tidak di koleksi Apidog. Diberi nomor di akhir §1 agar §1.4–§1.6 tidak bergeser, karena ketiganya dirujuk lintas dokumen |
| 56 | §3.1: tambah `409` (state `SCHEDULED`) dan `400 DEVICE_ID_REQUIRED` | Keduanya ada di Appendix A dan di ADR-005 tetapi tidak terdaftar sebagai respons endpoint. `409` bukan kasus tepi: ia keadaan **awal** sale, dan layar pra-buka adalah layar terpadat yang dilihat 300.000 orang |
| 57 | §3.2: tambah `429` (`rl:sse:{ip}`) dan `409`; `410` dinyatakan hanya untuk `CLOSED` | ADR-005 Bagian 4 menaruh `rl:sse:{ip}` di gerbang ALB sebagai pengganti WAF, tetapi kontrak tidak pernah mendaftarkannya sehingga tidak ada yang menegakkannya. Dan v2.4 menulis 410 sebagai "event sudah sold out", yang membalik keputusan `SOLD_OUT` reversibel: 410 hanya untuk `CLOSED` |
| 58 | §4.1 dinyatakan **mengubah state**: `GRANTED → SELECTING` + `Cache-Control: no-store` | ADR-005 Bagian 6 menugaskan transisi claim ke `booking-service` tanpa menyebut endpoint mana yang menjalankannya, sehingga `SELECTING` tidak dapat dicapai dan §4.3 — yang menuntut status `SELECTING` — tidak pernah dapat berhasil. Sebuah `GET` yang punya efek samping harus dikatakan, atau seseorang akan men-cache-nya |

> **Yang tidak berubah.** Tidak ada perubahan pada kredensial, TTL, alur antrian, alur
> booking, atau aturan pembayaran. Butir 49–51 memperbaiki kontrak terhadap API pihak ketiga;
> butir 52–58 menutup hal-hal yang ADR-005 sudah putuskan tetapi kontrak tidak pernah
> menuliskan.

> **Dampak ke artefak turunan.** `rusuh-booking-openapi.yaml` dan `rusuh-booking-apidog.json`
> keduanya diekspor dari keadaan yang mendahului beberapa keputusan v2.4 dan **tidak** dapat
> dianggap sinkron: CAPTCHA (`captcha_token` pada §3.1, `captcha.site_key` pada §2.2) tidak
> ada di satu pun dari keduanya, dan koleksi Apidog juga kehilangan `rejoined`,
> `already_used`, `PENDING_CANCELLATION` dan field `source` pada §4.2. Markdown ini adalah
> sumber kebenaran; keduanya diregenerasi darinya, bukan diedit sendiri-sendiri.

---

### v2.3 → v2.4

| # | Perubahan | Alasan |
|---|---|---|
| 41 | `expires_in` **1080 → 22740** detik (§1.3) | 1080 diturunkan dari perjalanan — benar untuk grant, kebetulan untuk sesi. Sesi akun dimulai saat **login**, sebelum antrian, jadi ia harus mencakup penantian juga: `SALE_WINDOW_SECONDS + 990 + 150`. Pada `ratePerMin` 318 dan antrian 300.000, posisi terakhir menanti ~15,7 jam, jadi sesi 18 menit mati di ruang tunggu untuk hampir seluruh antrian. Yang rusak bukan ruang tunggunya — §3.3 memulihkan lewat cookie — tapi §1.6 pada Detail Ticket Page: Daftar NIK Teman 401 tepat saat dibuka, memaksa NIK manual, memaksa Dukcapil, dan Dukcapil *fail closed* |
| 42 | **BREAKING** `Authorization` dihapus dari §4 dan §5 | v2.3 menerimanya sebagai opsional untuk mengisi `bookings.account_id`. Itu gagal secara diam-diam: sesi yang mati di ruang tunggu berarti klien tidak mengirim apa pun, baris tertulis `account_id = NULL`, dan pembelian customer login tercatat sebagai pembelian tamu — tanpa error, tanpa 401, tanpa jejak. v2.4 mengambilnya dari `token:{uuid}`, yang mengetahuinya sejak join ketika sesi masih pasti segar, dan menuliskannya pada INSERT saat GRANT. Atribusi berhenti bergantung pada jam klien, dan header itu tidak punya pekerjaan lagi |
| 43 | Kredensial antrian dihapus pada keadaan terminal; `CONFIRMED` meninggalkan **nisan** (§3.3) | `Max-Age` dan `EX SALE_WINDOW_SECONDS` berarti kredensial tetap hidup berjam-jam setelah berhenti berarti apa pun. Penghapusan **tidak** dilakukan pada grant (§3.3 memulihkan keadaan setelah grant — "kembali dari aplikasi bank"). `LAPSED` sempat dikecualikan juga, karena token itu menunggu grant keduanya; butir 47 menghapus kesempatan kedua itu, jadi `LAPSED` ikut dihapus bersama `EXPIRED` dan `ABANDONED`. Pada `CONFIRMED` ia dihapus tetapi diganti `settled:{uuid}` yang memuat `booking_id` saja: tanpa itu, tamu yang me-refresh sebelum email tiba melihat **layar kedaluwarsa** setelah membayar — "jangan PHP" dalam bentuk paling murni, dan salah secara faktual |
| 44 | §6.2 menerima `?t={eticket_token}` selain sesi akun | FR-26 mengirim tiket ke **masing-masing pemegang NIK**, dan pemegang kedua dan ketiga umumnya bukan pemilik akun. Rute yang hanya menerima sesi akun berarti tombol "Buka e-tiket" di email mereka menjawab 401 — jalur pengiriman utama requirement itu tidak berfungsi untuk orang-orang yang requirement itu ditulis untuk mereka. Berlaku juga bagi pembeli tamu, yang setelah butir 43 hanya memiliki email |
| 45 | Pengikatan device untuk grant anonim **didefinisikan** (§3.1, §4, §9) | ADR-005 Bagian 12 mengklaim grant anonim diikat ke "device fingerprint", tanpa header, tanpa algoritma, tanpa perilaku saat fingerprint berubah — tidak dapat diimplementasikan, padahal ia satu-satunya yang berdiri antara customer dan menjual slot antriannya. Dipilih `X-Device-Id`: acak 128 bit yang dibuat klien, disimpan sebagai HMAC, satu rebind dicatat. **Bukan** fingerprint pasif — canvas/WebGL rapuh dan memanen karakteristik perangkat tanpa dasar persetujuan (NFR-11, UU PDP) |
| 46 | Appendix A menambah `DEVICE_ID_REQUIRED`, `DEVICE_MISMATCH`, `ETICKET_TOKEN_INVALID` | Tiga kegagalan baru yang masing-masing dirutekan berbeda oleh frontend. `DEVICE_MISMATCH` sengaja menandai percobaan **ketiga**, bukan sekadar ganti browser |

| 47 | **`LAPSED` bersifat final — kesempatan kedua (`readmit`) dihapus seluruhnya** | Tiga alasan. **Keadilan FCFS**: pengurasan `readmit` mendahului antrian dan *mengurangi* `deficit` yang tersisa untuknya, sehingga customer yang sudah mendapat giliran dan membakar 300 detiknya dilayani **mendahului** orang di posisi 150.000 yang belum pernah mendapat giliran. FCFS memberi satu giliran, bukan giliran tanpa batas. **Biaya slot**: satu no-show menahan slot hingga 600 detik tanpa pernah memilih tiket, pada 300.000 orang untuk 10.000 tiket. **Tidak dapat dibedakan**: push yang tertahan OS dan customer yang pergi menghasilkan `LAPSED` identik, jadi membayari yang pertama ikut memberi hadiah pada yang kedua. Menjaga perangkat menyala selama perang tiket ada di sisi customer |
| 48 | `LAPSED` kini melepas **semua** kredensialnya: `grant:{token}`, `acct:{ev}:{account_id}`, `token:{uuid}` | Konsekuensi langsung butir 47, dan satu perbaikan yang terpisah: `grant:{token}` punya `EX 1080` sementara TTL booking 300 detik, sehingga sebelumnya grant yang sudah mati tetap lolos verifikasi ~780 detik lagi. Bukan lubang akses — klausa kepemilikan menemukan booking `LAPSED` dan validasi status menolaknya — tetapi penolakannya datang dari validasi status, bukan dari kredensialnya, dan satu-satunya kredensial yang dapat dicabut seketika justru tidak dicabut. Butir 47 juga membalikkan pemecahan tabel BR-07 yang sempat diperlukan: tanpa kesempatan kedua, tidak ada keadaan "LAPSED tapi masih menunggu" |

### v2.5 → v2.5.1 (koreksi, bukan perubahan perilaku)

| # | Perubahan | Alasan |
|---|---|---|
| 59 | §1.7: TTL token verifikasi **24 jam → 15 menit** | v2.5 mendokumentasikan 24 jam; implementasinya tidak pernah demikian. `Register` memakai `entity.MagicTokenTTL`, satu konstanta yang dipakai kedua `purpose`, dan nilainya 15 menit. Kontraknya yang salah, bukan kodenya — jadi ini koreksi dokumen, dan tidak ada kode yang berubah |
| 60 | §1.7: ditambahkan catatan bahwa §1.3 **juga** mengisi `email_verified_at` | Tanpa itu, TTL 15 menit terbaca sebagai jalan buntu permanen, dan tindak lanjut yang wajar adalah membangun `POST /auth/verify-email/resend`. Itu akan menduplikasi apa yang sudah dilakukan login: menebus tautan login membuktikan kendali atas alamat, dan kode sudah menandai verifikasi di jalur itu. Dua penulis untuk satu kolom dengan prasyarat berbeda adalah yang kelak menyimpang |

> **Mengapa 2.5.1 dan bukan 2.6.** Tidak ada perilaku yang berubah dan tidak ada kode yang
> disentuh; dua pernyataan tentang perilaku yang sudah berjalan diperbaiki. Menaikkan minor
> akan menyiratkan ada sesuatu yang perlu diintegrasikan ulang.

> **Yang tidak berubah.** `GRANT_TOKEN_TTL_SECONDS` tetap 1080 — perubahan 41 memisahkan dua
> TTL yang kebetulan sama, bukan menaikkan keduanya. Rotasi dan umur refresh token tidak
> tersentuh. Klausa kepemilikan §9 secara fungsional sama.

> **Konsekuensi yang perlu diputuskan, bukan diam-diam diterima.** Pada 22740 detik, claim
> `nik_verified` di dalam access token adalah **otorisasi** yang ter-cache enam jam — dan
> access token adalah JWT stateless, jadi umurnya *adalah* jendela pencabutannya. Pasangan
> yang semestinya menyertai butir 41 adalah mengeluarkan claim itu dan memeriksanya ke basis
> data, dengan biaya satu pembacaan per permintaan di bawah flash crowd. Terbuka sebagai
> ADR-005 Bagian 17 #11. Alternatif yang lebih aman — mempertahankan 1080 dan me-refresh di
> **ruang tunggu**, di mana refresh tidak berbahaya — ditolak karena menuntut penjadwalan di
> klien yang, bila dilewatkan, gagal secara diam-diam.

### v2.6 → v2.7

| # | Perubahan | Alasan |
|---|---|---|
| 61 | **BREAKING** Pause admission dihapus: `paused` hilang dari payload SSE dan §3.4; aturan "baca ulang §3.3 saat `paused` berubah" dihapus; `ADMISSION_PAUSED` → `PAYMENT_GATEWAY_UNAVAILABLE` | ADR-006 §4.3: BR-13a dan BR-22 dihapus. Breaker tetap fail closed tetapi tidak menghentikan antrian, dan tenggat tidak pernah digeser |
| 62 | **BREAKING** Event SSE `status` kini berarti **state sale berubah** (`{state, soldOutAt, serverTime}`), dikirim tiap transisi; indikator basi lama menjadi event `stale` | Klien mendapat sinyal eksplisit saat sold out, tanpa membandingkan payload sendiri |
| 63 | §3.1: respons join menambah `queue_token` | Klien mobile tidak punya cookie store; tanpa ini mereka tidak dapat mengirim `X-Queue-Token` |
| 64 | §3.1: `estimated_wait_minutes` dihapus; §3.5 baru: rumus estimasi untuk frontend | Satu sumber laju (`ratePerMin`), satu rumus tertulis untuk web dan mobile (ADR-006 §6.2) |
| 65 | §3.4: `paused` diganti `state` | Mode polling perlu mendeteksi `SOLD_OUT` / `CLOSED` |
| 66 | Sesi akun: `journey + pause` 990 → `journey` 930, margin 150 → 210 | `expires_in` tetap 22740; hanya rinciannya berubah |
| 67 | Envelope: `meta.error_code` pada setiap error; `data` selalu objek (`{}` bila kosong, tidak `null`) | Appendix A mendaftar kode tanpa tempat membawanya; Apidog sudah memakai `error_code` di sebagian respons. Satu aturan untuk semua |
| 68 | Appendix A: `UNAUTHORIZED`, `NOT_FOUND`, `SALE_NOT_OPEN`, `SALE_CLOSED`, `BOOKING_STATUS_INVALID` | Respons 401/404/409/410 sebelumnya tanpa kode |
| 69 | §3.1: contoh respons antrian kosong (`granted: true`); §3.2: respons 401 | Sudah ada di perilaku (FR-05, auth stream), belum di dokumen |
| 70 | OpenAPI YAML diarsipkan; Apidog = sumber kebenaran bentuk respons | Dua artefak mesin yang harus sinkron terlalu mahal dipelihara |
| 71 | `GET /bookings/{booking_id}/tickets` **dihapus**; §4.4 `tickets[].eticket_url` membawa `?t={eticket_token}` saat `PAID` | Isi endpoint itu sama dengan `tickets[]` di §4.4. Yang tidak dimiliki §4.4 hanyalah akses QR tanpa email bagi tamu, dan itu kini diberikan tautan bertoken (ADR-006 §8.8) |

---

> **Catatan untuk ADR-005.** Daftar tipe outbox di §9 langkah 4 memuat
> `account.verify` tetapi tidak `auth.magic_link`. Login tanpa kata sandi (§1.1)
> memakai kanal yang sama dan karenanya menerbitkan tipe itu. Layak ditambahkan ke
> tabel ADR agar pembaca berikutnya tidak membacanya sebagai improvisasi.

### v2.7 → v2.8

| # | Perubahan | Alasan |
|---|---|---|
| 68 | §3.1 + Appendix A: kode **`SOLD_OUT`** (409) untuk join pendatang baru saat `SOLD_OUT`; yang sudah mengantri tetap dipulihkan | Kursi yang kembali jatuh ke antrian yang sudah ada; nomor baru hampir tidak mungkin terlayani. Sama dengan perilaku "reject" ruang tunggu (Cloudflare Waiting Room) |
| 69 | Appendix A: **`QUEUE_NOT_READY`** (503) dan **`INTERNAL_ERROR`** (5xx) | Event yang belum di-seed bukan "belum dibuka"; envelope mewajibkan `error_code` pada setiap error, termasuk 5xx |
| 70 | §3.1: cookie `queue_token` menambah `Domain=rusuhtiket.com; Path=/` | Join di `api.*`, stream di `events.*`: cookie host-only tidak akan terkirim ke stream |
| 71 | §3.1: `sse_url` menjadi URL absolut `https://events.rusuhtiket.com/events/{event_id}/queue/stream` | Contoh lama `/queue/stream` bukan path §3.2 dan relatif terhadap host yang salah |
| 72 | §3.1: semua respons join didaftar (400 `VALIDATION_ERROR`, 401, 404, 410, 503 `QUEUE_NOT_READY` / `INTERNAL_ERROR`, `retry_after` + `Retry-After`) dan format `X-Device-Id` | Implementasi sudah menjawabnya; frontend perlu merutekan semuanya |
| 73 | §3.1: pemulihan berjalan **sebelum** `SOLD_OUT`/`QUEUE_FULL`; `queue_token` hanya memulihkan tempat milik pemanggil (login + token tamu → tempat sendiri) | Customer yang pindah perangkat tidak boleh ditolak karena antrian penuh; mengambil alih tempat tamu melewati BR-07 |
| 74 | §2.1: `GO_TO_TICKETS` dinyatakan tetap memanggil join | Grant, pengikatan device dan BR-07 hanya lahir dari join |

### v2.8 → v2.9

| # | Perubahan | Alasan |
|---|---|---|
| 75 | §3.1, §4, §9, Appendix A: `queue_token` hanya berlaku bagi pemiliknya, **tanpa rebind** (ADR-006 §8.9). Token tamu dari perangkat lain → `401 DEVICE_MISMATCH`; token akun dipakai akun lain / tamu → `401 UNAUTHORIZED`; `MAX_DEVICE_REBINDS` dihapus | Rebind menambah penghitung dan CAS di tiga service untuk kasus jarang. Konsekuensi diterima: tamu yang menghapus data situs mulai lagi dari belakang |
| 76 | §3.1: langkah klien pada kedua 401 itu — hapus token antrian (web: cookie dihapus server; mobile: hapus dari secure store), lalu join lagi tanpa token | Tanpa itu klien mengirim token yang sama selamanya dan selalu ditolak |
| 77 | §3.1: cookie `queue_token` `Path=/events/{event_id}` (dari `Path=/`) | Satu cookie per event; join event lain tidak lagi menimpa token event ini |
| 78 | §3.1: format `X-Device-Id` tidak lagi divalidasi (16–128 `[A-Za-z0-9_-]` dihapus); hanya wajib ada untuk tamu | Server hanya meng-HMAC nilainya; aturan format tidak menambah keamanan |
| 79 | Base URL tanpa `/v1` | Belum ada versi API. Cookie `Path=/events/{event_id}` hanya cocok bila join dilayani di `/events/...` tanpa prefiks |
| 80 | Domain: `my-rusuh.my.id` (API `api.*`, stream `events.*`, cookie `Domain=my-rusuh.my.id`) menggantikan `rusuhtiket.com` / `rusuh.id` pada jalur antrian | Domain yang dimiliki tim; cookie hanya terkirim bila API dan stream berada di bawah domain cookie yang sama |

### v2.9 → v2.10

| # | Perubahan | Alasan |
|---|---|---|
| 81 | **BREAKING** §3.1, §3.2, §3.4, Appendix A, Apidog: state antrian `SOLD_OUT` → `FULL_BOOKED`; error join 409 `SOLD_OUT` → `FULL_BOOKED` (`data.queue_state`, `data.full_booked_at`); SSE `status {state, fullBookedAt}`; `SOLD_OUT_HOLD_SECONDS` → `FULL_BOOKED_HOLD_SECONDS` | "Sold out" terdengar final, padahal state ini reversibel (kursi dari booking yang kedaluwarsa kembali). Event `sold-out` (final) dan state ini bertabrakan nama: klien membaca `SOLD_OUT` dan menutup stream |
| 82 | `CLOSED`, event SSE `sold-out` dan `SALE_CLOSED` (410) **dipertahankan** | `CLOSED` satu-satunya keadaan terminal; `sold-out` sekarang konsisten: hanya muncul saat sale benar-benar selesai |
| 83 | `status: SOLD_OUT` di §2.1 (halaman utama, FR-03c) **tidak diubah** | Sumbu stok yang terpisah dan dimiliki requirements; selaraskan dengan pemilik requirements (lihat catatan §2.1) |
| 84 | `SALE_CLOSED` (410) memakai `meta.message` "Tiket sudah habis" pada join dan stream; event `reconnect` mendapat skema `QueueStreamReconnect` (`data: {}`) | Pesan `CLOSED` harus menyatakan sold out; `data` wajib ada pada setiap frame Apidog, dan `{}` tidak cocok dengan empat skema lain |
| 85 | §3.2: respons `404 NOT_FOUND` dan `503 SERVICE_UNAVAILABLE` / `QUEUE_NOT_READY` didaftar (Apidog + kontrak) | Implementasi sse-server menjawabnya; klien perlu merutekan keduanya |

### v2.10 → v2.11

| # | Perubahan | Alasan |
|---|---|---|
| 86 | §3.1 FR-05: join menunggu grant bagi **setiap** pendatang yang nomornya muat di tempat kosong (`ticket_no ≤ now_serving + min(free_slots, JOIN_DIRECT_MAX)`), bukan hanya `now_serving + 1` (ADR-006 §8.11) | Batch pertama saat penjualan dibuka langsung ke Detail Ticket tanpa melihat ruang tunggu sekejap |
| 87 | §2.1: `granted: true` juga bisa menjadi jawaban untuk `JOIN_QUEUE`; klien merutekan berdasarkan respons | Halaman utama bisa masih `OPEN` saat ada tempat kosong (mis. tepat saat dibuka) |
| 88 | admission-ctl berhenti → join tidak menunggu (`free_slots` lebih tua dari 2 detik) | Tidak menahan setiap pendatang 1 detik saat antrian memang beku. Bentuk respons tidak berubah |

### v2.11 → v2.12

| # | Perubahan | Alasan |
|---|---|---|
| 89 | §3.1: **respons `granted: true` dihapus**; join tidak menunggu grant dan selalu menjawab `queued` atau `rejoined` (ADR-006 §8.14). Mencabut butir 86–88 | Tunggu ≤ 1 detik menahan Lambda per pendatang (hingga 500 sekaligus saat T0) dan menambah konfigurasi, field Redis dan cabang respons, demi menghemat sekitar satu tick. Satu jalur ke grant: stream + `/queue/me` |
| 90 | §2.1: `GO_TO_TICKETS` tetap memanggil join, tetapi klien menampilkan "Memproses…" sampai pesan stream pertama, bukan layar antrian | FR-05 dipenuhi oleh ruang tunggu yang berlangsung sekitar satu tick |
| 91 | Apendiks: tabel FR memberi FR-05 ⚠️ (menunggu persetujuan pemilik requirement) | Wording FR-05 mengasumsikan jalur langsung |
| 92 | §3.2: bentuk event stream seragam: `stale {stale, state, message?, serverTime}` (`reason` dihapus), `sold-out {state, message, serverTime}`, `reconnect {state, message, serverTime}`. Skema Apidog `QueueStreamStale`, `QueueStreamSoldOut`, `QueueStreamReconnect` diperbarui | Klien menangani setiap event dengan field yang sama (`state` selalu ada), dan teks tampil datang dari server. `message` bukan kode: klien tidak bercabang padanya |
