# Meeting Agenda — Rusuh Booking Tiket First Clarification
## Requirement Clarification — PO & Team
**Date:** 2026-10-14  
**Format:** Setiap pertanyaan memiliki opsi atau concern teknis — perlu keputusan dari PO/klien, bukan asumsi sepihak dari vendor.

**Legend:**
- 🔴 **MUST ASK** — Blocking arsitektur/implementasi, tidak bisa mulai coding tanpa jawaban
- 🟡 **NICE TO HAVE** — Bisa diasumsikan default yang reasonable, konfirmasi via assumptions doc

---

## 1️⃣ UMUM & ASUMSI

*Konteks proyek dan constraint teknis — set the stage sebelum masuk detail.*

**UA-1.** 🟡 Penerapan Development sesuai "Standard" itu hanya codebase, atau mencakup SDLC + project management? Perusahaan mana yang dimaksud (PT Server Jangan Tumbang atau TL)?
> **→ Keputusan yang dibutuhkan:** definisi deliverable dan konteks perusahaan.

**UA-2.** 🔴 TechRadar — apakah teknologi, framework, tools, dan library di FE/BE/Mobile dibebaskan, atau ada constraint dari klien?
> **→ Keputusan yang dibutuhkan:** ada mandatory tech stack atau bebas?

**UA-3.** 🔴 Bagaimana definisi "PHP" (app kita tidak menepati janji)? Berlaku di fase antrian, atau saat sudah masuk detail booking?
> *Ini salah satu pertanyaan paling penting secara arsitektur — jangan lewat.*
> **→ Keputusan yang dibutuhkan:** definisi PHP dan di fase mana berlakunya.

**UA-4.** 🔴 Worst case kalau server crash atau service mati di tengah transaksi — recovery apa yang diharapkan?
> *Concern: ada tradeoff antara at-least-once (mungkin double booking) vs at-most-once (mungkin lost booking). Ini keputusan bisnis.*
> **→ Keputusan yang dibutuhkan:** recovery behavior yang diharapkan.

**UA-7.** 🔴 Karena payment gateway third-party out of scope, mekanisme pembayaran yang direkomendasikan seperti apa? User input nominal sesuai harga, ada mock flow, atau ada cara lain?
> **→ Keputusan yang dibutuhkan:** mekanisme mock payment yang acceptable.

**UA-8.** 🔴 Email service menggunakan real provider (SendGrid, SES, dll) atau mock/console log cukup untuk assessment?
> **→ Keputusan yang dibutuhkan:** real email atau mock acceptable.

**UA-10.** 🟡 Kalau backend bermasalah, apa yang boleh ditampilkan ke user? Ada ketentuan dari NFR-01?
> *Concern: "halaman fallback statis" bisa dari CDN atau service worker. Tapi pesan yang ditampilkan dan apakah boleh ada retry mechanism perlu keputusan.*
> **→ Keputusan yang dibutuhkan:** konten fallback yang acceptable.

**UA-11.** 🟡 Bahasa apa saja yang didukung? Browser minimum apa? Ada target aksesibilitas?
> *Concern: kalau multi-bahasa masuk scope, ini menambah effort frontend signifikan.*
> **→ Keputusan yang dibutuhkan:** bahasa, minimum browser support, dan accessibility requirement.

**UA-12.** 🟡 Ada brand guideline atau design system dari klien, atau vendor bebas menentukan?
> **→ Keputusan yang dibutuhkan:** ada guideline atau tidak.

**UA-13.** 🟡 Apakah vendor perlu menyediakan Figma? Atau wireframe yang disediakan sudah cukup?
> **→ Keputusan yang dibutuhkan:** Figma masuk deliverable atau tidak.

**UA-14.** 🟡 "Dokumentasi sebaik mungkin" — definisinya apa? Technical Document, User Guidance, atau keduanya?
> **→ Keputusan yang dibutuhkan:** scope dokumentasi yang dimaksud.

**UA-15.** 🟡 Hierarki di Gambaran Umum — kalau ada perbedaan pandangan antara stakeholder, vendor ikut yang mana?
> **→ Keputusan yang dibutuhkan:** siapa decision maker tunggal.

