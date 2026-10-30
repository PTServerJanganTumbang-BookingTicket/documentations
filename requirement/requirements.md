# Rusuh Booking Tiket — Final Requirements

**Versi**: 1.1  
**Tanggal**: 26 Oktober 2026  
**Status**: Finalized  
**Sumber**:

- Tender Rusuh Booking Ticket.pdf (baseline)
- update-requirement-rusuh-booking-tiket.md (perubahan resmi)
- 2026-10-16-meeting-business-needs-cto-pov.md (klarifikasi CEO/CTO)
- 2026-10-21-meeting-notes-queue-algorithm.md (keputusan arsitektur)
- Usulan requirement tim (25 Oktober 2026) — **butir 1–8 dikonfirmasi** dan masuk
  dokumen ini; butir sisanya masih dalam klarifikasi dan **belum** menjadi requirement

> **Perubahan v1.1** — delapan fitur baru dikonfirmasi: NIK terdaftar per akun,
> Daftar NIK Teman, persetujuan data pribadi, kapasitas antrian, grace period
> pembayaran, email kegagalan, serta dua penolakan eksplisit (TTL antrian absolut dan
> push notification). Rinciannya di Bagian 11.

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

| ID     | Page          | Requirement                                                           | Priority |
| ------ | ------------- | --------------------------------------------------------------------- | -------- |
| FR-01  | Home          | Menampilkan daftar seluruh event yang tiketnya dijual online          | MUST     |
| FR-02  | Home          | Event available dan memiliki stok ditampilkan aktif/clickable         | MUST     |
| FR-03  | Home          | Event sold out ditampilkan non-aktif dengan label status              | MUST     |
| FR-03a | Home          | Menampilkan dua field terpisah: `status` (stok) dan `queue_state` (izin masuk) | MUST     |
| FR-03c | Home          | `status`: `AVAILABLE` / `SELLING_OUT_FAST` / `SOLD_OUT`               | MUST     |
| FR-03d | Home          | `queue_state`: `NOT_OPEN` / `DIRECT` / `OPEN` / `FULL`                | MUST     |
| FR-03e | Home          | Server mengirim blok `display` siap-render (`enabled`, `action`, `label_key`, `badge_key`) — frontend tidak memutuskan apa pun | MUST     |
| FR-03b | Home          | State `FULL` menonaktifkan tombol beli dengan label "Antrian Penuh" — customer tahu sebelum klik, bukan sesudah | MUST     |
| FR-04  | Home          | Saat klik event dan ada antrian → diarahkan ke Waiting Room           | MUST     |
| FR-05  | Home          | Jika tidak ada antrian → langsung ke Detail Ticket Page               | MUST     |
| FR-06  | Waiting Room  | Menampilkan posisi antrian customer ("Anda antrian ke-n")             | MUST     |
| FR-07  | Waiting Room  | Posisi diperbarui otomatis setiap antrian di depan berkurang          | MUST     |
| FR-08  | Waiting Room  | Customer otomatis dipindahkan ke Detail Page saat giliran tiba        | MUST     |
| FR-09  | Waiting Room  | Menampilkan estimasi waktu tunggu berdasarkan laju antrian            | COULD    |
| FR-10  | Waiting Room  | Posisi tetap konsisten saat refresh halaman                           | MUST     |
| FR-10a | Waiting Room  | Join antrian ditolak **`QUEUE_FULL`** (fail fast) saat kapasitas antrian event penuh — tanpa buffering | MUST     |
| FR-10d | Waiting Room  | Join antrian menuntut **CAPTCHA tak terlihat**, berlaku untuk tamu dan user login | MUST     |
| FR-10e | Waiting Room  | Token CAPTCHA diambil **sebelum** sale dibuka dan disegarkan berkala; tidak ada tantangan interaktif saat join | MUST     |
| FR-10f | Waiting Room  | Provider CAPTCHA tidak tersedia → join **tetap diterima** (fail-open) dengan alarm | MUST     |
| FR-11  | Detail Ticket | Menampilkan detail event beserta kategori tempat                      | MUST     |
| FR-12  | Detail Ticket | Setiap kategori menampilkan stok available dan harga                  | MUST     |
| FR-13  | Detail Ticket | Customer dapat membeli maksimal **3 tiket per transaksi**, dijumlahkan **lintas kategori** | MUST     |
| FR-13a | Detail Ticket | Satu transaksi boleh memuat kategori berbeda (mis. 1 Premium + 1 Festival + 1 VIP) | MUST     |
| FR-13b | Detail Ticket | Hold multi-kategori bersifat **all-or-nothing**: stok seluruh kategori diperiksa sebelum satu pun dikurangi | MUST     |
| FR-14  | Detail Ticket | Menolak pemilihan jumlah yang melebihi stok atau batas 3              | MUST     |
| FR-15  | Detail Ticket | Memvalidasi format NIK (16 digit) untuk **setiap tiket**              | MUST     |
| FR-15a | Detail Ticket | Memvalidasi keberadaan NIK ke Dukcapil API (**NIK saja** — nama dan tanggal lahir tidak dikumpulkan) | MUST     |
| FR-15b | Detail Ticket | NIK yang berasal dari NIK Terdaftar atau Daftar NIK Teman **tidak memanggil Dukcapil lagi** — bukti verifikasi tersimpan dipakai ulang | MUST     |
| FR-15c | Detail Ticket | Dukcapil gagal/timeout untuk NIK yang belum terverifikasi → **booking diblokir** (fail-closed), tidak ada pembayaran yang dapat dilakukan | MUST     |
| FR-16  | Detail Ticket | Saat lanjut ke pembayaran, stok tiket di-hold                         | MUST     |
| FR-17  | Detail Ticket | Menampilkan countdown TTL Booking (5 menit, dihitung dari saat slot diberikan) | MUST     |
| FR-18  | Detail Ticket | TTL habis → booking batal, hold dilepas, slot ke antrian berikutnya   | MUST     |
| FR-19  | Detail Ticket | Timer booking berjalan di server (server-authoritative)               | MUST     |
| FR-20  | Payment       | Customer dapat memilih metode pembayaran dari mitra tersedia          | MUST     |
| FR-20a | Payment       | **Dua** metode di POC: Transfer Bank VA (BCA/BNI/BRI/CIMB) dan E-Wallet GoPay | MUST     |
| FR-20b | Payment       | Kartu kredit dan Over the Counter **di luar cakupan POC** — keduanya tidak dapat menghormati TTL 10 menit di sisi gateway | MUST     |
| FR-21  | Payment       | Menampilkan countdown TTL Pembayaran (10 menit)                       | MUST     |
| FR-22  | Payment       | Pembayaran berhasil → email e-tiket ke customer                       | MUST     |
| FR-23  | Payment       | Pembayaran tidak selesai dalam 10 menit → booking batal, stok dilepas | MUST     |
| FR-23a | Payment       | TTL Pembayaran habis → booking masuk **Pending Cancellation** selama **grace 30 detik**; stok **tetap** di-hold | MUST     |
| FR-23b | Payment       | Callback sukses yang tiba dalam grace **tetap mengonfirmasi** booking | MUST     |
| FR-23c | Payment       | Callback sukses setelah pembatalan final → booking ditandai **Disputed**, dicatat untuk rekonsiliasi manual | MUST     |
| FR-23d | Payment       | Customer dapat mengganti metode pembayaran selama status `AWAITING_PAYMENT`; transaksi lama dibatalkan ke gateway | MUST     |
| FR-24  | Payment       | API pembayaran dapat dikonsumsi web dan mobile (shared API)           | MUST     |
| FR-25  | Payment       | Menangani callback payment gateway secara idempoten                   | MUST     |
| FR-26  | Notification  | Email tiket dikirim ke **masing-masing pemegang NIK**                 | MUST     |
| FR-27  | Notification  | Tiket berisi: QR code, kategori tiket, NIK pemegang (**nomor bangku tidak diimplementasikan di POC**) | MUST     |
| FR-28  | Notification  | Email expired dikirim jika pembayaran tidak selesai dalam TTL         | MUST     |
| FR-28a | Notification  | Email pemberitahuan **kegagalan pembayaran** dikirim pada tiga kondisi: (a) gateway menyatakan gagal, (b) TTL + grace berlalu tanpa callback, (c) callback sukses tiba setelah pembatalan (Disputed) | MUST     |
| FR-28b | Notification  | Email kondisi (c) menyatakan dana diterima, booking batal, dan refund sedang diproses — **wajib email**, bukan hanya log | MUST     |
| FR-29  | Auth          | **Login bersifat OPSIONAL** — bisa booking tanpa akun                 | MUST     |
| FR-30  | Auth          | Cukup input NIK dan email untuk melakukan pembelian                   | MUST     |
| FR-31  | Auth          | Jika buat akun, verifikasi email via link (bukan OTP)                 | SHOULD   |
| FR-31a | Auth          | Registrasi akun **wajib menyertakan NIK** (NIK Terdaftar), diverifikasi Dukcapil saat registrasi | MUST     |
| FR-31b | Auth          | Satu NIK hanya dapat terdaftar pada **satu akun** secara global       | MUST     |
| FR-31c | Auth          | Registrasi wajib disertai **persetujuan pemrosesan NIK pihak ketiga**; checkbox tidak boleh pre-checked | MUST     |
| FR-31d | Auth          | Rekam persetujuan bersifat **immutable** (timestamp + versi kebijakan); pencabutan hanya lewat tim operasional | MUST     |
| FR-32  | Auth          | Mendukung minimal 2 bahasa: Indonesia dan Inggris                     | SHOULD   |
| FR-32a | Profil        | Akun dapat menyimpan **Daftar NIK Teman** — maksimal **5 entri** per akun | MUST     |
| FR-32b | Profil        | NIK diverifikasi Dukcapil **saat ditambahkan**; NIK yang gagal validasi tidak disimpan | MUST     |
| FR-32c | Profil        | Daftar NIK Teman hanya menampilkan NIK ter-mask; NIK lengkap tidak pernah dikembalikan API | MUST     |
| FR-32d | Profil        | Menghapus entri Daftar NIK Teman **tidak** memengaruhi tiket yang sudah dibeli dengan NIK itu | MUST     |
| FR-33  | Dashboard     | Menampilkan metrik real-time: jumlah antrian aktif                    | MUST     |
| FR-34  | Dashboard     | Menampilkan laju konversi (antrian → pembelian)                       | MUST     |
| FR-35  | Dashboard     | Menampilkan expiry rate (booking tidak dibayarkan)                    | MUST     |
| FR-36  | Dashboard     | Menampilkan daftar booking **Disputed** beserta `order_id` dan nominal untuk rekonsiliasi manual | MUST     |

