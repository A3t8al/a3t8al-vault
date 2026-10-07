# A3t8al Vault

A single-file encrypted vault for Linux and iSH. The executable stores file names, metadata, and file contents inside an authenticated encrypted image.

## Security model

The vault protects the contents of `vault.img` against offline reading and accidental or malicious byte modification. Passwords are expanded with PBKDF2-HMAC-SHA256 (250,000 iterations), and the database is encrypted and authenticated with ChaCha20-Poly1305.

This release does **not** protect against keyloggers, malware on the host, a stolen password, weak passwords, rollback to an older image, process-memory inspection, or traffic/size analysis. It is not a replacement for age, LUKS, or a professionally audited encrypted filesystem.

## Included commands

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

The `--size` value is retained for command compatibility; this release grows the encrypted image as needed.

## Linux installation

Install OpenSSL 3 and copy the executable to your PATH:

```sh
sudo apt install libssl3
chmod +x vault
sudo cp vault /usr/local/bin/vault
vault selftest
```

## iSH installation

This release includes a 32-bit Intel executable intended for iSH. In iSH, install the runtime and copy the executable into the current directory:

```sh
apk update
apk add libcrypto3
chmod +x vault
./vault selftest
```

If your iSH image uses a different OpenSSL package name, run `apk search openssl` and install the package providing `libcrypto.so.3`.

Create and use a vault:

```sh
./vault create --size 64M vault.img 'use-a-long-unique-password'
./vault put vault.img 'use-a-long-unique-password' photo.jpg /documents/photo.jpg
./vault ls vault.img 'use-a-long-unique-password'
./vault get vault.img 'use-a-long-unique-password' /documents/photo.jpg restored.jpg
./vault fsck vault.img 'use-a-long-unique-password'
```

Keep `vault.img` private and make an independent backup. The image is the encrypted data container; files placed inside it are not stored as ordinary files on disk.

## Interactive mode

```sh
./vault open vault.img 'use-a-long-unique-password'
```

Available commands are `ls`, `rm PATH`, and `quit`.

## Image format

The image contains a random 16-byte salt, a random 12-byte nonce, authenticated ciphertext, and a 16-byte Poly1305 tag. The plaintext database is never written to the image unencrypted. A wrong password or modified byte is rejected with `tampering detected or wrong password`.

## Verification

From the development tree:

```sh
make test
make fuzz
```

The distributable package intentionally contains no source code. The binary was built with strict C11 warnings enabled and the 32-bit iSH build passed the cryptographic self-test.

## Limitations

This release rewrites the encrypted database on each change. It does not yet provide key slots, key rotation, snapshots, hidden volumes, encrypted size padding, a full WAL, rollback counters, Merkle indexing, or secure password prompting. Do not use it for high-value secrets without an independent security review.
