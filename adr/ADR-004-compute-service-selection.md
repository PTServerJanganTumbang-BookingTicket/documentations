# ADR-004: Pemilihan Compute Service untuk Backend

**Status**: In Review  
**Tanggal**: 2026  
**Decider**: Backend Team

---

## Konteks

Sistem Rusuh Booking Ticket memiliki dua jenis workload dengan karakteristik berbeda:

1. **SSE Server** (ADR-003) — Menahan koneksi persisten untuk push update `now_serving`
2. **REST API** — Stateless request-response untuk join queue, booking, payment

Keputusan compute service harus mempertimbangkan kedua workload ini. Tidak semua
AWS compute service cocok untuk koneksi persisten yang dibutuhkan SSE.

Requirement yang mendorong keputusan ini:

| ID     | Requirement                                                      |
| ------ | ---------------------------------------------------------------- |
| NFR-05 | Sistem mampu menangani minimal 1000 request per detik            |
| NFR-06 | Pembaruan posisi antrian memiliki latensi rendah (p95 < 1 detik) |
| NFR-07 | Sistem harus mampu menangani lonjakan trafik mendadak            |

---

## Opsi yang Dipertimbangkan

### Opsi A: AWS Lambda + API Gateway

Serverless compute yang auto-scale berdasarkan request.

**Karakteristik:**
- Eksekusi maksimum: 15 menit per invocation
- Billing: per 1ms execution time × memory allocated
- Concurrent executions: 1000 default (dapat dinaikkan)
- Cold start: 100ms - 1s (tergantung runtime dan package size)

**Kekuatan:**
- Scale to zero — tidak ada biaya saat idle
- Auto-scaling tanpa konfigurasi
- Managed infrastructure — tidak perlu patch/maintain server

**Kelemahan untuk SSE:**
- **Tidak mendukung koneksi persisten** — setiap invocation harus selesai dalam 15 menit
- API Gateway timeout: 29 detik (hard limit)
- Billing tidak efisien untuk long-held connections: 10.000 koneksi × 15 menit = 150.000 menit billing/cycle
- Lambda Response Streaming ada, tapi tetap terbatas 20 menit dan billing per-duration

**Perhitungan biaya SSE dengan Lambda (hipotetis):**
```
Asumsi: 10.000 koneksi SSE, koneksi rata-rata 30 menit, memory 256MB

Invocations: 10.000 × 2/jam = 20.000/jam (reconnect setiap 15 menit)
Duration: 20.000 × 15 menit × 60.000ms = 18.000.000.000 ms/jam
Cost: 18B ms × $0.0000166667/GB-s × 0.25GB = ~$75/jam = ~$1.800/hari

Ini HANYA untuk SSE, belum termasuk REST API.
```

### Opsi B: Amazon ECS Fargate

Container-based compute tanpa manage EC2 instances.

**Karakteristik:**
- Billing: per vCPU-hour + GB-memory-hour
- Minimum: 0.25 vCPU, 0.5 GB memory
- Scaling: task count (horizontal)
- No server management, tapi perlu define task definition

**Kekuatan:**
- **Mendukung koneksi persisten** — tidak ada timeout bawaan
- Predictable pricing untuk long-running workloads
- Auto-scaling via ECS Service Auto Scaling
- Dapat menggunakan Application Load Balancer untuk distribusi koneksi

**Kelemahan:**
- Tidak scale to zero — minimum 1 task running
- Cold start ~30-60 detik saat scaling up task baru
- Lebih kompleks dari Lambda: task definition, service, ALB, target group

**Perhitungan biaya SSE dengan Fargate:**
```
Asumsi: 10.000 koneksi SSE, setiap task handle 5.000 koneksi

Tasks needed: 2 tasks (dengan buffer)
Spec per task: 0.5 vCPU, 1 GB memory
Cost per task: $0.04048/vCPU-hour + $0.004445/GB-hour
             = $0.02024 + $0.004445 = ~$0.025/hour/task
Total: 2 × $0.025 × 24 × 30 = ~$36/bulan

Jauh lebih murah dari Lambda untuk workload ini.
```