**UA-16.** 🟡 Apakah boleh ada halaman tambahan di luar yang disebutkan di FR? (Error 404, koneksi terputus, fallback API error, tiket habis, toaster sukses/gagal)
> **→ Keputusan yang dibutuhkan:** apakah halaman-halaman ini masuk scope.

**UA-17.** 🟡 Apakah refund/cancellation flow masuk scope assessment?
> **→ Keputusan yang dibutuhkan:** refund masuk scope atau tidak.

**UA-18.** 🟡 Kalau customer pakai web dan mobile app, apakah quota NIK-nya shared?
> **→ Keputusan yang dibutuhkan:** quota NIK shared across platforms.

**UA-19.** 🟡 Apa KPI utama proyek ini — jumlah tiket terjual, revenue, atau conversion rate?
> **→ Keputusan yang dibutuhkan:** KPI yang menjadi tolok ukur sukses.

**UA-20.** 🟡 Apakah online menggantikan penjualan offline atau melengkapi?
> **→ Keputusan yang dibutuhkan:** positioning kanal online.

**UA-21.** 🟡 Berapa persen penjualan yang diharapkan berasal dari online?
> **→ Keputusan yang dibutuhkan:** target kontribusi online.

**UA-22.** 🟡 Apakah satu event bisa dijual melalui beberapa channel sekaligus?
> **→ Keputusan yang dibutuhkan:** multi-channel atau single channel.

**UA-23.** 🟡 Peak traffic berapa yang pernah terjadi sebelumnya?
> **→ Keputusan yang dibutuhkan:** baseline dari pengalaman sebelumnya.

**UA-24.** 🟡 Event terbesar yang pernah dijual berapa tiket?
> **→ Keputusan yang dibutuhkan:** scale reference dari pengalaman sebelumnya.

---

## 2️⃣ HOME PAGE

*Customer pertama kali buka website — entry point.*

**HP-1.** 🔴 Apakah ada state "Coming Soon" dengan countdown sebelum penjualan dibuka? Ini belum ada di FR.
> *Concern: tanpa halaman pre-queue, semua user akan hit server bersamaan tepat di momen pembukaan — ini adalah akar dari flash crowd problem. Ini bukan nice-to-have dari sisi teknis; ini arsitektur.*
> **→ Keputusan yang dibutuhkan:** apakah pre-queue/coming soon state masuk scope.

**HP-2.** 🟡 Seberapa real-time status "Available/Sold Out" harus diperbarui? Delay 10–30 detik dari CDN cache masih bisa diterima?
> *Concern: update real-time di home page sangat mahal dan biasanya tidak diperlukan.*
> **→ Keputusan yang dibutuhkan:** acceptable staleness untuk status di home page.

**HP-3.** 🟡 Saat semua stok sedang di-hold tapi belum terjual, labelnya "Sold Out" atau sesuatu yang lain?
> *Concern: "Sold Out" bersifat final dan bisa menyesatkan kalau stok masih bisa terbuka lagi karena expire.*
> **→ Keputusan yang dibutuhkan:** label dan messaging untuk state stok ter-hold.

---

## 3️⃣ WAITING ROOM

*Customer klik event dan masuk antrian.*

**WR-1.** 🔴 Apakah FCFS mutlak, atau ada jalur prioritas (VIP, sponsor, presale, member)?
> *Concern: jalur prioritas mengubah arsitektur queue secara signifikan — perlu multiple queue atau priority scoring. Ini keputusan bisnis fundamental.*
> **→ Keputusan yang dibutuhkan:** FCFS murni atau ada exception.

**WR-2.** 🔴 Ada batas maksimal jumlah orang yang boleh masuk antrian?
> *Concern: tanpa batas, 500.000 orang bisa antri untuk 5.000 tiket. Customer di posisi #490.000 harus menunggu ~45 jam.*
> **→ Keputusan yang dibutuhkan:** ada cap antrian atau tidak, dan berapa.

