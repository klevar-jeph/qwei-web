# Qwei website (Cloudflare Pages)

Static site — no build step.

| File | Purpose |
|---|---|
| `index.html` | Landing page |
| `pay/index.html` | Payment-request fallback page — served at `/pay` and `/pay/{reference}` (short opaque links resolve through the `pay-link-lookup` edge function; legacy `?v=1&r=…` links still render offline) |
| `_redirects` | 200-rewrites `/.well-known/assetlinks.json` → `app-links.json` and `/.well-known/apple-app-site-association` → `apple-app-site-association.json`, plus legacy Supabase-style links → the pay page |
| `app-links.json` | The Android App Links statement served at `/.well-known/assetlinks.json` |
| `apple-app-site-association.json` | iOS Universal Links statement (placeholder `appID` until the iOS Team ID exists) |

Deliberate layout choices:
- **No dot-folders.** Folders like `.well-known/` don't survive drag-and-drop
  uploads on Windows (they get silently skipped) — that is why the JSON lives
  at the root under a visible name and `_redirects` exposes it at the exact
  App Links path.
- **No `.html` in the pay URL.** Cloudflare auto-redirects `/pay.html` to
  `/pay`, which loops when combined with rewrite rules. A directory
  (`pay/index.html`) serves natively at `/pay` with no rewriting.

## Deploy

Primary: this folder is synced to the GitHub repo `klevar-jeph/qwei-web`,
which is connected to Cloudflare Pages — **pushing to `main` deploys**.

Manual fallback: Dashboard → Workers & Pages → qwei → Upload → drop/select
the **contents** of this folder.

## Configuration

### Store URLs (pay page)

Until Qwei is published, the pay page shows an honest "coming soon" line
instead of store buttons — **never a broken link**. When the listings exist,
edit the `STORES` config at the top of `pay/index.html`:

```js
var STORES = {
  android: 'https://play.google.com/store/apps/details?id=com.qwei.qwei',
  ios: 'https://apps.apple.com/app/qwei/id0000000000'
};
```

Empty string = unpublished. The page automatically puts the visitor's own
platform first (Android UA vs iPhone UA).

### iOS Universal Links (not yet active)

`apple-app-site-association.json` is a placeholder so
`/.well-known/apple-app-site-association` already serves valid JSON. Before
shipping iOS:

1. Replace `TEAMID` with the real Apple Team ID in that file.
2. In Xcode: Signing & Capabilities → + Associated Domains →
   `applinks:qwei.pages.dev`.
3. Re-deploy this site and verify the URL returns the updated JSON.
4. Test on a device with the app installed (Universal Links only verify on
   a real device).

### Backend lookup

`/pay/{reference}` pages fetch
`https://ngmkreuhhxfmhnwkmrbg.supabase.co/functions/v1/pay-link-lookup`
(POST `{"reference": "QWEI-…"}`). The function is public **by design** — it
returns only what a scanned QR already exposes (display name, amount,
currency, expiry, status). It never mutates anything; payments stay
authenticated and re-validated by `create-payment-intent`.

## Verify after deploy

1. `https://qwei.pages.dev/.well-known/assetlinks.json` — must return JSON
   with the app's package name (NOT the landing page HTML).
2. `https://qwei.pages.dev/app-links.json` — same JSON, served directly.
3. `https://qwei.pages.dev/.well-known/apple-app-site-association` — must
   return JSON (placeholder appID is fine pre-launch).
4. `https://qwei.pages.dev/pay?v=1&r=QWEI-SANITY-TEST&n=VGVzdA%3D%3D` — must
   render the payment-request preview card (status 200, no redirect loop).
5. `https://qwei.pages.dev/pay/QWEI-NOSUCH-REFERENCE` — must serve the pay
   page AT that URL (200, no redirect) and render the honest "could not be
   found" state. NOTE: `404.html` exists on purpose — it disables SPA mode
   (unmatched paths → landing page) and gives stray paths a real 404.
6. `https://qwei.pages.dev/pay/:ref` is a 200-rewrite to `/pay/` — the
   destination must stay in canonical directory form; pointing it at
   `/pay/index.html` triggers Cloudflare's automatic index.html → `/pay/`
   308, which strips the reference from the path.

## Important

- The project name must stay **`qwei`** — the Android app is registered for
  exactly `https://qwei.pages.dev/pay`. Any other name/domain means updating
  `PAY_WEB_BASE` in `.env`, `AndroidManifest.xml`, and
  `qwei_qr_payload.dart` (payLink / shortReference / isQweiPayLink).
- `app-links.json` lists the SHA-256 of the **debug keystore** (the current
  signing key). Switching to a dedicated release keystore = re-run
  `keytool -list -v` and replace the fingerprint, then re-upload.
- After the first verified deploy, re-check on the device:
  Settings → Apps → Qwei → Open by default → Add link. Or from a PC:
  `adb shell pm verify-app-links --re-verify com.qwei.qwei`.
