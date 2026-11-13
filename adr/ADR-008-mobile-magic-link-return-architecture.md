# ADR-008: Mobile Magic-Link Return Architecture

**Status**: Pending (Deferred) — ditunda ke iterasi berikutnya; lihat "Catatan Penundaan" di bawah
**Tanggal**: 7 November 2026 (Revised 12 November 2026; Deferred 13 November 2026)
**Decider**: Rosyid (architecture approval boundary)
**Terkait**: GH-43 Mobile Magic-Link Integration, ADR-0023 (fe-web), api-contracts.md §1.3
**Dokumen Referensi**: `fe-mobile-rusuh-war-ticketing/.kiro/specs/GH-43-mobile-magic-link-integration/`, `api/api-contracts.md` v2.9.0, `fe-web-rusuh-war-ticketing/src/core/data/account/account-gateway.service.ts`

---

## Decision Summary

| Item | Status |
|------|--------|
| **Architecture Direction** | Web-Mediated Return (Opsi B) |
| **Handoff Contract (Web FE → Mobile FE)** | **UNRESOLVED** |
| **Decision Completeness** | **INCOMPLETE** |
| **ADR Status** | **PENDING (DEFERRED)** — ditunda 13 November 2026 (H-1 tenggat penilaian) |

---

## Catatan Penundaan (Deferral Note)

ADR ini ditunda menjadi **Pending (Deferred)** pada 13 November 2026 (H-1 tenggat penilaian semestral), dengan alasan:

1. **Keterbatasan sumber daya menjelang tenggat.** Kapasitas tim yang tersisa diprioritaskan untuk menyelesaikan P0 mobile hardening (GH-51–GH-57) sebelum deadline, sehingga tidak ada bandwidth untuk menuntaskan kontrak handoff LEG B.
2. **Keputusan memerlukan input lintas tim yang tidak dapat dijadwalkan tepat waktu.** Pertanyaan terbuka Q1–Q6 menunggu keterlibatan Infra (hosting AASA/DAL), Web FE (deteksi mobile dan UX halaman handoff), serta penerimaan risiko keamanan LEG B oleh Rosyid; koordinasi lintas tim tersebut tidak realistis dilakukan pada H-1.
3. **Tidak ada dampak fungsional bagi pengguna.** LEG A (email → web) telah terbukti berjalan di produksi dan login magic-link tetap berfungsi penuh melalui alur web; yang ditunda adalah keputusan arsitektur, bukan fitur yang sudah berjalan.

Penundaan ini bukan penolakan terhadap Opsi B (Web-Mediated Return) — arah arsitektur tersebut tetap menjadi kandidat yang diutamakan dan pembahasan ADR ini dilanjutkan pada iterasi/semester berikutnya.

---

## Konteks

### Masalah

Mobile magic-link authentication memiliki dua celah integrasi:

1. **Token Transport Delta (SOLVED):** GH-6 implementation mengirim `GET /auth/verify?token={token}` tetapi backend membaca `Cookie: magic_token={token}` secara eksklusif. Solusi: modifikasi internal adapter untuk mengirim cookie header. **Telah dibuktikan** pada iPhone 12 + Expo SDK 58.0.24 + Expo Go.

2. **Email Return Path (BLOCKING):** Tidak ada mekanisme otoratif untuk mengembalikan user dari email magic-link ke mobile app.

### Credential Sensitivity

`magic_token` adalah kredensial autentikasi proyek ini. Backend memperlakukannya sebagai:
- Single-use: token invalidated setelah verifikasi sukses
- 15-minute TTL: token expired setelah 15 menit
- Session-granting: berhasil menukar token menghasilkan `access_token` + `refresh_token`

**PENTING:** TTL dan single-use mengurangi jendela replay, tetapi TIDAK menghilangkan risiko credential disclosure. Setiap exposure point harus dievaluasi berdasarkan semantik backend aktual, bukan asumsi bahwa "mirip OAuth berarti aman."

---

## Two-Leg Model

