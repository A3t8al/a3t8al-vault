# Design Notes

## Purpose

A3t8al Vault is a small encrypted container with a deliberately narrow command-line interface. It favors a simple format that can be audited over feature breadth.

## Cryptography

The implementation uses OpenSSL EVP for PBKDF2-HMAC-SHA256 and ChaCha20-Poly1305. Standard library primitives are used instead of inventing cryptographic algorithms. Each save operation generates a fresh salt and nonce, derives a fresh key, encrypts the complete database, and writes an authenticated image.

## Storage

The image layout is:

```text
16-byte random salt | 12-byte random nonce | ciphertext | 16-byte Poly1305 tag
```

The encrypted database contains a version, an entry count, and length-delimited records. Each record stores a path, mode, size, and bytes. No fixed plaintext magic value is written to the image.

## Atomic replacement

Writes are made to `IMAGE.tmp`, flushed, synced, and renamed over the old image. This reduces the chance of leaving a partially written image, but it is not a complete write-ahead log or rollback defense.

## Distribution

The public distribution is binary-only. The development source tree is kept separately so the release artifact contains the executable and English documentation rather than implementation source files.
