# Meeting Notes — Algoritma Queue & Admission Controller
## Internal PT Server Jangan Tumbang

| Atribut | Detail |
|---|---|
| **Tanggal** | 20 Oktober 2026 |
| **Durasi** | 1 jam 2 menit 33 detik |
| **Peserta** | Mochamad R. Ridho, Handoko, Akbar, Erma, Jaki |
| **Topik Utama** | Algoritma Queueing, Admission Controller, Adaptive Polling, Slot Management |

---

## 1. Konteks Diskusi

Meeting ini membahas mekanisme teknis **transisi dari Waiting Room → Detail Tiket Page → Payment**, khususnya bagaimana sistem memastikan user yang sudah masuk ke halaman detail tiket benar-benar mendapatkan slot tanpa terjadi "PHP" (kondisi masuk tapi tiket tidak tersedia).

---

## 2. Algoritma Admission Controller

### 2.1 Apa itu Admission Controller?

Admission Controller adalah komponen (service) yang mengatur **berapa banyak user dari antrian yang diizinkan masuk ke Detail Tiket Page** pada suatu waktu.

### 2.2 Formula Masuk

User diizinkan masuk ke Detail Tiket Page jika memenuhi **semua** kondisi berikut:

```
canEnter = (freeSlots > 0) AND (availableTickets > 0)
```

Di mana:

| Variabel | Definisi |
|---|---|
| `freeSlots` | Jumlah kasir/slot yang sedang idle (tidak sedang aktif diproses) |
| `availableTickets` | Jumlah tiket yang masih tersisa (belum dipesan/hold) |
| `readyUsers` | User yang sudah punya token tapi belum melanjutkan (re-entrant case) |

### 2.3 Kondisi Khusus: Re-entrant User

Jika user sudah mendapat akses token tapi menutup browser / keluar, kemudian login kembali dalam batas TTL 5 menit — user **tetap diarahkan ke Detail Tiket Page** (bukan kembali ke antrian). Ini menggunakan token yang sama selama belum expired.

### 2.4 Pengecekan Free Slot vs Tiket Tersisa

Sebelum membuka slot baru untuk user berikutnya, sistem harus **mengecek tiket tersisa terlebih dahulu**:

```
Contoh:
- Total tiket: 100
- Sudah terjual: 94
- Target slot aktif: 31
- Sisa tiket: 6

Karena sisa tiket (6) < target_slot × max_tiket_per_user (31 × 3 = 93),
maka free_slot = 0 → tidak membuka slot baru.
```

Slot baru hanya dibuka jika `sisa_tiket > 0` dan `freeSlots > 0`.

---

## 3. Mekanisme Update Posisi Antrian

### 3.1 Keputusan: Kalkulasi Posisi di Frontend

**Masalah:** Dengan 200.000 user concurrent, menghitung posisi antrian unik untuk setiap user di backend sangat mahal secara komputasi.

**Solusi yang Disepakati:**

| Komponen | Tanggung Jawab |
|---|---|
| **Backend** | Menyimpan dan mengirimkan `now_serving` — jumlah antrian yang sudah diproses (global, sama untuk semua user) |
| **Frontend** | Menyimpan `queue_number` milik user, menghitung `posisi = queue_number - now_serving` |

```
Contoh:
- User A: queue_number = 150
- Backend broadcast: now_serving = 100
- Frontend hitung: posisi = 150 - 100 = 50 (masih ada 50 antrian di depan)
```

**Keuntungan:** Backend hanya perlu meng-update satu nilai global (`now_serving`), bukan 200.000 nilai unik.

### 3.2 Penyimpanan Queue Number

`queue_number` disimpan di **persistent database** (backend) dan dikaitkan dengan `user_id`. Ini memungkinkan:
- User pindah device tetap mendapat nomor antrian yang sama
- Session recovery setelah browser ditutup
- Konsistensi lintas device selama user login

---

## 4. SSE vs Polling — Keputusan Arsitektur

### 4.1 Perbandingan

| Aspek | SSE | Adaptive Polling |
|---|---|---|
| **Koneksi** | Persistent, searah (server → client) | Per-request, stateless |
| **Skalabilitas** | Mahal di 200.000 koneksi simultan | Lebih mudah discale dengan CDN |
| **Auto-reconnect** | Built-in | Perlu implementasi manual |
| **Background/Kunci layar** | Koneksi bisa terputus di mobile | Tidak terpengaruh (request per interval) |
| **Network usage** | Lebih rendah (stream) | Lebih tinggi (repeated requests) |
| **CDN Cache** | Tidak bisa di-cache per user | Bisa di-cache untuk `now_serving` global |

