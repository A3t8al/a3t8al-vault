# Security Notes

## Reporting a security issue

Do not publish sensitive details of a suspected vulnerability in a public issue. Contact the project owner through the private project channel and include the release version, operating system, command used, and a minimal reproduction that does not contain private data.

## Password handling

Use a long, unique password. Do not reuse an account password. This release receives the password as a command-line argument, so local process inspection may expose it while a command is running. Do not use the vault on a device you do not trust.

## Backups

The encrypted image can be copied safely as an encrypted object, but an old valid copy can still be restored by an attacker with filesystem access. Keep backups versioned and protect their access separately.

## Plaintext boundaries

A file is protected while stored inside `vault.img`. A file supplied to `put` is read from the ordinary iSH filesystem, and a file produced by `get` is written as ordinary plaintext. Remove temporary plaintext copies when they are no longer required.

## Current security limitations

This release does not include secure password prompting, key slots, keyfiles, password rotation, rollback counters, encrypted size padding, snapshots, hidden volumes, memory locking, or a complete write-ahead log. It should not be used as the sole protection for high-value or regulated data.