---

## 3. Non-Functional Requirements

| ID     | Kategori      | Requirement                                                                   |
| ------ | ------------- | ----------------------------------------------------------------------------- |
| NFR-01 | Availability  | UX tetap terjaga apapun kondisi sistem; Waiting Room melindungi dari overload |
| NFR-02 | Reliability   | "Jangan PHP": customer yang sudah dapat slot dijamin selesai journey          |
| NFR-03 | Scalability   | Menangani flash crowd 200.000+ user dengan auto-scaling layer stateless; target rancangan **300.000 concurrent** |
| NFR-04 | Consistency   | Stok tidak oversell; pengurangan stok harus atomic                            |
| NFR-05 | Fairness      | Antrian FCFS; posisi adil dan tidak dapat dilangkahi. Proteksi: rate limit per IP + CAPTCHA tak terlihat di join + BR-07 untuk user login. Tanpa WAF, botnet terdistribusi tetap di luar jangkauan — degradasi yang diterima untuk POC |
| NFR-06 | Performance   | Pembaruan posisi antrian dan halaman utama memiliki latensi rendah (p95 < 1 detik) di bawah beban tinggi — **semua posisi**, sesuai tender asli |
| NFR-07 | Security      | Detail/Payment page diproteksi token; PCI-DSS via gateway                     |
| NFR-08 | Portability   | API platform-agnostic (REST/JSON) untuk web dan mobile                        |
| NFR-09 | Observability | Metrik antrian, konversi, expiry dipantau real-time                           |
| NFR-10 | Resilience    | Kegagalan email tidak membatalkan transaksi; async dengan retry + DLQ         |
| NFR-11 | Compliance    | Pemrosesan NIK pihak ketiga tunduk **UU PDP No. 27/2022**: persetujuan eksplisit, rekam immutable, NIK disimpan sebagai HMAC (bukan plaintext) |
| NFR-12 | Cost          | Dukcapil berbayar per panggilan; NIK Terdaftar dan Daftar NIK Teman ada untuk menekan volume. Volume aktual **bergantung pada rasio login** — estimasi tender memakai skenario tamu-berat |
| NFR-13 | Capacity      | Kapasitas antrian dibatasi **300.000 per event**; kelebihan ditolak fail-fast, tidak di-buffer |
| NFR-14 | Observability | Kesegaran data antrian terpantau di **dua arah**: hilangnya broadcast > 30 detik menyalakan indikator stale ke customer, dan `CtlHeartbeatAge` mem-page operasi. Ini deteksi, bukan pemulihan — pemulihannya ada di leader lease `admission-ctl` |

