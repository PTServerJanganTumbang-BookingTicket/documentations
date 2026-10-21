# ADR-003: Mekanisme Pengiriman Update Posisi Antrian ke Client

**Status**: In Review  
**Tanggal**: 2026  
**Decider**: Frontend & Backend Team

---

## Konteks

Setelah teknologi antrian (ADR-001) dan mekanisme penentuan posisi (ADR-002) ditetapkan,
pertanyaan berikutnya adalah **bagaimana server mengirimkan update `now_serving` ke client
secara real-time**. Ini adalah keputusan yang berdampak langsung pada pengalaman pengguna
di Virtual Waiting Room dan kemampuan sistem memenuhi requirement latensi.

Requirement yang mendorong keputusan ini:

| ID     | Requirement                                                                 |
| ------ | --------------------------------------------------------------------------- |
| FR-07  | Posisi antrian diperbarui otomatis setiap antrian di depan berkurang        |
| FR-08  | Customer otomatis dipindahkan ke Detail Ticket Page saat gilirannya tiba    |
| FR-10  | Posisi antrian tetap konsisten saat customer me-refresh halaman             |
| NFR-05 | Sistem mampu menangani minimal 1000 request per detik                       |
| NFR-06 | Pembaruan posisi antrian memiliki latensi rendah (p95 < 1 detik)            |

Dari ADR-002, kita sudah memutuskan bahwa server menyebarkan nilai `now_serving` yang
**identik untuk semua customer**, dan client menghitung posisinya sendiri
(`posisi = ticket_no − now_serving`). ADR ini membahas **bagaimana nilai tersebut
dikirimkan ke client**.

---

## Opsi yang Dipertimbangkan

### Opsi A: Long Polling

Client membuka HTTP request, server menahan respons sampai ada perubahan atau timeout.

```
Client ──GET /queue/serving──▶ Server
       (menunggu 30 detik...)
       ◀─────────────────────── { "now_serving": 31500 }
Client ──GET /queue/serving──▶ Server
       (menunggu 30 detik...)
       ...
```

**Kekuatan:**
- Universal browser support, termasuk browser lama
- Bekerja melalui semua proxy dan firewall
- Fallback yang mudah diimplementasikan

**Kelemahan:**
- Latensi inherent 1-3 detik (bergantung interval polling)
- Tidak dapat memenuhi p95 < 1 detik secara efisien (NFR-06)
- Reconnect logic harus diimplementasikan manual
- Connection cycling meningkatkan beban server

### Opsi B: WebSocket

Koneksi bidirectional yang persisten antara client dan server.

**Kekuatan:**
- Latensi terendah (~0ms push)
- Bidirectional — client bisa mengirim data ke server

**Kelemahan:**
- Overkill untuk use case read-only (waiting room hanya menerima update, tidak mengirim)
- Lebih kompleks: upgrade handshake, ping/pong keep-alive, manual reconnect
- Mungkin memerlukan konfigurasi proxy/load balancer khusus
- Overhead implementasi tidak sebanding dengan manfaat

### Opsi C: Server-Sent Events (SSE) — Dipilih

Koneksi unidirectional persisten menggunakan HTTP standar. Server mendorong event ke client.

```
Client ──EventSource('/queue/stream')──▶ Server
       ◀── event: stats
       ◀── data: {"now_serving": 31500, "throughput": 12}
       ◀── event: stats
       ◀── data: {"now_serving": 31501, "throughput": 12}
       ...
```

**Kekuatan:**
- Built-in reconnect di browser (`EventSource` API otomatis reconnect saat koneksi putus)
- Latensi mendekati nol (~0ms push)
- Menggunakan HTTP/1.1 standar — tidak perlu konfigurasi khusus
- CDN-friendly: payload `now_serving` identik untuk semua customer, dapat di-cache di edge
- Lebih sederhana dari WebSocket karena unidirectional

**Kelemahan:**
- Unidirectional only (tidak masalah untuk use case ini)
- Tidak support di Internet Explorer (acceptable — IE sudah EOL)

---

