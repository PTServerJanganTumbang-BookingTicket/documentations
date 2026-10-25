# Rusuh Booking Tiket — API Contracts

**Versi**: 2.0  
**Tanggal**: 24 Oktober 2026  
**Status**: Draft  
**Base URL**: `https://api.rusuh.id/v1`  
**Selaras dengan**: ADR-005 (Keputusan Arsitektur Final — SSE + ECS Fargate)

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
6. [User](#6-user)
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
| `meta.message` | string | Human-readable message (single message) |
| `meta.pagination` | object | Pagination info (only on list endpoints) |
| `data` | object/array | Response payload |

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
  }
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
    "refresh_token": "eyJhbG...",
    "expires_in": 300,
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

**Requirement Coverage**: FR-31, BR-09 (re-entrant user kembali ke halaman yang sama)

---

### 1.4 Register Account

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

**Requirement Coverage**: FR-29, FR-30 (login opsional, NIK + email untuk pembelian)

---

## 2. Events

### 2.1 List Events

Menampilkan daftar event. UI: Home Page "Daftar Event".

**Endpoint**: `GET /events`

**Query Parameters**:
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `filter` | string | No | `all`, `available`, `upcoming` |
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
      "status": "AVAILABLE",
      "has_queue": true,
      "queue_length": 1248,
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
      "has_queue": false,
      "queue_length": 0,
      "min_price": null,
      "max_price": null
    }
  ]
}
```

**Status Values**:
| Status | UI Display | Action |
|--------|------------|--------|
| `AVAILABLE` | Badge "AVAILABLE", Button "Beli Tiket" aktif | Clickable |
| `SOLD_OUT` | Badge "SOLD OUT", Button "Tidak Tersedia" disabled | Non-clickable |
| `COMING_SOON` | Badge "Segera hadir", Button disabled | Non-clickable |

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
    "has_queue": true,
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

**Requirement Coverage**: FR-11, FR-12

---

## 3. Queue

### 3.1 Join Queue

User bergabung ke antrian. Dipanggil saat klik "Beli Tiket" pada event dengan antrian aktif.

**Endpoint**: `POST /events/{event_id}/queue/join`

**Request**: (empty body)

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
    "estimated_wait_minutes": 4,
    "sse_url": "/queue/stream",
    "event": {
      "id": "evt_konser_a",
      "name": "Konser A — Jakarta"
    }
  }
}
```