### Opsi C: Amazon EC2

Self-managed virtual machines.

**Karakteristik:**
- Billing: per instance-hour (On-Demand) atau Reserved/Spot pricing
- Full control over OS, networking, software
- Scaling: manual atau via Auto Scaling Group

**Kekuatan:**
- **Mendukung koneksi persisten** — full control
- Paling murah untuk sustained workload (terutama Reserved Instances)
- Dapat fine-tune OS untuk high-connection scenarios (ulimit, sysctl)
- Spot Instances dapat menghemat hingga 90% untuk fault-tolerant workloads

**Kelemahan:**
- **Operational overhead tinggi** — patching, monitoring, recovery
- Tidak scale to zero
- Scaling lebih lambat (~2-5 menit untuk launch instance baru)
- Perlu manage: AMI, security groups, key pairs, EBS, dll.

**Perhitungan biaya SSE dengan EC2:**
```
Asumsi: 10.000 koneksi SSE, single t3.small cukup (2 vCPU, 2GB)

On-Demand: $0.0208/hour × 24 × 30 = ~$15/bulan
Reserved (1yr): ~$10/bulan
Spot: ~$5-7/bulan (tapi bisa interrupted)

Paling murah, tapi dengan operational overhead.
```

### Opsi D: AWS App Runner

Simplified container deployment (PaaS-like experience).

**Karakteristik:**
- Billing: per vCPU-hour + GB-memory-hour (mirip Fargate)
- Auto-scaling built-in
- Automatic deployments from source/container
- Managed HTTPS, load balancing

**Kekuatan:**
- **Mendukung koneksi persisten**
- Simplest deployment experience — no ALB/task def/service config
- Scale to zero available (dengan cold start trade-off)
- Built-in CI/CD dari GitHub/ECR

**Kelemahan:**
- Kurang fleksibel dari ECS (no sidecar, limited networking options)
- Scale to zero = cold start saat traffic datang
- Tidak bisa fine-tune connection handling sedetail ECS/EC2

**Perhitungan biaya SSE dengan App Runner:**
```
Mirip Fargate: ~$36/bulan untuk workload serupa
Dengan scale-to-zero: bisa lebih murah saat off-peak, tapi ada cold start penalty
```

---

## Perbandingan

### Kapabilitas

| Kapabilitas                    | Lambda        | ECS Fargate   | EC2           | App Runner    |
| ------------------------------ | ------------- | ------------- | ------------- | ------------- |
| Koneksi persisten (SSE)        | ❌ Tidak      | ✅ Ya         | ✅ Ya         | ✅ Ya         |
| Scale to zero                  | ✅ Ya         | ❌ Tidak      | ❌ Tidak      | ⚠️ Opsional   |
| Auto-scaling                   | ✅ Otomatis   | ✅ Configurable| ⚠️ ASG setup | ✅ Otomatis   |
| Operational overhead           | Sangat rendah | Rendah        | Tinggi        | Sangat rendah |
| Cold start                     | 100ms-1s      | 30-60s        | 2-5 menit     | 5-30s         |
| Fine-tune connection handling  | ❌ Tidak      | ⚠️ Terbatas   | ✅ Penuh      | ❌ Tidak      |
| VPC integration                | ✅ Ya         | ✅ Ya         | ✅ Ya         | ⚠️ Terbatas   |

### Biaya (estimasi 10.000 koneksi SSE, 24/7)

| Service      | Biaya/bulan | Catatan                                    |
| ------------ | ----------- | ------------------------------------------ |
| Lambda       | ~$1.800+    | Tidak praktis untuk SSE (billing per-ms)   |
| ECS Fargate  | ~$36        | Predictable, no server management          |
| EC2 On-Demand| ~$15        | Perlu manage server                        |
| EC2 Reserved | ~$10        | Commitment 1 tahun                         |
| App Runner   | ~$36        | Simplest setup                             |

### Perhitungan Biaya Lambda untuk REST API (Booking Flow)

Berbeda dengan SSE, Lambda **sangat cost-effective** untuk REST API stateless.

**Asumsi skenario event:**

