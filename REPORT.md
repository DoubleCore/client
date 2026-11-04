# Issue #4026 Implementation Report

## Summary

Added opt-in encrypted peer-cache content storage using XChaCha20-Poly1305. Task, persistent task, and persistent cache task files use independent 4 MiB chunks with fresh 24-byte random nonces and authenticated 16-byte tags. The implementation supports streaming writes, bounded-memory range reads, authenticated partial writes, integrity failures, and fail-fast key loading.

Encryption is disabled by default. Legacy plaintext is rejected when encryption is enabled, and retained encrypted files are rejected when encryption is disabled. Copy exports return authenticated plaintext; hard-link import/export is unavailable in encrypted mode. Plaintext behavior remains covered by the existing test suite.

## Algorithm and format rationale

XChaCha20-Poly1305 provides AEAD without requiring AES CPU instructions. Its 192-bit nonce allows random per-chunk, per-rewrite nonces without persistent nonce state. The 48-byte header stores the `DFENC001` format marker, chunk size, plaintext length, random file ID, and reserved bytes. Each fixed-size record stores a nonce, plaintext length, fixed ciphertext area, and tag. AAD binds the complete header, chunk index, and chunk length.

Authenticated 44-byte placeholders represent unwritten chunks. This avoids both whole-file zero encryption during creation and unsafe acceptance of all-zero records during later partial writes. Physical size is `48 + ceil(plaintext_length / 4 MiB) * (4 MiB + 44)`. No throughput benchmark was run; CPU, read-modify-write, placeholder initialization, and encrypted TCP buffered-copy costs are documented in `docs/storage-encryption.md`.

## Changed files

- `Cargo.toml`, `Cargo.lock`: added `chacha20poly1305 0.10.1` and its required lockfile dependencies; no unrelated package version changes.
- `dragonfly-client-storage/Cargo.toml`: used workspace `chacha20poly1305` and existing workspace `hex` dependencies.
- `dragonfly-client-config/src/dfdaemon.rs`: added `storage.encryption.enable`, `keyFile`, and `keyEnv`, defaults, and deserialization coverage.
- `dragonfly-client-storage/src/encryption.rs`: added key loading, XChaCha20-Poly1305 operations, format validation, authenticated placeholders, per-path locking, physical range mapping, and unit tests.
- `dragonfly-client-storage/src/io.rs`: added encrypted range reads and streaming writes, CRC32 calculation over plaintext, chunk-sized encrypted read buffering, and a checked raw-FD conversion for sendfile selection.
- `dragonfly-client-storage/src/content.rs`: retained the original public constructor and added an internal constructor that accepts the key loaded by `Storage`.
- `dragonfly-client-storage/src/content_linux.rs`: integrated encrypted creation, reads, writes, copies, space accounting, page-cache/writeback ranges, legacy-format checks, and Linux integration tests.
- `dragonfly-client-storage/src/content_macos.rs`: mirrored the content behavior, including all hard-link guards. macOS runtime tests were not available on the Linux host.
- `dragonfly-client-storage/src/lib.rs`: loads the key before metadata/content initialization and maps page-cache ranges to physical encrypted records.
- `dragonfly-client-storage/src/server/tcp.rs`: uses decrypted buffered copy for encrypted content while preserving plaintext `sendfile`; added an end-to-end TCP decryption test.
- `docs/storage-encryption.md`: added configuration, key generation, format, compatibility, export, and performance documentation.
- `PR_BODY.md`, `DONE`: added PR and completion summaries.

The new `chacha20poly1305` dependency is required for the selected AEAD implementation. `hex` was already in the workspace and is used only for key decoding and tests. Test keys and generated plaintext are confined to temporary directories; no secret or plaintext test fixture was added to git.

## Compatibility and operational behavior

- Exactly one of `keyFile` or `keyEnv` is required when enabled. Key files accept raw 32-byte or 64-character hexadecimal keys; environment variables accept hexadecimal keys. Missing, conflicting, unreadable, or malformed key configuration fails before storage initialization.
- Existing plaintext is not auto-migrated and cannot be read in encrypted mode. Retained ciphertext cannot be read, overwritten, copied, or linked as plaintext after encryption is disabled.
- Copies authenticate and decrypt to the export destination. All encrypted-mode hard-link paths reject the operation so they cannot expose ciphertext or import plaintext directly.
- Changing or losing the key makes retained content unavailable. Configuration and operational examples are in `docs/storage-encryption.md`.

## Verification

Environment for all Rust commands:

```bash
export PATH="$HOME/.cargo/bin:$PATH"
export CARGO_TARGET_DIR=/home/ps/LHL/target-dfc
```

Final commands and real results:

| Command | Result | Log |
| --- | --- | --- |
| `cargo fmt --all -- --check` | Passed; no diagnostics | `artifacts/logs/fmt.log` |
| `cargo build -p dragonfly-client-storage` | Passed, `Finished dev profile` | `artifacts/logs/build.log` |
| `cargo test -p dragonfly-client-storage --no-fail-fast` | **126 passed, 0 failed, 0 ignored**; doc-tests 0 | `artifacts/logs/test.log` |
| `cargo test -p dragonfly-client-config --no-fail-fast` | **31 passed, 0 failed, 0 ignored**; doc-tests 0 | `artifacts/logs/config-test.log` |
| `cargo clippy -p dragonfly-client-storage -- -D warnings` | Passed, `Finished dev profile` | `artifacts/logs/clippy.log` |

The pre-change storage baseline was 108 passed (`artifacts/logs/baseline-test.log`). Final coverage includes round-trip, empty/multi-chunk/ranged/large content, raw ciphertext, tamper and truncation detection, zeroed-record partial rewrite rejection, wrong keys, nonce uniqueness across chunks and rewrites, concurrent partial writes, raw/hex/environment keys, missing-key fail-fast before storage modification, legacy plaintext rejection, encrypted-to-disabled restart rejection, plaintext magic-prefix compatibility, persistent content, copy decryption, hard-link rejection, and TCP plaintext delivery.

macOS source paths mirror Linux behavior, but macOS compilation and runtime execution were not verified on this Linux x86_64 host. No performance benchmark numbers are claimed. `artifacts/` is intentionally not committed.
