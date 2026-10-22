# ADR-004: Pemilihan Compute Service untuk Backend

**Status**: Accepted  
**Tanggal**: 2026-10-22  
**Decider**: Backend Team

---

## Konteks

Sistem Rusuh Booking Ticket memiliki workload dengan karakteristik berbeda yang
membutuhkan compute service yang tepat. Dengan keputusan **Adaptive Polling** di ADR-003
(bukan SSE), semua endpoint menjadi stateless request-response.

### Requirements

| ID     | Requirement                                                      |
| ------ | ---------------------------------------------------------------- |
| NFR-03 | Menangani flash crowd 200.000-300.000 user concurrent            |
| NFR-05 | Sistem mampu menangani minimal 1000 request per detik            |
| NFR-06 | Pembaruan posisi antrian memiliki latensi rendah                 |
| NFR-07 | Sistem harus mampu menangani lonjakan trafik mendadak            |

---

## Workload Analysis

### Dengan Adaptive Polling (ADR-003)

Semua endpoint adalah **stateless HTTP request-response**:

| Endpoint | Karakteristik | Volume (300k users) |
|----------|---------------|---------------------|
| `GET /queue/status` | Read-only, CDN-cacheable | ~1.125 req/sec ke origin |
| `POST /queue/join` | Write, short-lived | ~50k total per event |
| `POST /queue/claim-token` | Write, critical path | ~10k total per event |
| `POST /booking/*` | Write, transactional | ~10k total per event |
| `POST /payment/*` | Write, external call | ~10k total per event |

**Key insight**: Dengan CDN absorbing 97% of `/queue/status` requests, origin load
sangat rendah (~1.125 req/sec untuk polling, bukan 37.500 req/sec).

---

## Opsi yang Dipertimbangkan

### Opsi A: AWS Lambda + API Gateway — **Dipilih**

**Karakteristik:**
- Serverless, auto-scale, pay-per-request
- Eksekusi maksimum: 15 menit (lebih dari cukup untuk request-response)
- Cold start: 100ms-1s

**Kekuatan:**
- **Scale to zero** — tidak ada biaya saat idle
- **Auto-scaling tanpa konfigurasi** — handles flash crowd naturally
- **Managed infrastructure** — tidak perlu patch/maintain
- Cocok untuk **semua endpoint** karena semuanya stateless

**Kelemahan:**
- Cold start pada traffic spike (mitigasi: provisioned concurrency)
- API Gateway hard limit 29 detik (tidak masalah untuk use case ini)

### Opsi B: Amazon ECS Fargate

**Karakteristik:**
- Container-based, always-on atau auto-scaling
- Billing per vCPU-hour + GB-memory-hour

**Kekuatan:**
- No cold start
- Predictable performance
- Lebih cocok untuk SSE/persistent connections

**Kelemahan:**
- **Tidak scale to zero** — minimum cost ~$36/bulan per service
- Overkill untuk stateless workload dengan CDN caching
- Scaling lebih lambat (30-60s untuk new tasks)

### Opsi C: Amazon EC2

**Kekuatan:**
- Paling murah untuk sustained workload
- Full control

**Kelemahan:**
- **Operational overhead tinggi**
- Tidak scale to zero
- Manual scaling management

### Opsi D: AWS App Runner

**Kekuatan:**
- Simpler than ECS
- Can scale to zero (with cold start)

**Kelemahan:**
- Cold start lebih lama dari Lambda
- Kurang mature tooling

---

## Keputusan

**AWS Lambda + API Gateway** untuk semua backend services.

---

## Alasan

### 1. Semua Workload Stateless

Dengan keputusan Adaptive Polling (ADR-003), tidak ada persistent connections.
Semua endpoint adalah standard HTTP request-response, ideal untuk Lambda.

### 2. Scale to Zero = Minimal Cost untuk Assessment

Tech assessment dengan frekuensi event rendah. Lambda biaya $0 saat idle, vs
Fargate minimum ~$36-72/bulan.

### 3. CDN Reduces Origin Load Dramatically

`/queue/status` (highest volume) di-cache CDN dengan 97% hit rate:
- Total requests: 37.500/sec
- Origin requests: ~1.125/sec

Lambda handles 1.125 req/sec dengan mudah.

### 4. Auto-scaling Handles Flash Crowd

Lambda auto-scales ke 1000+ concurrent executions tanpa konfigurasi.
Flash crowd 300k users → ~1.125 req/sec ke origin → well within Lambda capacity.

### 5. Cold Start Acceptable

