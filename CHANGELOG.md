# Changelog

## 0.13.2 — 2026-10-01

### Added
- A manual "Publish to Chrome Web Store" workflow that uploads a CI-built, hash-verified release through the Web Store API, gated by a reviewed environment.

### Security
- A browser still on an old password can no longer push over a synced vault it failed to open: a remote refused as foreign is never overwritten, however many times it was examined. Found by an adversarial review that reproduced the rotation case.
- A stored website is authoritative for the fill shortcut: an account bound to `example.com` no longer matches `example.net` by name. Name matching applies only to accounts with no site bound.

## 0.13.1 — 2026-10-01

### Security
- Sync adoption is decided by a revision counter inside the encrypted vault, not the manifest timestamp, so an older envelope replayed by someone with the Google account is refused. The recovery envelope is adopted only when its digest, also inside the vault, matches.
- Turning sync on pulls first: a copy already in Chrome sync that opens under this password is adopted rather than overwritten; one under another password blocks the switch with an explanation. A browser never pushes over a copy it has not seen, so a device still on an old password cannot re-upload an old-key vault.
- Push failures and unsynced changes show in the sync status line; the setting is saved only after the first push succeeds. Adoption during a popup open runs on the write queue with a lock check. The empty password never adopts anything, chunk counts are capped, and the generic settings handler cannot flip sync.
- Steam accounts always use a 30-second period.

## 0.13.0 — 2026-10-01

### Added
- Sync across your own Chrome browsers, off by default: the encrypted vault and recovery envelope are copied to Chrome sync in chunks; other browsers on the same Chrome profile pull a newer copy on unlock or popup open if it opens under the same master password. Never without a password.
- Custom account icons: choose an image in the edit panel; it is shrunk to 64 pixels and stored with the account.
- Steam Guard accounts: 5-symbol codes, by manual entry and from Aegis, 2FAS and Bitwarden imports, which no longer skip them.

## 0.12.1 — 2026-10-01

### Security
- A password change or protection toggle that collides with a lock mid-write now reports success with the new recovery code instead of failing after the vault was already re-keyed; the vault stays locked and opens with the new password.
- Learning a site on fill applies the same two-level-suffix rule as the fill shortcut, and hosting platforms such as `github.io` and `vercel.app` count as suffixes, so a tenant there is never mistaken for the platform.
- Enter on a focused row button activates that button; it no longer copies the code or advances a HOTP counter.

### Fixed
- Encrypted 2FAS and Bitwarden exports are recognised by their markers and refused with instructions.
- An entry with a fractional period is skipped on import instead of failing the whole file.
- The export reminder counter resets only once an export has been produced.

## 0.12.0 — 2026-10-01

### Changed
- Renamed to **AS Authenticator**. The project has been independent of the desktop app it once shared a vault format with since 0.8.0; the name now says so. Package, build artefacts (`as-authenticator-<version>.zip`), export file name, on-page overlay and all text updated. The vault format is unchanged: the associated-data strings bound into the encryption keep their historical prefix, so every existing vault, backup and export still opens.

## 0.11.0 — 2026-10-01

### Added
- Change master password in Settings, with the current password required; re-keys once, clears the hint and earlier backups, and issues a new recovery code.
- Lock when the screen locks (always) and after five idle minutes (on by default), via the `idle` permission.
- Import from other apps on the Backup tab: Aegis, 2FAS and Bitwarden JSON, and text files of otpauth links (Ente). Encrypted exports are refused with instructions.
- Duplicate detection: a secret already in the vault is refused on single add and skipped on bulk add, with counts.
- Keyboard navigation: Down from the search box enters the list, arrows move, Enter copies, Shift+Enter fills.
- Export reminder after ten changes since the last export.
- Accounts with a period other than 30 seconds show their own countdown next to the name.
- CI on GitHub Actions: both suites on every push; tagged releases built twice, compared, and published with hashes.

### Changed
- `minimum_chrome_version` 109, which the offscreen page, favicon lookup and WebAssembly policy require.

## 0.10.0 — 2026-10-01

### Added
- Suggested marker: when the popup opens on a site you hold an account for, that account sits under "Suggested for <host>" and the rest under "Accounts".
- Website per account: stored when a QR is scanned from a page, learned on the first fill from the popup, or set in the edit panel. Shown under the name, used for the brand mark and the favicon lookup, and treated as an exact match by the Suggested marker and the fill shortcut.
- An issuer entered as a host (for example `accounts.google.com`) shows the brand name with the site underneath.

### Security
- A fill teaches an account its site only when the account's name already matches that host, so one mistaken fill on a lookalike or unrelated page binds nothing. The edit panel remains the way to bind anything else, deliberately.
- A bulk Google Authenticator import no longer inherits the page it was shown on. A region capture records the page's host in the worker at capture time, not whichever tab is active when the popup reopens.
- A stored site must name one host: at least two labels, or localhost or an IP, and never a bare suffix such as `co.uk`, so no site acts as a wildcard for the fill shortcut.
- A stored site never overrides the issuer's own brand icon or displayed name; it only fills in when the name gives nothing.

## 0.9.1 — 2026-10-01

### Security
- The fill shortcut no longer accepts a bare issuer name under a two-level suffix such as `.co.uk` or `.com.ru`, where `paypal.com.ru` is a real registrable domain whose derived name is "paypal". There the issuer must spell the full host. Single-label suffixes such as `.com` work as before.
- Brand alias lookup is own-property only, so prototype names in an issuer never resolve.

## 0.9.0 — 2026-10-01

