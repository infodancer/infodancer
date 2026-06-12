# At-Rest Encryption Design

This document describes the design for at-rest encryption of stored messages in the infodancer mail stack.

*Updated 2026-06-12 for the maildancer monorepo: the former mail-deliver agent has been merged into mail-session, and session-manager now owns subprocess spawning. Component names and paths below reflect the consolidated layout.*

## Overview

Messages are encrypted before being written to the message store and decrypted after retrieval. The store handles opaque encrypted blobs only — plaintext never touches disk after delivery. Encryption and decryption happen in the privilege-separated subprocess layer (mail-session), not in the listener daemons.

The encryption primitive is NaCl box: X25519 key agreement + XSalsa20-Poly1305 AEAD. This is already implemented in `msgstore` via `golang.org/x/crypto/nacl/box`.

## Goals

- At-rest encryption: a compromised mail store cannot be read without the user's private key
- Spam scanning and Sieve filtering happen before encryption so they see plaintext
- No plaintext written to disk after delivery
- Key material is never stored on disk; it is derived at authentication time

## Non-Goals

- Transport encryption (that is handled by TLS/STARTTLS)
- End-to-end encryption between users (that is SCMP/SDMP territory)
- Server-side key escrow or admin recovery (open question — see below)

## Encryption Point

**Where:** the delivery pipeline in `internal/mail-session/deliver`, after the spam check and Sieve execution (stage 3), before the final mailbox write (stage 4, `deliverLocal`, which calls `dom.DeliveryAgent.Deliver()`).

**How:** When `DeliverRequest.EncryptionKeyHint` is non-empty, the pipeline resolves the hint to a public key via `auth.KeyProvider` and wraps `dom.DeliveryAgent` with `msgstore.EncryptingDeliveryAgent` before calling `Deliver()`. The hint is a key fingerprint or identifier — not the key itself.

`msgstore.EncryptingDeliveryAgent` is fully implemented and tested. It encrypts per-recipient using NaCl box and sets `Envelope.Encryption` metadata.

**Why here:** mail-session oneshot delivery already runs as the recipient's uid (spawned by session-manager with `SysProcAttr.Credential`), has access to the domain config, and is the natural privilege boundary for per-recipient operations.

### Sieve interaction

Sieve runs at stage 3, on plaintext, before the encryption point — same rationale as spam scanning. Two consequences:

- Header, body, and size tests work unmodified; nothing in Sieve needs to know about at-rest encryption.
- **Sieve `fileinto` currently bypasses the delivery-agent seam**: it writes via `FolderStore.DeliverToFolder`/`AppendToFolder` directly on the store, and `EncryptingDeliveryAgent` has no folder-delivery methods. Wiring encryption MUST extend the seam to cover folder writes, or a filtering user gets encrypted inbox mail and plaintext filed mail. Tracked as [maildancer#53](https://github.com/infodancer/maildancer/issues/53); that issue also specifies the guard test (deliver via fileinto with a key hint set, assert the on-disk blob is not plaintext).

## Decryption Point

**Where:** `mail-session`, after reading bytes from the message store, before they leave the process over gRPC. pop3d and imapd reach mail-session through session-manager's gRPC proxy and only ever see what mail-session returns.

**Interface:** `msgstore.DecryptingStore` wraps a `MessageStore` and intercepts `Retrieve`/`RetrieveFromFolder` to call `DecryptMessage` when a session key is set.

**Current state:** `msgstore.PassthroughDecryptingStore` is a no-op implementation that holds the session key but does not decrypt. A real implementation calling `msgstore.DecryptMessage` will be added when encryption is fully wired in.

**Why here:** IMAP FETCH with `Envelope`/`BodyStructure` options parses RFC 5322 structure from raw bytes. Decryption must happen before bytes reach the IMAP protocol layer — i.e., inside mail-session before the bytes are returned over gRPC.

## Key Model

### Derivation

Password-derived keys: a user's password plus a per-user salt produces an X25519 key pair via a KDF (Argon2id or HKDF — TBD). The function signature is defined in `auth/keys.go`:

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

session-manager — which authenticates the user and spawns mail-session with the user's uid/gid — must pass the user's decrypted private key to the mail-session subprocess securely.

**Convention:** fd 3 carries a versioned JSON key envelope.

### Envelope Format

```json
{"version":1,"key":"<base64-encoded 32 bytes>"}
```

The JSON envelope (rather than raw bytes) provides an extension point without a breaking protocol change. Future fields can be added to v1, or v2 can add new top-level keys:

| Field | Type | Description |
|-------|------|-------------|
| `version` | int | Always `1` for now |
| `key` | []byte (base64) | 32-byte NaCl private key |
| `algorithm` *(future)* | string | e.g. `"nacl-box"` |
| `key_id` *(future)* | string | Key fingerprint for logging/rotation |
| `expires` *(future)* | string (RFC3339) | Key expiry time |
| `keys` *(future v2)* | array | Multiple keys for keyring/device support |

### Protocol

1. After authenticating the user, session-manager derives or retrieves the user's private key
2. Creates an `os.Pipe()`
3. JSON-encodes the key envelope to the write end and closes it
4. Passes the read end as `cmd.ExtraFiles[0]` (fd 3 in the child)
5. mail-session JSON-decodes the envelope from fd 3 at startup, closes the fd immediately
6. Validates `version == 1` and `len(key) == 32`; falls through unchanged on any error
7. Calls `DecryptingStore.SetSessionKey()` with the key bytes
8. Calls `DecryptingStore.ClearSessionKey()` on exit

**No-encryption case:** fd 3 is absent. mail-session catches the EBADF error and uses the store unchanged.

**Security properties:**
- Key material never appears in argv or environment variables
- The fd is closed immediately after reading — key is held in memory only
- ClearSessionKey zeroes the bytes before releasing the slice

## Wire Protocol

### Delivery (smtpd → session-manager → mail-session)

`DeliverRequest` in `internal/mail-session/deliver/deliver.go` carries:

```go
EncryptionKeyHint string
```

Empty means no encryption. The hint format is a key fingerprint or identifier resolved by `auth.KeyProvider`.

The field travels as `encryption_key_hint` in the `DeliveryMetadata` protobuf message (`internal/mail-session/proto/mailsession/v1`), populated by smtpd's `SessionManagerDeliveryAgent` (`internal/smtpd/smtp/smdeliver.go`) and unpacked by mail-session's gRPC delivery server. It is currently carried end to end but not yet acted on by the pipeline.

### Storage (Envelope)

`msgstore.Envelope.Encryption *EncryptionInfo` carries per-recipient encryption metadata after delivery. The store writes this alongside the encrypted blob.

## Existing Implementation

The following is already implemented in the maildancer monorepo:

| Component | File | Status |
|-----------|------|--------|
| `EncryptingDeliveryAgent` | `msgstore/encrypting_delivery.go` | Implemented, tested — `Deliver()` only; no folder delivery (see maildancer#53) |
| `DecryptMessage()` | `msgstore/encrypting_delivery.go` | Implemented |
| `DecryptingStore` interface | `msgstore/store.go` | Defined |
| `PassthroughDecryptingStore` | `msgstore/decrypting_store.go` | Implemented (no-op) |
| `EncryptionInfo` | `msgstore/crypto.go` | Defined |
| `Envelope.Encryption` | `msgstore/delivery.go` | Defined |
| `KeyProvider` interface | `auth/agent.go` | Defined |
| `DeriveKeyPair` | `auth/keys.go` | Stub |
| `EncryptionKeyHint` in wire protocol | `internal/mail-session/deliver/deliver.go` + `proto/mailsession/v1` | Defined, carried end to end, not yet acted on |
| fd 3 key reading | `cmd/mail-session/main.go` (`maybeWrapWithDecryptingStore`, marked `── Encryption seam ──`) | Implemented |
| fd 3 pipe creation | session-manager spawn path (`internal/session-manager/manager`) | Not yet implemented |

## Outbound Queue Encryption and DKIM Signing

The outbound mail queue has a fundamentally different encryption model from the mailbox store.

### Why the queue cannot use per-user keys

Mailbox encryption protects the recipient's stored mail — encrypted with the recipient's public key, only they can decrypt. The server never holds private keys.

The outbound queue is different: the server must read the message to deliver it. It cannot be encrypted with a key only the user holds. The server is the sender's delivery agent and needs cleartext access.

### Domain key: signing and queue encryption

Each sending domain has an Ed25519 keypair used for DKIM signing. The server legitimately holds this private key (it must sign on behalf of the domain). This same key material serves double duty:

1. **DKIM signing** — authentication for SMTP receivers (RFC 6376)
2. **Queue body encryption** — at-rest protection for outbound messages

The Ed25519 signing key is converted to X25519 for encryption (a well-established derivation). One keypair per domain covers both uses.

### Signing and encryption flow

At **enqueue time** (session-manager's OutboundService receives the message from smtpd and has cleartext; DKIM signing lives in `internal/session-manager/dkim`):

1. Compute the DKIM-Signature header over the canonicalized message
2. Store the DKIM-Signature value in the envelope file
3. Encrypt the message body with the domain's X25519 public key (derived from the Ed25519 DKIM key)
4. Write the encrypted body to the queue

At **send time** (mail-remote, invoked by queue-manager):

- **SMTP delivery:** decrypt body with domain private key, prepend stored DKIM-Signature header, send
- **SDMP delivery:** decrypt body, send via SDMP (no DKIM needed)

Queue body encryption (steps 3-4 and the send-time decrypt) is design-only as of this update; DKIM signing at enqueue time is implemented.

### Transport-dependent behavior

| Submission | Delivery | DKIM signed? | Queue encrypted? |
|------------|----------|-------------|-----------------|
| SMTP (587) | SMTP     | Yes         | Yes (domain key) |
| SMTP (587) | SDMP     | Signature stored but unused | Yes (domain key) |
| SCMP       | SDMP     | No          | End-to-end encrypted by client |

SCMP-submitted messages arrive already encrypted by the client. The server is a blind relay — it cannot read, sign, or re-encrypt the content. SCMP messages can only be delivered via SDMP to domains that support the new protocol.

If a client needs to send to an SMTP-only domain, it submits via SMTP (port 587). The client checks SRV records at send time to determine transport availability; emergency fallback from SCMP to SMTP submission is possible but trades end-to-end encryption for deliverability (transport remains TLS-encrypted).

### DKIM key management

- One Ed25519 keypair per sending domain
- Private key stored in domain configuration (alongside TLS certs, hostname, etc.)
- Public key published in DNS: `selector._domainkey.example.com TXT "v=DKIM1; k=ed25519; p=<base64>"`
- Selector name is configurable per domain

### Relaying and ARC

When relaying a message received via SDMP that must be delivered via SMTP (future federation scenario), the server signs with its own domain key. The From: header won't match the DKIM signing domain, so DMARC `dkim=` alignment fails. ARC (Authenticated Received Chain, RFC 8617) headers should be added to preserve the authentication chain through the relay.

## Open Questions

- **Per-folder encryption:** Should only specific folders (e.g., INBOX, Sent) be encrypted, with others in plaintext for performance?
- **Key rotation:** When a user changes their password, how are existing encrypted messages re-encrypted? Batch job? Lazy on access?
- **Admin recovery:** Should there be an admin recovery key? Current design has no escrow — a forgotten password means permanent data loss.
- **Multiple device keys:** Should a user be able to have multiple key pairs (one per device) with the message encrypted to all of them?
- **Header preservation:** IMAP SEARCH and SORT require access to parsed headers. If messages are encrypted as opaque blobs, server-side search is impossible. Options: encrypt body only, store headers in plaintext alongside the encrypted blob, or require client-side search.
- **KDF choice:** Argon2id (memory-hard, widely recommended) vs HKDF (fast, deterministic). Argon2id is preferred for password-derived keys.
