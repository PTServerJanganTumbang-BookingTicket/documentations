# Update terhadap Requirement di Tender Rusuh Booking Ticket

**Tanggal Update**: Oktober 2026  
**Dokumen Referensi**: Tender Rusuh Booking Ticket.pdf  
**Disusun oleh**: Tim SJT

---

## 1. Latar Belakang

Setelah melakukan klarifikasi dengan CEO, CTO, dan Principal, terdapat beberapa perubahan pada
dokumen requirement yang perlu dicatat dan disosialisasikan kepada seluruh tim teknis. Perubahan
ini bersifat resmi dan menggantikan poin-poin terkait pada dokumen asli tender.

Klarifikasi dilakukan atas dasar penilaian ulang terhadap pengalaman pengguna, kebutuhan
aksesibilitas platform, dan strategi pencegahan scalping yang lebih efektif secara teknis.

---

## 2. Perubahan Login / Autentikasi

### Status Sebelumnya

Dokumen requirement asli tidak mendefinisikan requirement login secara eksplisit, namun
beberapa business rule (terutama BR-07) secara implisit mengasumsikan adanya identitas
customer yang terikat pada akun sebelum bergabung ke antrian.

### Keputusan Baru

Tim SJT telah menyampaikan analisis mengenai **pros dan cons** dari skema user tidak wajib
melakukan login, dan dari CEO telah mengkonfirmasi bahwa:

> **Requirement login bersifat opsional.** Customer dapat bergabung ke antrian dan melakukan
> pembelian tiket tanpa harus memiliki atau masuk ke akun terlebih dahulu.

### Perbandingan Skema

| Aspek | Login Wajib | Login Opsional (Keputusan Baru) |
|---|---|---|
| Friction masuk antrian | Tinggi — harus registrasi/login dulu | Rendah — langsung masuk antrian |
| Enforcement NIK | Di akun (sebelum antrian) | Di transaksi (saat booking) |
| Pencegahan multi-antrian | Via satu akun = satu slot | Via validasi NIK saat booking tiket |
| Risiko scalping | Lebih rendah (identitas terikat) | Ditangani lewat aturan NIK per tiket |
| Pengalaman pengguna | Lebih terbatas | Lebih inklusif dan cepat |

### Catatan Teknis

Dengan login opsional, pencegahan scalping berpindah dari layer **antrian** ke layer
**transaksi tiket**. Mekanisme NIK (Nomor Induk Kependudukan) menjadi satu-satunya kontrol
yang mengikat, dan divalidasi pada saat customer memilih tiket — bukan pada saat bergabung
antrian. Detail aturan NIK dijabarkan pada perubahan BR-01 dan BR-02 di bawah.

---

## 3. Perubahan Business Rules

### 3.1 BR-01 — Kuota Maksimal Pembelian Tiket

| | Sebelum | Sesudah |
|---|---|---|
| **Rumusan** | Maksimal pembelian 3 tiket per **NIK** per event (akumulasi seluruh kategori). Bila NIK tidak digunakan, kuota 3 dihitung per sesi pembelian. | Maksimal pembelian **3 tiket per 1 transaksi (antrian)** sebuah event. |
| **Basis pembatasan** | NIK lintas sesi dan perangkat | Per transaksi / per slot antrian |
| **Fallback jika tidak ada NIK** | Per sesi | Tidak relevan — dibatasi per transaksi |

**Penjelasan perubahan:**

Aturan lama mengikat kuota ke NIK secara kumulatif lintas sesi, yang membutuhkan
persistensi state NIK di database dan validasi lintas transaksi. Aturan baru menyederhanakan
ini: satu slot antrian boleh membeli maksimal 3 tiket dalam satu transaksi, tanpa melihat
berapa transaksi lain yang pernah dilakukan oleh NIK yang sama. Kontrol per-NIK kini
ditangani oleh BR-02 yang diperbarui (satu NIK per satu seat).

---

### 3.2 BR-02 — Pembatasan NIK per Tiket

| | Sebelum | Sesudah |
|---|---|---|
| **Rumusan** | Satu NIK yang sama tidak dapat melampaui **3 tiket** pada event yang sama, meskipun melalui sesi/perangkat berbeda. | Satu NIK dialokasikan hanya pada **satu tiket / seat** pada suatu event yang sama. |
| **Batas per NIK** | 3 tiket per event | 1 tiket per event |
| **Scope validasi** | Kumulatif lintas sesi | Per tiket pada saat pendaftaran NIK |

**Penjelasan perubahan:**

Aturan baru justru **lebih ketat** dalam hal per-NIK: satu NIK hanya boleh mengisi satu
kursi di satu event yang sama. Ini berarti dalam satu transaksi yang membeli 3 tiket,
ketiga tiket tersebut membutuhkan tiga NIK yang **berbeda**. Customer yang membeli untuk
keluarga atau rombongan harus mendaftarkan NIK masing-masing anggota.

