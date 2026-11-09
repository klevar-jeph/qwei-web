# Qwei website (Cloudflare Pages)

Static site — no build step. Contains:

| File | Purpose |
|---|---|
| `index.html` | Landing page |
| `pay.html` | Payment-request fallback page — target of the links encoded in shared Qwei QR codes |
| `_redirects` | Makes `/pay?...` serve `pay.html` (query strings preserved) |
| `.well-known/assetlinks.json` | Android App Links verification — lets tapping a link open Qwei directly, no "Open with" chooser |

## Deploy

Dashboard → Create Application → Pages → **Upload Direct Upload** →
drop the *contents* of this folder (4 items, including `.well-known`), project name **`qwei`**
→ the site appears at `https://qwei.pages.dev`.

## Important

- The project name MUST be `qwei` so the domain is `qwei.pages.dev` — the Android
  app only auto-opens links registered for exactly `https://qwei.pages.dev/pay`.
  (Any other name → delete/rename or use a custom domain and update the two
  registered values: `PAY_WEB_BASE` in `.env` and the `qwei.pages.dev` entries
  in `AndroidManifest.xml` / `_isWebPayLink` in `qwei_qr_payload.dart`.)
- `assetlinks.json` lists the SHA-256 fingerprint of the **debug keystore**
  (the app is currently signed with it). If you switch to a dedicated release
  keystore, re-run:
  `keytool -list -v -keystore <new keystore> -storepass <pass>` and replace the
  fingerprint here, then re-upload.
- Verification can take a few minutes after first deploy. To re-check on the
  device: Settings → Apps → Qwei → Open by default → Add link.