> Server sets cookie on join:
> ```
> Set-Cookie: queue_token=qt_xyz789; HttpOnly; Secure; SameSite=Strict; Max-Age=7200
> ```

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
    "estimated_wait_minutes": 4,
    "sse_url": "/queue/stream",
    "rejoined": true
  }
}
```

**Requirement Coverage**: FR-04, FR-10 (posisi konsisten saat refresh), NFR-05 (FCFS fairness)

---

### 3.2 Queue Status Stream (SSE)

Server-Sent Events stream untuk update posisi real-time. Client connect setelah join queue.

**Endpoint**: `GET /events/{event_id}/queue/stream`

**Cookies** (web): `queue_token=qt_xyz789`
**Headers** (mobile): `Authorization: Bearer qt_xyz789`

> Token **tidak pernah** dikirim lewat query string — ia akan tercatat di access log ALB,
> CloudWatch, dan history browser.

**Response** (200): `text/event-stream`

**Snapshot awal** — dikirim segera saat koneksi terbuka, sebelum broadcast berikutnya. Tanpa
ini posisi kosong sampai tick berikutnya, dan FR-10 tidak terpenuhi setelah refresh.

```
id: 31600
data: {"nowServing":31600,"state":"OPEN","ratePerMin":318,"paused":false,"serverTime":"2026-10-24T13:10:00.412Z"}
```

**Event default (tanpa nama)** — dikirim saat `nowServing` berubah, saat `paused` berubah,
atau tiap 10 detik sebagai heartbeat isi.

```
data: {"nowServing":31604,"state":"OPEN","ratePerMin":318,"paused":false,"serverTime":"2026-10-24T13:10:01.004Z"}
```

| Field | Tipe | Arti |
|---|---|---|
| `nowServing` | int | Nomor antrian global yang sedang dilayani |
| `state` | string | `OPEN` \| `closed_sold_out` |
| `ratePerMin` | int | Laju antrian untuk estimasi waktu tunggu (FR-09) |
| `paused` | bool | Admission dihentikan sementara (gangguan Dukcapil / payment gateway) |
| `serverTime` | string | Waktu server ISO-8601 — klien menyinkronkan jamnya (BR-10) |

> Frontend menghitung `position = ticket_no - nowServing`. Saat `paused = true`, tampilkan
> banner; jangan tampilkan error — antrian tidak hilang.

**Event: `grant`** — giliran customer tiba. Payload kosong; token **tidak** dikirim lewat SSE.

```
event: grant
data: {}
```

> Klien merespons dengan `GET /events/{event_id}/queue/me` untuk mengambil `access_token`.
> Grant hanya diterbitkan **setelah** slot diamankan dan baris booking tertulis, sehingga
> `/queue/me` tidak pernah menolak customer yang sudah menerima `grant` (NFR-02).

**Event: `status`** — indikator koneksi ke Redis, bukan status antrian.

```
event: status
data: {"stale": true}
```

**Event: `sold-out`** — tiket habis saat menunggu.

```
event: sold-out
data: {"message":"Tiket sudah habis"}
```

> **Klien WAJIB memanggil `eventSource.close()`** pada event ini. Tanpa itu `EventSource`
> reconnect otomatis dan seluruh crowd membanjiri ALB setelah sold out. Server menolak
> koneksi baru dengan `410 Gone` sebagai jaring pengaman.

**Event: `reconnect`** — task sedang shutdown; `retry:` berisi jeda berjitter 1–15 detik.

**Keepalive** — komentar SSE tiap **15 detik**, bukan event bernama:

```
: keepalive
```

> Komentar tidak memicu handler di klien dan berukuran 13 byte; sebuah event JSON tiap 30
> detik berukuran ~6× lebih besar dan tetap berisiko kena idle timeout ALB (default 60s,
> dikonfigurasi 3600s).

**Response** (410 - stream ditolak): event sudah sold out. `EventSource` berhenti permanen
pada status non-200 — ini disengaja.

**Requirement Coverage**: FR-07, FR-08, FR-09, FR-10, BR-10, NFR-06

---

### 3.3 Get Queue State (Re-entrancy)

Satu-satunya endpoint yang dipanggil klien saat kembali — setelah refresh, ganti tab, atau
kembali dari aplikasi bank. Mengembalikan **token yang sama** selama TTL belum habis.

**Endpoint**: `GET /events/{event_id}/queue/me`

**Cookies**: `queue_token=qt_xyz789`

**Response** (200): bentuk `data` mengikuti keadaan customer.

| Keadaan | `data` | Halaman tujuan |
|---|---|---|
| Masih mengantre | `{"state":"QUEUED","ticket_no":31842,"now_serving":31600}` | Waiting Room |
| Giliran tiba | `{"state":"GRANTED","access_token":"at_abc123","expires_at":"…"}` | Detail Ticket |
| Sedang memilih | `{"state":"SELECTING","access_token":"at_abc123","booking_id":"RB-00912","booking_expires_at":"…","server_time":"…"}` | Detail Ticket, countdown dilanjutkan |
| Menunggu bayar | `{"state":"AWAITING_PAYMENT","access_token":"at_abc123","booking_id":"RB-00912","payment_expires_at":"…","payment_ref":"pay_123","server_time":"…"}` | Payment, countdown dilanjutkan |
| Selesai | `{"state":"CONFIRMED","booking_id":"RB-00912"}` | Halaman tiket |
| Kedaluwarsa | `{"state":"EXPIRED"}` | Layar expired |

> `POST /queue/claim-token` dari v1.1 **dihapus**. Pada model itu klien meminta slot setelah
> `your_turn`, dan bisa menerima `429 Slot penuh` — memberi tahu customer bahwa gilirannya
> tiba lalu menolaknya, pelanggaran langsung NFR-02 "Jangan PHP". Pada v2.0 grant hanya
> diterbitkan setelah slot diamankan, jadi tidak ada jalur penolakan setelah `grant`.

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
    "paused": false,
    "server_time": "2026-10-24T13:10:00.412Z",
    "poll_after_ms": 5000
  }
}
```

