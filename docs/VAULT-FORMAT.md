# Vault format

AS Authenticator owns its vault format. The stored vault, every automatic backup, the recovery envelope and every export
use the same codec, implemented in `src/vault.js`.

## V5 (written since 0.8.0)

The whole account document is encrypted with AES-256-GCM under a key derived from the master password with
Argon2id: 64 MiB of memory, 3 passes, parallelism 1, 32-byte output. Every write draws a fresh 32-byte salt and a
fresh 12-byte nonce. The fixed parameters are bound as associated data, and a reader rejects any envelope whose
parameters differ, so a file cannot choose cheaper ones. The tag is 16 bytes and stored separately.

```json
{ "Version": 5, "Algorithm": "AES-256-GCM", "Kdf": "Argon2id", "Memory": 65536, "Iterations": 3, "Parallelism": 1,
  "Salt": "<base64 32 bytes>", "Nonce": "<base64 12 bytes>", "Tag": "<base64 16 bytes>", "Data": "<base64 ciphertext>" }
```

The associated-data strings bound into every ciphertext are frozen format constants:
`2fast:v5:AES-256-GCM:Argon2id:65536:3:1` and `2fast:v4:AES-256-GCM:PBKDF2-SHA256:600000`. Their prefix is a
historical artefact of the format, not a product name; changing a single byte would make every existing vault,
backup and export undecryptable, so they never change.

Argon2id comes from the vendored hash-wasm build (`vendor/hash-wasm-argon2.js`, MIT), a WebAssembly implementation,
which is why the extension-page content security policy allows `wasm-unsafe-eval`.

## V4 (read only)

Vaults and exports written before 0.8.0 use PBKDF2-SHA256 with 600,000 iterations under the same cipher and envelope
shape. They still open. A stored V4 vault is re-encrypted as V5 at the next unlock, and the automatic backups from the
PBKDF2 era are discarded at the same moment, so nothing under the weaker key stays in the profile. Nothing writes V4 any
more.

## The document inside

```json
{ "Version": 4, "Collection": [ <account> ], "GlobalCategories": [ <category> ] }
```

An account: `Label`, `Issuer`, `SecretByteArray` (base64 of the raw key bytes), `TotpSize` (6 to 8), `Period`
(seconds), `HashMode` (`0` SHA-1, `1` SHA-256, `2` SHA-512), `OTPType` (`totp`, `hotp` or `steam`; Steam Guard codes are 5 symbols, `TotpSize` 5), `Counter` for HOTP,
`IsFavourite` (pinned), optional `Icon` (a PNG data URL of at most 16 KB chosen by the user), optional `Site` (the account's website as a hostname, used for the icon, the Suggested marker
and the fill shortcut), and `SelectedCategories`, a list of category copies matched by `Guid`. A category has `Guid`,
`Name`, `UnicodeString` and `UnicodeIndex` (an icon id from the fixed set in `src/categories.js`). The inner `Version`
is historical and not used for anything.

Imported files are validated field by field before they replace the vault (`assertModel`), and are re-encrypted under
the current master password.
