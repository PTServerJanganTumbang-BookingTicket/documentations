# Rusuh Booking Tiket — Final Requirements

**Versi**: 1.0  
**Tanggal**: Oktober 2026  
**Status**: Finalized  
**Sumber**:
- Tender Rusuh Booking Ticket.pdf (baseline)
- update-requirement-rusuh-booking-tiket.md (perubahan resmi)
- 2026-10-16-meeting-business-needs-cto-pov.md (klarifikasi CEO/CTO)
- 2026-10-21-meeting-notes-queue-algorithm.md (keputusan arsitektur)

---

## 1. Gambaran Umum

Rusuh Booking Tiket adalah sistem pemesanan tiket konser online yang dirancang untuk
menangani lonjakan trafik ekstrem (flash crowd) saat penjualan tiket dibuka.

**Prinsip kunci:**
1. Jaga user experience apapun kondisi sistem di belakang
2. **Jangan PHP** — pengguna yang sudah masuk antrian dan memulai booking dijamin dapat
   menyelesaikan journey-nya sampai akhir tanpa diputus sepihak
3. **Transparansi** — apa yang terlihat di sistem harus mencerminkan kondisi aktual

---

## 2. Functional Requirements

| ID | Page | Requirement | Priority |
|----|------|-------------|----------|
| FR-01 | Home | Menampilkan daftar seluruh event yang tiketnya dijual online | MUST |
| FR-02 | Home | Event available dan memiliki stok ditampilkan aktif/clickable | MUST |
| FR-03 | Home | Event sold out ditampilkan non-aktif dengan label status | MUST |
| FR-04 | Home | Saat klik event dan ada antrian → diarahkan ke Waiting Room | MUST |
| FR-05 | Home | Jika tidak ada antrian → langsung ke Detail Ticket Page | MUST |
| FR-06 | Waiting Room | Menampilkan posisi antrian customer ("Anda antrian ke-n") | MUST |
| FR-07 | Waiting Room | Posisi diperbarui otomatis setiap antrian di depan berkurang | MUST |
| FR-08 | Waiting Room | Customer otomatis dipindahkan ke Detail Page saat giliran tiba | MUST |
| FR-09 | Waiting Room | Menampilkan estimasi waktu tunggu berdasarkan laju antrian | COULD |
| FR-10 | Waiting Room | Posisi tetap konsisten saat refresh halaman | MUST |
| FR-11 | Detail Ticket | Menampilkan detail event beserta kategori tempat | MUST |
| FR-12 | Detail Ticket | Setiap kategori menampilkan stok available dan harga | MUST |
| FR-13 | Detail Ticket | Customer dapat membeli maksimal **3 tiket per transaksi** | MUST |
| FR-14 | Detail Ticket | Menolak pemilihan jumlah yang melebihi stok atau batas 3 | MUST |
| FR-15 | Detail Ticket | Memvalidasi format NIK (16 digit) untuk **setiap tiket** | MUST |
| FR-16 | Detail Ticket | Saat lanjut ke pembayaran, stok tiket di-hold | MUST |
| FR-17 | Detail Ticket | Menampilkan countdown TTL Booking (5 menit) | MUST |
| FR-18 | Detail Ticket | TTL habis → booking batal, hold dilepas, slot ke antrian berikutnya | MUST |
| FR-19 | Detail Ticket | Timer booking berjalan di server (server-authoritative) | MUST |
| FR-20 | Payment | Customer dapat memilih metode pembayaran dari mitra tersedia | MUST |
| FR-21 | Payment | Menampilkan countdown TTL Pembayaran (10 menit) | MUST |
| FR-22 | Payment | Pembayaran berhasil → email e-tiket ke customer | MUST |
| FR-23 | Payment | Pembayaran tidak selesai dalam 10 menit → booking batal, stok dilepas | MUST |
| FR-24 | Payment | API pembayaran dapat dikonsumsi web dan mobile (shared API) | MUST |
| FR-25 | Payment | Menangani callback payment gateway secara idempoten | MUST |
| FR-26 | Notification | Email tiket dikirim ke **masing-masing pemegang NIK** | MUST |
| FR-27 | Notification | Tiket berisi: QR code, nomor bangku, NIK pemegang | MUST |
| FR-28 | Notification | Email expired dikirim jika pembayaran tidak selesai dalam TTL | MUST |
| FR-29 | Auth | **Login bersifat OPSIONAL** — bisa booking tanpa akun | MUST |
| FR-30 | Auth | Cukup input NIK dan email untuk melakukan pembelian | MUST |
| FR-31 | Auth | Jika buat akun, verifikasi email via link (bukan OTP) | SHOULD |
| FR-32 | Auth | Mendukung minimal 2 bahasa: Indonesia dan Inggris | SHOULD |
| FR-33 | Dashboard | Menampilkan metrik real-time: jumlah antrian aktif | MUST |
| FR-34 | Dashboard | Menampilkan laju konversi (antrian → pembelian) | MUST |
| FR-35 | Dashboard | Menampilkan expiry rate (booking tidak dibayarkan) | MUST |

---

