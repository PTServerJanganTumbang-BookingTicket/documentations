# Metode Pembayaran — Rusuh Booking Tiket

**Gateway**: Midtrans **Core API** (custom interface)
**Tanggal**: 25 Oktober 2026
**Status**: Accepted
**Selaras dengan**: ADR-005 Bagian 9, `api/api-contracts.md` §5

POC ini membuka **tiga** metode pembayaran dari katalog Midtrans. Masing-masing punya
dokumen sendiri karena bentuk alurnya berbeda — bukan sekadar nilai `payment_type` yang
berbeda.

| Metode | `payment_type` | Dokumen | Sifat |
|---|---|---|---|
| Kartu Kredit/Debit | `credit_card` | [01-credit-card.md](./01-credit-card.md) | Sinkron, hasil langsung, butuh 3DS |
| Transfer Bank (VA) | `bank_transfer` | [02-bank-transfer-va.md](./02-bank-transfer-va.md) | Asinkron, customer bayar di ATM/m-banking |
| E-Wallet (GoPay) | `gopay` | [03-e-wallet-gopay.md](./03-e-wallet-gopay.md) | Asinkron, customer bayar di aplikasi |

---

## Yang berlaku untuk ketiganya

**Satu endpoint charge.** `POST /v2/charge` dengan Basic Auth (`base64(ServerKey:)`).
Yang berbeda hanya isi body dan bentuk respons.

**`order_id` adalah pembawa idempotency.** Midtrans menolak `order_id` yang sudah dipakai,
sehingga percobaan ulang **wajib** memakai nilai baru. Sistem ini memakai
`{booking_id}:{payment_attempts}` — percobaan pertama `RB-00912:1`, setelah ditolak dan
customer menekan coba lagi menjadi `RB-00912:2`. Lihat ADR-005 Bagian 8.

**Status tidak pernah dipercaya dari frontend.** Midtrans menyatakannya eksplisit untuk
kartu maupun e-wallet: callback di browser dapat mengembalikan `pending` sementara verifikasi
masih berjalan. Kebenaran hanya datang dari **webhook** atau **Get Status API**.

**Verifikasi tanda tangan webhook**:

```
signature_key == SHA512(order_id + status_code + gross_amount + ServerKey)
```

Konkatenasi polos, tanpa pemisah. Tidak cocok → **abaikan notifikasi**, jangan proses.

> Dokumentasi Midtrans memuat **dua** skema tanda tangan. SHA256withRSA pada halaman
> *Security Specification* berlaku untuk lini API lain (B2B / Transactional API), **bukan**
> untuk webhook Core API. Mengimplementasikan skema yang salah membuat endpoint praktis
> tidak terverifikasi.

**Status sukses butuh dua field.** `transaction_status` saja tidak cukup:

| `transaction_status` | `fraud_status` | Perlakuan |
|---|---|---|
| `capture` | `accept` | **Lunas** (kartu) |
| `capture` | `deny` / `challenge` | **Bukan lunas** — jangan terbitkan tiket |
| `settlement` | `accept` | **Lunas** (semua metode) |
| `pending` | — | Belum dibayar; booking tetap `AWAITING_PAYMENT` |
| `deny`, `cancel`, `failure` | — | Gagal; kembali ke `HELD`, customer boleh pilih metode lain |
| `expire` | — | Kedaluwarsa; lepas stok dan NIK |

---

## Dua keputusan POC yang mempersempit cakupan

### 1. Hanya BCA, BNI, BRI, CIMB untuk VA

Midtrans mendokumentasikan kasus **reversal**: `settlement` dapat berubah menjadi `deny`,
biasanya dalam 1–5 menit, dan dana kembali ke customer sehingga pesanan harus dianggap
belum dibayar. Kasus ini hanya tercatat untuk **Permata, Mandiri Bill Payment, dan
Indomaret**.

Dengan membatasi POC pada empat bank di atas, kelas kegagalan itu **tidak dapat terjadi**,
dan tidak diperlukan handler reversal — yang jika dibangun akan berarti menarik kembali
tiket yang sudah diterbitkan dan e-tiket yang sudah terkirim.

Permata (`payment_type: permata`) dan Mandiri Bill (`payment_type: echannel`) memakai bentuk
request dan respons yang berbeda pula, sehingga pengecualiannya juga menyederhanakan kode.

> **Sebelum produksi**: bila Permata atau Mandiri Bill ditambahkan, handler reversal menjadi
> wajib — termasuk pembatalan tiket yang sudah terbit.

### 2. `custom_expiry` disetel menyamai TTL pembayaran

Expiry VA bawaan Midtrans adalah **24 jam**. TTL pembayaran sistem ini **10 menit**.
Tanpa `custom_expiry`, VA tetap dapat dibayar sepanjang hari setelah sweep melepas stoknya
dan menjualnya ke orang lain.

```json
"custom_expiry": {
  "order_time": "2026-10-25 10:15:00 +0700",
  "expiry_duration": "10",
  "unit": "minute"
}
```

`order_time` memakai WIB (GMT+7). Nilainya diisi `held_at`, anchor TTL pembayaran, bukan
waktu request — agar jendela Midtrans dan jendela kita berakhir pada detik yang sama.

Sebagai lapis kedua, sweep memanggil `POST /v2/{order_id}/expire` saat meng-expire booking.
Keduanya dipasang karena jam Midtrans dan jam kita tidak dijamin identik, dan selisih
beberapa detik cukup untuk menerima pembayaran atas kursi yang sudah dijual ulang.

> **`custom_expiry` tidak berlaku untuk kartu kredit.** Midtrans menyatakannya eksplisit.
> Jendela kartu ditentukan batas waktu 3DS, dan untuk kartu lapisan pelindungnya adalah
> pemanggilan `/expire` dari sweep.

---

## Referensi Midtrans

- [Core API — Custom Interface](https://docs.midtrans.com/docs/custom-interface-core-api)
- [Transaction Status Cycle](https://docs.midtrans.com/docs/transaction-status-cycle)
- [Receiving Notifications — formula signature_key](https://docs.midtrans.com/reference/receiving-notifications)
- [HTTP(S) Notification / Webhooks](https://docs.midtrans.com/docs/https-notification-webhooks)
- [Get Status API Requests](https://docs.midtrans.com/docs/get-status-api-requests)
- [Advanced Features — custom_expiry](https://docs.midtrans.com/docs/coreapi-advanced-features)
- [Error Code & Response Code](https://docs.midtrans.com/docs/error-code-and-response-code)

## Yang belum dikonfirmasi

| Item | Dampak |
|---|---|
| Jadwal retry webhook Midtrans saat endpoint kita gagal | Menentukan seberapa lama `OutboxLag` boleh tinggi sebelum notifikasi hilang |
| Kode respons HTTP yang diharapkan Midtrans | Desain kita selalu `200`; perlu dipastikan itu yang mencegah retry |
| Rate limit Get Status API | Membatasi agresivitas charge reconciler |
| Alamat IP Midtrans untuk allowlist resource policy | Diperlukan sebelum T-7 (ADR-005 Bagian 11) |

Keempatnya ada di halaman *Best Practices to Handle Notification* dan *IP Address* yang
belum ditarik.
