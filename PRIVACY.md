# Privacy policy

AS Authenticator is a browser extension that generates two-factor authentication codes. This policy describes what it
does with data. In short: everything stays on your device, and the developer receives nothing.

## What the extension stores

- The secrets of the accounts you add, their names, optional website and icon, and your settings. They are kept in the
  extension's storage inside your browser profile, encrypted with AES-256-GCM under a key derived from your master
  password with Argon2id. If you choose to use the extension without a password, the vault is stored encrypted under an
  empty password, which anyone with access to your browser profile could open; the extension tells you this when you
  choose it.
- While unlocked, the master password is held in the browser's memory-only session storage so the extension can stay
  unlocked between uses. It is cleared when you lock, when the screen locks, after the idle time you choose, and when
  the browser closes.
- Optional: a password hint you write, shown on the lock screen, and a recovery code envelope.

## What leaves your device

Nothing is sent to the developer or to any server operated by the developer. The extension has no analytics and no
account system.

- If you turn on "Sync across your Chrome browsers" (off by default), the encrypted vault and recovery envelope are
  copied to Chrome's own sync storage, which Google stores and delivers to your other browsers signed in to the same
  Google account. Only ciphertext travels; your password and settings do not. Turning the option off removes the copy.
- If you turn on "Check the clock" (off by default), the extension requests the current time from timeapi.io about once
  an hour. The request carries your IP address and the extension's identity and nothing else.
- Exported backup files are written where you choose and stay encrypted under your master password.

## What the extension reads on web pages

Only when you act, and only on the page you are on: when you choose to scan a QR code from the page, it captures the
visible tab or the region you drag to decode the code; when you choose to fill a code, it writes the code into the
login field on that page. It does not run on pages otherwise, has no access to other tabs, and keeps no browsing
history. Account icons may be looked up in Chrome's local favicon cache, which never contacts the web.

## Your choices

You can export, delete or reset the vault at any time from the extension. Removing the extension deletes everything it
stored on the device. Chrome sync data, if enabled, is removed when the option is turned off or the extension is reset.

## Contact

ayala.solutions@gmail.com. Security reports: see SECURITY.md in the repository.

Last updated: 2026-10-01.