| Parameter | Nilai | Keterangan |
|-----------|-------|------------|
| Durasi event | 2 jam | Typical flash sale window |
| Total tiket | 10.000 | Contoh ukuran event |
| Conversion rate | 30% | Tidak semua selesai booking |
| Rata-rata attempt/user | 2 | Retry, koreksi form |
| User dalam antrian | 50.000 | 5x jumlah tiket (high demand) |

**API calls per booking flow:**

| Kategori | Calls/user | Total calls | Avg duration | Memory |
|----------|------------|-------------|--------------|--------|
| Join antrian | 1 | 50.000 | 50ms | 256MB |
| Validasi status | 10 | 500.000 | 30ms | 128MB |
| Buat booking | 1.5 | 15.000 | 200ms | 256MB |
| Cek booking | 2 | 20.000 | 50ms | 128MB |
| Inisiasi payment | 1.2 | 12.000 | 300ms | 256MB |
| Payment callback | 1 | 10.000 | 150ms | 256MB |
| **Total** | | **~607.000** | | |

**Perhitungan duration (GB-seconds):**

```
Join antrian:     50.000 × 0.050s × 0.25GB   =    625 GB-s
Validasi status: 500.000 × 0.030s × 0.125GB  =  1.875 GB-s
Buat booking:     15.000 × 0.200s × 0.25GB   =    750 GB-s
Cek booking:      20.000 × 0.050s × 0.125GB  =    125 GB-s
Inisiasi payment: 12.000 × 0.300s × 0.25GB   =    900 GB-s
Payment callback: 10.000 × 0.150s × 0.25GB   =    375 GB-s
──────────────────────────────────────────────────────────
Total:                                         4.650 GB-s
```

**Biaya per event (Lambda pricing ap-southeast-1):**

```
Request cost:  $0.20 per 1 juta requests
Duration cost: $0.0000166667 per GB-second

Requests:  607.000 × $0.20/1.000.000     = $0.12
Duration:  4.650 × $0.0000166667         = $0.08
─────────────────────────────────────────────────
Total per event:                          ~$0.20
```

**Biaya bulanan berdasarkan frekuensi event:**

| Events/bulan | Biaya Lambda | Catatan |
|--------------|--------------|---------|
| 1 event | $0.20 | Single concert sale |
| 4 events | $0.80 | Weekly events |
| 10 events | $2.00 | Busy month |
| 30 events | $6.00 | Daily events |

**Perbandingan dengan Fargate untuk REST API:**

| Aspek | Lambda | Fargate (always on) |
|-------|--------|---------------------|
| Biaya idle | $0 | ~$36/bulan |
| Biaya 4 events/bulan | $0.80 | ~$36/bulan |
| Scale to zero | ✅ Ya | ❌ Tidak |
| Auto-scaling | ✅ Otomatis | ⚠️ Perlu config |

**Kesimpulan:** Lambda menghemat **~97% biaya** untuk REST API dengan traffic bursty/event-driven.

### Total Biaya Arsitektur (Hybrid: SSE + REST)

| Komponen | Service | Biaya/bulan |
|----------|---------|-------------|
| SSE Server | ECS Fargate | ~$36 |
| REST API | Lambda | ~$2-6 |
| API Gateway | - | ~$2 (600k requests) |
| ElastiCache Redis | cache.t3.micro | ~$12 |
| ALB | Application LB | ~$16 + data |
| CloudFront | - | ~$1-5 (tergantung traffic) |
| **Total (hybrid)** | | **~$70-80/bulan** |

**Perbandingan dengan single Fargate:**

| Arsitektur | Biaya/bulan | Trade-off |
|------------|-------------|-----------|
| Single Fargate | ~$65-75 | Simpler, no scale-to-zero |
| Hybrid (Lambda + Fargate) | ~$70-80 | Complex, REST scales to zero |

Untuk assessment dengan traffic rendah, perbedaan biaya tidak signifikan.

### Kesesuaian per Workload

