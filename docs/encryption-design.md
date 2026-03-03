# At-Rest Encryption Design

This document describes the design for at-rest encryption of stored messages in the infodancer mail stack.

## Overview

Messages are encrypted before being written to the message store and decrypted after retrieval. The store handles opaque encrypted blobs only — plaintext never touches disk after delivery. Encryption and decryption happen in the subprocess layer (mail-deliver, mail-session), not in the listener daemons.

The encryption primitive is NaCl box: X25519 key agreement + XSalsa20-Poly1305 AEAD. This is already implemented in `msgstore` via `golang.org/x/crypto/nacl/box`.

## Goals

- At-rest encryption: a compromised mail store cannot be read without the user's private key
- Spam scanning happens before encryption so the scanner sees plaintext
- No plaintext written to disk after delivery
- Key material is never stored on disk; it is derived at authentication time

## Non-Goals

- Transport encryption (that is handled by TLS/STARTTLS)
- End-to-end encryption between users
- Server-side key escrow or admin recovery (open question — see below)

## Encryption Point

**Where:** `mail-deliver`, after the spam check, before calling `dom.DeliveryAgent.Deliver()`.

**Seam:** `mail-deliver/internal/deliver/deliver.go`, marked with the comment block `── Encryption seam ──`.

**How:** When `DeliverRequest.EncryptionKeyHint` is non-empty, `mail-deliver` resolves the hint to a public key via `auth.KeyProvider` and wraps `dom.DeliveryAgent` with `msgstore.EncryptingDeliveryAgent` before calling `Deliver()`. The hint is a key fingerprint or identifier — not the key itself.

`msgstore.EncryptingDeliveryAgent` is fully implemented and tested. It encrypts per-recipient using NaCl box and sets `Envelope.Encryption` metadata.

**Why here:** mail-deliver already runs as the recipient's uid, has access to the domain config, and is the natural privilege boundary for per-recipient operations.

## Decryption Point

**Where:** `mail-session`, after reading bytes from the message store, before returning them over the pipe to pop3d/imapd.

**Interface:** `msgstore.DecryptingStore` wraps a `MessageStore` and intercepts `Retrieve`/`RetrieveFromFolder` to call `DecryptMessage` when a session key is set.

**Current state:** `msgstore.PassthroughDecryptingStore` is a no-op implementation that holds the session key but does not decrypt. A real implementation calling `msgstore.DecryptMessage` will be added when encryption is fully wired in.

**Why here:** IMAP FETCH with `Envelope`/`BodyStructure` options parses RFC 5322 structure from raw bytes. Decryption must happen before bytes reach the IMAP protocol layer — i.e., inside mail-session before the bytes are returned over the pipe.

## Key Model

### Derivation

Password-derived keys: a user's password plus a per-user salt produces an X25519 key pair via a KDF (Argon2id or HKDF — TBD). The function signature is defined in `auth`:

```go
func DeriveKeyPair(password, username string, salt []byte) (pub, priv []byte, err error)
```

Currently a stub. The output is two opaque 32-byte slices compatible with `nacl/box`.

### Storage

- **Public key:** stored in the domain key backend, accessible to `auth.KeyProvider`
- **Private key:** never stored; re-derived from the user's password at authentication time

### Future: Keyring

The design supports a future upgrade where the password unlocks a per-user keyring (e.g., a sealed key blob stored on disk) rather than deriving the key directly from the password each time. The `DeriveKeyPair` interface accommodates this migration — callers treat the output as opaque.

## Key Passing to mail-session

The spawning daemon (pop3d or imapd) must pass the user's decrypted private key to the mail-session subprocess securely.

**Convention:** fd 3 carries the 32-byte raw private key.

**Protocol:**
1. After authentication, the daemon derives or retrieves the user's private key
2. Creates an `os.Pipe()`
3. Writes the 32-byte key to the write end and closes it
4. Passes the read end as `cmd.ExtraFiles[0]` (fd 3 in the child)
5. mail-session reads up to 32 bytes from fd 3 at startup, closes the fd immediately
6. Calls `DecryptingStore.SetSessionKey()` with the key bytes
7. Calls `DecryptingStore.ClearSessionKey()` on exit

**No-encryption case:** Write 0 bytes to the pipe. mail-session reads 0 bytes and does not set a session key.

**Security properties:**
- Key material never appears in argv or environment variables
- The fd is closed immediately after reading — key is held in memory only
- ClearSessionKey zeroes the bytes before releasing the slice

## Wire Protocol

### Delivery (smtpd → mail-deliver)

`DeliverRequest` in `mail-deliver/protocol/protocol.go` carries:

```go
EncryptionKeyHint string `json:"encryption_key_hint,omitempty"`
```

Empty means no encryption. The hint format is a key fingerprint or identifier resolved by `auth.KeyProvider`. The field is `omitempty` for backward compatibility — existing requests without it deserialize cleanly.

The same field is mirrored in smtpd's internal wire type (`smtpd/internal/maildeliver/wire.go`).

### Storage (Envelope)

`msgstore.Envelope.Encryption *EncryptionInfo` carries per-recipient encryption metadata after delivery. The store writes this alongside the encrypted blob.

## Existing Implementation

The following is already implemented and tested in `msgstore`:

| Component | File | Status |
|-----------|------|--------|
| `EncryptingDeliveryAgent` | `msgstore/encrypting_delivery.go` | Implemented, tested |
| `DecryptMessage()` | `msgstore/encrypting_delivery.go` | Implemented |
| `DecryptingStore` interface | `msgstore/store.go` | Defined |
| `PassthroughDecryptingStore` | `msgstore/decrypting_store.go` | Implemented (no-op) |
| `EncryptionInfo` | `msgstore/crypto.go` | Defined |
| `Envelope.Encryption` | `msgstore/delivery.go` | Defined |
| `KeyProvider` interface | `auth/agent.go` | Defined |
| `DeriveKeyPair` | `auth/keys.go` | Stub |
| `EncryptionKeyHint` in wire protocol | `mail-deliver/protocol/protocol.go` | Defined |
| fd 3 key reading | `mail-session/cmd/mail-session/main.go` | Plumbing only |
| fd 3 pipe creation | `pop3d`, `imapd` subprocess spawn | Plumbing only |

## Open Questions

- **Per-folder encryption:** Should only specific folders (e.g., INBOX, Sent) be encrypted, with others in plaintext for performance?
- **Key rotation:** When a user changes their password, how are existing encrypted messages re-encrypted? Batch job? Lazy on access?
- **Admin recovery:** Should there be an admin recovery key? Current design has no escrow — a forgotten password means permanent data loss.
- **Multiple device keys:** Should a user be able to have multiple key pairs (one per device) with the message encrypted to all of them?
- **Header preservation:** IMAP SEARCH and SORT require access to parsed headers. If messages are encrypted as opaque blobs, server-side search is impossible. Options: encrypt body only, store headers in plaintext alongside the encrypted blob, or require client-side search.
- **KDF choice:** Argon2id (memory-hard, widely recommended) vs HKDF (fast, deterministic). Argon2id is preferred for password-derived keys.