---

## 4. Business Rules

| ID    | Status     | Rule                                                                              |
| ----- | ---------- | --------------------------------------------------------------------------------- |
| BR-01 | ✏️ UPDATED | Maksimal **3 tiket per transaksi** (bukan per NIK kumulatif)                      |
| BR-02 | ✏️ UPDATED | **1 NIK = 1 seat** per event (beli 3 tiket = 3 NIK berbeda)                       |
| BR-03 | —          | TTL Booking = **5 menit sejak slot diberikan** (`granted_at`), bukan sejak halaman dibuka                                   |
| BR-04 | —          | TTL Booking habis → booking batal, hold dilepas, slot ke antrian berikutnya       |
| BR-05 | —          | TTL Pembayaran = **10 menit** sejak booking dibuat                                |
| BR-06 | —          | TTL Pembayaran habis → booking batal, stok dilepas, email expired dikirim         |
| BR-07 | ✔ RESTORED | **Satu keterlibatan aktif per event** untuk customer login (antrian ATAU booking). Dilepas saat terminal — setelah `CONFIRMED` boleh antre lagi, dari belakang. **Tidak berlaku untuk tamu** |
| BR-08 | —          | Stok yang sudah dibayar bersifat final; stok di-hold tidak bisa dibeli orang lain |
| BR-09 | ➕ NEW     | Re-entrant user dalam TTL kembali ke halaman yang sama dengan token yang sama     |
| BR-10 | ➕ NEW     | **Server time** sebagai source of truth untuk semua operasi waktu                 |
| BR-11 | ➕ NEW     | Akun wajib punya NIK terverifikasi; **1 NIK = 1 akun** secara global              |
| BR-12 | ➕ NEW     | Maksimal **5 NIK tersimpan** per akun; entri hanya lahir dari validasi Dukcapil yang berhasil |
| BR-13 | ➕ NEW     | Dukcapil **fail-closed**: NIK belum terverifikasi + Dukcapil tidak dapat dihubungi → booking diblokir |
| BR-13a | ➕ NEW     | Breaker Dukcapil atau payment gateway OPEN → **admission di-pause**; tidak ada grant baru dan tidak ada posisi antrian yang hilang |
| BR-14 | ➕ NEW     | Grace **30 detik** setelah TTL Pembayaran; stok tetap di-hold → perjalanan maksimum **15,5 menit** |
| BR-15 | ➕ NEW     | Grace menutup **latensi callback**, bukan pembayaran terlambat — batas bayar ke gateway tetap 10 menit |
| BR-16 | ➕ NEW     | Kapasitas antrian **300.000** per event; join di atas itu ditolak `QUEUE_FULL`    |
| BR-17 | ❌ REJECTED | ~~TTL antrian absolut (Session TTL Antrian)~~ — posisi hanya dilepas oleh grace setelah slot diberikan |
| BR-18 | ❌ REJECTED | ~~Push notification giliran tiba~~ — tidak dibangun di POC                        |
| BR-20 | ❌ REMOVED | ~~Grace antrian 120 detik antara GRANT dan klaim~~ — jam booking mulai saat slot diberikan |
| BR-19 | ➕ NEW     | Refund Disputed **tidak otomatis** di POC — ditandai, dilaporkan, dieksekusi manual |
| BR-21 | ➕ NEW     | Sold out bersifat **sementara**: kursi dari booking kedaluwarsa dijual kembali pada posisi antrian yang sama. Final setelah 930 detik |
| BR-22 | ➕ NEW     | Pause admission menambah **60 detik** (`PAUSE_TTL_GRACE_SECONDS`) ke tenggat booking dan pembayaran yang sedang berjalan |