## 3. Non-Functional Requirements

| ID | Kategori | Requirement |
|----|----------|-------------|
| NFR-01 | Availability | UX tetap terjaga apapun kondisi sistem; Waiting Room melindungi dari overload |
| NFR-02 | Reliability | "Jangan PHP": customer yang sudah dapat slot dijamin selesai journey |
| NFR-03 | Scalability | Menangani flash crowd 200.000+ user dengan auto-scaling layer stateless |
| NFR-04 | Consistency | Stok tidak oversell; pengurangan stok harus atomic |
| NFR-05 | Fairness | Antrian FCFS; posisi adil dan tidak dapat dilangkahi |
| NFR-06 | Performance | Pembaruan posisi p95 < 1 detik (untuk posisi ≤100) |
| NFR-07 | Security | Detail/Payment page diproteksi token; PCI-DSS via gateway |
| NFR-08 | Portability | API platform-agnostic (REST/JSON) untuk web dan mobile |
| NFR-09 | Observability | Metrik antrian, konversi, expiry dipantau real-time |
| NFR-10 | Resilience | Kegagalan email tidak membatalkan transaksi; async dengan retry + DLQ |

---

## 4. Business Rules

| ID | Status | Rule |
|----|--------|------|
| BR-01 | ✏️ UPDATED | Maksimal **3 tiket per transaksi** (bukan per NIK kumulatif) |
| BR-02 | ✏️ UPDATED | **1 NIK = 1 seat** per event (beli 3 tiket = 3 NIK berbeda) |
| BR-03 | — | TTL Booking = **5 menit** di Detail Ticket Page |
| BR-04 | — | TTL Booking habis → booking batal, hold dilepas, slot ke antrian berikutnya |
| BR-05 | — | TTL Pembayaran = **10 menit** sejak booking dibuat |
| BR-06 | — | TTL Pembayaran habis → booking batal, stok dilepas, email expired dikirim |
| BR-07 | ❌ DELETED | ~~Satu customer satu slot aktif~~ — tidak berlaku lagi |
| BR-08 | — | Stok yang sudah dibayar bersifat final; stok di-hold tidak bisa dibeli orang lain |
| BR-09 | ➕ NEW | Re-entrant user dalam TTL kembali ke halaman yang sama dengan token yang sama |
| BR-10 | ➕ NEW | **Server time** sebagai source of truth untuk semua operasi waktu |

---

## 5. Architecture Decisions (Locked)

| Aspect | Decision |
|--------|----------|
| Transport | **Adaptive Polling** (bukan SSE/WebSocket) |
| Position calculation | Di **frontend**: `posisi = ticket_no - now_serving` |
| Broadcast value | Server hanya kirim `now_serving` (global, identik semua user) |
| Polling interval | Dinamis: 10s (>1000) → 5s (101-1000) → 2s (11-100) → 1s (1-10) |
| Admission formula | `canEnter = (freeSlots > 0) AND (availableTickets > 0)` |
| Time authority | Server-side; frontend sync setiap 30 detik |

---

## 6. Admission Controller

Komponen yang mengatur berapa user dari antrian yang diizinkan masuk ke Detail Ticket Page.

### Formula

```
canEnter = (freeSlots > 0) AND (availableTickets > 0)
```

| Variabel | Definisi |
|----------|----------|
| `freeSlots` | Jumlah slot yang sedang idle (tidak aktif diproses) |
| `availableTickets` | Jumlah tiket yang masih tersisa |

### Kapan Slot Dibebaskan

1. User menyelesaikan pembayaran (sukses)
2. TTL Booking habis (5 menit tanpa konfirmasi)
3. TTL Pembayaran habis (10 menit)
4. User secara eksplisit membatalkan

---

## 7. Flow End-to-End

```
[Home Page]
    │
    ├── User klik "Beli Tiket" → Join Queue → dapat ticket_no
    │
    ↓
[Waiting Room]
    │
    ├── Adaptive polling: GET /queue/status → { now_serving }
    ├── Frontend hitung: posisi = ticket_no - now_serving
    ├── Tampilkan posisi & estimasi waktu tunggu
    │
    ├── Saat posisi = 0:
    │   └── Frontend hit: POST /queue/claim-token
    │       ├── Backend validasi: freeSlots > 0 AND availableTickets > 0
    │       ├── Sukses → access_token (TTL 5 menit) → redirect ke Detail Tiket
    │       └── Gagal (tiket habis / slot penuh) → tampilkan "Tiket Habis"
    │
    ↓
[Detail Ticket Page]
    │
    ├── Akses dijaga: access_token harus valid
    ├── Pilih kategori & jumlah tiket (max 3)
    ├── Input NIK + email per tiket (1 NIK per seat)
    ├── Countdown TTL Booking (5 menit, server-synced)
    │
    ↓
[Payment Page]
    │
    ├── Pilih metode pembayaran
    ├── Countdown TTL Pembayaran (10 menit)
    ├── Polling status pembayaran
    └── Sukses → release slot → now_serving++ → tiket diterbitkan via email
```

---

## 8. Ruang Lingkup

### Dalam Lingkup (In Scope)