Untuk assessment scope:
- Typical cold start: 100-300ms (Go/Node dengan optimized package)
- Acceptable untuk non-real-time use case
- Production dapat gunakan Provisioned Concurrency jika needed

---

## Analisis Biaya: 300.000 Concurrent Users

### Per-Event Cost (2 jam flash crowd)

| Endpoint | Requests | Avg Duration | Memory | GB-seconds | Cost |
|----------|----------|--------------|--------|------------|------|
| `/queue/status` | 8.1M* | 30ms | 128MB | 30.375 | $0.51 |
| `/queue/join` | 50k | 50ms | 256MB | 625 | $0.01 |
| `/queue/claim-token` | 15k | 100ms | 256MB | 375 | $0.01 |
| `/booking/*` | 30k | 200ms | 256MB | 1.500 | $0.03 |
| `/payment/*` | 20k | 300ms | 256MB | 1.500 | $0.03 |
| **Total Lambda** | | | | | **~$0.59** |

*8.1M = 1.125 req/sec × 7.200 sec (2 jam)

| Service | Cost per Event |
|---------|----------------|
| Lambda duration | ~$0.59 |
| Lambda requests | ~$0.03 |
| API Gateway | ~$29 |
| **Total compute** | **~$30** |

### Perbandingan dengan Fargate (Always-On)

| Aspect | Lambda | Fargate |
|--------|--------|---------|
| Per-event cost | ~$30 | ~$1.16 |
| Monthly idle cost | $0 | ~$72 |
| 1 event/month | ~$30 | ~$73 |
| 2 events/month | ~$60 | ~$74 |
| 4 events/month | ~$120 | ~$77 |

**Breakeven: ~2.5 events/bulan**

Untuk assessment dengan <2 events/bulan, Lambda lebih ekonomis.

### Total System Cost per Event (2 jam, 300k users)

| Component | Cost |
|-----------|------|
| Lambda + API Gateway | ~$30 |
| CloudFront (requests + data) | ~$252 |
| ElastiCache Redis | ~$0.50 |
| Data Transfer | ~$5 |
| **Total per event** | **~$288** |

### Monthly Cost Projection

| Events/month | Total Cost |
|--------------|------------|
| 0 (idle) | ~$12 (Redis only) |
| 1 | ~$300 |
| 2 | ~$588 |
| 4 | ~$1.164 |

---

## Arsitektur

```
┌─────────────────────────────────────────────────────────────────────┐
│                         CloudFront CDN                              │
│   • Cache GET /queue/status (TTL 1s) — 97% hit rate                 │
│   • Pass-through untuk POST endpoints                               │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        API Gateway (HTTP API)                       │
│   • Rate limiting                                                   │
│   • Request validation                                              │
│   • JWT authorization                                               │
└─────────────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐     ┌───────────────┐     ┌───────────────┐
│ Queue Lambda  │     │Booking Lambda │     │Payment Lambda │
│               │     │               │     │               │
│ • /queue/*    │     │ • /booking/*  │     │ • /payment/*  │
│ • join        │     │ • create      │     │ • initiate    │
│ • status      │     │ • confirm     │     │ • callback    │
│ • claim-token │     │ • cancel      │     │ • status      │
└───────────────┘     └───────────────┘     └───────────────┘
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        ElastiCache Redis                            │
│   • now_serving counter                                             │
│   • Slot tracking                                                   │
│   • Token storage                                                   │
│   • Distributed locking                                             │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        Amazon SQS + Lambda                          │
│   • Email notification queue                                        │
│   • Booking expiry processor                                        │
│   • Payment callback processor                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Detail Teknis

### Lambda Configuration

```yaml
# SAM template
QueueStatusFunction:
  Type: AWS::Serverless::Function
  Properties:
    Handler: bootstrap
    Runtime: provided.al2023
    Architectures: [arm64]
    MemorySize: 128
    Timeout: 10
    Environment:
      Variables:
        REDIS_URL: !Ref RedisEndpoint
    Events:
      Api:
        Type: HttpApi
        Properties:
          Path: /queue/status
          Method: GET

QueueJoinFunction:
  Type: AWS::Serverless::Function
  Properties:
    Handler: bootstrap
    Runtime: provided.al2023
    Architectures: [arm64]
    MemorySize: 256
    Timeout: 10
    Events:
      Api:
        Type: HttpApi
        Properties:
          Path: /queue/join
          Method: POST

