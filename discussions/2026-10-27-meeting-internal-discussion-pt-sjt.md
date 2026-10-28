# Internal Discussion PT Server Jangan Tumbang

**Tanggal:** 26 Oktober 2026  
**Waktu:** 16.00 – 16.30 WIB  
**Fasilitator:** mochamad_r_ridho  
**Topik:** Business Requirement Rusuh War Tiket Online  
**Sumber:** Internal_Discussion_PT_Server_Jangan_Tumbang.pdf (Microsoft Loop export)

> ⚠️ Konten dihasilkan sebagian oleh AI Loop Copilot — beberapa poin mungkin perlu dikonfirmasi terhadap transkrip asli.

---

## Keputusan (D)

| ID | Keputusan |
|----|-----------|
| D1 | Satu akun dapat menyimpan hingga **6 NIK** (1 NIK pemilik + maks 5 NIK teman) |
| D2 | Penyimpanan NIK teman **wajib melalui consent** saat proses registrasi; setelah consent diberikan, NIK langsung dapat dipilih di detail tiket tanpa input ulang |
| D3 | Kapasitas antrean ditetapkan hingga **300.000 pengguna**; jika melebihi kapasitas, sistem menampilkan notifikasi "antrean penuh" dan tidak mengizinkan pengguna baru masuk |
| D4 | Pengguna yang sudah melakukan pembayaran mendapat **grace period 30 detik**, termasuk saat status masih `pending` atau `pending cancellation` |
| D5 | Transaksi dianggap **gagal** apabila payment gateway mengirim callback gagal, atau tidak ada callback hingga batas waktu sekitar **5–10 menit** (perlu diselaraskan dengan SLA mitra ~3 menit) |
| D6 | **Refund otomatis out of scope** untuk saat ini; penanganan dilakukan melalui **admin/back office** dengan dashboard manual |
| D7 | Setiap percobaan pembayaran memiliki **Payment ID tersendiri** meskipun terikat pada booking yang sama, agar transaksi lama dan pengganti dapat dilacak terpisah |
| D8 | Infrastruktur disediakan pada **2 server / minimal 2 instance** untuk high availability dan kemudahan scale-up |

---

## Open Items / Pertanyaan Terbuka (O)

| ID | Pertanyaan | Konteks |
|----|-----------|---------|
| O1 | **Grace period idle di antrean** — berapa lama? | Diskusi menyebut ~2 menit; belum final. Pengguna idle perlu diterminasi atau diarahkan ulang |
| O2 | **Alur penggantian metode pembayaran** | Jika VA Mandiri gagal lalu user ingin pindah ke VA BRI, apakah cukup dalam konteks booking sama atau harus transaksi baru? |
| O3 | **Definisi dan trigger final email kegagalan** | Kapan tepatnya email dikirim — saat callback gagal diterima atau setelah timeout? |
| O4 | **SSE vs polling** — keputusan metode sinkronisasi belum final | Perhitungan biaya (cost model/spreadsheet di GitHub) masih perlu divalidasi; FE perlu tahu mekanisme ini agar requirement bisa dikunci |
| O5 | **TTL virtual account dan QRIS/GoPay** — mekanisme kedaluwarsa tiap metode pembayaran belum dipastikan | Contoh durasi: 5 atau 10 menit |
| O6 | **Batas waktu konfirmasi payment gateway** — belum selaras dengan SLA mitra | Diskusi menyebut 3 menit (SLA mitra) vs 5–10 menit; perlu keputusan final |
| O7 | **UI pemilihan NIK teman** — pilihan otomatis + fitur pencarian masih dalam pembahasan | Mencakup apakah NIK yang belum terdaftar bisa ditambah via input manual |
| O8 | **Estimasi posisi/waktu tunggu ketika antrean penuh** — apakah ditampilkan ke user? | Memperjelas UX ketika queue full |

---

## Ringkasan per Topik

### 1. Registrasi Pengguna & Consent Data (PDP)
- Satu akun: 1 NIK pemilik + maks 5 NIK teman = total **6 NIK**.
- NIK teman yang belum terdaftar dapat ditambahkan via input manual (UX masih dibahás).
- Penyimpanan NIK teman wajib melalui **consent record** saat registrasi.
- Setelah consent diberikan, NIK tersimpan dapat langsung dipilih di halaman detail tiket.