Ini secara efektif memblokir scalper, karena memborong banyak tiket membutuhkan NIK
yang valid dan unik untuk setiap kursi — bukan hanya satu NIK untuk beberapa tiket.

---

### 3.3 BR-07 — Diluruskan, bukan dihapus

| | Sebelum | Sesudah |
|---|---|---|
| **Rumusan** | Satu customer hanya memegang satu slot aktif pada satu waktu untuk event yang sama. | **Satu keterlibatan aktif per event** — posisi antrian **atau** booking yang belum terminal. Berlaku untuk customer **login**; tidak berlaku untuk customer tamu. |
| **Status** | Aktif | **Aktif, dengan cakupan yang dipersempit** |
| **Jumlah event** | Tidak dibatasi | **Tidak dibatasi** — satu keterlibatan per event, berapa pun jumlah eventnya |

**Mengapa versi "dihapus" perlu dikoreksi.**

Pembacaan sebelumnya menilai BR-07 sebagai aturan **anti-scalping**, lalu menyimpulkan
dengan benar bahwa BR-02 sudah menanganinya: memborong tiket menuntut NIK unik per kursi,
sehingga slot tambahan tidak menghasilkan tiket tambahan. Kesimpulan itu tetap benar.

Yang terlewat adalah bahaya yang berbeda. Slot bukan hanya pintu menuju tiket — ia adalah
**sumber daya yang terbatas**. Jumlah customer yang boleh berada di Detail Ticket Page secara
bersamaan dibatasi `MAX_CONCURRENT_SLOTS`. Satu orang yang memegang lima slot sekaligus
memakai lima unit dari kuota itu dan **memperlambat masuknya semua orang lain**, meskipun ia
tidak akan pernah bisa menukarnya menjadi lebih banyak tiket. Itu kerugian *throughput* dan
keadilan, bukan kerugian scalping — dan itulah yang BR-07 lindungi di arsitektur ini.

**Rumusan baru, dan apa yang dilepasnya.**

Satu akun boleh memegang satu keterlibatan aktif per event. Keterlibatan itu dilepas begitu
mencapai keadaan terminal:

| Keadaan | Boleh antre lagi? |
|---|---|
| Masih mengantre, atau booking belum selesai | Tidak — join kedua memulihkan yang sudah ada |
| **Pembayaran selesai (`CONFIRMED`)** | **Ya** — boleh antre lagi untuk pembelian berikutnya |
| Kedaluwarsa / dibatalkan | Ya |

Antre lagi berarti **dari belakang**: nomor antrian baru, posisi lama tidak dipulihkan. Kuota
tetap dijaga BR-02 — pembelian kedua menuntut NIK yang berbeda lagi.

Join kedua dari akun yang sama **tidak ditolak**, melainkan memulihkan keterlibatan yang ada
dan menerbitkan ulang tokennya. Menolak akan menghukum customer yang hanya berpindah
perangkat; sebagai efek sampingnya, customer login mendapat pemulihan antrian lintas
perangkat yang tidak tersedia bagi tamu.

**Batasan yang tetap berlaku dari pembacaan sebelumnya:**
- **Tiket dibatasi** oleh aturan 1 NIK per tiket (BR-02)
- **NIK yang sama tidak bisa didaftarkan lebih dari satu kali** untuk suatu event yang sama
- Slot tambahan tidak menghasilkan tiket tambahan, karena setiap kursi menuntut NIK unik

**Kelemahan yang diterima secara sadar.**

BR-07 hanya dapat ditegakkan terhadap customer yang teridentifikasi. Customer tamu tidak
punya identitas stabil, sehingga strategi optimal pihak yang berniat buruk adalah **tidak
login** — artinya BR-07 membatasi pengguna yang kooperatif dan bukan yang adversarial.

Ini dapat diterima **hanya** karena proteksi sebenarnya bagi semua orang berada di tempat
lain: **CAPTCHA tak terlihat pada join antrian**, ditambah rate limit per IP. BR-07 adalah
lapis kedua, bukan pertahanan utama. Tanpa CAPTCHA, memulihkan BR-07 hampir tidak berarti.

---

## 4. Ringkasan Perubahan

| Poin | Jenis Perubahan | Ringkasan |
|---|---|---|
| Login | **Diubah** | Dari implisit wajib → opsional (dikonfirmasi CEO) |
| BR-01 | **Diubah** | Kuota 3 tiket: dari per-NIK kumulatif → per transaksi |
| BR-02 | **Diubah** | Kuota NIK: dari 3 tiket per NIK → 1 NIK per tiket/seat |
| BR-07 | **Diluruskan** | Satu keterlibatan aktif **per event**, customer login saja; jumlah event tidak dibatasi. Alasannya okupansi slot, bukan scalping |
| Proteksi bot | **Ditambahkan** | CAPTCHA tak terlihat pada join antrian — pertahanan utama yang membuat BR-07 bermakna |

