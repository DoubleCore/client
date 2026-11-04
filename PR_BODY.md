## Summary

- Add opt-in XChaCha20-Poly1305 encryption for peer-cache task, persistent task, and persistent cache task content.
- Load a 32-byte key from exactly one configured file or environment source before storage initialization.
- Preserve plaintext protocol/export behavior through authenticated decrypting readers while preventing raw-file and hard-link bypasses.

## Changes

- Add `storage.encryption.enable`, `keyFile`, and `keyEnv` configuration with disabled-by-default behavior.
- Store content in authenticated 4 MiB chunks with fresh random 24-byte nonces, per-file AAD binding, validated lengths, and authenticated unwritten-chunk placeholders.
- Add bounded-memory encrypted range reads and streaming writes, partial-write locking, integrity/wrong-key errors, physical range accounting, and Linux/macOS content integration.
- Reject legacy plaintext in encrypted mode and retained ciphertext in plaintext mode rather than silently returning or corrupting the wrong format.
- Decrypt copy exports and encrypted TCP uploads; reject encrypted-mode hard links and retain plaintext `sendfile`.
- Document key generation, configuration, compatibility, disk layout, and performance tradeoffs in `docs/storage-encryption.md`.

## Test plan

- `cargo fmt --all -- --check`
- `cargo build -p dragonfly-client-storage`
- `cargo test -p dragonfly-client-storage --no-fail-fast` — 126 passed, 0 failed
- `cargo test -p dragonfly-client-config --no-fail-fast` — 31 passed, 0 failed
- `cargo clippy -p dragonfly-client-storage -- -D warnings`

Coverage includes round-trip and range reads, large/empty content, tamper/truncation/wrong-key rejection, nonce uniqueness, concurrent partial writes, key validation and fail-fast startup, format compatibility, persistent content, copy/hard-link behavior, and TCP plaintext delivery. macOS behavior mirrors Linux but was not runtime-tested on the Linux host.

Refs #4026
