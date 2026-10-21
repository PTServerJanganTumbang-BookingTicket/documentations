# Business Needs - CTO POV
## Rusuh Tiket Booking
**Sumber:** Interview Alignment Requirements — Shadrach (CTO/CEO)
**Tanggal:** 16 Oktober 2026

---

## 1. Transparansi & Anti-PHP (No False Promise)

Prinsip utama yang paling ditekankan CTO: **sistem harus jujur kepada user**.

- Jika tiket tersedia, user bisa membeli. Jika sudah sold out hari ini, tampilkan sold out.
- Apa yang terlihat di sistem harus benar-benar mencerminkan kondisi aktual — available ya available, sold out ya sold out.
- Jangan sampai memberikan harapan bahwa tiket masih available, tapi saat masuk antrian ternyata antrian sudah penuh.
- Yang sering terjadi dan tidak diinginkan: customer mendapat error, padahal sebenarnya tiketnya sudah tidak ada.

---

## 2. Manajemen Antrian (Queue Management)

- Jika ada maksimum kapasitas antrian, harus ditegakkan. Jika antrian penuh, lebih baik langsung gagal (fail fast).
- Queue bersifat per-event — tiap event memiliki antrian sendiri.
- Di wireframe ada estimasi waktu tunggu. Harapannya angka tersebut adalah estimasi yang bisa lebih cepat, bukan lebih lambat — jika janji 15 menit tapi ternyata 20 menit, itu termasuk PHP.
- User boleh membatalkan nomor antrian dan bisa mengantri ulang, tapi dengan nomor baru yang lebih belakang.

---

## 3. Mekanisme Pembayaran & Locking Tiket

- Waktu untuk membayar adalah 10 menit.
- Jika user tidak membayar dalam 10 menit, tiket harus bisa dibuka lagi. Tapi pastikan user benar-benar sudah melewati 10 menit sebelum slot dibuka untuk orang lain.
- Jangan sampai saat user sedang ingin membayar, tiket yang sedang dipegang malah dibuka untuk orang lain.
- Ketika user sudah masuk ke halaman pembayaran (payment page), hanya satu NIK yang bisa mengakses payment page tersebut. Orang lain tidak bisa masuk ke halaman yang sama.
- Sampai user tidak membayar selama 10 menit atau menyatakan tidak mau membayar, barulah waktu itu lepas dan orang baru bisa masuk ke proses pemesanan.

---

## 4. Identitas User & Registrasi

- Yang penting user bisa booking. Syaratnya adalah memiliki KTP — user auto-teregistrasi dengan memasukkan NIK.
- Form pemesanan perlu ditambahkan field NIK dan email di awal.
- Saat login/buat akun, pastikan email terverifikasi dengan benar.
- Metode verifikasi email yang lebih baik adalah via verification link (klik link di email), bukan OTP — lebih simpel untuk user.
- Minimal dua bahasa yang harus ada: Indonesia dan Inggris.

---

## 5. Aturan Tiket per NIK (Anti-Calo)

- Satu bangku diikat ke satu NIK dan satu unit. Prinsipnya: satu bangku = satu NIK = satu event.
- Alasan utama aturan ini: menghindari praktik percaloan (scalping).
- Satu NIK bisa membeli maksimal 3 tiket pada satu event yang sama.
- Satu NIK hanya bisa menempati satu bangku. Jika NIK sudah terikat ke satu bangku, NIK tersebut tidak bisa membeli untuk dirinya lagi di event yang sama.
- User bisa membeli tiket untuk orang lain, tapi dia tidak bisa membeli tiket dengan NIK-nya sendiri lagi (karena NIK-nya sudah ada di satu bangku).

---

## 6. Informasi pada Tiket yang Diterbitkan

- Tiket cukup memuat QR code, nomor bangku, dan NIK. Tidak perlu informasi berlebihan.
- NIK yang tercantum di tiket adalah NIK orang yang akan duduk di bangku tersebut, bukan selalu NIK pembeli.
- Di gate event, penjaga akan membandingkan NIK di tiket dengan KTP yang dibawa, lalu mencocokkan dengan wajah. Jika cocok, diizinkan masuk.
- Email tiket dikirimkan ke masing-masing pemesan (jika beli 3 tiket, dikirim ke 3 orang).

---

## 7. Validasi NIK (Dukcapil)

