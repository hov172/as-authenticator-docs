# Security policy

AS Authenticator keeps TOTP and HOTP secrets in an AES-256-GCM vault inside the browser profile, keyed by Argon2id. Anything that could expose those
secrets, open the vault without its password, or make the extension reach the network beyond the opt-in clock check and the account-icon favicon.ico loads from the
account's own site is in scope and taken seriously.

## Reporting a vulnerability

Email **ayala.solutions@gmail.com** with "AS Authenticator security" in the subject. Include the version (Settings, About
line), steps to reproduce, and what an attacker gains. Please do not open a public issue for anything exploitable.

You will get an acknowledgement within 3 business days and a fix or a decision within 30 days for anything confirmed.
Credit is given in the changelog unless you ask otherwise.

## Scope

In scope: everything under `src/`, `popup/`, `manifest.json` and the derived manifests in `tools/manifests.mjs`, the build in `build.sh`, and the vault format described in
`docs/VAULT-FORMAT.md`.

Out of scope: the vendored `vendor/jsQR.js` and `vendor/hash-wasm-argon2.js` (report upstream), the browsers themselves (Chrome, Edge, Brave, Firefox, Safari), and attacks that need the user's unlocked browser profile or machine, which the threat model already concedes
(see the accepted risks in `docs/security-audit-report-2026-09-30.md`).

## Continuous integration

`.github/workflows/ci.yml` runs the unit and end-to-end suites in real Chrome on every push and pull request. On a
`v*` tag it builds twice, fails unless the two hash files are identical, and publishes the three zips (Chromium, Firefox, Safari) and their hash files as a
GitHub release using the runner's own `gh`, with no third-party release action. Every action is pinned by commit.

## Verifying a release

Every release ships `as-authenticator-<version>.sha256`, `-firefox.sha256` and `-safari.sha256`, a SHA-256 for each packaged file and for the zip itself,
produced by `npm run build`. The build is reproducible: check out the release tag, run `npm run build`, and the zip and
hash file match the published ones byte for byte. To check an installed copy against a release:

```
cd <unpacked extension folder>
shasum -a 256 -c as-authenticator-<version>.sha256 --ignore-missing
```

The vendored jsQR is the unmodified 1.4.0 npm release (`bc40c8a15196236b2314db0856f72ca0b49980cd5413b8c852a7349f5fee0859`),
and the vendored hash-wasm Argon2 build is the unmodified 4.12.0 npm release (`dcec617a2e1b700fa132d1583a186cb70611113395e869f2dd6cc82b415d3094`).

## What the extension never does

- No host permissions, no content scripts, no remote code, no analytics. The `favicon` permission reads Chrome's local
  favicon database only as the last resort for account icons, after the site's own favicon.ico over HTTPS (see PRIVACY.md); the `idle` permission only tells the worker when
  the screen locks or the computer idles, so it can lock the vault. The content security policy allows
  WebAssembly (`wasm-unsafe-eval`) solely for the vendored Argon2id build; scripts remain `'self'` only. The `offscreen` and `clipboardWrite`
  permissions exist only to replace the clipboard with a blank after a copied code's grace period; nothing reads it except the
  `clipboardRead` permission (Chrome, Edge, Brave, Firefox), which the Add tab's "Scan the copied QR code" tile uses once per click to decode a copied QR image locally. Safari has no such permission: there the same click asks the host app's extension handler, over `nativeMessaging`, for the pasteboard image; the handler answers that one message only.
- No network request of its own except a GET to timeapi.io when "Check the clock" is on, and, for account icons with no
  bundled brand mark, an HTTPS image load of `favicon.ico` from the account's own host, then its registrable domain (no icon service).
  With "Sync across your browsers" on, also off by default, the browser's own sync (Chrome sync or Firefox Sync) carries the
  encrypted vault envelope between the user's browsers; the extension never contacts a server of its own.
- No password reset or decryption bypass. A forgotten password is recoverable only with the recovery code the user saved.
