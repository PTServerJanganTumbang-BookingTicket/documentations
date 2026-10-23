# ADR-003: Mekanisme Pengiriman Update Posisi Antrian ke Client

**Status**: Accepted  
**Tanggal**: 2026-10-23  
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

### Opsi A: Server-Sent Events (SSE) — **Dipilih**

Koneksi unidirectional persisten menggunakan HTTP standar. Server mendorong event ke client.

**Kekuatan:**
- Latensi mendekati nol (~0ms push) — memenuhi NFR-06 untuk SEMUA posisi
- Built-in reconnect di browser (`EventSource` API)
- Beban komputasi O(1) — broadcast satu nilai global ke semua connections
- Native HTTP — tidak perlu upgrade protocol seperti WebSocket
- Auto-reconnect dengan `Last-Event-ID` untuk resume

**Kelemahan:**
- Koneksi persisten langsung ke origin (tidak CDN-cacheable)
- Biaya always-on untuk capacity (~$475/bulan untuk 300k)
- Mobile carrier dapat memutus koneksi idle (mitigasi: heartbeat)

### Opsi B: WebSocket

Koneksi bidirectional persisten.

**Kekuatan:**
- Latensi terendah
- Bidirectional communication

**Kelemahan:**
- Overkill untuk use case read-only (server → client only)
- Kompleksitas lebih tinggi dari SSE
- Perlu custom reconnect logic (tidak built-in seperti EventSource)

### Opsi C: Adaptive Polling

Client melakukan HTTP polling dengan interval yang menyesuaikan berdasarkan posisi antrian.

**Kekuatan:**
- CDN-cacheable — response `now_serving` identik dapat di-cache di edge
- Scale to zero saat tidak ada event
- Universal browser support

**Kelemahan:**
- Latensi inherent (1-10 detik tergantung tier)
- **Tidak memenuhi NFR-06 untuk posisi >100** (p95 > 1 detik)
- Request volume tinggi meski di-absorb CDN
- Higher cost per event (~$300 vs ~$14 untuk SSE)

---

## Analisis Biaya: 300.000 Concurrent Users

### SSE Architecture

```
300.000 koneksi SSE persisten

Per-instance capacity (Go/Node dengan tuning): ~20.000 koneksi
ECS Fargate tasks needed: 300.000 ÷ 20.000 = 15 tasks (dengan buffer: 20 tasks)
```

### Perbandingan Biaya per Event (2 jam flash crowd)

| Komponen | SSE | Adaptive Polling |
|----------|-----|------------------|
| Compute | ~$1.16 | ~$0.84 |
| API Gateway | — | ~$47 |
| CDN Requests | — | ~$247 |
| Data Transfer | ~$13 | ~$5 |
| ALB | ~$0.17 | — |
| **Total per event** | **~$14** | **~$300** |

### Perbandingan Biaya Bulanan

| Skenario | SSE | Adaptive Polling |
|----------|-----|------------------|
| Always-on capacity | ~$475/bulan | ~$0 |
| Per event cost | ~$14 | ~$300 |

**Breakeven: ~1.6 events/bulan** — SSE lebih murah jika ≥2 events/bulan

### Perbandingan Non-Biaya

| Aspek | SSE | Adaptive Polling |
|-------|-----|------------------|
| Latensi update | ~0ms (instant) | 1-10s |
| p95 < 1s (SEMUA posisi) | ✅ | ❌ |
| NFR-06 compliance | ✅ Full | ⚠️ Partial (tier 1-2 only) |
| UX responsiveness | Excellent | Good (tier 1-2), Fair (tier 3-4) |
| Complexity | Medium | Low |

---

## Keputusan

**Server-Sent Events (SSE)** dipilih sebagai mekanisme pengiriman update posisi antrian.

---

## Alasan

### 1. Full NFR-06 Compliance

NFR-06 mensyaratkan p95 < 1 detik. SSE memberikan latensi ~0ms untuk **semua posisi**,
bukan hanya tier 1-2. Ini memberikan UX yang konsisten di seluruh waiting room.

### 2. Lower Per-Event Cost

Pada ~$14/event vs ~$300/event untuk polling, SSE **21x lebih murah per event**.
Meskipun ada biaya always-on capacity, ini ter-offset jika ada ≥2 events/bulan.

### 3. Better User Experience

- Update posisi instant — user melihat perubahan segera
- Auto `your_turn` notification — tidak perlu menunggu polling interval
- Consistent experience — tidak ada "tier" dengan responsiveness berbeda

### 4. Simpler Client Logic

Dengan SSE, client tidak perlu manage polling intervals atau tier calculation.
Cukup connect ke stream dan react ke events.

### 5. Native Browser Support

`EventSource` API sudah mature dengan built-in:
- Auto-reconnect on connection drop
- `Last-Event-ID` untuk resume dari event terakhir
- No external dependencies

---

## Detail Teknis

### SSE Endpoint

```
GET /queue/stream?token={queue_token}

Headers:
Accept: text/event-stream
```

### Event Types

**1. Status Event** (broadcast setiap 1-2 detik)
```
event: status
data: {"now_serving": 31600, "throughput": 12, "server_time": "2026-10-23T13:10:00Z"}
```

