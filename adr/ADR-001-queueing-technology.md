# ADR-001: Teknologi Antrian untuk Virtual Waiting Room

**Status**: **Accepted — sebagian digantikan ADR-005** (30 Oktober 2026)  
**Tanggal**: 2026  
**Decider**: Backend Team

> **Keputusannya berlaku; rincian implementasinya tidak.** Model counter Redis yang
> dipilih di sini tetap menjadi dasar ADR-005. Yang sudah bergerak sejak dokumen ini:
> `queue:{event_id}:state` kini bernilai `SCHEDULED | OPEN | SOLD_OUT | CLOSED` (bukan
> `"open" | "closed_sold_out"`), dan `grant:{token}` hidup `EX 1080`, bukan `EX 900`.
> Sejak kontrak API v2.3, kredensial yang disimpan di key itu bernama **grant token** dan
> dikirim di header `X-Grant-Token`, bukan di `Authorization: Bearer` — slot itu kini milik
> token sesi akun saja (ADR-005, "Empat kredensial, empat slot").
> Untuk skema key dan TTL yang mengikat, pakai **ADR-005 Bagian 4 dan Bagian 7**.

---

## Konteks

Sistem Rusuh Booking Tiket dirancang untuk menangani lonjakan trafik ekstrem (*flash crowd*)
saat penjualan tiket konser dibuka. Salah satu komponen kritis adalah **Virtual Waiting Room** —
ruang tunggu virtual yang mengatur giliran masuk ke halaman pembelian secara adil dan terkendali.

Requirement yang mendorong keputusan ini:

| ID | Requirement |
|----|-------------|
| FR-06 | Sistem menampilkan posisi antrian customer secara tepat ("Anda antrian ke-n") |
| FR-07 | Posisi antrian diperbarui otomatis setiap antrian di depan berkurang |
| FR-08 | Customer otomatis dipindahkan ke Detail Ticket Page saat gilirannya tiba |
| FR-10 | Posisi antrian tetap konsisten saat customer me-refresh halaman |
| NFR-04 | Antrian bersifat FIFO — posisi adil dan tidak bisa dilangkahi |
| NFR-05 | Pembaruan posisi antrian memiliki latensi rendah (p95 < 1 detik) di bawah beban tinggi |
| NFR-06 | Sistem harus mampu menangani lonjakan trafik mendadak jauh melebihi kapasitas tiket |

Tantangan teknis yang membedakan ini dari antrian konvensional: teknologi yang dipilih harus
mampu menjawab dua pertanyaan sekaligus dengan efisien —

1. **"Berapa total customer dalam antrian?"** — dibutuhkan untuk estimasi waktu tunggu
2. **"Customer X saat ini berada di posisi ke berapa?"** — dibutuhkan untuk tampilan FR-06

Tidak semua teknologi antrian mampu menjawab keduanya secara akurat dan cepat di bawah
kondisi flash crowd.

---

## Opsi yang Dipertimbangkan

### Opsi A: RabbitMQ

**Kekuatan:**
- Jumlah pesan dalam antrian tersedia via Management API (`messages` field), akurat
- FIFO terjamin secara default
- Mature, battle-tested

**Kelemahan:**
- **Tidak ada query posisi per customer.** Tidak ada mekanisme native untuk mengetahui
  "customer X berada di posisi ke berapa" — ini memblokir FR-06 sepenuhnya
- Berbasis model *consumer delivery*, bukan *positional observer* — customer harus menjadi
  consumer aktif, tidak cocok untuk waiting room yang menampilkan nomor urut
- Membutuhkan layer tambahan di luar RabbitMQ untuk menjawab pertanyaan posisi

### Opsi B: Apache Kafka

**Kekuatan:**
- Throughput sangat tinggi untuk pengiriman event
- Posisi per consumer bisa diketahui via consumer offset

**Kelemahan:**
- **Consumer offset hanya berlaku per partition, bukan per individu customer** — satu
  Lambda/consumer per customer tidak scalable
- Query posisi lintas consumer tidak tersedia secara native
- Tidak ada operasi atomic "ambil dari depan" (setara `ZPOPMIN`) untuk pemberian slot
- Kompleksitas operasional tinggi: butuh Kafka cluster, ZooKeeper/KRaft, Schema Registry

### Opsi C: Amazon SQS (FIFO)

**Kekuatan:**
- Terkelola penuh, zero operational overhead
- FIFO terjamin pada FIFO Queue

**Kelemahan:**
- `GetQueueAttributes` → `ApproximateNumberOfMessages` bersifat **approksimasi**, bukan
  nilai pasti — tidak memenuhi kebutuhan tampilan posisi akurat di FR-06