## Keputusan

**Server-Sent Events (SSE)** dengan model broadcast dipilih sebagai mekanisme pengiriman
update posisi antrian ke client.

---

## Alasan

### 1. Unidirectional sesuai dengan use case

Waiting room bersifat read-only dari perspektif client — server mendorong update posisi,
client tidak pernah mengirim data. Kapabilitas bidirectional WebSocket tidak dibutuhkan
dan hanya menambah kompleksitas tanpa nilai tambah.

### 2. Built-in reconnect memenuhi FR-10

API `EventSource` browser secara native melakukan reconnect saat koneksi terputus. Ini
memenuhi FR-10 (posisi konsisten saat refresh) tanpa kode tambahan. Long Polling dan
WebSocket memerlukan implementasi reconnect manual.

### 3. Model broadcast memungkinkan CDN caching dan beban server O(1)

Dari ADR-002, server menyebarkan `now_serving` yang identik untuk semua customer.
Dengan SSE, payload ini dapat di-cache di CDN edge nodes dengan TTL 1-2 detik.

**Perhitungan cache hit rate:**
```
Asumsi:
- 50.000 customer aktif
- Polling interval rata-rata: 10 detik
- CDN cache TTL: 2 detik

Request rate ke CDN: 50.000 / 10 = 5.000 request/detik
Cache refresh rate: 1 / 2 = 0.5 request/detik ke origin
Cache hit rate: (5.000 - 0.5) / 5.000 = 99.99%

Hasil:
- Tanpa CDN:  5.000 request/detik ke origin
- Dengan CDN: ~1 request/2 detik ke origin (penurunan 10.000x)
```

Per-user position (model ZSET) tidak dapat di-cache karena setiap customer mendapat
nilai berbeda.

### 4. Memenuhi NFR-06 (p95 < 1 detik)

Push-based delivery mencapai latensi mendekati nol. Long Polling memiliki latensi
inherent dari interval polling yang sulit memenuhi p95 < 1 detik secara efisien.

**Perhitungan latensi Long Polling:**
- Interval polling tipikal: 3-10 detik (trade-off antara freshness vs server load)
- Dengan interval 3 detik, worst-case latency = 3 detik (perubahan terjadi tepat setelah poll)
- Average latency = interval / 2 = 1.5 detik
- Untuk mencapai p95 < 1 detik, interval harus < 2 detik → 30+ request/menit/customer
- Pada 50.000 customer: 25.000+ request/detik ke origin — tidak praktis

**Latensi SSE:**
- Event dikirim segera saat `now_serving` berubah
- Network latency only (~50-200ms typical)
- p95 < 1 detik tercapai secara natural

### 5. Kompleksitas lebih rendah dari WebSocket

Tidak ada upgrade handshake, tidak perlu ping/pong keep-alive, tidak perlu logic
reconnect manual — semua ditangani oleh browser dan HTTP standar.

### Mengapa Bukan Long Polling?

| Aspek                  | Long Polling        | SSE                  |
| ---------------------- | ------------------- | -------------------- |
| Latensi                | 1-3 detik           | ~0ms (push)          |
| p95 < 1 detik          | Sulit dicapai       | ✅ Mudah dicapai     |
| Reconnect              | Manual              | ✅ Built-in browser  |
| CDN caching            | Tidak efektif*      | ✅ Efektif           |
| Connection overhead    | Tinggi (cycling)    | Rendah (persistent)  |

*Long Polling + CDN tidak efektif karena: (1) cache response = data stale, atau
(2) bypass cache = tidak ada manfaat CDN.

#### Mengapa Long Polling Tidak Dapat Memenuhi p95 < 1 Detik

**Definisi p95 latency:** 95% dari semua update posisi harus sampai ke client dalam
waktu < 1 detik sejak perubahan terjadi di server.