| Workload              | Lambda | ECS Fargate | EC2  | App Runner |
| --------------------- | ------ | ----------- | ---- | ---------- |
| SSE Server            | ❌     | ✅          | ✅   | ✅         |
| REST API (stateless)  | ✅     | ✅          | ✅   | ✅         |
| Background jobs       | ✅     | ✅          | ✅   | ❌         |
| Event-driven triggers | ✅     | ⚠️          | ⚠️   | ❌         |

---

## Keputusan

**Arsitektur hybrid** dipilih:

1. **SSE Server → ECS Fargate** (atau App Runner untuk simplicity)
2. **REST API → AWS Lambda + API Gateway**
3. **Background jobs (admission controller, expiry sweeper) → AWS Lambda**

### Alasan Pemilihan Hybrid

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CloudFront                                    │
│                    (CDN + SSL termination)                              │
└─────────────────────────────────────────────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
            ▼                 ▼                 ▼
┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐
│   SSE Endpoint    │ │  Queue Endpoints  │ │ Booking/Payment   │
│                   │ │                   │ │    Endpoints      │
│   ECS Fargate     │ │    Lambda         │ │     Lambda        │
│   atau App Runner │ │ + API Gateway     │ │  + API Gateway    │
│                   │ │                   │ │                   │
│ • Persistent conn │ │ • Stateless       │ │ • Stateless       │
│ • Long-running    │ │ • Scale to 0      │ │ • Scale to 0      │
│ • Predictable $   │ │ • Pay per req     │ │ • Pay per req     │
└───────────────────┘ └───────────────────┘ └───────────────────┘
            │                 │                 │
            └─────────────────┼─────────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │   ElastiCache     │
                    │     (Redis)       │
                    │                   │
                    │ • Queue state     │
                    │ • Pub/Sub events  │
                    │ • Grants/tokens   │
                    └───────────────────┘
```

### Mengapa Tidak Full Lambda?

Lambda tidak dapat menahan koneksi SSE yang dibutuhkan untuk real-time updates:
- API Gateway hard limit: 29 detik
- Lambda max: 15 menit (terlalu pendek untuk session waiting room)
- Billing per-ms membuat long-held connections sangat mahal

### Mengapa Tidak Full ECS/EC2?

Stateless REST API lebih cocok di Lambda:
- Scale to zero saat tidak ada traffic (hemat biaya off-peak)
- Auto-scaling tanpa konfigurasi
- Tidak perlu manage capacity untuk bursty traffic

### Mengapa ECS Fargate untuk SSE, bukan EC2?

- Operational overhead lebih rendah
- Tidak perlu patch OS, manage AMI
- Scaling lebih mudah (adjust task count)
- Biaya tidak jauh berbeda untuk workload ini (~$36 vs ~$15-20/bulan)
- Assessment scope: simplicity lebih penting dari cost optimization maksimal

### Kapan Pilih App Runner vs ECS Fargate?

| Pilih App Runner jika:              | Pilih ECS Fargate jika:                  |
| ----------------------------------- | ---------------------------------------- |
| Tim kecil, perlu deploy cepat       | Butuh sidecar containers                 |
| Tidak butuh VPC peering kompleks    | Butuh fine-grained networking            |
| OK dengan default scaling behavior  | Butuh custom scaling policies            |
| Aplikasi sederhana (single container)| Multi-container architecture             |

Untuk assessment ini, **App Runner** cukup dan lebih sederhana. Untuk production
dengan requirement networking kompleks, **ECS Fargate** lebih fleksibel.

---

## Detail Teknis

### ECS Fargate Task Definition (SSE Server)

```json
{
  "family": "rusuh-sse-server",
  "cpu": "512",
  "memory": "1024",
  "networkMode": "awsvpc",
  "containerDefinitions": [
    {
      "name": "sse-server",
      "image": "xxx.dkr.ecr.ap-southeast-1.amazonaws.com/rusuh-sse:latest",
      "portMappings": [
        { "containerPort": 8080, "protocol": "tcp" }
      ],
      "environment": [
        { "name": "REDIS_URL", "value": "redis://xxx.cache.amazonaws.com:6379" }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 5,
        "retries": 3
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/rusuh-sse-server",
          "awslogs-region": "ap-southeast-1",
          "awslogs-stream-prefix": "sse"
        }
      }
    }
  ]
}
```

### Lambda Function (REST API)

```yaml
# serverless.yml atau SAM template
QueueJoinFunction:
  Type: AWS::Serverless::Function
  Properties:
    Handler: cmd/queue-join/main
    Runtime: provided.al2
    Architectures: [arm64]
    MemorySize: 256
    Timeout: 10
    Environment:
      Variables:
        REDIS_URL: !Ref RedisEndpoint
    Events:
      Api:
        Type: Api
        Properties:
          Path: /queue/join
          Method: POST