**WR-3.** 🔴 Kalau customer menutup tab lalu kembali, posisinya tetap atau ada grace period? Berapa lama? Dan mekanisme token-nya apa — random UUID di cookie, user ID login, atau kombinasi?
> *Concern: tanpa grace period, customer yang koneksinya putus sesaat kehilangan posisi antrian. Mekanisme token menentukan bagaimana "refresh tidak kehilangan posisi" diimplementasikan.*
> **→ Keputusan yang dibutuhkan:** grace period, durasi, dan mekanisme token.

**WR-4.** 🔴 Kalau customer membuka tab kedua atau perangkat lain di event yang sama — tab baru menggantikan yang lama, atau ditolak?
> *Concern: dua sesi aktif untuk satu NIK di satu event bisa dipakai untuk "save slot".*
> **→ Keputusan yang dibutuhkan:** apakah multi-session diizinkan.

**WR-5.** 🔴 Berapa lama slot dibiarkan "kosong" jika customer mendapat giliran tapi tidak pernah masuk ke Detail Ticket Page?
> *Concern: slot yang di-hold tapi tidak digunakan memblokir customer lain.*
> **→ Keputusan yang dibutuhkan:** timeout untuk slot yang tidak digunakan.

**WR-6.** 🔴 Bagaimana sistem memberitahu customer mobile bahwa gilirannya sudah tiba, ketika app sedang di background?
> *Concern: di mobile, browser/app yang di-background bisa di-suspend. WebSocket/SSE koneksi bisa terputus. Customer mungkin tidak tahu gilirannya sudah tiba.*
> **→ Keputusan yang dibutuhkan:** mekanisme notifikasi untuk mobile background state.

**WR-7.** 🔴 FR-07 bilang posisi diperbarui "setiap antrian berkurang". Apakah harus per pergerakan, atau update berkala tiap 3–5 detik sudah diterima?
> *Concern: update real-time per-step sangat mahal di skala flash crowd.*
> **→ Keputusan yang dibutuhkan:** frekuensi update yang acceptable.

**WR-8.** 🟡 Kalau antrian jauh melebihi stok, apakah customer di belakang perlu diberi tahu bahwa tiket kemungkinan habis?
> *Concern: memberi tahu bisa mengurangi frustrasi, tapi bisa juga menyebabkan customer keluar padahal tiket masih bisa terbuka karena expire.*
> **→ Keputusan yang dibutuhkan:** ada notifikasi "kemungkinan habis" atau tidak.

**WR-9.** 🟡 Progress bar di wireframe menghitung dari apa? Dari posisi awal saat join?
> **→ Keputusan yang dibutuhkan:** definisi dan formula progress bar.

**WR-10.** 🟡 Fairness lebih penting daripada speed? Jika harus memilih, mana yang diprioritaskan?
> **→ Keputusan yang dibutuhkan:** prioritas antara fairness dan speed.

**WR-11.** 🟡 Apakah perlu konfirmasi sebelum customer membatalkan antrian?
> **→ Keputusan yang dibutuhkan:** ada konfirmasi cancel atau tidak.

---

## 4️⃣ DETAIL TICKET PAGE

*Customer dapat giliran, pilih tiket dan isi data.*

**DT-1.** 🔴 Mana yang lebih penting: tidak oversell, atau transaksi cepat?
> *Concern: ada tradeoff antara locking yang ketat (no oversell tapi slower) vs optimistic (faster tapi risk oversell).*
> **→ Keputusan yang dibutuhkan:** prioritas antara data integrity dan speed.

**DT-2.** 🔴 Satu NIK untuk seluruh order, atau tiap tiket butuh NIK dan nama pemegang sendiri? Validasi cukup format 16 digit, atau perlu integrasi API kependudukan?
> *Concern: NIK per tiket lebih akurat tapi jauh lebih kompleks. Integrasi API kependudukan menambah dependency dan latency.*
> **→ Keputusan yang dibutuhkan:** scope NIK (per order/per tiket) dan kedalaman validasi.

**DT-3.** 🔴 Kenapa dipilih 3 tiket sebagai limit? Apakah bisa dikonfigurasi per event?
> *Concern: beberapa event mungkin butuh limit berbeda. Hardcode vs configurable mempengaruhi desain.*
> **→ Keputusan yang dibutuhkan:** limit configurable atau fixed.