### 2. Manajemen Antrean
- Kapasitas antrean: **300.000 pengguna**. Jika penuh → sistem blok user baru masuk + tampilkan pesan.
- Pengguna yang tidak aktif (idle) perlu ditangani: diberi waktu ~2 menit, lalu diterminasi atau diarahkan ulang *(belum final, O1)*.
- Status tiket harus membedakan: **tersedia / habis / ditutup** agar user tidak terus mengantre sia-sia.
- Ketika sebagian pengguna keluar, kapasitas kembali tersedia → user berikutnya bisa masuk.
- UX perlu menampilkan estimasi posisi/waktu tunggu ketika antrean penuh *(O8)*.

### 3. Polling & Sinkronisasi Status
- Metode sinkronisasi (**SSE vs polling**) belum diputuskan *(O4)*.
- Polling adaptif dipertimbangkan dengan interval 10 detik, 30 detik, atau 1 menit.
- Risiko: perangkat sleep → backend tidak menerima update → user terlihat masih antre meski tidak aktif.
- Keputusan metode ini **blocker** untuk requirement sisi FE.

### 4. Pembatalan Antrean
- Pengguna harus bisa membatalkan antrean secara eksplisit.
- Pembatalan harus mengirim sinyal ke backend agar posisi antrean berkurang (bukan hanya maju secara natural).

### 5. Pembayaran & Grace Period
- **Grace period 30 detik** setelah pembayaran, termasuk saat status `pending` / `pending cancellation`.
- Payment gateway bisa terlambat merespons → status sementara tampil sebagai `pending`, bukan langsung sukses.
- **Timeout kegagalan:** ~5–10 menit (perlu selaras SLA mitra ~3 menit) *(O6)*.
- TTL virtual account dan QRIS/GoPay masih perlu dipastikan *(O5)*.

### 6. Notifikasi & Email Pembayaran
- Alur notifikasi: `pending` dulu → email lanjutan setelah konfirmasi sukses.
- Email kegagalan dikirim ketika: callback gagal **atau** timeout tanpa callback.
- Definisi dan trigger final email masih perlu dikunci *(O3)*.

### 7. Refund Pembayaran
- **Refund otomatis: out of scope** (skenario diprioritaskan ke penanganan admin).
- Jika pembayaran sukses di internal tapi `pending` 30 detik lalu gagal di gateway:
  - Metode di luar ekosistem terintegrasi → dana ke **wallet**, user tarik manual.
  - Ekosistem terintegrasi (GoPay, Tokopedia) → refund langsung ke saldo/metode terkait.
- Akan disediakan **dashboard back office** untuk proses refund manual.

### 8. Penggantian Metode Pembayaran
- Skenario: VA Mandiri gagal → user ingin pindah ke VA BRI.
- Durasi TTL yang lebih panjang (~10 menit) dapat mengakomodasi skenario ini tanpa harus membuat transaksi baru yang tidak terlacak.
- Setiap percobaan pembayaran tetap punya **Payment ID sendiri** dalam booking yang sama *(D7)*.
- Alur penggantian metode pembayaran masih perlu dikunci *(O2)*.

### 9. Biaya Infrastruktur (SSE vs Polling)
- Perbandingan biaya SSE vs polling mengacu pada cost model/spreadsheet di GitHub.
- Biaya dipengaruhi oleh **jumlah request/hit**, bukan hanya volume data.
- Model polling bisa lebih mahal saat frekuensi request tinggi; SSE punya biaya koneksi tetap.
- Perhitungan perlu divalidasi ulang sebelum dijadikan acuan *(O4)*.
- Skala 300.000 pengguna jauh lebih mahal dari 20.000 pengguna — perlu dihitung ulang di titik break-even.

### 10. Kapasitas & Skalabilitas
- Acuan awal: ~20.000 koneksi; requirement target: 300.000 pengguna.
- Deployment: **2 server / minimal 2 instance** untuk HA dan scale-up *(D8)*.

---

## Keterkaitan dengan Gap Analysis Existing

| Gap ID | Relevansi |
|--------|-----------|
| GAP-C4 / GAP-O9 | O4 mempertegas bahwa keputusan SSE vs polling **masih terbuka** — kontradiksi ADR-002 (FCM/APNs) vs QUEUE doc ("pure polling") belum resolved |
| GAP-O4 | D3 mengkonfirmasi batas antrean 300.000 dan perilaku "full queue" |
| GAP-O6 | O1 memperdalam isu idle user — grace period 2 menit di antrean belum ada di dokumen existing |
| GAP-O7 | O6 memperjelas TTL payment gateway (5–10 menit vs SLA mitra 3 menit) — perlu keputusan satu angka |
| — | D1 (maks 6 NIK/akun) adalah aturan **baru** yang belum ada di requirements.md maupun ADR existing |
| — | D6 (refund out of scope) dan D7 (Payment ID per attempt) adalah aturan **baru** — perlu ditambahkan ke requirements |