> `poll_after_ms` ditentukan **server**, mengikuti profil di `requirements.md` §5:
> 10s (>1000) → 5s (101–1000) → 2s (11–100) → 1s (1–10). Klien tidak menanam interval sendiri.

**Requirement Coverage**: FR-07 (fallback), NFR-08

---

## 4. Booking

### 4.1 Get Ticket Selection Page

Menampilkan kategori tiket dengan stok. UI: "Pilih tempatmu di konser."

**Endpoint**: `GET /events/{event_id}/tickets`

**Headers**:
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

**Requirement Coverage**: FR-11, FR-12, FR-13, FR-17, BR-01, BR-02

---

### 4.2 Validate NIK

Validasi NIK ke Dukcapil sebelum submit booking. UI: dipanggil saat input NIK blur atau sebelum "Lanjut ke Pembayaran".

**Endpoint**: `POST /nik/validate`

**Headers**:
```
Authorization: Bearer {access_token}
```

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

> **Fail closed.** Baik NIK yang ditolak Dukcapil maupun Dukcapil yang tidak dapat dihubungi
> sama-sama memblokir pembayaran. `POST /bookings` menolak permintaan kecuali **setiap** NIK
> di dalamnya sudah terverifikasi. Konsekuensinya Dukcapil adalah single point of failure yang
> diterima secara sadar: saat ia mati, admission di-pause dan penjualan berhenti (ADR-005
> Bagian 8).

**Requirement Coverage**: FR-15a (validasi NIK ke Dukcapil)

---

### 4.3 Create Booking

Submit pemilihan tiket. UI: "Lanjut ke Pembayaran" button.

**Endpoint**: `POST /bookings`

**Headers**:
```
Authorization: Bearer {access_token}
```

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

