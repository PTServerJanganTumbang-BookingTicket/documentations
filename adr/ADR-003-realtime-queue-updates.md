# ADR-003: Mekanisme Pengiriman Update Posisi Antrian ke Client

**Status**: Accepted  
**Tanggal**: 2026-10-22  
**Decider**: Frontend & Backend Team

---

## Konteks

Setelah teknologi antrian (ADR-001) dan mekanisme penentuan posisi (ADR-002) ditetapkan,
pertanyaan berikutnya adalah **bagaimana server mengirimkan update `now_serving` ke client
secara real-time**. Ini adalah keputusan yang berdampak langsung pada pengalaman pengguna
di Virtual Waiting Room dan kemampuan sistem memenuhi requirement latensi.

### Requirements

| ID     | Requirement                                                                 |
| ------ | --------------------------------------------------------------------------- |
| FR-07  | Posisi antrian diperbarui otomatis setiap antrian di depan berkurang        |
| FR-08  | Customer otomatis dipindahkan ke Detail Ticket Page saat gilirannya tiba    |
| FR-10  | Posisi antrian tetap konsisten saat customer me-refresh halaman             |
| NFR-03 | Menangani flash crowd 200.000-300.000 user concurrent                       |
| NFR-06 | Pembaruan posisi antrian memiliki latensi rendah (p95 < 1 detik untuk posisi ≤100) |

Dari ADR-002, kita sudah memutuskan bahwa server menyebarkan nilai `now_serving` yang
**identik untuk semua customer**, dan client menghitung posisinya sendiri
(`posisi = ticket_no − now_serving`). ADR ini membahas **bagaimana nilai tersebut
dikirimkan ke client**.

---

## Opsi yang Dipertimbangkan

### Opsi A: Server-Sent Events (SSE)

Koneksi unidirectional persisten menggunakan HTTP standar. Server mendorong event ke client.

**Kekuatan:**
- Built-in reconnect di browser (`EventSource` API)
- Latensi mendekati nol (~0ms push)
- Beban komputasi O(1) — broadcast satu nilai global

**Kelemahan:**
- **Tidak dapat memanfaatkan CDN edge caching** — SSE adalah stream persisten
- 300.000 koneksi persisten langsung ke origin
- Biaya always-on tinggi (~$475/bulan untuk 300k capacity)
- Mobile carrier dapat memutus koneksi idle

### Opsi B: WebSocket

Koneksi bidirectional persisten.

**Kekuatan:**
- Latensi terendah
- Bidirectional communication

**Kelemahan:**
- Overkill untuk use case read-only
- Kompleksitas lebih tinggi dari SSE
- Sama seperti SSE: tidak CDN-cacheable, koneksi persisten mahal

### Opsi C: Adaptive Polling — **Dipilih**

Client melakukan HTTP polling dengan interval yang menyesuaikan berdasarkan posisi antrian.

**Kekuatan:**
- **CDN-cacheable** — response `now_serving` identik dapat di-cache di edge
- Scale to zero saat tidak ada event
- Universal browser support
- Failure blast radius rendah

**Kelemahan:**
- Latensi inherent (1-10 detik tergantung tier)
- Request volume tinggi (tapi di-absorb CDN)

---

## Analisis Biaya: 300.000 Concurrent Users

### Distribusi User dan Polling Interval

| Tier | Posisi | Interval | Est. Users | Requests/sec |
|------|--------|----------|------------|--------------|
| 1 | 1-10 | 1s | 1.000 | 1.000 |
| 2 | 11-100 | 2s | 7.000 | 3.500 |
| 3 | 101-1.000 | 5s | 40.000 | 8.000 |
| 4 | >1.000 | 10s | 250.000 | 25.000 |
| **Total** | | | **298.000** | **37.500 req/s** |

### CDN Cache Effectiveness

```
now_serving = nilai IDENTIK untuk semua users
Cache TTL: 1 detik

Total requests ke CDN: 37.500/sec
CDN cache hit rate: ~97% (1 miss per TTL per edge location)
Requests ke origin: ~1.125/sec (30 edge locations × 37.5 req/TTL)
```

### Perbandingan Biaya per Event (2 jam flash crowd)