- Tidak ada mekanisme query posisi per pesan sama sekali
- Pull-based polling only — tidak ada push

### Opsi D: PostgreSQL Queue

**Kekuatan:**
- Posisi dan jumlah antrian akurat dan exact via `COUNT(*)`
- ACID — durabilitas penuh
- Familiar bagi sebagian besar tim

**Kelemahan:**
- `COUNT(*)` pada tabel dengan 50.000+ baris tanpa optimasi menjadi bottleneck di bawah
  flash crowd — tidak memenuhi NFR-06
- Tidak horizontally scalable tanpa layer cache di atasnya
- Tidak memiliki built-in mekanisme real-time update

### Opsi E: BullMQ (Redis-backed job queue library)

**Kekuatan:**
- `queue.getWaitingCount()` akurat dan O(1) karena berbasis `LLEN` Redis
- Event-driven via Redis Pub/Sub

**Kelemahan:**
- Tidak ada query posisi per job secara native
- Abstraksi dirancang untuk *job processing* (worker mengonsumsi task), bukan untuk
  customer yang mengamati posisi mereka sendiri dalam antrian
- Menambahkan dependency library di atas Redis tanpa nilai tambah untuk use case ini

### Opsi F: Redis ZSET + ZRANK (Model v1)

Setiap customer ditambahkan ke Redis Sorted Set dengan skor berupa timestamp bergabung:

```
ZADD queue:event:{id}  <join_timestamp>  <customer_token>
ZCARD queue:event:{id}           → total antrian  O(1)
ZRANK queue:event:{id}  <token>  → posisi customer  O(log N)
```

**Kekuatan:**
- `ZCARD` O(1) untuk total antrian
- `ZRANK` O(log N) untuk posisi per customer — akurat
- FIFO otomatis via skor timestamp
- Atomic join via `ZADD NX`

**Kelemahan:**
- Server harus memanggil `ZRANK` secara individual untuk setiap customer saat hendak
  menyebarkan pembaruan posisi — **tidak ada response yang bisa dibagi antar customer**
- Dengan 50.000 customer aktif, pembaruan posisi berarti 50.000 query ZRANK secara bersamaan
- Customer yang meninggalkan antrian tanpa pamit (*abandoned*) harus dihapus aktif via `ZREM`

### Opsi G: Redis Counter Model — INCR + now_serving (Model v2, Dipilih)

Menggunakan dua counter global per event sebagai pengganti sorted set:

```
queue:{event}:issued       STR  ← INCR saat customer bergabung → nomor tiket unik
queue:{event}:now_serving  STR  ← dimajukan saat slot tersedia
```

Saat bergabung, `INCR` dijalankan secara atomic dan menghasilkan nomor tiket unik —
analogis dengan mesin nomor antre. Posisi dihitung oleh **client**, bukan server:

```
posisi = ticket_no − now_serving
```

**Kekuatan:**
- `INCR` bersifat atomic — tidak ada dua customer yang mendapatkan nomor sama meski
  puluhan ribu request datang bersamaan
- `now_serving` adalah satu nilai global yang **identik** untuk semua customer —
  server tidak perlu menghitung posisi per customer
- Seluruh operasi hot path adalah O(1)
- Customer abandoned tidak memerlukan cleanup aktif — nomornya dilewati `now_serving` secara alami
- Posisi bersifat *monotonically decreasing* — tidak pernah terlihat naik dari perspektif customer

**Kelemahan:**
- Terdapat "celah" pada sekuens nomor akibat customer abandoned, sehingga posisi yang
  ditampilkan adalah estimasi konservatif (bisa lebih besar dari posisi nyata)
- `now_serving` harus dimajukan via Lua script untuk menjamin atomicity read-advance-write

---

## Keputusan

**Redis dengan model ticket-number counter (INCR + now_serving)** digunakan sebagai engine
Virtual Waiting Room.

---

## Alasan

### Redis vs. alternatif lain

| Kapabilitas | Redis Counter | RabbitMQ | Kafka | SQS FIFO | PostgreSQL | BullMQ |
|---|---|---|---|---|---|---|
| Total antrian (exact) | ✅ Derived: `issued − now_serving` | ✅ via API | ⚠️ Approx | ⚠️ Approx | ✅ COUNT(*) lambat | ✅ LLEN |
| Posisi per customer | ✅ O(1), di client | ❌ Tidak ada | ❌ Tidak praktis | ❌ Tidak ada | ✅ Lambat di skala | ❌ Tidak ada |
| FIFO | ✅ Via INCR sequence | ✅ | ✅ per partition | ✅ | ✅ | ✅ |
| Operasi atomic join | ✅ INCR | ✅ | ✅ | ✅ | ✅ via lock | ✅ |
| Scalable di flash crowd | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Kompleksitas operasional | Rendah | Sedang | Tinggi | Sangat rendah | Rendah | Rendah |