**2. Your Turn Event** (ketika posisi = 0)
```
event: your_turn
data: {"message": "Giliran Anda!", "claim_url": "/queue/claim-token"}
```

**3. Heartbeat Event** (setiap 30 detik untuk keep-alive)
```
event: heartbeat
data: {"server_time": "2026-10-23T13:10:30Z"}
```

**4. Sold Out Event** (jika tiket habis saat menunggu)
```
event: sold_out
data: {"message": "Maaf, tiket sudah habis"}
```

### Client Implementation

```javascript
function connectToQueue(queueToken, ticketNo) {
  const eventSource = new EventSource(`/queue/stream?token=${queueToken}`);
  
  eventSource.addEventListener('status', (e) => {
    const { now_serving, throughput, server_time } = JSON.parse(e.data);
    const position = ticketNo - now_serving;
    const etaMinutes = position / throughput;
    updateUI(position, etaMinutes);
  });
  
  eventSource.addEventListener('your_turn', async (e) => {
    eventSource.close();
    await claimAccessToken();
    window.location.href = '/booking';
  });
  
  eventSource.addEventListener('sold_out', (e) => {
    eventSource.close();
    showSoldOutMessage(JSON.parse(e.data).message);
  });
  
  eventSource.onerror = () => {
    // EventSource auto-reconnects, tapi bisa add custom logic
    console.log('Connection lost, reconnecting...');
  };
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
│                    Application Load Balancer                        │
│   • Sticky sessions untuk SSE connections                           │
│   • Health checks pada /health endpoint                             │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      ECS Fargate (SSE Servers)                      │
│   • 20 tasks × 0.5 vCPU × 2GB memory                                │
│   • ~15.000-20.000 connections per task                             │
│   • Subscribe to Redis Pub/Sub for now_serving updates              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        ElastiCache Redis                            │
│   • now_serving counter per event                                   │
│   • Pub/Sub channel untuk broadcast updates                         │
│   • queue_token validation                                          │
│   • Slot tracking (active bookings)                                 │
└─────────────────────────────────────────────────────────────────────┘
```

### Connection Management

| Aspect | Strategy |
|--------|----------|
| Max connections/task | 20.000 (tuned dengan ulimit dan epoll) |
| Heartbeat interval | 30 detik (before mobile NAT timeout) |
| Reconnect | Built-in EventSource + `Last-Event-ID` |
| Graceful shutdown | Drain connections over 30s before task termination |

---

## Konsekuensi

### Positif

- Latensi ~0ms untuk update posisi — excellent UX
- Full NFR-06 compliance untuk semua posisi
- Lower cost per event (~$14 vs ~$300)
- Simpler client implementation (no polling logic)
- Native browser support dengan auto-reconnect

### Negatif

- Biaya always-on capacity (~$475/bulan)
- Tidak dapat leverage CDN caching
- Perlu manage persistent connections dan Redis Pub/Sub
- Mobile compatibility memerlukan heartbeat untuk NAT timeout

### Risiko & Mitigasi

| Risiko | Likelihood | Impact | Mitigasi |
|--------|------------|--------|----------|
| Connection storm saat reconnect | Medium | High | Exponential backoff di client |
| Mobile NAT timeout | Medium | Medium | Heartbeat 30 detik |
| Redis Pub/Sub bottleneck | Low | High | Redis Cluster, connection pooling |
| Fargate task failure | Low | Medium | Multi-AZ, auto-scaling, health checks |

---

## Catatan Implementasi

### ECS Fargate Tuning

```dockerfile
# Increase file descriptor limits for high connection count
RUN echo "* soft nofile 65535" >> /etc/security/limits.conf
RUN echo "* hard nofile 65535" >> /etc/security/limits.conf
```

### Redis Pub/Sub Pattern

```go
// Publisher (Booking Service)
func publishNowServing(eventID string, nowServing int) {
    redis.Publish(ctx, fmt.Sprintf("queue:%s", eventID), nowServing)
}

// Subscriber (SSE Server)
func subscribeToEvent(eventID string) <-chan int {
    pubsub := redis.Subscribe(ctx, fmt.Sprintf("queue:%s", eventID))
    ch := make(chan int)
    go func() {
        for msg := range pubsub.Channel() {
            nowServing, _ := strconv.Atoi(msg.Payload)
            ch <- nowServing
        }
    }()
    return ch
}
```

---

## Referensi

### Dokumen Internal
- `Tender Rusuh Booking Ticket.pdf` — FR-07, FR-08, FR-10, NFR-06
- ADR-001 — Teknologi Antrian untuk Virtual Waiting Room
- ADR-002 — Mekanisme Penentuan Posisi Antrian per Customer
- `api-contracts.md` — SSE endpoint specification

### AWS Pricing (ap-southeast-1, Oktober 2026)
- Fargate: $0.04048 per vCPU-hour, $0.004445 per GB-hour
- ALB: $0.0225 per hour + $0.008 per LCU-hour
- ElastiCache Redis: ~$0.068 per hour (cache.t3.medium)
- Data Transfer: $0.09 per GB (first 10TB)
