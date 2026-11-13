# Rusuh Booking Tiket — Dokumentasi

Sistem penjualan tiket konser online yang tahan *flash crowd*: ratusan ribu orang datang bersamaan saat penjualan
dibuka, sementara tiketnya hanya ribuan. Customer melewati empat halaman: **Home → Waiting Room → Detail Ticket →
Payment**. Halaman pembayaran web dipakai ulang di dalam aplikasi mobile lewat WebView.

Repo ini berisi rekaman meeting/diskusi, requirement, keputusan arsitektur, kontrak API, dan desain sistem. Kode ada di tiga repo terpisah
(lihat [Repo kode](#repo-kode)).

**Tim — PT. Server Jangan Tumbang**

| Nama | Peran |
|---|---|
| Ahmad Muzaki | Web |
| Akbar T Hartomo | Web |
| Mochamad Rosyid Ridho | Mobile |
| Handoko D Wicaksono | Backend |
| Erma S Nurmasyita | Backend |

---

## Guidance

| Urutan | Baca | Untuk memahami |
|---|---|---|
| 1 | [`requirement/requirements.md`](requirement/requirements.md) | Apa yang dibangun: Functional Requirement (FR), Non Functional Requirement (NFR), aturan bisnis, dan hasil diskusi yang final |
| 2 | [`high-level-design.md`](high-level-design.md) | Gambaran sistem sesuai implementasi: komponen, alur customer, data, deployment |
| 3 | [`adr/ADR-005-final-architecture-decision.md`](adr/ADR-005-final-architecture-decision.md) | Penjelasan dari keputusan arsitektur keseluruhan |
| 4 | [`api/api-contracts.md`](api/api-contracts.md) | Kontrak API: perilaku setiap endpoint. Bentuk respons persisnya di [`api/rusuh-booking-apidog.json`](api/rusuh-booking-apidog.json) (Apidog) atau [`api/rusuh-booking-openapi.yaml`](api/rusuh-booking-openapi.yaml) (OpenAPI 3.1) |
| 5 | [`infra-cost-model/rusuh-queue-cost-model.xlsx`](infra-cost-model/rusuh-queue-cost-model.xlsx) | Perhitungan biaya pengiriman update antrian: perbandingan WebSocket, polling, dan SSE (titik impas SSE) |

Ringkasan cara kerjanya dalam satu analogi: antrian bank. Setiap orang mengambil nomor, satu papan menampilkan "nomor
yang sedang dilayani" untuk semua orang, dan satu petugas memanggil nomor berikutnya hanya bila ada tempat dan masih ada
tiket.

---

## List Dokumentasi

| Folder / file | Isi |
|---|---|
| [`requirement/`](requirement/) | Tender asli (PDF), perubahan requirement resmi, requirement final, rencana implementasi awal |
| [`high-level-design.md`](high-level-design.md) | Desain sistem tingkat tinggi (v3, sesuai implementasi). Versi sebelumnya (v2.1.0) ada di [riwayat git](https://github.com/PTServerJanganTumbang-BookingTicket/documentations/blob/9408e55/architecture/high-level-design.md) |
| [`adr/`](adr/) | Architecture Decision Records. Versi lengkap ADR-005 dan ADR-006 sebelum diringkas dapat dibuka dari tautan di header masing-masing |
| [`api/`](api/) | [Deskripsi Kontrak API](api/api-contracts.md), [koleksi Apidog](api/rusuh-booking-apidog.json) dan [OpenAPI 3.1](api/rusuh-booking-openapi.yaml) |
| [`payment-method/`](payment-method/) | Dua metode pembayaran Midtrans yang dipakai pada aplikasi: Virtual Account dan GoPay |
| [`infra-cost-model/`](infra-cost-model/) | [Perhitungan biaya update antrian](infra-cost-model/rusuh-queue-cost-model.xlsx) (spreadsheet: model biaya per transport, titik impas SSE, temuan) |
| [`discussions/`](discussions/) | Notulen rapat dengan pemilik tender dan diskusi internal tim |

---

## Keputusan arsitektur (ADR)

| ADR | Judul | Status | Isinya dalam satu kalimat |
|---|---|---|---|
| [001](adr/ADR-001-queueing-technology.md) | Teknologi antrian | Accepted, sebagian digantikan ADR-005 | Antrian berbasis counter di Redis |
| [002](adr/ADR-002-queue-position-mechanism.md) | Mekanisme posisi antrian | Accepted, sebagian digantikan ADR-005 | Posisi = nomor customer − nomor yang sedang dilayani |
| [003](adr/ADR-003-realtime-queue-updates.md) | Pengiriman update posisi | Superseded oleh ADR-005 | Pilihan transport real-time (kini SSE) |
| [004](adr/ADR-004-compute-service-selection.md) | Pemilihan compute | Superseded oleh ADR-005 | Pilihan compute (kini Lambda + ECS Fargate) |
| [005](adr/ADR-005-final-architecture-decision.md) | **Arsitektur final** | **Accepted** | SSE di ECS Fargate untuk antrian, Lambda untuk API, Redis + Aurora, outbox untuk email |
| [006](adr/ADR-006-queue-admission-amendments.md) | Amandemen antrian | **Accepted** | Giliran yang tahan crash, formula admission, satu pemberi giliran, broadcast saja |
| [007](adr/ADR-007-mobile-webview-payment-handoff.md) | Pembayaran di WebView mobile | **Accepted** | Kode sekali pakai ditukar dengan session pembayaran untuk satu booking |
| [008](adr/ADR-008-mobile-magic-link-return-architecture.md) | Kembali ke aplikasi dari magic link | Pending (Deferred) | Ditunda ke iterasi berikutnya; arah kembali lewat web |

ADR yang lebih baru mengubah ADR yang lebih lama lewat dokumennya sendiri; bagian yang diubah ditandai di dokumen lama.

---

## Repo kode

| Repo | Isi | Teknologi |
|---|---|---|
| `be-rusuh-war-ticketing` | Seluruh backend: 10 service, migrasi database, template deployment | Go 1.26, AWS Lambda, ECS Fargate, Redis, Aurora PostgreSQL, CloudFormation |
| `fe-web-rusuh-war-ticketing` | Aplikasi web, termasuk halaman pembayaran untuk WebView | Next.js 16, React 19, TypeScript |
| `fe-mobile-rusuh-war-ticketing` | Aplikasi mobile | Expo, React Native, TypeScript |
