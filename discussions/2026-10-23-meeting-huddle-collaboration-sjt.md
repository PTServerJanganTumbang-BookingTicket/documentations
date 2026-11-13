# Hasil Meeting — Huddle Collaboration PT. SJT

- **Tanggal:** Jumat, 23 Oktober 2026
- **Durasi:** ~2 jam 37 menit (06:40 WIB, auto-transcribed)
- **Proyek:** Rusuh Booking Tiket (codename *Rusuh Online* — War Ticketing)
- **Peserta (dari transkrip):** akbar_t_hartomo, Handoko, Nur (CFO), Dinda (CEO), Bayu, Sugeng/Tukul (referensi meeting sebelumnya), tim FE & BE.

> ⚠️ **Catatan sumber:** transkrip dihasilkan oleh auto-transcription, jadi beberapa nama/istilah tidak akurat dan banyak obrolan non-teknis. Ringkasan ini mengekstrak **keputusan & poin requirement** yang substantif; frasa yang ambigu ditandai *(perlu konfirmasi)*.

---

## TL;DR — Poin Kunci Bisnis

1. **Prinsip bisnis utama (CFO/Nur):** *yang penting tiket habis terjual.* Risiko kesalahan input ditanggung user, bukan mengganjal transaksi — **KECUALI** validasi NIK (lihat #3).
2. **Validasi NIK ke Dukcapil = HARD GATE.** Jika Dukcapil gagal/`timeout`/NIK tidak valid → tidak boleh lanjut ke payment. Ini satu-satunya titik yang memblokir happy flow.
3. **Kuota: 1 NIK = 1 seat; maksimal 3 tiket per transaksi** (butuh 3 NIK berbeda).
4. **Nama ikut dicantumkan di tiket** (diinput user, bukan diambil dari Dukcapil) — untuk pencocokan KTP↔tiket oleh petugas di venue. Validasi Dukcapil hanya mengecek NIK valid/tidak.
5. **Nomor seat/bangku di-generate otomatis** (tidak pilih manual), dan untuk 1 transaksi multi-tiket diusahakan **berderet/bersebelahan**.
6. **Real-time antrian: pakai polling (adaptive), bukan SSE** untuk antrian; SSE/hybrid hanya opsi fallback jika skala naik. Keputusan diperkuat oleh perhitungan biaya.
7. **Simpan NIK/nama hanya untuk user teregistrasi, dengan user consent eksplisit** saat registrasi + saat menambah penumpang baru.

---

## 1. Model Identitas, NIK & Validasi Dukcapil

### Keputusan
- **Kuota:** 1 NIK = 1 seat. Satu transaksi booking boleh 3 NIK → 3 tiket. (menegaskan arah di requirement terbaru).
- **Validasi Dukcapil wajib & blocking:**
  - Trigger validasi **setelah user selesai input NIK** (via tombol/aksi, bukan on-keystroke) untuk menghemat jumlah hit ke Dukcapil.
  - Jika gagal / `timeout` (mis. 3x retry habis) / NIK invalid → field di-*disable*, **tidak bisa lanjut ke payment**. "Harus valid."
  - Konsensus akhir: NIK **harus valid** sebelum submit — berbeda dari ide awal "loloskan saja demi tiket terjual".
- **Nama di tiket:**
  - Nama **diinput user** dan ikut tercetak di e-tiket (dipakai petugas venue mencocokkan KTP ↔ tiket; tiket pakai QR code).
  - Sistem **tidak mengambil nama dari Dukcapil** (hindari menyimpan/menarik data pribadi lebih dari perlu). Validasi Dukcapil cukup cek NIK valid/invalid.
  - *(Perlu konfirmasi)* apakah nama diverifikasi harus cocok dengan data Dukcapil, atau cukup free-text. Ada kekhawatiran salah ketik nama (NIK valid tapi nama beda) → arah: tetap loloskan jika NIK valid, nama tanggung jawab user.

### Caching NIK (diusulkan, belum final)
- Ide Handoko: **simpan NIK yang sudah pernah tervalidasi** di DB (NIK ~tetap sepanjang hidup) agar tidak hit Dukcapil berulang → hemat biaya.
- Keberatan: menyimpan NIK orang lain = isu **PDP/compliance**. Disepakati hanya boleh untuk:
  - User **teregistrasi** yang menyimpan NIK dirinya + NIK orang lain yang "dititipkan", **dengan consent**.
  - Guest / non-login → **selalu validasi ulang ke Dukcapil** (tidak di-cache).

### Open / perlu diputuskan
- Perjanjian/kontrak penggunaan **mock Dukcapil vs Dukcapil asli** (untuk dev pakai mock, disepakati di awal).
- Proses **refund** bila terjadi salah input yang lolos — diputuskan **out of scope** teknis utama, ditangani back office / manual (dokumen). Masih perlu journey jelas.

---

## 2. Login, Registrasi & Passenger List

### Keputusan
- **Login opsional** (ada mode non-login/guest). Login pakai **magic link via email** (OTP/one-time link).
- **User teregistrasi** dapat:
  - Menyimpan **NIK + nama** diri sendiri dan **daftar penumpang lain** ("passenger list" di profil, mirip Traveloka).
  - Saat booking tinggal **pilih dari passenger list** (autofill), tanpa ketik ulang — NIK yang sudah pernah divalidasi tidak perlu re-validate.
  - Menambah passenger baru → **harus validasi Dukcapil + consent (TnC)**.
- **Consent/PDP:** saat registrasi user harus centang consent bahwa NIK & nama disimpan; consent kedua saat menambah passenger baru.

### Kontrak API login yang dibahas
- Response magic link: envelope **`{ meta, data, message }`**.
  - `meta`: `code` (200/201/400/409/429...), `message` (success/error).
  - `data`: objek (mis. berisi `link`); bila tidak ada data → objek kosong `{}` / `0`.
  - `message`: untuk pesan error ditaruh di sini (satu pesan, bukan list — field-level error tidak dipakai).
- **Expired & resend:** magic link punya masa berlaku (ada `expired_in`); sediakan mekanisme **resend/kirim ulang** (tombol resend saat countdown habis).
- **Standar HTTP status code** mengikuti standar partner **TL** (referensi kontrak): 200/201, 400/401/404/409/429 dsb. Pakai yang mandatory.

### Open
- Halaman **registrasi** belum ada di Figma (perlu ditambah: nama, NIK, consent).
- Perlu keputusan field metadata standar di envelope (align dengan IDP/standar TL).

---

## 3. Seat / Nomor Bangku

- **Auto-generate**, tidak ada pemilihan manual bangku (mengurangi kompleksitas).
- Untuk multi-tiket dalam 1 transaksi → seat **diusahakan berderet** (mis. A1, A2, A3) bukan terpisah-pisah, agar rombongan duduk berdekatan.
- Tipe tiket / kategori: Festival, Tribun, VIP, dll. *(daftar final perlu dikonfirmasi)*. VIP biasanya seat terbaik; festival umumnya berdiri (seat kurang relevan).
- Figma **belum final** → **belum bisa jadi kontrak** sampai detail seat & field disepakati.

---

## 4. Antrian (Queue) & Real-time Update

### Keputusan arah
- **Mekanisme real-time antrian: polling (adaptive interval), bukan SSE.**
  - Interval dinamis: melebar/menyempit **1 → 2 → 4 → 8 detik** tergantung posisi; makin dekat giliran makin cepat (hingga ~1 detik).
  - Latensi telat 1–3 detik di antrian **dapat diterima** ("tidak harus real-time ketat").
- **SSE / hybrid = opsi fallback** untuk jangka panjang bila skala naik; implementasi bisa di-*switch* lewat feature flag/config. Opsi lain yang disebut: managed WebSocket/pub-sub (AppSync-style) — lebih mahal.
- **Sisa tiket (stock) di halaman detail/booking** juga butuh update berkala — kemungkinan endpoint streaming/polling terpisah. *(Belum final apakah SSE atau polling.)*

### NFR terkait
- **NFR-0006 (latensi):** P95 pembaruan posisi antrian **< 1 detik** untuk skala **±300.000** concurrent *(interpretasi: ini soal response time endpoint, bukan wajib push real-time)*. Perlu klarifikasi definisi "latensi" (response time FPI vs frekuensi perubahan angka antrian).
- **NFR-0009 (observability):** metric antrian, **laju konversi**, dan **rate expired** dipantau real-time di dashboard.

### Open
- Definisi tegas "di bawah 1 detik" (apakah response time atau freshness).
- CEO/CFO sempat minta "real-time"; tim teknis klarifikasi polling tetap memenuhi "real-time cukup". Perlu sign-off akhir dari C-level.

---

## 5. Biaya Infrastruktur — SSE vs Polling (ringkasan hitungan)

Perhitungan kasar yang dibahas (asumsi 1 event, ~300.000 connection, durasi ~2 jam):

| Pendekatan | Komponen | Estimasi |
|---|---|---|
| **SSE (ECS)** | ~20 service, 0.5 vCPU / 2GB; biaya nyala ~**$14/event** + ~$475 bila 20 task selalu nyala | mahal untuk maintenance connection terus-menerus; ECS tidak bisa scale-to-zero |
| **Polling (Lambda + CDN)** | scale-to-zero (maintenance bulanan ~$0); per-event ~**$300** karena volume request besar (jutaan hit / 2 jam). Mayoritas biaya ada di **CDN** (mis. 95% cache hit → origin/Lambda hanya ~0.84 USD, sisanya CDN) | lebih murah untuk 1 event; CDN jadi komponen dominan |

**Kesimpulan arah:** 
- **1 event → polling lebih murah.**
- **>2 event / skala besar & sering → SSE/ECS bisa lebih efisien** (biaya connection relatif tetap).
- Karena bisnis pasti jual **banyak event**, arah akhir condong ke **polling sekarang + jalur migrasi ke SSE/hybrid** bila volume naik.
- Angka masih **kasar**, perlu dilengkapi di **ADR + spreadsheet cost model** (sudah ada `infra-cost-model/rusuh-queue-cost-model.xlsx`).

---

## 6. Payment & Token / Session

### Alur payment
- User pilih metode → redirect ke **payment gateway** → gateway redirect balik ke aplikasi dengan **deep link + query param**.
- **Callback diproses di Back End** (webhook): BE menerima call back, update stok/booking, lalu arahkan user ke halaman **sukses/failed**.
- Setelah sukses: 
  - **User login** → redirect ke halaman dengan konteks login (history/booking).
  - **Guest / non-login** → redirect ke **home**.
- **Payment gateway: mock/sandbox** untuk dev; **jangan copy kontrak mitra existing** mentah-mentah — desain ulang kontrak sendiri mengikuti kebutuhan bisnis (arahan: "jangan ketiplek"). Jika sandbox balas 200 → sistem anggap sukses.
- Otorisasi deep link dari gateway: opsi **Basic Auth atau Bearer token** *(perlu diputuskan)*.

### Token & session (login + non-login)
- Alur: masuk antrian → dapat **queue token** (cookie) → saat posisi antrian = 0 → **tukar jadi access token** untuk akses booking.
- **Penyimpanan token:**
  - **Refresh token → set di Cookie** (HttpOnly, lebih aman).
  - **Access token → dikembalikan di response body** (dan/atau cookie). *(Masih diperdebatkan; arah: cookie untuk refresh, body untuk access.)*
- **Non-login (guest) token** harus **terikat device** (`device_id` di dalam JWT + signature) → token tidak bisa dipindah ke device lain.
- **Mobile constraint:** mobile (native) sulit menyimpan cookie seperti browser → kemungkinan pakai **local storage / secure store**; token di-set ulang dari response body. **PWA** masih bisa pakai engine browser (cookie).
- **Logout/expired:** token yang sudah expired harus **di-blacklist di BE** dan semua API diputus (tidak cukup hanya hapus di client).
- **Join queue vs booking** = **dua tahap token berbeda**: `device_id` dibawa di header saat join queue; access token didapat setelah posisi 0.

### Open
- Final: access token di body vs cookie (keputusan security + kemudahan mobile).
- Standarisasi header auth untuk callback payment (Basic vs Bearer).

---

## 7. Standar Kontrak API & Error Handling

- **Envelope seragam** `{ meta, data, message }` untuk semua response.
- **Pagination** (bila list): `limit`, `offset`, `total_data`, `total_page` ditaruh di **`meta`**.
- **Error:** satu `message` utama (bukan list field-error). Business error di-handle sebagai **error boundary di Front End** (mapping), tidak perlu bisnis-code khusus di BE *(disepakati: "tidak perlu business error code")*.
- **HTTP status** mengikuti standar partner **TL** (ada referensi infeksi/standar internal).

---

## 8. Observability & Dashboard (NFR-0009)

- Butuh **dashboard monitoring** (Grafana + Sentry) untuk:
  - **Laju konversi** (funnel: masuk antrian → berhasil bayar). Contoh: antrian 3.000 → sukses beli 100.
  - **Metric antrian**, **rate expired** (hold 5 menit / 10 menit).
  - Front-end observability (Sentry) untuk melihat di langkah mana user banyak gagal/drop.
- Grafana/CloudWatch sudah tersedia → tim bebas desain panel.
- CEO minta eksplisit "harus ada dashboard-nya" sebagai bukti aktif & data-driven, bukan asumsi.

---

## Ringkasan Keputusan (Decisions)

| # | Keputusan | Status |
|---|---|---|
| D1 | 1 NIK = 1 seat; maks 3 tiket/transaksi (3 NIK) | ✅ Sepakat |
| D2 | Validasi Dukcapil **blocking** — gagal/timeout = tidak bisa payment | ✅ Sepakat |
| D3 | Validasi NIK di-trigger via aksi setelah input (hemat hit Dukcapil) | ✅ Sepakat |
| D4 | Nama diinput user & tercetak di tiket; nama tidak diambil dari Dukcapil | ✅ Sepakat (verifikasi nama *perlu konfirmasi*) |
| D5 | Seat auto-generate, multi-tiket diusahakan berderet | ✅ Sepakat |
| D6 | Login opsional; magic link via email + resend | ✅ Sepakat |
| D7 | Passenger list + simpan NIK hanya untuk user teregistrasi + consent (PDP) | ✅ Sepakat |
| D8 | Guest selalu re-validate Dukcapil (tidak di-cache) | ✅ Sepakat |
| D9 | Antrian pakai polling adaptif (1→2→4→8s); SSE/hybrid = fallback | ✅ Sepakat arah |
| D10 | Payment callback diproses BE (webhook); sukses→konteks login, guest→home | ✅ Sepakat |
| D11 | Payment gateway mock/sandbox; desain kontrak sendiri (jangan copy mitra) | ✅ Sepakat |
| D12 | Envelope `{meta,data,message}`; pagination di meta; HTTP status ala TL | ✅ Sepakat |
| D13 | Guest token terikat device_id (JWT+signature); refresh di cookie | ✅ Arah, detail open |
| D14 | Dashboard observability (konversi, antrian, expired) via Grafana + Sentry | ✅ Sepakat |

## Open Items (perlu keputusan lanjut)

| # | Open Question |
|---|---|
| O1 | Apakah nama harus diverifikasi cocok dengan Dukcapil, atau cukup free-text user? |
| O2 | Perjanjian penggunaan mock Dukcapil vs Dukcapil asli (legal/kontrak) |
| O3 | Journey refund untuk salah input yang lolos (back office / manual) |
| O4 | Definisi tegas NFR-0006 "latensi < 1 detik" (response time vs freshness) |
| O5 | Access token: body vs cookie (security + kemudahan mobile/PWA) |
| O6 | Header auth untuk callback payment (Basic vs Bearer) |
| O7 | Mekanisme storage token di mobile native vs PWA |
| O8 | Sisa tiket (stock) real-time: SSE atau polling terpisah? |
| O9 | Daftar final kategori tiket (Festival/Tribun/VIP/...) & aturan seat per kategori |
| O10 | Figma final → kapan bisa jadi kontrak (detail seat & field registrasi) |
