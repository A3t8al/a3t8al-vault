# A3t8al Vault — Technical Design

<p align="center"><img src="assets/logo.svg" alt="A3t8al Vault" width="520"></p>

![System architecture](assets/architecture.png)

## 1. Scope

A3t8al Vault is a single-file encrypted container for a command-line environment. The primary target is iSH on iPhone, where a small executable can manage private files without presenting the container as a mounted filesystem.

The design prioritizes a small operational surface, authenticated storage, deterministic failure on corruption, and a distribution that can be used without installing a compiler in iSH.

## 2. Distribution model

The release is binary-only:

- `vault-ish` — 32-bit Intel Linux executable intended for iSH.
- `vault-linux-amd64` — 64-bit Linux executable.
- `README.md` — installation, operation, security, and troubleshooting guide.
- `DESIGN.md` — this technical design.
- `SHA256SUMS` — release integrity hashes.

Development sources are maintained separately and are not included in the binary release.

## 3. Runtime environment

The iSH build is a 32-bit Intel Linux executable. iSH provides a user-mode Linux environment on iPhone and supplies an Alpine-style package manager. The executable requires the OpenSSL 3 runtime library exposing `libcrypto.so.3`.

The iSH installation procedure therefore installs `libcrypto3`, copies the executable into the iSH filesystem, sets the executable bit, and runs the built-in cryptographic self-test.

## 4. Cryptographic construction

The password-processing and encryption pipeline is:

```text
password
   |
   v
PBKDF2-HMAC-SHA256, 250,000 iterations, random 16-byte salt
   |
   v
32-byte encryption key
   |
   v
ChaCha20-Poly1305 with a fresh random 12-byte nonce
   |
   v
ciphertext and 16-byte authentication tag
```

The implementation uses OpenSSL EVP for the standardized primitives. It does not define a new cipher, MAC, KDF, or nonce construction.

Every complete save generates a fresh salt and nonce. The tag is checked before plaintext is accepted. A failed tag, malformed record stream, or incorrect password results in a failure rather than partial output.

## 5. On-disk image format

The image is intentionally simple:

```text
+----------------------+ 0
| random salt (16)     |
+----------------------+
| random nonce (12)    |
+----------------------+
| encrypted database   |
| variable length      |
+----------------------+
| Poly1305 tag (16)    |
+----------------------+
```

No fixed plaintext magic value, password, path, or file content is written to the image.

The encrypted database contains:

```text
uint32 version
uint64 record_count
record[record_count]
```

Each record contains a path length, data length, mode, UTF-8 path bytes, and file bytes. The current implementation uses a complete database rewrite for each update.

## 6. Write path

A save operation performs the following steps:

1. Serialize the in-memory record set.
2. Generate a new salt and nonce.
3. Derive a key with PBKDF2-HMAC-SHA256.
4. Encrypt and authenticate the serialized database.
5. Write the new image to `IMAGE.tmp`.
6. Flush and synchronize the temporary file.
7. Rename the temporary image over the previous image.
8. Clear temporary key and plaintext buffers where supported by the runtime.

This reduces the risk of leaving a half-written destination after a process interruption. It is not a complete transactional WAL and does not provide rollback detection.

## 7. Command interface

The command interface deliberately keeps paths explicit:

```text
vault create --size SIZE IMAGE PASSWORD
vault put IMAGE PASSWORD LOCAL_FILE /remote/path
vault get IMAGE PASSWORD /remote/path LOCAL_FILE
vault ls IMAGE PASSWORD
vault rm IMAGE PASSWORD /remote/path
vault fsck IMAGE PASSWORD
vault open IMAGE PASSWORD
vault selftest
```

The `--size` option is retained for interface compatibility. The current implementation grows the image as records are added rather than reserving a fixed block device.

## 8. iSH-specific operational considerations

iSH is an application environment on iPhone, not a native iOS filesystem driver. The vault is therefore a command-line container rather than a folder that appears in the Files application.

Files are imported into the iSH working directory, inserted into `vault.img`, and should be removed from the working directory after successful insertion. Files extracted with `get` are ordinary plaintext files and require separate handling.

iOS may suspend or terminate an application. Users should keep an independent backup of `vault.img` and run `fsck` after an interrupted operation.

## 9. Threat model

The design addresses offline inspection and modification of the image. It does not address endpoint compromise, password capture, process-memory inspection, rollback, traffic analysis, or plaintext generated outside the container.

The password is currently accepted as a command-line argument. This keeps the interface portable in iSH but can expose the password to local process inspection while the command is running. A future release should add a hidden terminal prompt and optional keyfile support.

## 10. Deliberate non-goals in this release

The following features are outside the current implementation:

- Multiple password slots and password rotation.
- Keyfiles or hardware-backed keys.
- Fixed-size encrypted block allocation and size padding.
- Snapshots and copy-on-write storage.
- Hidden volumes or plausible deniability.
- Merkle indexing and rollback counters.
- A full write-ahead log with replay and recovery records.
- A mountable FUSE-like filesystem.

These omissions are documented so that the compact release is not mistaken for a complete encrypted filesystem.

## 11. Validation

The development build is compiled with strict C11 warnings. The release binaries pass:

- PBKDF2 and ChaCha20-Poly1305 round-trip self-test.
- Authentication-failure test after tag modification.
- Create, put, get, list, remove, and `fsck` integration tests.
- Corruption smoke tests that modify random image bytes.
- 32-bit iSH-targeted compilation and self-test.