---

## 5. Architecture Decisions (Locked)

| Aspect               | Decision                                                                                       |
| -------------------- | ---------------------------------------------------------------------------------------------- |
| Transport            | **Server-Sent Events (SSE)** — ECS Fargate mengelola koneksi persisten                        |
| Position calculation | Di **frontend**: `posisi = ticket_no - now_serving`                                            |
| Broadcast value      | Server push `now_serving` (global, identik semua user) via SSE                                 |
| Fallback interval    | Dinamis: 10s (>1000) → 5s (101–1000) → 2s (11–100) → 1s (1–10) — untuk REST fallback `/queue/{ev}/serving` saat SSE tidak tersedia |
| Admission formula    | `canEnter = (freeSlots > 0) AND (availableTickets > 0)`                                        |
| Time authority       | Server-side; `serverTime` di setiap payload SSE dan `server_time` di setiap respons REST — frontend menghitung offset jam sekali, tidak ada endpoint waktu tersendiri |
| Queue capacity       | **300.000 per event**; `QUEUE_FULL` fail-fast. Halaman utama menampilkan `status` + `queue_state` + `display` — lihat FR-03a..e |
| Payment grace        | **30 detik** `PENDING_CANCELLATION` setelah TTL Pembayaran; stok tetap di-hold, callback sukses masih diterima |
| Queue session        | **Tidak ada TTL antrian absolut dan tidak ada grace antrian**; slot yang di-grant tapi tidak diklaim direbut kembali oleh TTL Booking itu sendiri (maks 2× re-admit) |
| Anchor TTL           | Booking dari `granted_at`, pembayaran dari `held_at` — berurutan, maksimum **15,5 menit** |
| NIK verification     | **Fail-closed**; bukti verifikasi tersimpan (akun + Daftar NIK Teman + cache 24 jam) dipakai ulang untuk menekan biaya Dukcapil |
| Degradasi dependensi | Circuit breaker Dukcapil **atau** payment gateway OPEN (= dependensi sedang tidak dapat dipakai) → **pause admission**: tidak ada grant baru, antrian berhenti maju, tidak ada posisi yang hilang. Berlaku sama untuk kedua dependensi, tanpa pengecualian |
| TTL saat pause       | Tenggat yang sedang berjalan digeser **+60 detik** (`PAUSE_TTL_GRACE_SECONDS`, dapat dikonfigurasi) **saat pause dimulai** — bukan saat berakhir, karena sweep tetap berjalan dan yang kedaluwarsa di tengah pause sudah terlambat ditolong |
| Sold out             | **Reversibel** selama 930 detik: kursi yang dilepas booking kedaluwarsa dijual kembali pada posisi antrian yang sama. Sesudah itu `CLOSED`, stream ditutup `410 Gone` |
| Payment gateway      | **Midtrans Core API** — Bank Transfer VA (BCA/BNI/BRI/CIMB) dan E-Wallet GoPay. Kartu kredit dan OTC di luar cakupan: keduanya tidak dapat menghormati TTL 10 menit di sisi gateway |
| Proteksi bot         | Rate limit Redis per IP **lalu** CAPTCHA tak terlihat di join (urutan itu wajib), plus BR-07 untuk user login. Provider CAPTCHA mati → fail-open |
| Disputed refund      | Dicatat dan dilaporkan; **eksekusi manual** di POC |