- Sistem gate (validasi KTP di lokasi event) belum masuk scope. Penjaga gate akan mengecek tiket + NIK secara manual — jika sinkron, diizinkan masuk.
- Untuk real case, validasi NIK seharusnya dilakukan ke Dukcapil.
- Dukcapil memiliki SLA — maksimal 5–10 detik per request.
- Setiap validasi ke Dukcapil ada biayanya (cost per call).
- Validasi NIK sebaiknya dilakukan saat registrasi saja, agar cost lebih terkontrol.
- **Untuk keperluan POC, bagian validasi Dukcapil ini bisa di-skip dulu.**

---

## 8. Observability & Dashboard Operasional

- Ada permintaan dashboard. Jika real-time, harus berbentuk dashboard.
- Dari sisi CTO/CPO: ingin ada web dashboard dengan tampilan yang bisa dipantau.
- Key metrics yang harus tersedia secara real-time: **antrian, laju konversi, expiry rate**.
- Laju konversi = step-by-step dari antrian ke pembelian. Expiry rate = berapa banyak tiket yang tidak dibayarkan.
- CTO butuh web dashboard. Implementasinya bebas (bisa Grafana, custom, dll).

---

## 9. Multi-Device & Session Management

- Multi-device login diperbolehkan, tapi session-nya sama.
- Jika sedang menunggu pembayaran 10 menit di handphone, login di laptop akan menampilkan state yang sama — countdown 10 menit tetap berjalan dari server.
- Boleh login di banyak device, tapi maksimum pembelian tetap 3 tiket per event.

---

## 10. Metode Pembayaran

- Dari wireframe ada 3 pilihan pembayaran: virtual account, e-wallet, dan transfer bank.
- Rekomendasi menggunakan payment gateway/mitra pembayaran.
- Harus memperhatikan PCI-DSS compliance — tidak boleh menyimpan nomor kartu kredit secara langsung.
- User harus diinformasikan ke sistem pembayaran mana mereka akan terhubung, sehingga mereka tahu dan merasa aman.

---

## 11. Pengalaman Buruk yang Harus Dihindari (Pain Points)

Berdasarkan pengalaman CTO sebagai user:

- Pengalaman paling menyebalkan: sudah sampai di halaman pembayaran, berhasil bayar, tapi kemudian di-refund karena tiket sudah terjual. Artinya pembayaran terjadi tapi gagal mendapat tiket.
- Sistem harus memastikan jika user sudah berhasil masuk ke payment page dan membayar, tiket **pasti** tersedia untuknya.
- CTO ingin sistemnya digunakan oleh orang organik yang benar-benar manusia — bukan bot.

---

## Ringkasan Prioritas (dari sudut pandang CTO)

| # | Kebutuhan | Prioritas |
|---|-----------|-----------|
| 1 | Transparansi availability tiket (no PHP) | 🔴 Critical |
| 2 | Locking tiket saat di payment page | 🔴 Critical |
| 3 | 1 NIK = 1 bangku per event (anti-calo) | 🔴 Critical |
| 4 | Payment window 10 menit + auto-release | 🔴 Critical |
| 5 | Email verified + NIK di awal | 🟠 High |
| 6 | Real-time dashboard (antrian, konversi, expiry) | 🟠 High |
| 7 | Payment gateway dengan 3 metode | 🟠 High |
| 8 | Anti-bot mechanism | 🟠 High |
| 9 | Multi-device dengan shared session | 🟡 Medium |
| 10 | Validasi NIK ke Dukcapil | 🟡 Medium (skip untuk POC) |
| 11 | Multi-bahasa (ID + EN) | 🟡 Medium |
| 12 | Queue cancellation | 🟢 Nice to have |

---

## Catatan Open Items

Hal-hal yang masih perlu dikonfirmasi ke stakeholder lain:

1. **Login mandatory vs optional** — CTO memberi kebebasan ke tim untuk memutuskan, dengan catatan harus ada reasoning/justifikasi yang kuat saat presentasi.
2. **NIK teman di tiket** — apakah perlu input NIK tiap orang yang dibelikan, atau cukup NIK pembeli saja. Perlu konfirmasi ke level operasional.
3. **Jumlah concurrent payment gate** — CTO menyarankan satu gate per tiket (satu orang satu waktu di payment page), tapi implementasi detailnya diserahkan ke tim.