### LEG A: Email → Web FE (EXISTING PRODUCTION)

```
Email Link
    │
    │  {WEB_BASE_URL}/login/verify#token={magic_token}
    │
    ▼
Web FE Browser
```

**Evidence Source:** `fe-web-rusuh-war-ticketing/src/modules/auth/controllers/verify-login.controller.ts` lines 14-16

**Security Properties (LEG A):**

| Property | Status | Evidence |
|----------|--------|----------|
| Token in URL fragment (`#token=`) | ✅ VERIFIED | Web FE source code |
| Fragment NOT sent to server | ✅ VERIFIED | HTTP specification, fragment never in request |
| Fragment NOT in HTTP access logs | ✅ VERIFIED | HTTP specification |
| Fragment NOT in Referer header | ✅ VERIFIED | HTTP specification |
| Fragment NOT in browser history path | ✅ VERIFIED | Fragment excluded from history URL |
| Token extracted client-side only | ✅ VERIFIED | `readToken()` function reads `window.location.hash` |

**LEG A Disposition:** ACCEPTABLE — fragment-based transport provides credential isolation from server-side logging.

---

### LEG B: Web FE → Mobile FE (UNRESOLVED)

```
Web FE
    │
    │  <HANDOFF_MECHANISM_UNRESOLVED>
    │
    ▼
Mobile FE
```

**Evidence Source:** NONE — no authoritative contract exists.

**Security Properties (LEG B):**

| Property | Status | Notes |
|----------|--------|-------|
| Credential representation | **UNRESOLVED** | Could be query param, fragment, path, state, or other |
| OS deep-link exposure | **UNRESOLVED** | OS may log deep-link URLs; credential visibility depends on mechanism |
| HTTP exposure (fallback) | **UNRESOLVED** | HTTPS Universal/App Link fallback would expose token to web server |
| Server logging | **UNRESOLVED** | Depends on mechanism selected |
| Referer behavior | **UNRESOLVED** | Depends on mechanism selected |
| Analytics exposure | **UNRESOLVED** | Application may log handoff URLs |
| Application logging | **UNRESOLVED** | Mobile FE may log received URLs |
| Fallback behavior | **UNRESOLVED** | What happens if app not installed or cannot open |

**LEG B Disposition:** REQUIRES EXPLICIT RISK ACCEPTANCE — cannot be assessed without authoritative handoff contract.

**CRITICAL:** LEG B does NOT automatically inherit LEG A's fragment-based security properties. The handoff mechanism may use a fundamentally different credential representation.

---

## Perilaku Backend Saat Ini

Notification service menghasilkan link email:
```
{APP_BASE_URL}/auth/login?token={token}
```

dimana `APP_BASE_URL` adalah URL **Web Frontend**, bukan mobile app. Komentar di `render.go` line 156:
> "It points at the frontend, not the API: the API reads the magic token from a cookie (contract §1.3), so the app route is what sets the cookie and then calls the endpoint"

---

## Web Frontend Flow (LEG A Evidence)

Alur web yang berfungsi saat ini (`fe-web-rusuh-war-ticketing/`):

1. **Email → Web FE:** Link email membuka `/login/verify#token={token}` (token di fragment, tidak dikirim ke server)
2. **Web FE → Web BFF:** Client-side POST ke `/api/auth/verify` dengan `{ token }` di body
3. **Web BFF → Backend:** BFF inject cookie header:
   ```typescript
   // account-gateway.service.ts:125
   const response = await callBackend("/auth/verify", {
     headers: { cookie: `magic_token=${encodeURIComponent(token)}` },
   });
   ```
4. **Backend → Web BFF:** Backend memverifikasi token dari cookie, return session credentials
5. **Web BFF → Web FE:** BFF menyimpan credentials di httpOnly cookies, return success response
6. **Web FE:** Redirect ke halaman sesuai `redirect_context`

---

## Arsitektur Kandidat

### Opsi A: Direct Mobile Deep-Link

Email langsung membuka mobile app via deep-link.