| Komponen | SSE | Adaptive Polling |
|----------|-----|------------------|
| Compute (Fargate/Lambda) | ~$1.16 | ~$0.84 |
| API Gateway | — | ~$47 |
| CDN Requests | — | ~$247 |
| Data Transfer | ~$13 | ~$5 |
| ALB | ~$0.17 | — |
| **Total per event** | **~$14** | **~$300** |

### Perbandingan Biaya Bulanan

| Skenario | SSE | Adaptive Polling |
|----------|-----|------------------|
| Always-on capacity | ~$475/bulan | ~$0 |
| 1 event/bulan | ~$489 | ~$300 |
| 2 events/bulan | ~$503 | ~$600 |
| 4 events/bulan | ~$531 | ~$1.200 |

**Breakeven: ~1.6 events/bulan**

### Perbandingan Non-Biaya

| Aspek | SSE | Adaptive Polling |
|-------|-----|------------------|
| Latensi update | ~0ms | 1-10s (by tier) |
| p95 < 1s (tier 1-10) | ✅ | ✅ (1s interval) |
| p95 < 1s (tier >100) | ✅ | ❌ (2-10s) |
| CDN leverage | ❌ | ✅ |
| Scale to zero | ❌ | ✅ |
| Mobile compatibility | ⚠️ (timeout risk) | ✅ |
| Complexity | Medium | Low |
| Failure blast radius | High | Low |

---

## Keputusan

**Adaptive Polling** dipilih sebagai mekanisme pengiriman update posisi antrian.

---

## Alasan

### 1. CDN-Cacheable = Massive Cost Reduction di Scale

Pada 300.000 users, SSE membutuhkan 300k koneksi persisten ke origin. Dengan polling,
CDN meng-absorb ~97% requests karena `now_serving` identik untuk semua users.

### 2. Scale to Zero untuk Assessment Scope

Untuk tech assessment dengan frekuensi event rendah (<2/bulan), polling **jauh lebih
murah** karena tidak ada biaya always-on. SSE membutuhkan ~$475/bulan capacity stand-by.

### 3. Latensi Acceptable untuk Posisi Jauh

NFR-06 mensyaratkan p95 < 1 detik untuk **posisi ≤100** (tier 1-2). Tier ini menggunakan
interval 1-2 detik, memenuhi requirement. User di posisi >1000 tidak membutuhkan
update per-detik — 10 detik cukup karena mereka masih jauh dari giliran.

### 4. Operational Simplicity

- Tidak perlu manage persistent connections
- Tidak perlu sticky sessions atau Redis Pub/Sub untuk SSE fanout
- Standard HTTP request/response — semua tooling dan monitoring sudah mature
- Failure di satu request tidak cascade ke user lain

### 5. Mobile-Friendly

Mobile carriers sering memutus koneksi idle (NAT timeout 30-120s). SSE membutuhkan
heartbeat dan reconnect logic. Polling naturally resistant terhadap network interruptions.

---

## Detail Teknis

### Polling Tiers

| Posisi | Interval | Rationale |
|--------|----------|-----------|
| 1-10 | 1 detik | Near-admission, UX critical |
| 11-100 | 2 detik | Soon to enter, responsive feel |
| 101-1.000 | 5 detik | Moderate wait, balance cost/UX |
| >1.000 | 10 detik | Long wait, cost efficiency |

### Endpoint

```
GET /queue/status?event_id={event_id}

Response:
{
  "now_serving": 31500,
  "throughput": 12,        // users/menit (untuk ETA calculation)
  "server_time": 1698912000,
  "cache_ttl": 1
}

Headers:
Cache-Control: public, max-age=1
```

### Client Implementation

```javascript
const POLL_INTERVALS = {
  CRITICAL: 1000,   // posisi 1-10
  SOON: 2000,       // posisi 11-100
  MODERATE: 5000,   // posisi 101-1000
  FAR: 10000        // posisi >1000
};

function getInterval(position) {
  if (position <= 10) return POLL_INTERVALS.CRITICAL;
  if (position <= 100) return POLL_INTERVALS.SOON;
  if (position <= 1000) return POLL_INTERVALS.MODERATE;
  return POLL_INTERVALS.FAR;
}

async function pollQueue() {
  const myTicketNo = localStorage.getItem('ticket_no');
  
  const response = await fetch(`/queue/status?event_id=${eventId}`);
  const { now_serving, throughput, server_time } = await response.json();
  
  const position = myTicketNo - now_serving;
  const etaMinutes = position / throughput;
  
  updateUI(position, etaMinutes);
  
  if (position <= 0) {
    // Giliran tiba — claim token
    await claimAccessToken();
    return;
  }
  
  // Schedule next poll with adaptive interval
  const interval = getInterval(position);
  setTimeout(pollQueue, interval);
}
```