ClaimTokenFunction:
  Type: AWS::Serverless::Function
  Properties:
    Handler: bootstrap
    Runtime: provided.al2023
    Architectures: [arm64]
    MemorySize: 256
    Timeout: 10
    ReservedConcurrentExecutions: 100  # Limit untuk protect Redis
    Events:
      Api:
        Type: HttpApi
        Properties:
          Path: /queue/claim-token
          Method: POST
```

### Cold Start Optimization

```go
// Use minimal dependencies
// Lazy-load non-critical components
// Keep deployment package small (<10MB)
// Use ARM64 for faster cold starts

var redisClient *redis.Client
var once sync.Once

func init() {
    // Initialize only essentials
}

func handler(ctx context.Context, event events.APIGatewayV2HTTPRequest) (events.APIGatewayV2HTTPResponse, error) {
    once.Do(func() {
        // Lazy init Redis connection
        redisClient = redis.NewClient(&redis.Options{
            Addr: os.Getenv("REDIS_URL"),
        })
    })
    // ... handle request
}
```

### API Gateway Configuration

```yaml
# HTTP API (not REST API) for lower latency and cost
HttpApi:
  Type: AWS::Serverless::HttpApi
  Properties:
    StageName: prod
    CorsConfiguration:
      AllowOrigins:
        - "https://rusuh.ticket.example.com"
      AllowMethods:
        - GET
        - POST
      AllowHeaders:
        - Content-Type
        - Authorization
    DefaultRouteSettings:
      ThrottlingBurstLimit: 5000
      ThrottlingRateLimit: 10000
```

---

## Background Jobs

### Booking Expiry Processor

```yaml
BookingExpiryFunction:
  Type: AWS::Serverless::Function
  Properties:
    Handler: bootstrap
    Runtime: provided.al2023
    MemorySize: 256
    Timeout: 60
    Events:
      Schedule:
        Type: Schedule
        Properties:
          Schedule: rate(1 minute)
          Description: Check and expire stale bookings
```

### Email Notification Processor

```yaml
EmailNotificationFunction:
  Type: AWS::Serverless::Function
  Properties:
    Handler: bootstrap
    Runtime: provided.al2023
    MemorySize: 256
    Timeout: 30
    Events:
      SQS:
        Type: SQS
        Properties:
          Queue: !GetAtt EmailQueue.Arn
          BatchSize: 10
```

---

## Konsekuensi

### Positif

- **Scale to zero** — biaya minimal saat tidak ada event
- **Auto-scaling** — handles flash crowd 300k tanpa konfigurasi
- **Operational simplicity** — no servers to manage
- **Pay-per-use** — biaya proporsional dengan actual usage
- **Consistent with ADR-003** — semua workload stateless, cocok untuk Lambda

### Negatif

- Cold start pada traffic spike (100-300ms)
- API Gateway cost dapat signifikan pada high volume
- 29 detik timeout limit (tidak masalah untuk use case ini)

### Risiko & Mitigasi

| Risiko | Likelihood | Impact | Mitigasi |
|--------|------------|--------|----------|
| Cold start burst | Medium | Low | Provisioned Concurrency untuk critical paths |
| Lambda throttling | Low | High | Request Reserved Concurrency increase |
| Redis connection exhaustion | Medium | High | Connection pooling, ReservedConcurrency limit |
| API Gateway timeout | Low | Medium | Optimize function, async for long operations |

---

## Catatan untuk Production

Jika sistem berevolusi ke production dengan high-frequency events (≥2.5/bulan),
pertimbangkan:

1. **Fargate untuk core services** — lower per-request cost
2. **Provisioned Concurrency** — eliminate cold starts
3. **Reserved Capacity** — discount pricing
4. **Hybrid architecture** — Lambda untuk bursty, Fargate untuk steady-state

---

## Referensi

### AWS Documentation
- [Lambda quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)
- [API Gateway HTTP APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api.html)
- [Lambda Provisioned Concurrency](https://docs.aws.amazon.com/lambda/latest/dg/provisioned-concurrency.html)

### AWS Pricing (ap-southeast-1, Oktober 2026)
- Lambda: $0.20 per 1M requests, $0.0000166667 per GB-second
- API Gateway HTTP API: $1.00 per 1M requests (first 300M), $0.90 thereafter
- ElastiCache Redis (cache.t3.micro): ~$0.017/hour

### ADR Terkait
- ADR-001 — Teknologi Antrian untuk Virtual Waiting Room
- ADR-002 — Mekanisme Penentuan Posisi Antrian per Customer
- ADR-003 — Mekanisme Pengiriman Update Posisi Antrian ke Client (Adaptive Polling)