Hanya Redis yang mampu menjawab kedua pertanyaan kritis (total antrian dan posisi per customer)
secara akurat dengan kompleksitas operasional rendah. SQS dan Kafka gugur karena posisi
per customer tidak tersedia. PostgreSQL gugur karena tidak scalable di flash crowd. RabbitMQ
dan BullMQ gugur karena tidak mendukung query posisi.

### Counter model (v2) vs. ZSET (v1)

Model ZSET membutuhkan satu query `ZRANK` per customer setiap kali posisi perlu diperbarui.
Pada 50.000 customer aktif, ini berarti 50.000 operasi berbeda dengan 50.000 hasil berbeda
— tidak ada satupun yang bisa digunakan bersama antar customer.

Model counter membalik paradigma ini: server menyimpan dan menyebarkan **satu nilai global**
(`now_serving`), dan setiap client menghitung posisinya sendiri secara lokal
(`ticket_no − now_serving`). Beban komputasi posisi dipindahkan sepenuhnya ke client.

```
v1 — server harus menghitung 50.000 jawaban berbeda:
  ZRANK(alice) → 16.717   ZRANK(bob) → 16.716   ZRANK(charlie) → 16.715 ...

v2 — server menyediakan satu jawaban, semua client hitung sendiri:
  now_serving = 31.500
  Alice: 48.217 − 31.500 = 16.717   ← Alice yang hitung, bukan server
  Bob:   48.216 − 31.500 = 16.716   ← Bob yang hitung, bukan server
```

---

## Struktur Data Redis

```
# State antrian per event
queue:{event_id}:issued       STR   ← INCR saat customer join → nomor tiket unik
queue:{event_id}:now_serving  STR   ← dimajukan oleh admission controller
queue:{event_id}:state        STR   ← "open" | "closed_sold_out"

# Per customer (disimpan saat join, dibaca saat polling/push)
token:{uuid}                  HASH  ← ticket_no, account_id, event_id, issued_at

# Pemberian slot (dibuat admission controller, dibaca client)
grant:{token}                 STR   ← EX 900, tanda slot telah diberikan
```

---

## Konsekuensi

**Positif:**
- Semua operasi hot path adalah O(1) — tidak ada penalti performa saat antrian membengkak
  hingga ratusan ribu customer
- `now_serving` yang identik untuk semua customer memungkinkan response di-cache dan
  disebarkan secara efisien tanpa komputasi per customer di server
- Customer abandoned tidak memerlukan cleanup aktif — nomornya dilewati `now_serving`
  secara alami, menyederhanakan lifecycle management
- Posisi bersifat monotonically decreasing — lebih intuitif secara UX dan mencegah
  "loncat balik" yang membingungkan customer
- Estimasi waktu tunggu (FR-09) tersedia tanpa infrastruktur tambahan dari delta
  `now_serving` per unit waktu

**Negatif / Risiko:**
- **Posisi yang ditampilkan adalah estimasi konservatif** karena celah pada sekuens nomor
  akibat customer abandoned. Customer bisa melihat posisinya lebih besar dari posisi nyata.
  Mitigasi: komunikasikan ke product bahwa ini adalah *upper bound*, bukan posisi eksak.
- **Redis adalah in-memory store.** Kegagalan Redis dapat menyebabkan kehilangan data antrian.
  Mitigasi **sebagian**: Aurora PostgreSQL berperan sebagai *system of record*, dan stok,
  `nik_claims` serta booking yang sudah di-grant memang dapat dibangun ulang dari Aurora.
  **Posisi antrian yang belum di-grant tidak dapat** — `ticket_no` hanya ada di Redis dan
  baris Aurora baru dibuat saat GRANT. Pernyataan "state Redis dapat dibangun ulang" pada
  versi awal dokumen ini terlalu luas; lihat ADR-005 Bagian 16 dan Bagian 17 #7.
- **Admission controller harus atomic.** Dua invokasi bersamaan dapat memajukan `now_serving`
  dua kali jika tidak dilindungi.
  Mitigasi: gunakan Lua script untuk membungkus operasi read-advance-write menjadi satu
  langkah atomic; atau set reserved concurrency = 1 pada Lambda admission controller.

---

## Referensi

- `Tender Rusuh Booking Ticket.pdf` — FR-06, FR-07, FR-08, FR-10, NFR-04, NFR-05, NFR-06