```

### Auto Scaling Configuration

**ECS Service Auto Scaling:**
```
- Target: CPU utilization 70%
- Min tasks: 2
- Max tasks: 10
- Scale-out cooldown: 60s
- Scale-in cooldown: 300s
```

**Lambda:**
- Auto-scales automatically (no config needed)
- Reserved concurrency dapat di-set jika perlu limit

---

## Aksi Implementasi

### Infrastructure

| Aksi                          | Service        | Komponen                           |
| ----------------------------- | -------------- | ---------------------------------- |
| Setup ECS Cluster             | ECS Fargate    | Cluster, Service, Task Definition  |
| Setup ALB                     | ECS            | Target Group, Listener, Health Check|
| Setup Lambda functions        | Lambda         | Functions, API Gateway             |
| Setup ElastiCache Redis       | ElastiCache    | Cluster, Security Group            |
| Setup CloudFront              | CloudFront     | Distribution, Origins, Behaviors   |
| Setup VPC                     | VPC            | Subnets, Security Groups, NAT      |

### CI/CD

| Aksi                          | Tool           | Deskripsi                          |
| ----------------------------- | -------------- | ---------------------------------- |
| Container build & push        | ECR + CodeBuild| Build image, push ke ECR           |
| Lambda deployment             | SAM/Serverless | Deploy functions                   |
| ECS deployment                | CodeDeploy     | Blue/green atau rolling update     |
| Infrastructure as Code        | Terraform/CDK  | Manage semua resources             |

---

## Konsekuensi

### Positif

- SSE dapat berjalan dengan koneksi persisten (requirement terpenuhi)
- REST API scale to zero saat off-peak (hemat biaya)
- Operational overhead rendah (Fargate + Lambda = minimal server management)
- Scaling otomatis untuk kedua workload
- Separation of concerns: SSE server dan REST API dapat di-deploy independen

### Negatif

- Kompleksitas arsitektur lebih tinggi dari single-service
- Dua deployment pipeline (container + Lambda)
- Perlu manage networking antara services (VPC, security groups)

### Risiko & Mitigasi

| Risiko                                    | Likelihood | Impact | Mitigasi                                           |
| ----------------------------------------- | ---------- | ------ | -------------------------------------------------- |
| ECS task crash memutus semua SSE koneksi  | Low        | High   | Multiple tasks + ALB health check + client reconnect|
| Lambda cold start saat traffic spike      | Medium     | Low    | Provisioned concurrency untuk critical functions   |
| Redis failure memutus koordinasi          | Low        | High   | ElastiCache Multi-AZ + automatic failover          |
| Cost overrun saat unexpected traffic      | Medium     | Medium | AWS Budgets alert + Lambda concurrency limits      |

---

## Referensi

### AWS Documentation
- [AWS Lambda quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html) — Timeout, memory, concurrency limits
- [API Gateway quotas](https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html) — Integration timeout 29 detik
- [ECS Fargate pricing](https://aws.amazon.com/fargate/pricing/) — vCPU dan memory pricing
- [App Runner pricing](https://aws.amazon.com/apprunner/pricing/) — Compute dan memory pricing
- [EC2 pricing](https://aws.amazon.com/ec2/pricing/) — On-Demand, Reserved, Spot pricing

### ADR Terkait
- ADR-001 — Teknologi Antrian untuk Virtual Waiting Room
- ADR-002 — Mekanisme Penentuan Posisi Antrian per Customer
- ADR-003 — Mekanisme Pengiriman Update Posisi Antrian ke Client (SSE)
