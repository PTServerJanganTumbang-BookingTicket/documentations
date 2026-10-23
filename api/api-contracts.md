# Rusuh Booking Tiket — API Contracts

**Versi**: 1.0  
**Tanggal**: Oktober 2026  
**Status**: Draft  
**Base URL**: 

---

## Daftar Isi

1. [Authentication](#1-authentication)
2. [Events](#2-events)
3. [Queue](#3-queue)
4. [Booking](#4-booking)
5. [Payment](#5-payment)
6. [User](#6-user)
7. [Common Response Codes](#7-common-response-codes)

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
  "success": true,
  "message": "Login link sent to your email",
  "expires_in": 900
}
```

**Response** (429 - Rate Limited):
```json
{
  "success": false,
  "error": "TOO_MANY_REQUESTS",
  "message": "Please wait before requesting another link",
  "retry_after": 60
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
  "success": true,
  "message": "New login link sent",
  "expires_in": 900
}
```

---

### 1.3 Verify Magic Link

Verifikasi token dari link email. Redirect target dari email link.

**Endpoint**: `GET /auth/verify?token={token}`

**Response** (200):
```json
{
  "success": true,
  "access_token": "eyJhbG...",
  "refresh_token": "eyJhbG...",
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
  "nik": "0000000000000000"
}
```

**Response** (200):
```json
{
  "success": true,
  "message": "Verification link sent to your email",
  "user_id": "usr_456"
}
```

**Validation Errors** (400):
```json
{
  "success": false,
  "error": "VALIDATION_ERROR",
  "details": [
    { "field": "nik", "message": "NIK must be exactly 16 digits" }
  ]
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
| `page` | int | No | Default: 1 |
| `limit` | int | No | Default: 10, Max: 50 |

**Response** (200):
```json
{
  "success": true,
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
    },
    {
      "id": "evt_konser_c",
      "name": "Konser C — Surabaya",
      "category": "LIVE MUSIC",
      "location": {
        "venue": "GOR",
        "city": "Surabaya"
      },
      "date": "2027-02-02",
      "image_url": "https://cdn.rusuh.id/events/konser-c.jpg",
      "status": "AVAILABLE",
      "has_queue": false,
      "queue_length": 0,
      "min_price": 500000,
      "max_price": 1500000
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 3,
    "has_next": false
  }
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
  "success": true,
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
  "success": true,
  "data": {
    "queue_token": "qt_xyz789",
    "ticket_no": 32748,
    "now_serving": 31500,
    "position": 1248,
    "estimated_wait_minutes": 4,
    "sse_url": "/queue/stream?token=qt_xyz789",
    "event": {
      "id": "evt_konser_a",
      "name": "Konser A — Jakarta"
    }
  }
}
```

**Response** (409 - Already in Queue):
```json
{
  "success": true,
  "data": {
    "queue_token": "qt_xyz789",
    "ticket_no": 32748,
    "now_serving": 31520,
    "position": 1228,
    "estimated_wait_minutes": 4,
    "sse_url": "/queue/stream?token=qt_xyz789",
    "rejoined": true
  }
}
```

**Requirement Coverage**: FR-04, FR-10 (posisi konsisten saat refresh), NFR-05 (FCFS fairness)

---

### 3.2 Queue Status Stream (SSE)

Server-Sent Events stream untuk update posisi real-time. Client connect setelah join queue.

**Endpoint**: `GET /queue/stream`

**Query Parameters**:
| Param | Type | Required | Description |
|-------|------|----------|-------------|
| `token` | string | Yes | Queue token dari join response |

**Headers**:
```
Accept: text/event-stream
```

**Response**: SSE stream dengan event types berikut:

**Event: `status`** (broadcast setiap 1-2 detik)
```
event: status
data: {"now_serving": 31600, "throughput": 12, "server_time": "2026-10-23T13:10:00Z"}
```

> Frontend menghitung: `position = ticket_no - now_serving`

**Event: `your_turn`** (ketika posisi = 0)
```
event: your_turn
data: {"message": "Giliran Anda!", "claim_url": "/queue/claim-token"}
```

**Event: `heartbeat`** (setiap 30 detik untuk keep-alive)
```
event: heartbeat
data: {"server_time": "2026-10-23T13:10:30Z"}
```

**Event: `sold_out`** (jika tiket habis saat menunggu)
```
event: sold_out
data: {"message": "Maaf, tiket sudah habis"}
```

**Connection Handling**:
- Client harus reconnect jika connection drop
- Gunakan `Last-Event-ID` header untuk resume dari event terakhir
- Server close stream setelah `your_turn` atau `sold_out` event

**Requirement Coverage**: FR-07 (posisi diperbarui otomatis), NFR-06 (p95 < 1 detik)

---

### 3.3 Claim Access Token

Dipanggil saat `position = 0`. Mendapatkan token untuk masuk Detail Ticket Page.

**Endpoint**: `POST /queue/claim-token`

**Headers**:
```
X-Queue-Token: qt_xyz789
```

**Response** (200):
```json
{
  "success": true,
  "data": {
    "access_token": "at_abc123",
    "expires_at": "2026-10-23T13:15:00Z",
    "ttl_seconds": 300,
    "redirect_url": "/events/evt_konser_a/tickets"
  }
}
```

**Response** (410 - Tickets Gone):
```json
{
  "success": false,
  "error": "TICKETS_UNAVAILABLE",
  "message": "Maaf, tiket sudah habis"
}
```

**Response** (429 - Slots Full):
```json
{
  "success": false,
  "error": "SLOTS_FULL",
  "message": "Slot penuh, silakan tunggu",
  "retry_after": 5
}
```

**Requirement Coverage**: FR-08 (otomatis dipindahkan ke Detail Page), NFR-07 (token protection)

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
  "success": true,
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

### 4.2 Create Booking

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
      "email": "buyer@example.com"
    },
    {
      "category_id": "cat_vip",
      "nik": "3171xxxxxxxxxx33",
      "email": "friend@example.com"
    }
  ]
}
```

**Response** (201):
```json
{
  "success": true,
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
  "success": false,
  "error": "VALIDATION_ERROR",
  "details": [
    { "field": "tickets[0].nik", "message": "NIK must be exactly 16 digits" },
    { "field": "tickets", "message": "Maximum 3 tickets per order" }
  ]
}
```

**Response** (409 - NIK Already Used):
```json
{
  "success": false,
  "error": "NIK_ALREADY_USED",
  "message": "NIK 3171xxxxxxxxxx22 sudah digunakan untuk event ini",
  "details": [
    { "nik_masked": "3171********xx22", "index": 0 }
  ]
}
```

**Response** (409 - Insufficient Stock):
```json
{
  "success": false,
  "error": "INSUFFICIENT_STOCK",
  "message": "Stok VIP tidak mencukupi",
  "available": 1,
  "requested": 2
}
```

**Requirement Coverage**: FR-13, FR-14, FR-15, FR-16, NFR-04 (atomic stock), BR-01, BR-02

---

### 4.3 Get Booking Detail

Detail booking untuk Payment Page dan Success Page.

**Endpoint**: `GET /bookings/{booking_id}`

**Headers**:
```
Authorization: Bearer {access_token}
```

**Response** (200 - Pending Payment):
```json
{
  "success": true,
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
  "success": true,
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

List metode pembayaran yang tersedia. UI: Payment Method selection.

**Endpoint**: `GET /payments/{booking_id}/methods`

**Headers**:
```
Authorization: Bearer {access_token}
```

**Response** (200):
```json
{
  "success": true,
  "data": {
    "methods": [
      {
        "id": "va",
        "name": "Virtual Account",
        "description": "Verifikasi otomatis, tanpa unggah bukti.",
        "icon_url": "https://cdn.rusuh.id/icons/va.svg",
        "enabled": true
      },
      {
        "id": "ewallet",
        "name": "E-Wallet",
        "description": "Bayar praktis dengan dompet digital.",
        "icon_url": "https://cdn.rusuh.id/icons/ewallet.svg",
        "enabled": true
      },
      {
        "id": "bank_transfer",
        "name": "Transfer Bank",
        "description": "Transfer melalui rekening bank pilihan.",
        "icon_url": "https://cdn.rusuh.id/icons/bank.svg",
        "enabled": true
      }
    ]
  }
}
```

**Requirement Coverage**: FR-20

---

### 5.2 Initiate Payment

Memulai pembayaran dengan metode terpilih. UI: "Bayar Sekarang" button.

**Endpoint**: `POST /payments/{booking_id}`

**Headers**:
```
Authorization: Bearer {access_token}
```

**Request**:
```json
{
  "payment_method": "va"
}
```

**Response** (200):
```json
{
  "success": true,
  "data": {
    "payment_id": "pay_123",
    "status": "PENDING",
    "method": "va",
    "instructions": {
      "bank": "BCA",
      "va_number": "8801234567890",
      "amount": 1500000,
      "expires_at": "2026-10-23T13:25:00Z"
    },
    "redirect_url": null,
    "ttl": {
      "expires_at": "2026-10-23T13:25:00Z",
      "ttl_seconds": 280,
      "server_time": "2026-10-23T13:20:20Z"
    }
  }
}
```

**Requirement Coverage**: FR-20, FR-21, FR-24 (API shared web/mobile)

---

### 5.3 Get Payment Status

Polling status pembayaran. UI: Payment Page polling.

**Endpoint**: `GET /payments/{booking_id}/status`

**Headers**:
```
Authorization: Bearer {access_token}
```

**Response** (200 - Pending):
```json
{
  "success": true,
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
  "success": true,
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
  "success": true,
  "data": {
    "status": "EXPIRED",
    "expired_at": "2026-10-23T13:25:00Z",
    "message": "Waktu pembayaran habis. Booking dibatalkan."
  }
}
```

**Requirement Coverage**: FR-22, FR-23

---

### 5.4 Payment Gateway Callback (Webhook)

Callback dari payment gateway. Internal endpoint.

**Endpoint**: `POST /payments/webhook`

**Headers**:
```
X-Callback-Signature: sha256=...
```

**Request** (dari gateway):
```json
{
  "transaction_id": "pay_123",
  "external_id": "RB-00912",
  "status": "PAID",
  "paid_amount": 1500000,
  "paid_at": "2026-10-23T13:23:00Z",
  "payment_method": "va",
  "metadata": {}
}
```

**Response** (200):
```json
{
  "success": true,
  "message": "Callback processed"
}
```

> Endpoint ini idempoten — callback duplikat tidak mengubah state yang sudah final.

**Requirement Coverage**: FR-25 (idempoten callback), NFR-07 (PCI-DSS via gateway)

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
| `page` | int | No | Default: 1 |

**Response** (200):
```json
{
  "success": true,
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
  ],
  "pagination": {
    "page": 1,
    "total": 1,
    "has_next": false
  }
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
  "success": true,
  "data": {
    "ticket_id": "TKT-001",
    "event": {
      "name": "Konser A — Jakarta",
      "date": "2026-12-25",
      "time": "19:00",
      "venue": "GBK, Jakarta"
    },
    "category": "VIP",
    "seat": "A-15",
    "holder": {
      "nik_masked": "3171********xx22"
    },
    "qr_code": "data:image/png;base64,iVBORw0KGgo...",
    "qr_value": "TKT-001|evt_konser_a|3171xxxxxxxxxx22",
    "instructions": "Tunjukkan QR code dan identitas pemegang saat masuk venue."
  }
}
```

**Requirement Coverage**: FR-27 (QR code, nomor bangku, NIK pemegang)

---

## 7. Common Response Codes

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

### Coverage Matrix

| Req ID | Endpoint | Status |
|--------|----------|--------|
| FR-01 | `GET /events` | ✅ Covered |
| FR-02 | `GET /events` (status AVAILABLE) | ✅ Covered |
| FR-03 | `GET /events` (status SOLD_OUT) | ✅ Covered |
| FR-04 | `POST /events/{id}/queue/join` | ✅ Covered |
| FR-05 | `GET /events/{id}` + direct to tickets | ✅ Covered (has_queue flag) |
| FR-06 | Queue join response + status | ✅ Covered |
| FR-07 | `GET /queue/status` | ✅ Covered |
| FR-08 | `POST /queue/claim-token` | ✅ Covered |
| FR-09 | Queue join response (estimated_wait) | ✅ Covered |
| FR-10 | Re-join returns same ticket_no | ✅ Covered (409 rejoined) |
| FR-11 | `GET /events/{id}` | ✅ Covered |
| FR-12 | `GET /events/{id}/tickets` | ✅ Covered |
| FR-13 | `POST /bookings` (max 3) | ✅ Covered |
| FR-14 | Validation error response | ✅ Covered |
| FR-15 | NIK validation (16 digit) | ✅ Covered |
| FR-16 | `POST /bookings` (creates hold) | ✅ Covered |
| FR-17 | Session TTL in response | ✅ Covered |
| FR-18 | Booking expires → TTL logic | ⚠️ Backend logic (not API visible) |
| FR-19 | `server_time` in responses | ✅ Covered |
| FR-20 | `GET /bookings/{id}/payment-methods` | ✅ Covered |
| FR-21 | Payment TTL in response | ✅ Covered |
| FR-22 | Payment PAID status | ✅ Covered |
| FR-23 | Payment EXPIRED status | ✅ Covered |
| FR-24 | REST/JSON API | ✅ Covered |
| FR-25 | Webhook idempotent | ✅ Covered |
| FR-26 | Email per NIK | ⚠️ Backend logic (not API visible) |
| FR-27 | E-ticket content | ✅ Covered |
| FR-28 | Email expired | ⚠️ Backend logic (not API visible) |
| FR-29 | Login optional | ✅ Covered (can book without auth) |
| FR-30 | NIK + email for purchase | ✅ Covered |
| FR-31 | Magic link verification | ✅ Covered |
| FR-32 | Multi-language | 🔲 Header `Accept-Language` (to add) |
| FR-33..35 | Dashboard metrics | 🔲 Admin API (separate doc) |

### UI-API Mapping

| UI Screen | Endpoints Used |
|-----------|---------------|
| Home Page | `GET /events` |
| Waiting Room | `POST /events/{id}/queue/join`, `GET /queue/status`, `POST /queue/claim-token` |
| Detail Ticket | `GET /events/{id}/tickets`, `POST /bookings` |
| Payment | `GET /bookings/{id}`, `GET /payments/{id}/methods`, `POST /payments/{id}`, `GET /payments/{id}/status` |
| Success | `GET /bookings/{id}`, `GET /tickets/{id}/eticket` |
| Login | `POST /auth/magic-link`, `POST /auth/magic-link/resend`, `GET /auth/verify` |
| Register | `POST /auth/register` |
| My Tickets | `GET /users/me/tickets`, `GET /tickets/{id}/eticket` |

### Findings dari UI Review

| Finding | Severity | Notes |
|---------|----------|-------|
| ✅ NIK per tiket | OK | UI shows "NIK Tiket 1", "NIK Tiket 2" — matches BR-02 |
| ✅ Max 3 tiket | OK | UI shows "Maks. 3 tiket per pesanan" — matches BR-01 |
| ✅ Countdown timer | OK | UI shows 04:47 / 09:42 countdown — matches FR-17, FR-21 |
| ✅ Re-entrant flow | OK | "You're logged in" shows "Booking context restored" — matches BR-09 |
| ✅ Order ID | OK | "Order #RB-00912" visible — matches booking_id format |
| ✅ Payment methods | OK | VA, E-Wallet, Transfer Bank shown — matches FR-20 |
| ⚠️ Category sold out | Minor | UI shows "Habis" badge but +/- buttons still visible (should be disabled) |
| ⚠️ NIK input count | Minor | UI shows 2 NIK inputs for 2 tickets — dynamic form needed |

---

## 9. Security Considerations

| Aspect | Implementation |
|--------|----------------|
| Queue Token | Short-lived, validated server-side, required for SSE stream |
| Access Token | JWT, 5 min TTL, required for booking/payment |
| NIK | Masked in responses (3171********xx22), stored encrypted |
| Payment | Via payment gateway (PCI-DSS compliant), no card data stored |
| Rate Limiting | Per IP on queue join, per user on auth |
| CSRF | Origin validation on state-changing endpoints |
| SSE Auth | Queue token validated on stream connection |

---

## Appendix A: Error Codes

| Code | Description |
|------|-------------|
| `VALIDATION_ERROR` | Input validation failed |
| `NIK_ALREADY_USED` | NIK sudah digunakan untuk event ini |
| `INSUFFICIENT_STOCK` | Stok tiket tidak mencukupi |
| `TICKETS_UNAVAILABLE` | Tiket sudah habis |
| `SLOTS_FULL` | Slot booking penuh |
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
| Transport | SSE (`GET /queue/stream`) |
| Broadcast interval | 1-2 detik |
| Heartbeat | 30 detik |
| Reconnect | Auto dengan `Last-Event-ID` |

**Keuntungan SSE untuk Queue:**
- Update instan tanpa polling overhead
- Server push `now_serving` identik ke semua client
- Efisien untuk 300k concurrent connections
- Built-in reconnection support

### Payment Status: Polling

| Aspect | Value |
|--------|-------|
| Transport | REST polling (`GET /payments/{id}/status`) |
| Interval | 3-5 detik |
| Duration | Max 10 menit (TTL pembayaran) |

**Alasan Polling untuk Payment:**
- Bounded duration (max 10 menit)
- Limited concurrent users (hanya yang di payment page)
- Simpler implementation, no persistent connection needed
- Payment gateway webhook sebagai source of truth