- Menampilkan daftar event dan status ketersediaannya
- Sistem antrian virtual (virtual waiting room) dengan posisi antrian real-time
- Pemilihan kategori tiket dengan harga berbeda dan pembatasan maksimal 3 tiket/transaksi
- Alur pembayaran dengan timeout 10 menit dan pelepasan stok otomatis
- Notifikasi email untuk tiket sukses maupun booking expired
- API pembayaran yang dapat dikonsumsi oleh web dan mobile apps
- Halaman pembayaran berbasis web yang reusable di mobile (WebView)
- Dashboard operasional real-time (antrian, konversi, expiry rate)

### Di Luar Lingkup (Out of Scope)

- Pembuatan/manajemen event oleh admin (back-office)
- Implementasi internal mitra pembayaran (via payment gateway pihak ketiga)
- Program loyalti, diskon, dan voucher
- Validasi NIK ke Dukcapil (skip untuk POC)
- Sistem gate check-in di lokasi event

---

## 9. Aktor & Peran

| Aktor | Tipe | Peran |
|-------|------|-------|
| Customer | Manusia | Memilih event, mengantre, memilih tiket, dan membayar |
| Queue Service | Sistem | Mengatur antrian virtual, token akses, dan posisi antrian |
| Booking Service | Sistem | Mengelola stok tiket, reservasi sementara (hold), dan status booking |
| Payment Gateway | Sistem Eksternal | Memproses pembayaran dan mengirim callback status |
| Notification Service | Sistem | Mengirim email e-tiket atau pemberitahuan booking expired |
| Mobile App | Sistem Klien | Mengonsumsi API pembayaran yang sama dengan web |

---

## 10. Istilah

| Istilah | Definisi |
|---------|----------|
| Waiting Room | Ruang tunggu virtual tempat customer menunggu giliran masuk ke halaman pembelian |
| Access Token / Slot | Hak akses sementara yang diberikan kepada customer untuk masuk ke Detail Ticket Page |
| Hold / Reservasi | Penguncian stok tiket sementara saat customer berhasil masuk ke halaman payment |
| TTL Booking | Batas waktu customer berada di Detail Ticket Page (5 menit) |
| TTL Pembayaran | Batas waktu untuk menyelesaikan pembayaran (10 menit) |
| NIK | Nomor Induk Kependudukan (16 digit) — basis pembatasan 1 seat per NIK |
| `now_serving` | Nilai global yang menunjukkan nomor antrian yang sedang dilayani |
| `ticket_no` | Nomor antrian unik customer, diberikan saat join queue |

---

## 11. Ringkasan Perubahan dari Tender Asli

| Poin | Jenis | Ringkasan |
|------|-------|-----------|
| Login | ✏️ UPDATED | Dari implisit wajib → **opsional** (dikonfirmasi CEO) |
| BR-01 | ✏️ UPDATED | Kuota 3 tiket: dari per-NIK kumulatif → **per transaksi** |
| BR-02 | ✏️ UPDATED | Kuota NIK: dari 3 tiket per NIK → **1 NIK per seat** |
| BR-07 | ❌ DELETED | Satu slot aktif per customer tidak lagi berlaku |
| BR-09 | ➕ NEW | Re-entrant user dalam TTL kembali ke halaman yang sama |
| BR-10 | ➕ NEW | Server time sebagai source of truth |
| FR-26..28 | ➕ NEW | Email ke pemegang NIK, bukan hanya pembeli |
| FR-29..32 | ➕ NEW | Login opsional, verifikasi via link, multi-bahasa |
| FR-33..35 | ➕ NEW | Dashboard operasional real-time |
| Transport | 🔧 DECIDED | Adaptive polling (bukan SSE/WebSocket) |
| Position calc | 🔧 DECIDED | Di frontend (`ticket_no - now_serving`) |

---

## 12. Asumsi

- Data event (jadwal, kategori, harga, kapasitas) tersedia dari sistem back-office
- Customer memiliki email valid untuk pengiriman e-tiket
- Mitra pembayaran menyediakan mekanisme callback status
- Validasi NIK ke Dukcapil di-skip untuk POC (dilakukan manual di gate event)
- Target skala: 200.000 concurrent users pada flash crowd

---

## 13. Referensi Dokumen

| Dokumen | Path |
|---------|------|
| Tender asli | `requirement/Tender Rusuh Booking Ticket.pdf` |
| Update requirement | `requirement/update-requirement-rusuh-booking-tiket.md` |
| Meeting CTO POV | `discussions/2026-10-16-meeting-business-needs-cto-pov.md` |
| Meeting Queue Algorithm | `discussions/2026-10-21-meeting-notes-queue-algorithm.md` |
| ADR-001 Queueing Technology | `adr/ADR-001-queueing-technology.md` |
| ADR-002 Queue Position Mechanism | `adr/ADR-002-queue-position-mechanism.md` |
| ADR-003 Realtime Queue Updates | `adr/ADR-003-realtime-queue-updates.md` |
| ADR-004 Compute Service Selection | `adr/ADR-004-compute-service-selection.md` |