> **Satu baris per kursi** (BR-02: 1 NIK = 1 seat). `holder_email` adalah alamat tujuan
> e-tiket kursi itu — FR-26 menuntut tiket sampai ke tiap pemegang NIK, bukan hanya pembeli.
> Tidak ada field nama: Dukcapil hanya memvalidasi NIK.
>
> **Panggilan inilah yang memulai TTL pembayaran.** "Booking dibuat" pada BR-05 adalah saat
> ini: kursi dipilih, NIK diisi, stok di-hold. `payment_ttl.ttl_seconds` karena itu selalu
> **600** — setiap customer mendapat 10 menit penuh, berapa pun lama ia memilih kursi.
> TTL booking (5 menit) berhenti berlaku begitu booking ini dibuat; keduanya berurutan,
> tidak tumpang tindih.
>
> **Idempoten.** Submit yang diulang tidak menjalankan hold kedua. Server mengunci baris
> booking (`SELECT … FOR UPDATE` atas `access_token`); bila status sudah `HELD` atau lebih
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
        "nik_masked": "3171********xx22",
        "holder_name": "JOHN DOE"
      },
      {
        "category": "VIP",
        "price": 1500000,
        "nik_masked": "3171********xx33",
        "holder_name": "JANE DOE"
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

**Response** (400 - NIK Name Mismatch):
```json
{
  "meta": {
    "code": 400,
    "message": "NIK 3171********xx22: nama tidak cocok"
  },
  "data": {}
}
```

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
Authorization: Bearer {access_token}
```

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
        "eticket_url": "/tickets/TKT-001/eticket"
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

**Requirement Coverage**: FR-21, FR-22

---

## 5. Payment

### 5.1 Get Payment Methods

Metode pembayaran yang tersedia beserta **parameter yang dibutuhkan klien**. UI: Payment
Method selection.

**Endpoint**: `GET /payments/{booking_id}/methods`

**Headers**:
```
Authorization: Bearer {access_token}
```

**Response** (200):
```json
{
  "meta": { "code": 200, "message": "Success" },
  "data": {
    "methods": [
      {
        "id": "credit_card",
        "name": "Kartu Kredit/Debit",
        "description": "Verifikasi 3D Secure, hasil langsung.",
        "icon_url": "https://cdn.rusuh.id/icons/card.svg",
        "enabled": true,
        "requires_tokenization": true,
        "client_key": "SB-Mid-client-xxxxxxxxxxxx"
      },
      {
        "id": "bank_transfer",
        "name": "Transfer Bank / Virtual Account",
        "description": "Verifikasi otomatis, tanpa unggah bukti.",
        "icon_url": "https://cdn.rusuh.id/icons/bank.svg",
        "enabled": true,
        "requires_tokenization": false,
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
        "enabled": true,
        "requires_tokenization": false
      }
    ]
  }
}
```

| Field | Arti |
|---|---|
| `id` | Dikirim kembali sebagai `method` pada §5.2. Nilainya sama dengan `payment_type` Midtrans |
| `requires_tokenization` | `true` → klien **wajib** memanggil Midtrans.js lebih dulu dan mengirim `token_id` |
| `client_key` | Hanya untuk kartu. Client key Midtrans untuk tokenisasi di browser. **Server key tidak pernah dikirim ke klien** |
| `banks` | Hanya untuk `bank_transfer`. Daftar bank yang dibuka |
| `enabled` | `false` saat breaker payment gateway terbuka; UI menonaktifkan pilihan |

**Keputusan lingkup POC — hanya empat bank.** Midtrans mendokumentasikan kasus *reversal*
(`settlement` berubah menjadi `deny` dalam 1–5 menit, dana kembali ke customer) **hanya**
untuk Permata, Mandiri Bill Payment, dan Indomaret. Ketiganya tidak dibuka, sehingga kelas
kegagalan itu tidak dapat terjadi dan handler reversal tidak diperlukan. Lihat
`payment-method/02-bank-transfer-va.md`.

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
Authorization: Bearer {access_token}
```

Bentuk request berbeda per metode:

**Kartu** — `token_id` diperoleh klien dari Midtrans.js sebelum memanggil endpoint ini:
```json
{ "method": "credit_card", "token_id": "48111111sHfSakAvHvFQFEjTivUV1114" }
```

**Transfer Bank**:
```json
{ "method": "bank_transfer", "bank": "bca" }
```

**E-Wallet**:
```json
{ "method": "gopay" }
```

**Response** (200) — bentuk `data.instructions` mengikuti metode:

**Kartu, 3DS diperlukan**:
```json
{
  "meta": { "code": 200, "message": "Payment initiated" },
  "data": {
    "payment_id": "RB-00912:1",
    "method": "credit_card",
    "status": "PENDING",
    "instructions": { "redirect_url": "https://api.midtrans.com/v2/3ds/redirect/..." },
    "ttl": { "expires_at": "2026-10-25T10:25:00Z", "ttl_seconds": 540, "server_time": "2026-10-25T10:16:00Z" }
  }
}
```

**Transfer Bank**:
```json
{
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

**Response** (409 - status tidak sesuai): booking belum `HELD` atau sudah terminal.

> **Setelah penolakan, panggil endpoint ini lagi — bukan `/retry`.** Jalur `deny` menaikkan
> `payment_attempts` sendiri, sehingga `order_id` yang diturunkan sudah baru dan Midtrans
> menerimanya. `/retry` hanya untuk membatalkan pembayaran yang masih `pending` (§5.3).

**Response** (422 - `token_id` tidak valid atau sudah dipakai): token Midtrans **sekali
pakai**; klien harus melakukan tokenisasi ulang.

**Response** (503 - `ADMISSION_PAUSED`): breaker payment gateway terbuka.

### Expiry yang dikirim ke Midtrans

Untuk `bank_transfer` dan `gopay`, charge menyertakan `custom_expiry` yang **menyamai TTL
pembayaran**:

```json
"custom_expiry": { "order_time": "<held_at, WIB>", "expiry_duration": "10", "unit": "minute" }
```

Tanpa ini, expiry VA bawaan Midtrans adalah **24 jam** — VA tetap dapat dibayar sepanjang
hari setelah sweep melepas stoknya dan menjualnya ke orang lain. `order_time` diisi
`held_at`, bukan waktu request, agar jendela Midtrans berakhir bersama jendela kita (BR-05).

`custom_expiry` **tidak berlaku untuk kartu**; jendelanya ditentukan batas waktu 3DS.
Untuk ketiga metode, sweep juga memanggil `POST /v2/{order_id}/expire` saat meng-expire
booking, sebagai lapis kedua terhadap selisih jam.

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
Authorization: Bearer {access_token}
```

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

**Response** (404): `booking_id` tidak ada **atau bukan milik pemegang token**. Keduanya
menjawab sama — `403` akan mengonfirmasi bahwa booking itu ada.

**Response** (409 - status tidak sesuai): booking tidak dalam `AWAITING_PAYMENT`. Termasuk
`HELD` setelah penolakan: tidak ada transaksi Midtrans yang hidup, panggil §5.2.

**Response** (409 - `PAYMENT_WINDOW_CLOSED`): TTL sudah lewat, atau sisanya di bawah
`MIN_RETRY_REMAINING_SECONDS`. Menerbitkan instrumen baru dengan sisa 15 detik hanya
memindahkan kegagalan ke tempat yang lebih menyakitkan.

**Response** (429 - `MAX_ATTEMPTS_EXCEEDED`): melewati `MAX_PAYMENT_ATTEMPTS`. Tiap
percobaan menerbitkan VA/QR baru di Midtrans.

**Response** (503): Midtrans tidak dapat dihubungi. `payment_attempts` **tidak** dinaikkan;
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
Authorization: Bearer {access_token}
```

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
    "reason": "Kartu ditolak penerbit",
    "can_retry": true,
    "attempt": 2,
    "payment_expires_at": "2026-10-25T10:25:00Z",
    "server_time": "2026-10-25T10:19:40Z"
  }
}
```

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
| `EXPIRED` / `LAPSED` | `EXPIRED` | Layar kedaluwarsa |

**Requirement Coverage**: FR-22, FR-23

---

### 5.5 Payment Gateway Callback (Webhook)

Callback dari payment gateway. Endpoint internal — tidak dipanggil klien.

**Endpoint**: `POST /payments/webhook`

**Headers**:
```
X-Callback-Signature: sha256=...
```

> Dilindungi **resource policy API Gateway** (allowlist IP gateway) ditambah verifikasi HMAC.
> Tidak ada CAPTCHA dan tidak ada rate limit per-IP pada endpoint ini.

**Request** (dari gateway):
```json
{
  "event_id": "evt_9f2a1c7b",
  "transaction_id": "pay_123",
  "external_id": "RB-00912",
  "status": "PAID",
  "paid_amount": 1500000,
  "paid_at": "2026-10-24T13:23:00Z",
  "payment_method": "va"
}
```

**Verifikasi tanda tangan**:

```
signature_key == SHA512(order_id + status_code + gross_amount + ServerKey)
```

Konkatenasi polos tanpa pemisah. Tidak cocok → **abaikan notifikasi**, jangan proses, jangan
ubah status apa pun.

> Dokumentasi Midtrans memuat dua skema tanda tangan. SHA256withRSA pada halaman *Security
> Specification* berlaku untuk lini API lain, **bukan** webhook Core API.

**Idempotency**: `event_id` disimpan di `payment_events.gateway_event_id` dengan constraint
`UNIQUE`. Pengiriman ulang memicu unique violation, seluruh transaksi batal, dan endpoint
tetap menjawab `200`.

> **Kuncinya adalah ID event milik gateway, bukan `booking_id`.** Memakai `booking_id` akan
> menelan setiap event sah berikutnya untuk booking yang sama (pending → settled → refund).

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

## 6. User

### 6.1 Get My Tickets

List tiket user. UI: "Tiket saya" menu.

**Endpoint**: `GET /users/me/tickets`

**Headers**:
```
Authorization: Bearer {access_token}
```

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

**Endpoint**: `GET /tickets/{ticket_id}/eticket`

**Headers**:
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

## 7. Common Response Structure

### Success Response

```json
{
  "meta": {
    "code": 200,
    "message": "Success"
  },
  "data": { ... }
}
```

### Success Response with Pagination

```json
{
  "meta": {
    "code": 200,
    "message": "Success",
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
    "message": "NIK must be exactly 16 digits"
  },
  "data": {}
}
```

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
| FR-05 | `POST /events/{id}/queue/join` → `granted: true` saat antrian kosong | ✅ |
| FR-06 | Join response + SSE payload `nowServing` | ✅ |
| FR-07 | `GET /events/{id}/queue/stream`, fallback `…/queue/serving` | ✅ |
| FR-08 | SSE `grant` → `GET /events/{id}/queue/me` | ✅ |
| FR-09 | `ratePerMin` pada payload SSE | ✅ |
| FR-10 | Snapshot awal pada stream + `GET …/queue/me` | ✅ |
| FR-11 | `GET /events/{id}` | ✅ |
| FR-12 | `GET /events/{id}/tickets` | ✅ |
| FR-13 | `POST /bookings` (maks 3, ditegakkan di Lua) | ✅ |
| FR-14 | `POST /bookings` → `NO_STOCK` / `QTY_EXCEEDED` | ✅ |
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
| FR-26 | `holder_email` per kursi pada `POST /bookings` | ✅ |
| FR-27 | `GET /tickets/{id}/eticket` — QR, kategori, NIK. **Nomor bangku di luar lingkup POC** | ✅ |
| FR-28 | Webhook `EXPIRE` / sweep → email expired | ⚙️ Backend |
| FR-29 | Semua endpoint pembelian dapat diakses tanpa akun | ✅ |
| FR-30 | `POST /bookings` hanya butuh NIK + email | ✅ |
| FR-31 | `POST /auth/magic-link`, `GET /auth/verify` | ✅ |
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
| NFR-07 | `access_token` pada endpoint booking/payment; tidak ada data kartu | ✅ |
| NFR-08 | REST/JSON, platform-agnostic | ✅ |
| NFR-09 | CloudWatch EMF, namespace `RusuhBooking/Queue` | ⚙️ Observability |
| NFR-10 | Kegagalan email tidak mengubah status booking; `GET /bookings/{id}/tickets` selalu tersedia | ✅ |
| BR-01 | Maks 3 tiket per `POST /bookings` | ✅ |
| BR-02 | Satu entri per kursi, satu NIK masing-masing; `UNIQUE (event_id, nik_hmac)` menolak NIK yang sudah dipakai booking lain di event yang sama | ✅ |
| BR-03/04 | `booking_expires_at` = `slot_claimed_at` + 5 menit | ✅ |
| BR-05/06 | `payment_expires_at` = `held_at` + 10 menit — berurutan setelah TTL booking, penuh bagi setiap customer | ✅ |
| BR-09 | `GET /events/{id}/queue/me` mengembalikan token yang sama | ✅ |
| BR-10 | `server_time` pada tiap respons; klien menghitung offset jam sekali dan merender semua countdown terhadapnya | ✅ |

Legenda: ✅ terpenuhi di kontrak · ⚙️ di luar permukaan API (backend atau observability) ·
⚠️ terpenuhi sebagian, keterbatasan dicatat

---

## 9. Security Considerations

| Aspect | Implementation |
|--------|----------------|
| Queue Token | `queue_token` cookie (HttpOnly, Secure), set via `Set-Cookie` on queue join |
| Access Token | JWT, returned in response body; client stores and sends via `Authorization: Bearer` |
| Refresh Token | Returned in response body on login; client stores securely (Keychain/Keystore/memory) |
| NIK | Masked in responses (3171********xx22), stored encrypted |
| Payment | Via payment gateway (PCI-DSS compliant), no card data stored |
| Rate Limiting | Per IP on queue join, per user on auth |
| CSRF | `SameSite=Strict` on queue cookie; Origin validation on state-changing endpoints |
| **Object ownership** | Setiap endpoint yang menerima `booking_id` atau `ticket_id` dari path mengikatnya ke pemegang token: `WHERE id = $1 AND queue_token = $accessToken`. Token yang sah membuktikan pemegangnya punya slot, bukan bahwa slot itu miliknya. Tidak cocok → **404**, bukan 403 |
| SSE Auth | `queue_token` cookie validated on stream connection |

---

## Appendix A: Error Codes

| Code | Description |
|------|-------------|
| `VALIDATION_ERROR` | Input validation failed |
| `NIK_ALREADY_USED` | NIK sudah digunakan untuk event ini |
| `NIK_NOT_FOUND` | NIK tidak terdaftar di Dukcapil |
| `NIK_INACTIVE` | NIK terdaftar tapi status tidak aktif |
| `NIK_VALIDATION_FAILED` | Validasi NIK ke Dukcapil gagal |
| `NIK_UNVERIFIED` | Ada NIK dalam booking yang belum lolos `POST /nik/validate` |
| `DUKCAPIL_UNAVAILABLE` | Dukcapil tidak dapat dihubungi — pembayaran diblokir (fail closed) |
| `QTY_EXCEEDED` | Melebihi 3 tiket per transaksi (BR-01) |
| `INSUFFICIENT_STOCK` | Stok tiket tidak mencukupi |
| `TICKETS_UNAVAILABLE` | Tiket sudah habis |
| `TERMS_NOT_ACCEPTED` | Syarat & ketentuan dan persetujuan PDP belum diterima |
| `PAYMENT_ALREADY_INITIATED` | Charge sudah berjalan; respons sebelumnya diputar ulang |
| `CARD_TOKEN_INVALID` | `token_id` Midtrans tidak valid atau sudah dipakai — tokenisasi ulang |
| `BANK_NOT_SUPPORTED` | Bank di luar bca/bni/bri/cimb |
| `MAX_ATTEMPTS_EXCEEDED` | Melewati batas percobaan pembayaran per booking |
| `PAYMENT_ALREADY_SETTLED` | Pembayaran lama sudah lunas; pembatalan tidak dilakukan |
| `PAYMENT_WINDOW_CLOSED` | TTL pembayaran habis atau sisanya terlalu pendek untuk ganti metode |
| `ADMISSION_PAUSED` | Admission dihentikan sementara (breaker Dukcapil / payment gateway) |
| `BOOKING_EXPIRED` | TTL booking habis |
| `PAYMENT_EXPIRED` | TTL pembayaran habis |
| `TOO_MANY_REQUESTS` | Rate limited |
| `TOKEN_EXPIRED` | Access token expired |
| `TOKEN_INVALID` | Access token invalid |

---

## Appendix B: Real-time Strategy

### Queue Status: SSE (Server-Sent Events)

| Aspect | Value |
|--------|-------|
| Transport | SSE (`GET /events/{event_id}/queue/stream`) |
| Auth | Cookie `HttpOnly` (web) / header `Authorization` (mobile) — tidak pernah query string |
| Broadcast | Saat `nowServing` berubah, saat `paused` berubah, atau tiap 10 detik |
| Tick admission | 500 ms |
| Keepalive | Komentar `: keepalive` tiap **15 detik** (bukan event bernama) |
| Snapshot awal | Ya — dikirim saat koneksi terbuka (FR-10) |
| Reconnect | Otomatis dengan `Last-Event-ID`; `retry:` berjitter saat shutdown |
| Sold out | Event `sold-out` → klien **wajib** `close()`; koneksi baru ditolak `410 Gone` |

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
| 13 | Webhook: switch per tipe event + `gateway_event_id UNIQUE` | Sebelumnya hanya menangani `PAID`; "idempoten" dinyatakan tanpa mekanisme |
| 14 | Webhook: `PAID` atas booking `EXPIRED` → `payment.refund` | Uang diterima setelah stok dijual ulang sebelumnya tidak punya jalur |
| 15 | Snapshot awal pada stream | FR-10 sebelumnya ditandai terpenuhi tanpa mekanisme |
| 16 | Matriks coverage dikoreksi | FR-10, FR-19 dan FR-25 sebelumnya ditandai ✅ tanpa dasar |