---

## 6. Admission Controller

Komponen yang mengatur berapa user dari antrian yang diizinkan masuk ke Detail Ticket Page.

### Formula

```
canEnter = (freeSlots > 0) AND (availableTickets > 0)
```

| Variabel           | Definisi                                            |
| ------------------ | --------------------------------------------------- |
| `freeSlots`        | Jumlah slot yang sedang idle (tidak aktif diproses) |
| `availableTickets` | Jumlah tiket yang masih tersisa                     |

### Kapan Slot Dibebaskan

1. User menyelesaikan pembayaran (sukses)
2. TTL Booking habis (5 menit tanpa konfirmasi)
3. TTL Pembayaran habis (10 menit) **+ grace 30 detik** — stok dilepas hanya setelah grace,
   bukan saat TTL habis
4. User secara eksplisit membatalkan
5. Slot diberikan tetapi user tidak pernah membuka Detail Ticket Page sampai TTL Booking
   habis — **tidak ada jendela grace terpisah**; jam 5 menit sudah berjalan sejak slot
   diberikan (maksimum dua kali re-admit)

> **Tidak ada pelepasan karena TTL antrian.** Menutup tab tidak melepas posisi; yang melepas
> hanyalah kelima kondisi di atas. Lihat BR-17.

### Kapasitas Antrian

Antrian dibatasi **300.000 posisi per event**. Join di atas batas ditolak `QUEUE_FULL` secara
fail-fast — tanpa buffering dan tanpa memberi nomor antrian yang tidak akan pernah terlayani.
State antrian (`NOT_OPEN` / `DIRECT` / `OPEN` / `FULL`) ditampilkan di halaman utama agar customer melihat
antrian penuh sebelum mengklik.

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
    ├── SSE stream: GET /queue/{event}/stream → server push { now_serving, paused, serverTime }
    ├── Frontend hitung: posisi = ticket_no - now_serving
    ├── Tampilkan posisi & estimasi waktu tunggu
    │
    ├── Saat menerima event "grant" (server push saat giliran tiba):
    │   └── Frontend hit: GET /queue/{event}/me
    │       ├── Sukses → access_token → redirect ke Detail Tiket
    │       └── Tiket habis → tampilkan "Tiket Habis"
    │
    ├── Fallback (SSE tidak tersedia): GET /queue/{event}/serving → { now_serving }
    │   interval adaptif: 10s (>1000) → 5s (101–1000) → 2s (11–100) → 1s (1–10)
    │
    ↓