**Status: NOT FEASIBLE — Tidak Ada Bukti Otoratif**

- Tidak ada konfigurasi custom scheme di notification service
- Tidak ada Universal Link configuration (iOS)
- Tidak ada App Link configuration (Android)
- Tidak ada email template mobile-specific
- Tidak ada platform detection di email generation

**Membutuhkan:**
- Backend notification service modification
- iOS Associated Domains configuration
- Android App Links verification
- Apple App Site Association file hosting
- Digital Asset Links JSON hosting

### Opsi B: Web-Mediated Return (PREFERRED ARCHITECTURE DIRECTION)

Web Frontend tetap menjadi landing surface email, kemudian melakukan handoff ke mobile app.

```
┌─────────┐              ┌─────────┐              ┌─────────────────┐
│  Email  │ ──LEG A───▶ │ Web FE  │ ──LEG B───▶ │   Mobile FE     │
└─────────┘   (proven)   └─────────┘  (unresolved) └─────────────────┘
```

**Architecture Direction Rationale:**
1. Tidak membutuhkan backend notification service modification
2. Menggunakan infrastruktur web yang sudah ada dan berfungsi (LEG A)
3. Memberikan fallback ke web flow jika mobile handoff gagal
4. Perubahan terlokalisir di Web FE dan Mobile FE

**IMPORTANT:** Selecting Opsi B establishes architecture direction only. The LEG B handoff contract remains UNRESOLVED.

### Opsi C: Backend Platform Detection

Backend notification service mendeteksi platform saat magic-link request dan menghasilkan link yang berbeda.

**Status: HIGH COMPLEXITY**

- Membutuhkan platform parameter dari mobile app saat request magic-link
- Membutuhkan conditional logic di notification service
- Membutuhkan maintenance dua email templates
- Masih membutuhkan deep-link configuration di mobile

**Tidak direkomendasikan** karena Opsi B memberikan hasil yang sama dengan perubahan yang lebih terlokalisir.

---

## Analisis Keamanan

### LEG A Security Assessment

| Exposure Point | Status | Rationale |
|----------------|--------|-----------|
| Browser history | **ACCEPTABLE** | Fragment excluded from history URL |
| HTTP access logs | **ACCEPTABLE** | Fragment never sent to server |
| Referer header | **ACCEPTABLE** | Fragment excluded from Referer |
| Analytics | **ACCEPTABLE** | Fragment not in standard analytics URL capture |
| Web FE logging | **ACCEPTABLE** | Token in client memory only; not logged by existing code |

**LEG A Disposition:** ACCEPTABLE

### LEG B Security Assessment

| Exposure Point | Status | Rationale |
|----------------|--------|-----------|
| OS deep-link visibility | **UNRESOLVED** | OS may log/display deep-link URLs; mechanism-dependent |
| HTTP access logs (fallback) | **REQUIRES EXPLICIT RISK ACCEPTANCE** | HTTPS Universal/App Link fallback exposes token to web server if app not installed |
| Referer header | **UNRESOLVED** | Depends on handoff mechanism |
| Analytics | **UNRESOLVED** | Application may capture handoff URLs |
| Mobile FE logging | **UNRESOLVED** | Mobile app may log received deep-link URLs |
| URL persistence | **UNRESOLVED** | Deep-link URL may persist in system; mechanism-dependent |

**LEG B Disposition:** REQUIRES EXPLICIT RISK ACCEPTANCE — cannot approve without authoritative contract and risk assessment.

### Cookie Persistence Assessment

| Item | Status | Evidence |
|------|--------|----------|
| Explicit Cookie header transmission | **PROVEN** | iPhone 12 + Expo SDK 58.0.24 spike |
| `credentials:'omit'` prevents native persistence | **UNRESOLVED** | No runtime evidence |
| Automatic Cookie replay on subsequent requests | **UNRESOLVED** | Requires R5 qualification |

**IMPORTANT:** Do NOT attribute persistence security properties to `credentials:'omit'` without runtime evidence. Automatic Cookie persistence/replay behavior remains **UNRESOLVED** and is owned by GH-43 R5 qualification.