### Claim Token (saat giliran tiba)

```
POST /queue/claim-token
Authorization: Bearer {queue_token}

Response (success):
{
  "access_token": "eyJ...",
  "expires_at": 1698912300,
  "redirect_url": "/booking"
}

Response (slot penuh):
{
  "error": "NO_SLOT_AVAILABLE",
  "retry_after": 5
}
```

### Arsitektur

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CloudFront CDN                              │
│   • Cache /queue/status dengan TTL 1 detik                          │
│   • 97% hit rate → origin hanya terima ~1.125 req/sec               │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ ~3% cache miss
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      API Gateway + Lambda                           │
│   • GET /queue/status — read now_serving dari Redis                 │
│   • POST /queue/claim-token — atomic slot acquisition               │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        ElastiCache Redis                            │
│   • now_serving counter per event                                   │
│   • Slot tracking (active bookings)                                 │
│   • queue_token validation                                          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Konsekuensi

### Positif

- Biaya rendah untuk assessment scope (scale to zero)
- CDN absorbs 97% traffic pada 300k scale
- Operational simplicity — standard HTTP, no persistent connections
- Mobile-friendly — no NAT timeout issues
- Failure isolation — satu request gagal tidak affect user lain

### Negatif

- Latensi 1-10 detik (tidak real-time seperti SSE)
- User di posisi jauh mungkin merasa update "lambat"
- CDN request cost tinggi pada frequent events (~$247/event untuk 300k users)

### Risiko & Mitigasi

| Risiko | Likelihood | Impact | Mitigasi |
|--------|------------|--------|----------|
| CDN cache miss spike | Low | Medium | Multiple edge locations, stale-while-revalidate |
| User frustration (update lambat) | Medium | Low | Show "last updated X seconds ago" di UI |
| Thundering herd saat cache expire | Low | Medium | Jittered polling start, staggered TTL |
| Origin overload saat CDN down | Low | High | Origin rate limiting, circuit breaker |

---

## Alternatif yang Dipertimbangkan

### Hybrid: SSE untuk Tier 1-10, Polling untuk sisanya

**Dipertimbangkan** tetapi **ditolak** karena:
- Kompleksitas dua transport mechanism
- Tier 1-10 hanya ~1.000 users — polling 1s sudah cukup responsive
- Marginal UX improvement tidak worth complexity

### SSE dengan Always-On Capacity

**Dipertimbangkan** tetapi **ditolak** untuk assessment scope karena:
- Biaya ~$475/bulan untuk capacity stand-by
- Cocok jika ≥2 major events/bulan
- Dapat dipertimbangkan untuk production dengan traffic tinggi

---

## Catatan untuk Production

Jika sistem berevolusi ke production dengan ≥2 major events/bulan, **SSE layak
dipertimbangkan ulang** karena:

1. Biaya per-event SSE lebih rendah (~$14 vs ~$300)
2. Latensi real-time meningkatkan UX
3. Complexity trade-off lebih acceptable dengan tim yang lebih besar

Keputusan ini dapat di-revisit di ADR terpisah saat requirement berubah.

---

## Referensi

### Dokumen Internal
- `Tender Rusuh Booking Ticket.pdf` — FR-07, FR-08, FR-10, NFR-06
- ADR-001 — Teknologi Antrian untuk Virtual Waiting Room
- ADR-002 — Mekanisme Penentuan Posisi Antrian per Customer
- `2026-10-21-meeting-notes-queue-algorithm.md` — Keputusan adaptive polling

### AWS Pricing (ap-southeast-1, Oktober 2026)
- CloudFront: $0.0090 per 10k HTTPS requests, $0.085/GB transfer
- Lambda: $0.20 per 1M requests, $0.0000166667 per GB-second
- API Gateway: $3.50 per 1M requests
- Fargate: $0.04048 per vCPU-hour, $0.004445 per GB-hour