[Detail Ticket Page]
    │
    ├── Akses dijaga: access_token harus valid
    ├── Pilih kategori & jumlah tiket (max 3)
    ├── Pilih NIK per tiket:
    │   ├── NIK Terdaftar (akun)      → tanpa panggilan Dukcapil
    │   ├── Daftar NIK Teman (maks 5) → tanpa panggilan Dukcapil
    │   └── Ketik manual              → Dukcapil; gagal/timeout = booking diblokir
    ├── Countdown TTL Booking (5 menit sejak slot diberikan, server-synced)
    │
    ↓
[Payment Page]
    │
    ├── Pilih metode pembayaran (VA / GoPay)
    ├── Countdown TTL Pembayaran (10 menit)
    ├── Polling status pembayaran
    ├── Ganti metode → batalkan transaksi lama ke gateway, buat yang baru
    │
    ├── Sukses → release slot → now_serving++ → tiket diterbitkan via email
    │
    └── TTL habis → Pending Cancellation (grace 30 detik, stok MASIH di-hold)
            ├── callback sukses tiba → booking CONFIRMED
            └── grace berlalu        → booking batal, stok dilepas, email kegagalan
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
- Akun opsional dengan NIK terdaftar dan Daftar NIK Teman (maksimal 5) untuk menekan
  panggilan Dukcapil
- Grace period 30 detik setelah TTL Pembayaran agar callback yang terlambat tidak
  menghanguskan pembayaran yang sah
- Pelaporan booking Disputed untuk rekonsiliasi

### Di Luar Lingkup (Out of Scope)

- Pembuatan/manajemen event oleh admin (back-office)
- Implementasi internal mitra pembayaran (via payment gateway pihak ketiga)
- Program loyalti, diskon, dan voucher
- Sistem gate check-in di lokasi event
- **Push notification** (FCM/APNs) untuk giliran tiba atau posisi mendekati — email dan
  halaman Waiting Room adalah satu-satunya kanal pemberitahuan di POC
- **Refund otomatis** untuk booking Disputed — ditandai dan dilaporkan, dieksekusi manual
  lewat dashboard gateway
- **TTL antrian absolut** (Session TTL Antrian) — ditolak, lihat BR-17
- Nomor bangku (seat numbering) — kategori saja, lihat FR-27

---

## 9. Aktor & Peran

| Aktor                | Tipe             | Peran                                                                |
| -------------------- | ---------------- | -------------------------------------------------------------------- |
| Customer             | Manusia          | Memilih event, mengantre, memilih tiket, dan membayar                |
| Queue Service        | Sistem           | Mengatur antrian virtual, token akses, dan posisi antrian            |
| Booking Service      | Sistem           | Mengelola stok tiket, reservasi sementara (hold), dan status booking |
| Payment Gateway      | Sistem Eksternal | Memproses pembayaran dan mengirim callback status                    |
| Notification Service | Sistem           | Mengirim email e-tiket atau pemberitahuan booking expired            |
| Mobile App           | Sistem Klien     | Mengonsumsi API pembayaran yang sama dengan web                      |

---

## 10. Istilah