---

## Handoff Mechanism Candidates (ILLUSTRATIVE ONLY)

The following are illustrative examples of possible LEG B handoff mechanisms. **These are NOT approved contracts.**

| Mechanism Class | Illustrative Example | Credential Exposure | Fallback Behavior |
|-----------------|---------------------|---------------------|-------------------|
| Custom URL scheme | `<APP_SCHEME>://<PATH>?token=<TOKEN>` | Token in URL visible to OS | Fails silently or error |
| Android Intent | `intent://<PATH>?token=<TOKEN>#Intent;...` | Token in Intent URL | Opens browser or fails |
| iOS Universal Link | `https://<DOMAIN>/<PATH>?token=<TOKEN>` | Token exposed to server on fallback | Opens in browser |
| Android App Link | `https://<DOMAIN>/<PATH>?token=<TOKEN>` | Token exposed to server on fallback | Opens in browser |
| Fragment-based | `<APP_SCHEME>://<PATH>#token=<TOKEN>` | Depends on OS handling | Unknown |

**ILLUSTRATIVE ONLY — NOT AN APPROVED CONTRACT**

The authoritative handoff contract will be established when:
1. Platform ownership determines available mechanisms
2. Security/risk disposition is explicitly accepted
3. Rosyid approves the selected mechanism

---

## Decisions Required

### Decision 1: Architecture Direction

| Item | Status |
|------|--------|
| **Decision** | Web-Mediated Return (Opsi B) |
| **Disposition** | **PROPOSED** — awaiting Rosyid approval |

### Decision 2: LEG B Handoff Contract

| Item | Status |
|------|--------|
| **Decision** | **UNRESOLVED** |
| **Blocking Evidence** | No authoritative handoff mechanism defined |
| **Required Approvals** | See Ownership Matrix below |

### Decision 3: Security/Risk Disposition

| Item | Status |
|------|--------|
| **LEG A** | ACCEPTABLE |
| **LEG B** | **REQUIRES EXPLICIT RISK ACCEPTANCE** |
| **Cookie Persistence** | **UNRESOLVED** (GH-43 R5) |

---

## Ownership Matrix

| Decision/Work Item | Owner | Evidence Source | Status |
|--------------------|-------|-----------------|--------|
| **Architecture Direction (Opsi B)** | Rosyid | ADR-008 | PROPOSED |
| **LEG B Handoff Mechanism Class** | Mobile FE + Infra | Platform capabilities | UNRESOLVED |
| **Custom URL Scheme Registration** | Mobile FE | Expo/React Native config | UNRESOLVED |
| **Universal Link Configuration** | Mobile FE + Infra | AASA file hosting | UNRESOLVED |
| **App Link Configuration** | Mobile FE + Infra | DAL file hosting | UNRESOLVED |
| **AASA/DAL File Hosting** | Infra | Domain/CDN ownership | UNRESOLVED |
| **Web FE Mobile Detection** | Web FE | Browser UA detection | NOT STARTED |
| **Web FE Handoff UI** | Web FE | UX decision | UNRESOLVED |
| **Mobile FE Deep-Link Handler** | Mobile FE (GH-43) | Linking configuration | NOT STARTED |
| **Cookie Header Transport** | Mobile FE (GH-43 T1) | Adapter implementation | READY |
| **Cookie Persistence Qualification** | Mobile FE (GH-43 R5) | Runtime testing | UNRESOLVED |
| **Backend Notification Service** | Backend | — | NO CHANGE REQUIRED |
| **Backend GET /auth/verify** | Backend | — | NO CHANGE REQUIRED |

**Correction:** Native deep-link handling is Mobile FE / platform ownership, NOT Backend ownership. Backend is responsible only for notification service and auth endpoints.

---

## GH-43 / GH-7 Boundary

### GH-43 Scope (Contract + Integration)

GH-43 exists to resolve the integration contract required by GH-7:

| Item | GH-43? | Rationale |
|------|--------|-----------|
| Authoritative return/handoff contract definition | ✅ | Contract must exist before presentation consumes it |
| Cookie header transport implementation (T1) | ✅ | Integration delta fix |
| Cookie transport unit tests (T2, T3) | ✅ | Contract verification |
| CI gate verification (T4) | ✅ | Quality gate |
| iOS persistence qualification (T5) | ✅ | Platform qualification |
| Android transport + persistence qualification (T6) | ✅ | Platform qualification |
| Deep-link configuration (T7) | ✅ | Contract-level linking setup |
| Return path integration tests (T8) | ✅ | Contract-level evidence |
| CONTRACT-BASELINE update (T9) | ✅ | Contract documentation |

### GH-7 Scope (Presentation)

GH-7 consumes the completed GH-43 contract:

| Item | GH-7? | Rationale |
|------|--------|-----------|
| VerifyScreen UI component | ✅ | Presentation layer |
| Auth flow navigation | ✅ | Presentation routing |
| Error display for TOKEN_INVALID/EXPIRED | ✅ | Presentation UX |
| Session state management after verify | ✅ | Presentation state |

**Boundary Rule:** GH-7 remains parked until GH-43 contract is complete and approved.

---

## Pertanyaan Terbuka

| # | Pertanyaan | Owner | Status |
|---|-----------|-------|--------|
| Q1 | Which handoff mechanism class (custom scheme, Universal Link, App Link, other)? | Mobile FE + Infra | **UNRESOLVED** |
| Q2 | Who will host AASA/DAL files if Universal/App Links selected? | Infra | **UNRESOLVED** |
| Q3 | What is the UX for Web FE mobile detection page? | Product + Web FE | **UNRESOLVED** |
| Q4 | What credential representation in LEG B (query, fragment, state)? | Mobile FE + Security | **UNRESOLVED** |
| Q5 | What fallback behavior when app not installed? | Product + Mobile FE | **UNRESOLVED** |
| Q6 | Is LEG B credential exposure risk acceptable? | Rosyid (risk acceptance) | **UNRESOLVED** |

---

## Konsekuensi

### Jika Opsi B Architecture Direction Diterima

**Enables:**
- GH-43 cookie transport fix (T1-T4) dapat dilanjutkan — tidak blocked oleh LEG B
- GH-43 platform qualification (T5-T6) dapat dilanjutkan — tidak blocked oleh LEG B
- Roadmap untuk LEG B handoff contract resolution

**Remains Blocked:**
- GH-43 T7-T9 — blocked sampai LEG B handoff contract resolved
- GH-7 R4 — blocked sampai GH-43 contract complete

### Jika Opsi B Architecture Direction Ditolak

GH-43 R4 dan GH-7 R4 tetap **BLOCKED** sampai alternatif arsitektur ditetapkan.

---

## Referensi

1. Backend auth.go: Cookie-only token read (lines 57-68)
2. Web BFF account-gateway.service.ts: Cookie header injection (line 125)
3. Web FE verify-login.controller.ts: Fragment token extraction (lines 14-16)
4. api-contracts.md §1.3: GET /auth/verify contract
5. GH-43 Spec: `.kiro/specs/GH-43-mobile-magic-link-integration/`
6. Physical device spike: iPhone 12 + Expo SDK 58.0.24 transport verification

---

## ADR Status

| Item | Value |
|------|-------|
| **Architecture Direction** | Web-Mediated Return (Opsi B) — PROPOSED |
| **LEG B Handoff Contract** | **UNRESOLVED** |
| **Decision Completeness** | **INCOMPLETE** |
| **ADR Status** | **Pending (Deferred 2026-11-13)** — keterbatasan sumber daya dan input lintas tim pada H-1 tenggat penilaian; Rosyid remains approval boundary |

**GH-43 R4 status: DOR_BLOCKED** — requires LEG B handoff contract resolution.

**GH-7 status: PARKED (DEFERRED)** — requires GH-43 contract completion; ditunda bersama ADR-008 ke iterasi berikutnya.