**Model Long Polling:**
```
Timeline server: ───────●───────────────●───────────────●─────▶
                     change₁          change₂          change₃
                     (t=0)            (t=5s)           (t=12s)

Timeline client (poll interval 3s):
    poll₁──────────poll₂──────────poll₃──────────poll₄──────────▶
    (t=0)          (t=3)          (t=6)          (t=9)

    Kapan client TAHU tentang change₁?
    - change₁ terjadi di t=0
    - poll₂ di t=3 → client tahu di t=3
    - Latency = 3 detik ❌
```

**Analisis matematis:**

Dengan polling interval `I` detik, waktu tunggu hingga client mengetahui perubahan
berdistribusi **uniform** antara 0 dan I detik (tergantung kapan perubahan terjadi
relatif terhadap poll berikutnya).

```
Distribusi latency (uniform):
- Minimum: 0 detik (perubahan tepat sebelum poll)
- Maximum: I detik (perubahan tepat setelah poll)
- Average: I/2 detik
- p95: 0.95 × I detik

Untuk mencapai p95 < 1 detik:
  0.95 × I < 1
  I < 1.05 detik

Artinya polling interval harus < 1.05 detik (~1 poll/detik/customer)
```

**Implikasi polling interval < 1 detik:**

```
Asumsi: 50.000 customer aktif, interval 1 detik

Request rate: 50.000 request/detik ke server
Per menit: 3.000.000 request
Per jam: 180.000.000 request

Bandingkan dengan SSE:
- 50.000 koneksi persistent
- Update broadcast: 1 event × 50.000 delivery (via held connections)
- Tidak ada request overhead — data di-push langsung
```

**Mengapa ini tidak praktis:**

| Aspek | Polling (1s interval) | SSE |
|-------|----------------------|-----|
| Request rate | 50.000/detik | ~0 (connections held) |
| Server CPU | Tinggi (handle requests) | Rendah (hold + broadcast) |
| Network overhead | Tinggi (HTTP headers setiap request) | Rendah (event stream) |
| Load balancer cost | Tinggi (request counting) | Rendah (connection counting) |
| CDN benefit | ❌ Tidak bisa cache (TTL harus < 1s) | ✅ Bisa cache broadcast |

**Alternatif: Long Polling dengan "hold until change"**

Variasi long polling di mana server menahan response sampai ada perubahan:

```
Client ──GET /position──▶ Server (holds request)
                          ... menunggu perubahan ...
       ◀─────────────────── { now_serving: 31501 } (saat ada perubahan)
Client ──GET /position──▶ Server (holds request)
       ...
```

Masalah dengan pendekatan ini:
1. **Timeout handling kompleks** — harus handle server/proxy/client timeout
2. **Tidak scalable** — server menahan 50.000 pending requests
3. **Tidak seperti SSE** — tidak ada built-in reconnect, tidak ada event typing
4. **Resource usage** — setiap pending request = thread/goroutine blocked

Pada dasarnya, "hold until change" long polling **adalah SSE dengan implementasi
manual dan tanpa keuntungan protokol bawaan**. Jika butuh real-time push, SSE
adalah abstraksi yang tepat.

**Kesimpulan:**

Long Polling dapat memenuhi p95 < 1 detik secara teoritis (dengan interval < 1s),
tetapi **tidak praktis** karena:
1. 50.000 request/detik membebani server dan network
2. CDN tidak dapat membantu (TTL harus sangat pendek)
3. Biaya infrastructure meningkat signifikan
4. SSE mencapai hasil yang sama dengan resource jauh lebih efisien

### Mengapa Bukan WebSocket?

| Aspek                  | WebSocket           | SSE                  |
| ---------------------- | ------------------- | -------------------- |
| Directionality         | Bidirectional       | Unidirectional       |
| Use case fit           | Overkill            | ✅ Tepat             |
| Reconnect              | Manual              | ✅ Built-in          |
| Kompleksitas           | Tinggi              | Rendah               |
| Proxy compatibility    | Mungkin perlu setup | ✅ HTTP standar      |

---

## Detail Teknis

### Tipe Event

| Event      | Payload                                            | Penerima       | Cacheable |
| ---------- | -------------------------------------------------- | -------------- | --------- |
| `stats`    | `{now_serving, throughput, timestamp}`             | Semua customer | ✅ Ya     |
| `promoted` | `{access_token}`                                   | Satu customer  | ❌ Tidak  |