**DT-4.** 🔴 Kalau customer sudah beli tiket untuk suatu event, apakah bisa re-queue lagi untuk beli tambahan (sampai limit 3)?
> *Concern: kalau diizinkan, queue uniqueness constraint harus berdasarkan NIK + jumlah tiket yang sudah dibeli.*
> **→ Keputusan yang dibutuhkan:** apakah re-queue diizinkan setelah pembelian.

**DT-5.** 🔴 Apakah login wajib sebelum antre? Email untuk e-tiket diambil dari akun atau diisi di form?
> *Concern: login wajib menambah friction di momen flash crowd. Kalau guest checkout, bagaimana validasi NIK-nya?*
> **→ Keputusan yang dibutuhkan:** login required atau tidak.

**DT-6.** 🔴 Setelah masuk payment, countdown booking 5 menit berhenti dan diganti countdown 10 menit? FR-18 masih ambigu.
> *Concern: kalau kedua TTL berjalan bersamaan, customer bisa expire di booking sebelum selesai bayar.*
> **→ Keputusan yang dibutuhkan:** TTL booking dan payment bersamaan atau bergantian.

**DT-7.** 🟡 Apakah stok per kategori ditampilkan real-time, atau cukup snapshot saat halaman dibuka?
> **→ Keputusan yang dibutuhkan:** expectation freshness data stok.

**DT-8.** 🟡 Kalau stok habis tepat saat klik "Lanjut ke Pembayaran", UX yang diharapkan apa?
> **→ Keputusan yang dibutuhkan:** flow saat race condition stok.

**DT-9.** 🟡 Kalau payment expire, quota NIK langsung reset atau ada cooldown period?
> **→ Keputusan yang dibutuhkan:** behavior quota setelah payment expire.

**DT-10.** 🟡 Limit 3 tiket itu per event atau global untuk semua event?
> **→ Keputusan yang dibutuhkan:** scope limit tiket.

**DT-11.** 🟡 Jika terjadi race condition, siapa yang menang?
> **→ Keputusan yang dibutuhkan:** conflict resolution policy.

---

## 5️⃣ PAYMENT

*Customer bayar dan dapat e-tiket.*

**PAY-1.** 🔴 Payment gateway dari mana? Mau pakai apa?
> *Concern: kalau real gateway (Midtrans, Xendit), ada API key dan sandbox yang perlu disiapkan. Kalau mock, scope lebih sederhana.*
> **→ Keputusan yang dibutuhkan:** real payment gateway atau simulated.

**PAY-2.** 🔴 Untuk VA dan transfer — apakah customer menunggu di halaman ini sambil melihat instruksi bayar? Bagaimana UI status "pending"?
> *Concern: kalau VA/transfer masuk scope, perlu polling status atau webhook dari gateway.*
> **→ Keputusan yang dibutuhkan:** VA/transfer masuk scope dan UX pending state.

**PAY-3.** 🔴 Apakah customer boleh kembali dari halaman payment ke detail ticket untuk mengubah pilihan?
> *Concern: kalau boleh, bagaimana treatment TTL? Kalau tidak, flow jadi one-way.*
> **→ Keputusan yang dibutuhkan:** back navigation diizinkan atau tidak.

**PAY-4.** 🔴 Apakah booking bisa dilanjutkan jika customer sempat close tab/browser?
> *Concern: ini menyentuh prinsip "Jangan PHP". Kalau bisa dilanjutkan, perlu state persistence.*
> **→ Keputusan yang dibutuhkan:** booking persistence policy.

**PAY-5.** 🟡 Apakah halaman "Lihat E-Tiket" termasuk scope? QR statis atau bisa divalidasi di gate?
> **→ Keputusan yang dibutuhkan:** scope e-tiket.

**PAY-6.** 🟡 Apakah partial payment / split payment masuk scope?
> **→ Keputusan yang dibutuhkan:** single payment atau partial diizinkan.

**PAY-7.** 🟡 Apakah perlu reminder sebelum payment timeout (misalnya "2 menit lagi")?
> **→ Keputusan yang dibutuhkan:** ada reminder atau tidak.

**PAY-8.** 🟡 Berapa target payment success rate?
> **→ Keputusan yang dibutuhkan:** target metric.

---
*Updated: 2026-10-14*