| Istilah             | Definisi                                                                             |
| ------------------- | ------------------------------------------------------------------------------------ |
| Waiting Room        | Ruang tunggu virtual tempat customer menunggu giliran masuk ke halaman pembelian     |
| Access Token / Slot | Hak akses sementara yang diberikan kepada customer untuk masuk ke Detail Ticket Page |
| Hold / Reservasi    | Penguncian stok tiket sementara saat customer berhasil masuk ke halaman payment      |
| TTL Booking         | Batas waktu memilih tiket — 5 menit **sejak slot diberikan**, bukan sejak halaman dibuka |
| TTL Pembayaran      | Batas waktu untuk menyelesaikan pembayaran (10 menit)                                |
| NIK                 | Nomor Induk Kependudukan (16 digit) — basis pembatasan 1 seat per NIK                |
| `now_serving`       | Nilai global yang menunjukkan nomor antrian yang sedang dilayani                     |
| `ticket_no`         | Nomor antrian unik customer, diberikan saat join queue                               |
| NIK Terdaftar       | NIK yang terikat permanen pada satu akun, diverifikasi Dukcapil saat registrasi       |
| Daftar NIK Teman    | Sampai 5 NIK teman tersimpan di profil akun, masing-masing sudah terverifikasi Dukcapil saat ditambahkan; dipakai ulang di checkout tanpa panggilan baru |
| Persetujuan Data Pribadi | Persetujuan eksplisit saat registrasi untuk memproses NIK pihak ketiga (UU PDP No. 27/2022); wajib, tidak pre-checked, pencabutan lewat tim operasional |
| Consent Record      | Rekam immutable bukti persetujuan: timestamp, versi kebijakan, identitas akun         |
| Grace Period        | 30 detik setelah TTL Pembayaran habis; callback sukses masih diterima, stok masih di-hold |
| Pending Cancellation | Status transisional selama Grace Period — belum batal, belum terkonfirmasi           |
| Disputed            | Callback "berhasil" tiba setelah booking dibatalkan final; perlu rekonsiliasi dan refund manual |
| Kapasitas Antrian   | Batas maksimum posisi antrian per event (300.000); kelebihan ditolak `QUEUE_FULL`     |
| State Antrian       | `queue_state`: `NOT_OPEN` / `DIRECT` / `OPEN` / `FULL` per event — sumbu izin masuk    |
| CAPTCHA Tak Terlihat | Verifikasi berbasis skor tanpa teka-teki; token diambil sebelum sale buka agar tidak menambah waktu manusia ke dalam perlombaan `INCR issued` |
| Keterlibatan Aktif  | Satu posisi antrian atau booking yang belum terminal; dasar penegakan BR-07 untuk customer login |
| Status Event        | `status`: `AVAILABLE` / `SELLING_OUT_FAST` / `SOLD_OUT` — sumbu stok, terpisah dari antrian |
| Sold Out Sementara  | Stok nol tetapi kursi masih dapat kembali dari booking yang kedaluwarsa; stream SSE tetap terbuka dan antrian tidak kehilangan posisinya |
| Sale Ditutup        | Keadaan terminal setelah 930 detik tanpa kemungkinan kursi kembali; stream ditutup dan koneksi baru ditolak `410 Gone` |
| Pause TTL Grace     | Tambahan 60 detik pada tenggat booking dan pembayaran yang sedang berjalan, diberikan saat admission di-pause agar gangguan sistem tidak memakan waktu customer |
| Display Block       | Jawaban siap-render dari server (`enabled`, `action`, `label_key`, `badge_key`); frontend tidak menghitung presedensi |
| CAS                 | *Compare-And-Swap* — mengubah nilai hanya bila ia **masih** sama dengan yang dibaca, dalam satu operasi atomik; pihak yang kalah balapan mendapat 0 baris dan berhenti sebelum efek sampingnya, sehingga dua proses bersamaan tetap aman tanpa lock |

---

## 11. Ringkasan Perubahan dari Tender Asli

| Poin          | Jenis      | Ringkasan                                                 |
| ------------- | ---------- | --------------------------------------------------------- |
| Login         | ✏️ UPDATED | Dari implisit wajib → **opsional** (dikonfirmasi CEO)     |
| BR-01         | ✏️ UPDATED | Kuota 3 tiket: dari per-NIK kumulatif → **per transaksi** |
| BR-02         | ✏️ UPDATED | Kuota NIK: dari 3 tiket per NIK → **1 NIK per seat**      |
| BR-07         | ✔ RESTORED | Satu keterlibatan aktif per event, user login saja — alasannya okupansi slot, bukan scalping |
| BR-09         | ➕ NEW     | Re-entrant user dalam TTL kembali ke halaman yang sama    |
| BR-10         | ➕ NEW     | Server time sebagai source of truth                       |
| FR-26..28     | ➕ NEW     | Email ke pemegang NIK, bukan hanya pembeli                |
| FR-29..32     | ➕ NEW     | Login opsional, verifikasi via link, multi-bahasa         |
| FR-33..35     | ➕ NEW     | Dashboard operasional real-time                           |
| Transport     | 🔧 DECIDED | **Server-Sent Events (SSE)** via ECS Fargate — REST polling tetap sebagai fallback (jaringan blokir SSE / degraded mode) |
| Position calc | 🔧 DECIDED | Di frontend (`ticket_no - now_serving`)                   |
| Fail-closed   | 🔧 DECIDED | Dukcapil gagal/timeout → booking diblokir (BR-13)         |
| Pause         | 🔧 DECIDED | Breaker Dukcapil / gateway OPEN → pause admission (BR-13a) |
| Grace antrian | ❌ REMOVED | Jam booking mulai saat slot diberikan (BR-03, BR-20)      |
| Sold out      | 🔧 DECIDED | Reversibel 930 detik, lalu `CLOSED` (BR-21)              |
| TTL saat pause| 🔧 DECIDED | +60 detik saat pause mulai, dapat dikonfigurasi (BR-22)  |
| Event status  | 🔧 DECIDED | Dua field + `display` server-side (FR-03a..e)             |
| Multi-kategori| 🔧 DECIDED | Satu transaksi boleh lintas kategori, all-or-nothing (FR-13a..b) |
| Gateway       | 🔧 DECIDED | Midtrans Core API — dua metode (VA + GoPay)              |
| CAPTCHA       | 🔧 DECIDED | Tak terlihat, di join antrian, fail-open (FR-10d..f)      |