### Format Event SSE

```
event: stats
data: {"now_serving": 31500, "throughput": 12, "timestamp": 1698912000}

event: promoted
data: {"access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."}
```

### Perhitungan Posisi di Client

```javascript
// Client menyimpan ticket_no dari response POST /queue/join (tidak pernah berubah)
const myTicketNo = localStorage.getItem('ticket_no'); // e.g., 48217

// Setiap menerima event 'stats'
eventSource.addEventListener('stats', (e) => {
  const { now_serving, throughput } = JSON.parse(e.data);
  const posisi = myTicketNo - now_serving;
  const etaMenit = posisi / throughput;
  updateUI(posisi, etaMenit);
});

// Saat menerima event 'promoted' — redirect ke halaman booking
eventSource.addEventListener('promoted', (e) => {
  const { access_token } = JSON.parse(e.data);
  window.location.href = `/booking?token=${access_token}`;
});
```

### Arsitektur

```
┌─────────────────────────────────────────────────────────────────────┐
│                           CDN Edge                                  │
│   Cache: event stats (TTL 1-2 detik, identik untuk semua customer)  │
└─────────────────────────────────────────────────────────────────────┘
                              ▲
                              │ SSE stream
                              │
┌─────────────────────────────────────────────────────────────────────┐
│                         SSE Server                                  │
│  - Broadcast event 'stats' setiap 1-2 detik                         │
│  - Kirim event 'promoted' ke customer tertentu via Redis Pub/Sub    │
└─────────────────────────────────────────────────────────────────────┘
                              ▲
                              │ Redis Pub/Sub
                              │
┌─────────────────────────────────────────────────────────────────────┐
│                        Queue Service                                │
│  - Menyimpan state antrian di Redis (ADR-001)                       │
│  - Publish update stats                                             │
│  - Publish event promotion per customer                             │
└─────────────────────────────────────────────────────────────────────┘
```

### Flow Bergabung Antrian (Bagaimana Client Mendapat ticket_no)

