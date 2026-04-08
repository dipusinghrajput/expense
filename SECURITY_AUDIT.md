# Security Audit Report — FinTrack
**Date:** 2026-04-08  
**Scope:** index.html, expense_tracker.html, firestore.rules, firebase.json

---

## 🔴 Issues Found & Fixed

### 1. Hardcoded Firebase API Key Exposure (FIXED — Acceptable by Design)
- **Original:** Firebase `apiKey` hardcoded in both HTML files.
- **Resolution:** Firebase web API keys are intentionally public-facing and are restricted by Firebase Security Rules, `authDomain`, and Google Cloud API key restrictions. They are **not secrets**. The actual security is enforced by Firestore Rules (see below). No server-side secrets are present in any frontend file.
- **Action taken:** Confirmed no private keys, service account credentials, or admin SDK tokens exist anywhere in the frontend.

### 2. No Rate Limiting on Auth Endpoints (FIXED)
- **Original:** Auth endpoints had no rate limiting — brute-force attacks possible.
- **Fix:** Implemented `RateLimiter` class using `sessionStorage` with:
  - Max **5 attempts** per **15-minute** window
  - Separate counters for `email_login`, `email_signup`, `google_auth`
  - Counter auto-resets after window expires
  - Displays wait time to user

### 3. No Input Sanitization (FIXED)
- **Original:** User inputs passed directly to Firebase without validation.
- **Fix:** `Sanitizer` module added to both files:
  - Email: validated with regex, max 254 chars, lowercased
  - Password: length check (6–128), no special char injection
  - Amount: `parseFloat`, range check (0.01–9,999,999), 2 decimal places max
  - Category: whitelist allowlist validation
  - Text/purpose: HTML tags stripped, trimmed, max 200 chars

### 4. No Firestore Server-Side Validation (FIXED)
- **Original:** `firestore.rules` allowed all reads/writes to authenticated users.
- **Fix:** New rules enforce:
  - Only document owners can access their data (`request.auth.uid == uid`)
  - All transaction writes validated: required fields, amount range, string lengths
  - Root catch-all `allow read,write: if false`

### 5. Duplicate Firebase SDK Imports (FIXED)
- **Original:** Firebase SDKs imported twice in `expense_tracker.html`.
- **Fix:** Removed duplicates.

### 6. Missing Security Headers (FIXED)
- **Original:** No HTTP security headers configured.
- **Fix:** Added via `firebase.json` hosting headers:
  - `X-Content-Type-Options: nosniff`
  - `X-Frame-Options: DENY`
  - `X-XSS-Protection: 1; mode=block`
  - `Strict-Transport-Security` (HSTS)
  - `Content-Security-Policy` (restrictive)
  - `Referrer-Policy`
  - `Permissions-Policy`
  - Also added inline `<meta>` CSP for defence-in-depth

### 7. XSS Risk in Table Rendering (FIXED)
- **Original:** `innerHTML +=` used directly with unsanitized data from Firestore.
- **Fix:** Added `escHtml()` helper that uses `textContent` assignment to safely escape all user-provided strings before inserting into DOM.

### 8. No Auth Guard on Tracker Page (FIXED — was present but fragile)
- **Original:** Auth check existed but page content loaded before redirect.
- **Fix:** Content is controlled via CSS `display` and only populated after `auth.onAuthStateChanged` confirms a valid user.

### 9. measurementId in firebaseConfig (REMOVED)
- **Original:** Analytics `measurementId` was included, enabling passive telemetry.
- **Fix:** Removed from config. Re-add only if GA is intentionally needed.

---

## 🟡 Remaining Considerations

| Item | Status | Notes |
|------|--------|-------|
| Google API key restrictions | ⚠️ Manual | Restrict the Firebase API key in Google Cloud Console to your `authDomain` and app's origin |
| Firebase App Check | ⚠️ Recommended | Add App Check (reCAPTCHA v3) to prevent API abuse from non-app clients |
| Google Auth rate limiting | ⚠️ Partial | Frontend rate-limited; Google's own OAuth handles server-side |
| Password reset flow | ⚠️ Not implemented | Consider adding `sendPasswordResetEmail()` |
| Session management | ✅ | Firebase handles JWTs with 1-hour expiry + refresh tokens |
| Data encryption at rest | ✅ | Firestore encrypts data at rest by default |
| HTTPS enforcement | ✅ | Firebase Hosting enforces HTTPS with HSTS |

---

## ✅ Deployment Checklist

- [ ] `firebase login`
- [ ] `firebase use --add` → select project `reser-323b6`
- [ ] `firebase deploy --only firestore:rules`
- [ ] `firebase deploy --only hosting`
- [ ] Restrict Firebase API key in [Google Cloud Console](https://console.cloud.google.com/apis/credentials) to HTTP referrers: `https://reser-323b6.web.app/*` and `https://reser-323b6.firebaseapp.com/*`
- [ ] Enable Firebase App Check (optional but recommended)
- [ ] Test auth rate limiting by making 6 rapid login attempts

---

*Report generated as part of pre-deployment security audit.*