### Added
- Brand icons on account rows: a bundled set of 140-plus brand marks (Simple Icons, CC0) matched by issuer name, alias, domain, or the label's mail provider, drawn as a glyph on the brand colour. When no mark matches, Chrome's local favicon cache is consulted through the `favicon` permission; a site Chrome has never seen keeps the letter avatar. No network involved.

## 0.8.0 — 2026-10-01

### Changed
- The vault key now comes from Argon2id (64 MiB, 3 passes, parallelism 1) instead of PBKDF2. New writes of the vault, backups, the recovery envelope and exports use the V5 envelope. An older V4 vault is upgraded at the next unlock, and its PBKDF2-era backups are discarded at that moment so no weaker copy lingers. V4 files can still be imported. This ends vault-file compatibility with the former desktop companion app; the extension is an independent project.
- The fill shortcut matches the site's registrable name only (github for www.github.com), so a lookalike host such as github.evil.com gets nothing. The popup's ranking keeps the friendlier label match, since a click still chooses.
- A hint made of the password's symbols is refused even when the password has no letters or digits.
- The extension-page CSP allows `wasm-unsafe-eval` for the vendored hash-wasm Argon2 build.

### Added
- Camera scanning: "Scan with the camera" on the Add tab opens a tab that uses the webcam and adds the account when it sees a QR code.
- Pin: the star on a row keeps that account near the top. Drag a row onto another to change the order. Site matches still come first, then pinned, then your order.

## 0.7.0 — 2026-10-01

### Added
- HOTP (counter-based) accounts: by URI, manual entry, or Google Authenticator import. Rows show the counter and a skip button; copy, fill and skip advance it.
- Manual entry on the Add tab for sites that show a text secret: issuer, account, secret, type, digits, algorithm, period or counter.
- Keyboard shortcuts: Ctrl+Shift+U (Cmd on Mac) fills the account matching this site, Ctrl+Shift+Y starts QR selection. Both only while unlocked; change them at chrome://extensions/shortcuts.
- Clipboard clearing: 30 seconds after a copy the clipboard is replaced with a blank, popup open or not. On by default; off in Settings.

### Changed
- Site matching for ranking and the fill shortcut ignores names shorter than three characters, so "me" or "x" no longer matches every host.
- New permissions: `offscreen` and `clipboardWrite`, for the clipboard clear. Still no host permissions.

## 0.6.0 — 2026-10-01

### Added
- Hidden codes: the eye button in the header masks every code and remembers the choice. Copy and Fill still work; tapping a masked code shows it for five seconds.
- Reproducible builds: the same commit yields the same zip bytes, so a published hash file can be verified by rebuilding from source.

## 0.5.1 — 2026-10-01

### Security
- Changing the password, turning protection off, or recovering with a code now clears automatic backups in the same write, so no copy of the vault stays encrypted under the old key.
- Autofill fills the page itself whenever it has a candidate field; a subframe can no longer win by focusing its own input.
- A clock offset is applied only while the clock check is on, and turning it off zeroes it. The time request sends no referrer or credentials.
- The worker validates single accounts and categories on add, not only bulk imports.

## 0.5.0 — 2026-10-01

### Added
- Recovery code: shown once when a password is set, or regenerated from Settings. "Use a recovery code" on the lock screen opens the vault, takes a new password, and issues a fresh code.
- Google Authenticator import: scan or paste an `otpauth-migration://` export to add every account at once. Counter-based entries are skipped and named.
- Automatic local backups: the previous vault is kept on every change, the last seven, with Restore on the Backup tab. A backup made under an earlier password asks for it.
- Opt-in clock check in Settings: corrects codes when the machine clock drifts. Off by default; the only network request the extension makes.
- `SECURITY.md` with a disclosure address, and the build now writes a SHA-256 for every packaged file.

## 0.4.0 — 2026-09-30

### Changed
- Master password minimum is 8 characters (was 6).

### Added
- Password hint: optional, set when creating the vault or in Settings, shown on the lock screen. It may not contain the password and is cleared when password protection is turned on or off.
- "Forgot your password? Start over" on the lock screen. After a confirmation it deletes the vault and every setting, the same as removing the extension. There is still no way to recover a forgotten password.

## 0.3.0 — 2026-09-30

### Changed
- Master password minimum is 6 characters (was 12).
- Password protection is optional. The first unlock offers "Continue without a password", and Settings has "Require password to open". Off re-encrypts the vault under the empty password, hides the lock controls and auto-lock, and warns on the Backup tab that exports are unprotected. On asks for a new password and re-encrypts.

## 0.2.0 — 2026-09-30

Security release. Every finding from the 2026-09-30 audit is closed; see
`docs/security-audit-report-2026-09-30.md`.

### Security
- Autofill fills only the single best-scoring frame, so a third-party iframe never also receives a live code.
- Importing a vault requires the current vault to be unlocked, asks for the file's password in a masked field with an explicit confirmation, validates every account and category, and re-encrypts the accounts under your current master password.
- New vaults need a master password of at least 12 characters.
- Auto-lock cannot be undone by a decrypt or write that was in flight when it fired.
- The region picker ignores synthetic page events; the worker verifies the sender is the active tab's top frame, the vault is unlocked, and the rectangle is well-formed.
- A parked QR selection is discarded on lock and expires after five minutes.
- Content security policy sets `object-src 'none'`.

### Fixed
- Vaults larger than about 100 KB no longer fail to save.

### Changed
- `npm run build` packages the extension into `dist/<name>-<version>.zip`.
- The About line reads the version from the manifest.

## 0.1.0

Initial scaffold: unlock, add by URI or QR, copy, fill, edit, delete, categories, export and import, settings.