```
┌─────────────────────────────────────────────────────────────────────┐
│ 1. User klik "Masuk Antrian"                                        │
│         │                                                           │
│         ▼                                                           │
│ 2. Frontend: POST /queue/join                                       │
│    Body: { event_id, device_fingerprint }                           │
│         │                                                           │
│         ▼                                                           │
│ 3. Backend:                                                         │
│    - INCR queue:{event_id}:issued → menghasilkan ticket_no          │
│    - Buat signed queue_token (JWT berisi ticket_no + event_id)      │
│    - Return: { ticket_no: 48217, queue_token: "eyJ..." }            │
│         │                                                           │
│         ▼                                                           │
│ 4. Frontend menyimpan di localStorage:                              │
│    - ticket_no (untuk perhitungan posisi di UI)                     │
│    - queue_token (untuk autentikasi SSE)                            │
│         │                                                           │
│         ▼                                                           │
│ 5. Frontend connect: EventSource('/queue/stream?token=eyJ...')      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Aksi Implementasi

### Backend

| Aksi                      | Deskripsi                                                      | Endpoint/Komponen        |
| ------------------------- | -------------------------------------------------------------- | ------------------------ |
| Endpoint join antrian     | Terima request join, assign ticket_no, return signed token     | `POST /queue/join`       |
| Endpoint SSE stream       | Return `text/event-stream`, validasi queue_token               | `GET /queue/stream`      |
| Broadcast stats loop      | Publish `now_serving`, `throughput` setiap 1-2 detik           | Background goroutine     |
| Promotion per customer    | Kirim event `promoted` dengan `access_token` saat giliran tiba | Redis Pub/Sub channel    |
| Heartbeat                 | Kirim `:keepalive` comment setiap 30 detik untuk cegah timeout | SSE handler              |
| Endpoint status antrian   | Validasi queue_token, return posisi saat ini (untuk reconnect) | `GET /queue/status`      |

### Frontend

| Aksi                      | Deskripsi                                                      | Komponen                 |
| ------------------------- | -------------------------------------------------------------- | ------------------------ |
| Join antrian              | Call `POST /queue/join`, simpan response di localStorage       | Handler tombol join      |
| Connect SSE               | `new EventSource('/queue/stream?token=...')`                   | Halaman waiting room     |
| Handle event `stats`      | Hitung `posisi = ticket_no - now_serving`, update UI           | SSE event listener       |
| Handle event `promoted`   | Redirect ke `/booking?token=access_token`                      | SSE event listener       |
| Validasi reconnect        | Saat reconnect, call `GET /queue/status` untuk verifikasi      | EventSource `onopen`     |
| UI status koneksi         | Tampilkan indikator "Menghubungkan kembali..." jika putus      | Komponen status          |
| Persist across refresh    | Baca ticket_no/queue_token dari localStorage saat page load    | Inisialisasi halaman     |

---

## Konsekuensi

### Positif

- Memenuhi semua functional requirement (FR-07, FR-08, FR-10)
- Mencapai p95 < 1 detik (NFR-06) dengan delivery berbasis push
- Implementasi sederhana dengan API browser native
- CDN-friendly broadcast mengurangi beban origin server
- Automatic reconnection meningkatkan reliability

### Negatif

- Menahan satu koneksi TCP per client (acceptable di skala yang diharapkan)
- Tidak support Internet Explorer (acceptable — IE sudah EOL)
- Membutuhkan sticky sessions atau Redis Pub/Sub untuk deployment multi-instance

### Risiko & Mitigasi

| Risiko                                             | Likelihood | Impact | Mitigasi                                                              |
| -------------------------------------------------- | ---------- | ------ | --------------------------------------------------------------------- |
| Mobile carrier memutus koneksi SSE yang idle       | Medium     | Medium | Kirim heartbeat setiap 30 detik; client auto-reconnect                |
| Server instance failure memutus semua koneksi      | Low        | High   | Multiple instance di belakang load balancer; client auto-reconnect    |
| CDN tidak support SSE streaming                    | Low        | Medium | Verifikasi CDN support SSE; fallback ke direct origin jika diperlukan |

**Catatan tentang mobile carrier timeout:**
Mobile carrier (terutama di jaringan 3G/4G) sering menerapkan NAT timeout 30-120 detik
untuk koneksi idle guna menghemat resource. Koneksi TCP yang tidak mengirim data dalam
periode ini dapat diputus secara diam-diam tanpa notifikasi ke client maupun server.
Heartbeat interval 30 detik dipilih sebagai nilai konservatif di bawah batas minimum
umum. Referensi: RFC 5765 (SIP Outbound) merekomendasikan keep-alive ≤29 detik untuk
NAT traversal; praktik serupa berlaku untuk long-lived HTTP connections.

---

## Referensi

### Dokumen Internal
- `Tender Rusuh Booking Ticket.pdf` — FR-07, FR-08, FR-10, NFR-05, NFR-06
- ADR-001 — Teknologi Antrian untuk Virtual Waiting Room
- ADR-002 — Mekanisme Penentuan Posisi Antrian per Customer

### Spesifikasi Web
- [MDN: Server-Sent Events](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events) — Dokumentasi API EventSource
- [HTML Living Standard: Server-sent events](https://html.spec.whatwg.org/multipage/server-sent-events.html) — Spesifikasi formal SSE termasuk reconnection behavior
- [MDN: EventSource.readyState](https://developer.mozilla.org/en-US/docs/Web/API/EventSource/readyState) — State management dan auto-reconnect

### Network & Mobile
- [RFC 5765: Session Initiation Protocol (SIP) Outbound](https://datatracker.ietf.org/doc/html/rfc5765) — Section 3.5.1 merekomendasikan keep-alive ≤29 detik untuk NAT traversal
- [Cloudflare: Server-Sent Events](https://developers.cloudflare.com/workers/examples/websockets/#server-sent-events-sse) — CDN SSE support reference