### Tambahan v1.1 — usulan tim 25 Oktober 2026, butir 1–8

| # | Fitur | Jenis | Masuk sebagai |
| - | ----- | ----- | ------------- |
| 1 | Daftar NIK Teman (maks 5, terverifikasi saat ditambahkan) | ➕ NEW | FR-32a..d, BR-12 |
| 2 | NIK Terdaftar per akun, unik global | ➕ NEW | FR-31a..b, BR-11 |
| 3 | Persetujuan data pribadi + consent record immutable | ➕ NEW | FR-31c..d, NFR-11 |
| 4 | TTL antrian absolut (Session TTL Antrian) | ❌ REJECTED | BR-17 — grace setelah slot sudah menutupinya |
| 5 | Grace period 30 detik + Pending Cancellation + Disputed | ➕ NEW | FR-23a..c, FR-36, BR-14..15, BR-19 |
| 6 | Email pemberitahuan kegagalan pembayaran | ➕ NEW | FR-28a..b |
| 7 | Push notification giliran tiba | ❌ REJECTED | Out of scope — email + Waiting Room saja |
| 8 | Kapasitas antrian 300.000 + state antrian di halaman utama | ➕ NEW | FR-03a..b, FR-10a, NFR-13, BR-16 |

> **Butir 9 dan sesudahnya belum menjadi requirement.** Termasuk usulan mengganti SSE
> dengan adaptive polling, yang bertentangan dengan keputusan Transport di Bagian 5 dan
> akan membatalkan ADR-003 sampai ADR-005 beserta model biayanya. Keputusannya menunggu
> klarifikasi.


---

## 12. Asumsi

- Data event (jadwal, kategori, harga, kapasitas) tersedia dari sistem back-office
- Customer memiliki email valid untuk pengiriman e-tiket
- Mitra pembayaran menyediakan mekanisme callback status
- Validasi NIK dilakukan via Dukcapil API (**NIK saja** — nama dan tanggal lahir tidak dikumpulkan maupun dikirim)
- Target skala: 300.000 concurrent users pada flash crowd (NFR-03 mensyaratkan 200.000+)
- NIK yang sudah divalidasi Dukcapil dianggap tidak berubah statusnya; tidak ada revalidasi
  berkala atas NIK tersimpan
- Volume panggilan Dukcapil **tidak dapat diturunkan dari asumsi adopsi login**: seluruhnya
  tamu ≈ 10.000 panggilan per 10.000 tiket, seluruhnya login dengan NIK tersimpan ≈ nol.
  Rentang ~5×; estimasi tender memakai skenario tamu-berat dan menyajikan penghematan NIK
  tersimpan sebagai potensi (NFR-12)
- Mitra pembayaran (Midtrans) menyediakan pembatalan transaksi aktif, sehingga penggantian
  metode pembayaran tidak meninggalkan dua tagihan hidup untuk satu booking

---

## 13. Referensi Dokumen

| Dokumen                           | Path                                                       |
| --------------------------------- | ---------------------------------------------------------- |
| Tender asli                       | `requirement/Tender Rusuh Booking Ticket.pdf`              |
| Update requirement                | `requirement/update-requirement-rusuh-booking-tiket.md`    |
| Meeting CTO POV                   | `discussions/2026-10-16-meeting-business-needs-cto-pov.md` |
| Meeting Queue Algorithm           | `discussions/2026-10-21-meeting-notes-queue-algorithm.md`  |
| ADR-001 Queueing Technology       | `adr/ADR-001-queueing-technology.md`                       |
| ADR-002 Queue Position Mechanism  | `adr/ADR-002-queue-position-mechanism.md`                  |
| ADR-003 Realtime Queue Updates    | `adr/ADR-003-realtime-queue-updates.md`                    |
| ADR-004 Compute Service Selection | `adr/ADR-004-compute-service-selection.md`                 |
| ADR-005 Final Architecture        | `adr/ADR-005-final-architecture-decision.md`               |
| API Contracts                     | `api/api-contracts.md`                                     |
| Metode Pembayaran (Midtrans)      | `payment-method/README.md`                                 |