### 4.2 Keputusan: Pure Adaptive Polling untuk Seluruh Waiting Room

**Keputusan final:** Seluruh mekanisme Waiting Room — baik update posisi antrian maupun transisi ke Detail Tiket Page — menggunakan **adaptive polling**. SSE tidak digunakan.

**Alasan:**
- SSE membutuhkan persistent connection per user. Dengan 200.000 user concurrent, ini menciptakan beban infrastruktur yang signifikan (connection management, memory per connection, load balancer sticky session).
- Adaptive polling bersifat stateless dan CDN-cacheable, sehingga beban ke origin server jauh lebih rendah.
- Interval polling yang semakin pendek saat antrian mendekati 0 sudah cukup responsif untuk UX — delay maksimal 1 detik saat giliran hampir tiba.
- Implementasi lebih sederhana dan tidak ada risiko koneksi terputus di background (mobile).

**Pemetaan mekanisme:**

| Fase | Mekanisme | Keterangan |
|---|---|---|
| Update posisi antrian (Waiting Room) | Adaptive polling | Interval dinamis berdasarkan posisi |
| Deteksi giliran tiba (posisi = 0) | Adaptive polling (interval 1 detik) | Frontend deteksi via polling, lalu claim token |
| Transisi ke Detail Tiket | Redirect setelah token diterima | Tidak ada SSE/push dari server |

### 4.3 Adaptive Polling Interval

Interval polling disesuaikan dengan posisi antrian untuk mengurangi beban server:

```
posisi > 1.000  → interval: 10 detik
posisi 101–1.000 → interval: 5 detik
posisi 11–100   → interval: 2 detik
posisi 1–10     → interval: 1 detik
posisi = 0      → request access token (satu kali hit)
```

### 4.4 CDN Caching untuk `now_serving`

`now_serving` adalah data global (sama untuk semua user), sehingga bisa di-cache di CDN (Cloudflare, AWS CloudFront, dll). Walau ada 200.000 request per interval, CDN hanya meneruskan **satu request ke origin** per TTL cache.

```
200.000 user polling → CDN (cache hit) → 1 request ke origin server
```

Ini secara signifikan mengurangi load ke backend.

---

## 5. Manajemen Slot & Turnover

### 5.1 Kapan Slot Dibebaskan?

Slot aktif dibebaskan dan tersedia untuk user berikutnya ketika:

1. User menyelesaikan pembayaran (sukses)
2. TTL Booking habis (5 menit, tanpa konfirmasi)
3. TTL Pembayaran habis (10 menit)
4. User secara eksplisit membatalkan

### 5.2 Contoh Skenario Turnover

```
Kondisi awal: 31 slot aktif, 100 tiket tersisa

Event: 2 user berhasil bayar (masing-masing beli 3 tiket = 6 tiket terjual)
→ now_serving += 2
→ freeSlots += 2
→ availableTickets = 94

Cek admission: freeSlots (2) > 0 AND availableTickets (94) > 0
→ 2 user berikutnya dari antrian diizinkan masuk
```

---

## 6. Timezone & Server Time

**Keputusan:** Semua time-based operation (TTL, waktu buka event, countdown) menggunakan **server time**, bukan client time.

- Frontend melakukan sinkronisasi waktu dengan server secara periodik (`TIMER_SYNC_INTERVAL_MS: 30.000ms`)
- Event time ditampilkan dalam timezone-aware format (kirim timezone info bersama data event)
- User di Australia yang aksesnya "lebih cepat" secara wall clock tidak mendapat keuntungan karena antrian dihitung di server

---

## 7. Flow End-to-End (Rangkuman)

```
[Home Page]
    │
    ├── User klik "Beli Tiket" → Join Queue → dapat queue_number dari backend
    │
    ↓
[Waiting Room]
    │
    ├── Adaptive polling: GET /queue/status → terima { now_serving, total_capacity }
    ├── Frontend hitung: posisi = queue_number - now_serving
    ├── Tampilkan posisi & estimasi waktu tunggu
    │
    ├── Saat posisi = 0:
    │   └── Frontend hit: POST /queue/claim-token
    │       ├── Backend validasi: freeSlots > 0 AND availableTickets > 0
    │       ├── Sukses → terima access_token (TTL 5 menit) → redirect ke Detail Tiket
    │       └── Gagal (tiket habis / slot penuh) → tampilkan pesan "Tiket Habis"
    │
    ↓
[Detail Tiket Page]
    │
    ├── Akses dijaga: access_token harus valid
    ├── Pilih kategori & jumlah tiket (max 3)
    ├── Input NIK + email per tiket
    ├── Polling stok tiket setiap 10 detik
    ├── Countdown TTL Booking (5 menit, server-synced)
    │
    ↓
[Payment Page]
    │
    ├── Pilih metode pembayaran
    ├── Countdown TTL Pembayaran (10 menit)
    ├── Polling status pembayaran setiap 3 detik
    └── Sukses → release slot → now_serving++ → tiket diterbitkan
```

