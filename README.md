# Qwei website (Cloudflare Pages)

Static site — no build step.

| File | Purpose |
|---|---|
| `index.html` | Landing page |
| `pay/index.html` | Payment-request fallback page — target of the links encoded in shared Qwei QR codes (served at `/pay?...`) |
| `_redirects` | 200-rewrites `/.well-known/assetlinks.json` → `app-links.json`, and the legacy Supabase-style links → the pay page |
| `app-links.json` | The Android App Links statement served at `/.well-known/assetlinks.json` |

Deliberate layout choices:
- **No dot-folders.** Folders like `.well-known/` don't survive drag-and-drop
  uploads on Windows (they get silently skipped) — that is why the JSON lives
  at the root under a visible name and `_redirects` exposes it at the exact
  App Links path.
- **No `.html` in the pay URL.** Cloudflare auto-redirects `/pay.html` to
  `/pay`, which loops when combined with rewrite rules. A directory
  (`pay/index.html`) serves natively at `/pay` with no rewriting.

## Deploy

Dashboard → Workers & Pages → qwei → Upload → drop/select the **contents**
of this folder (or upload `qwei-site.zip` from the repo root).

## Verify after deploy

1. `https://qwei.pages.dev/.well-known/assetlinks.json` — must return JSON
   with the app's package name (NOT the landing page HTML).
2. `https://qwei.pages.dev/app-links.json` — same JSON, served directly.
3. `https://qwei.pages.dev/pay?v=1&r=QWEI-SANITY-TEST&n=VGVzdA%3D%3D` — must
   render the payment-request preview card (status 200, no redirect loop).

## Important

- The project name must stay **`qwei`** — the Android app is registered for
  exactly `https://qwei.pages.dev/pay`. Any other name/domain means updating
  `PAY_WEB_BASE` in `.env`, `AndroidManifest.xml`, and `_isWebPayLink` in
  `qwei_qr_payload.dart`.
- `app-links.json` lists the SHA-256 of the **debug keystore** (the current
  signing key). Switching to a dedicated release keystore = re-run
  `keytool -list -v` and replace the fingerprint, then re-upload.
- After the first verified deploy, re-check on the device:
  Settings → Apps → Qwei → Open by default → Add link. Or from a PC:
  `adb shell pm verify-app-links --re-verify com.qwei.qwei`.