---

## 8. Architectural Decision Records (ADR)

### ADR-001: Kalkulasi Posisi Antrian di Frontend

| | |
|---|---|
| **Status** | Diputuskan |
| **Tanggal** | 20 Oktober 2026 |

**Konteks:** Dengan 200.000 user concurrent, backend tidak bisa efficiently menghitung posisi unik setiap user per request polling.

**Alternatif:**
1. Backend mengirim posisi unik per user → mahal, O(n) komputasi per polling interval
2. Backend hanya broadcast `now_serving` (global) → frontend kalkulasi posisi lokal

**Keputusan:** Alternatif 2. Backend broadcast `now_serving`. Frontend hitung `posisi = queue_number - now_serving`.

**Konsekuensi:** `queue_number` harus disimpan di persistent storage (backend DB) untuk mendukung device switching.

---

### ADR-002: Pure Adaptive Polling untuk Seluruh Mekanisme Waiting Room

| | |
|---|---|
| **Status** | Diputuskan |
| **Tanggal** | 20 Oktober 2026 |

**Konteks:** Pilihan antara SSE (persistent connection) dan polling untuk update posisi antrian dan deteksi transisi ke Detail Tiket.

**Alternatif:**
1. Pure SSE — persistent connection per user, auto-reconnect, real-time push dari server
2. Pure Adaptive Polling — stateless, scalable via CDN, interval dinamis
3. Kombinasi: Polling untuk posisi antrian + SSE untuk notifikasi transisi

**Keputusan:** **Alternatif 2 — Pure Adaptive Polling** untuk seluruh siklus Waiting Room, termasuk deteksi saat giliran tiba (posisi = 0) dan trigger claim token. SSE tidak digunakan.

**Alasan utama:** Menjaga beban trafik server tetap rendah. SSE dengan 200.000 persistent connection simultan memerlukan infrastruktur connection management yang mahal. Adaptive polling bersifat stateless dan dapat dioptimalkan via CDN caching untuk data `now_serving` yang bersifat global.

**Pros:**
- CDN-cacheable untuk data global `now_serving` — 200.000 request hanya menghasilkan 1 hit ke origin
- Tidak perlu manage persistent connection per user
- Stateless — tidak ada risiko koneksi terputus di background mobile
- Interval adaptif secara aktif mengurangi beban: antrian panjang = polling lebih jarang

**Cons:**
- Slight delay maksimal 1 detik saat giliran hampir tiba (diterima karena UX masih wajar)
- Polling request tetap terjadi walau tidak ada perubahan (dimitigasi dengan interval adaptif)

---

### ADR-003: Server Time sebagai Source of Truth

| | |
|---|---|
| **Status** | Diputuskan |
| **Tanggal** | 20 Oktober 2026 |

**Konteks:** User dari timezone berbeda (WIB, AWST, AST) mengakses event yang dibuka pada waktu spesifik.

**Keputusan:** Semua TTL, waktu buka tiket, dan countdown menggunakan server time. Frontend sync setiap 30 detik.

**Konsekuensi:** Event API harus menyertakan timezone info. Frontend tidak boleh mengandalkan `Date.now()` untuk operasi kritis.

---

## 9. Poin Terbuka / Follow-up

| No | Item | Owner | Status |
|---|---|---|---|
| 1 | Finalisasi figma dengan Signal UI styling (warna + komponen) | Akbar | In Progress |
| 2 | Update API doc berdasarkan keputusan ADR-001 & ADR-002 | Handoko | Todo |
| 3 | Format event time di API response (sertakan timezone) | Backend Team | Todo |
| 4 | Validasi Mermaid diagram admission controller dengan logika final | Handoko/Erma | Todo |
| 5 | Figma sebagai UI/API contract antara fe-web, mobile, dan backend | Akbar | In Progress |

---

## 10. Referensi

- Business Requirements: `documentation/business-needs-cto-pov.md`
- Gap Analysis & Edge Cases: `documentation/gap-analysis-edge-cases.md`
- War Ticketing Edge Cases: `documentation/war_ticketing_edge_cases.md`
- Solution Analysis: `documentation/solution-analysis.md`
- Domain Constants: `src/@core/domain/constants/booking.constants.ts`

---

*Dokumen ini dibuat berdasarkan rekaman meeting Internal PT Server Jangan Tumbang, 20 Oktober 2026.*
