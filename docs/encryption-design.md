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

**Implemented** (maildancer#53 / PR #54).

**Where:** the delivery pipeline in `internal/mail-session/deliver`, as stage 3.5 (`encrypt.go`) — after the spam check and Sieve execution (stage 3, which evaluate plaintext), before any write.

**How:** When `DeliverRequest.EncryptionKeyHint` is non-empty, the pipeline resolves the recipient's public key via the domain auth agent's `auth.KeyProvider` and encrypts the message once with `msgstore.EncryptMessage` (NaCl box; the blob is self-describing: ephemeral public key || nonce || ciphertext). Every write path then stores the same encrypted bytes — the stage-4 keep path (with `Envelope.Encryption` metadata set), Sieve `fileinto` via `DeliverToFolder`, the imap4flags `AppendToFolder` variant, and the local copy from `redirect :copy`. The hint requests encryption; the key lookup is by the recipient's base localpart, matching the key backend's per-username `.pub` files.

Encrypting in the pipeline rather than by wrapping `dom.DeliveryAgent` was a deliberate choice: by the time any storage call runs, only ciphertext exists, so no delivery path can bypass the seam — the earlier design (wrap the delivery agent with `msgstore.EncryptingDeliveryAgent`) left Sieve's direct `FolderStore` writes uncovered (maildancer#53). `EncryptingDeliveryAgent` still exists for other callers but the pipeline does not use it.

**Fail-closed:** encryption requested but unsatisfiable (no key provider, no key on file, encryption failure) temp-fails the delivery — never a silent plaintext fallback. The SMTP reason stays generic; the detail goes to the server log.

**Why here:** mail-session oneshot delivery already runs as the recipient's uid (spawned by session-manager with `SysProcAttr.Credential`), has access to the domain config, and is the natural privilege boundary for per-recipient operations.

**Not yet active in production:** smtpd does not set `EncryptionKeyHint`, so deliveries remain plaintext until the submission side opts in. Activating it requires the retrieval path (below) first.

### Sieve interaction

Sieve runs at stage 3, on plaintext, before the encryption point — same rationale as spam scanning. Header, body, and size tests work unmodified; nothing in Sieve needs to know about at-rest encryption. The guard test (`TestEncrypt_SieveFileInto`) pins the evaluate-then-encrypt ordering: a header condition matches the plaintext Subject while the on-disk folder blob is asserted to be ciphertext.

## Decryption Point

**Where:** `mail-session`, after reading bytes from the message store, before they leave the process over gRPC. pop3d and imapd reach mail-session through session-manager's gRPC proxy and only ever see what mail-session returns.

**Interface:** `msgstore.DecryptingStore` wraps a `MessageStore` and intercepts `Retrieve`/`RetrieveFromFolder` to call `DecryptMessage` when a session key is set.

**Implemented** (maildancer#55 / PR #57). The wrapper decrypts retrieval and serves undecryptable content raw — plaintext stored before encryption was enabled, or a blob for a different key, comes back unchanged rather than erroring, so mixed mailboxes work. Two correctness properties worth knowing:

- The wrapper preserves `FolderStore` when the underlying store has it (mail-session's `session.Session` type-asserts at construction; a wrapper hiding the interface would silently disable IMAP folders).
- Folder writes encrypt: `AppendToFolder` (IMAP APPEND — drafts, saved sent mail) and `DeliverToFolder` encrypt with the public key derived from the session private key, so client appends cannot reintroduce plaintext into an encrypted mailbox.

**Why here:** IMAP FETCH with `Envelope`/`BodyStructure` options parses RFC 5322 structure from raw bytes. Decryption must happen before bytes reach the IMAP protocol layer — i.e., inside mail-session before the bytes are returned over gRPC.

**Known limitations:** POP3 LIST/STAT and IMAP RFC822.SIZE report stored (encrypted) sizes, +72 bytes per encrypted message vs. what RETR/FETCH returns. Sessions authenticated without a password (OAUTHBEARER) cannot decrypt the key file, so encrypted messages are served as raw blobs.

## Key Model

### Derivation

Password-derived keys: a user's password plus a per-user salt produces an X25519 key pair via a KDF (Argon2id or HKDF — TBD). The function signature is defined in `auth/keys.go`:

```go
func DeriveKeyPair(password, username string, salt []byte) (pub, priv []byte, err error)
```

Currently a stub. The output is two opaque 32-byte slices compatible with `nacl/box`.

**What is actually implemented is the keyring model, not direct derivation:** the `auth/passwd` backend stores a sealed private key per user (`keys/<user>.key`: salt || nonce || secretbox under an Argon2id password-derived key) alongside the raw 32-byte public key (`keys/<user>.pub`). `Agent.Authenticate` unseals the private key at login and returns it in `AuthSession.PrivateKey`. `DeriveKeyPair` (direct password-to-key derivation, no stored key file) remains a stub and an open alternative.

**Provisioning gap:** nothing writes `.key` files yet — `auth/passwd` can decrypt them but has no encrypt counterpart, and userctl has no key-provisioning command. Until that exists, user encryption keys can only be created by hand.

### Storage

- **Public key:** stored in the domain key backend (`keys/<user>.pub`, raw 32 bytes), accessible to `auth.KeyProvider`
- **Private key:** stored sealed (`keys/<user>.key`); unsealed only at authentication time, held in memory for the session

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
| Pipeline encryption stage 3.5 (`maybeEncrypt`) | `internal/mail-session/deliver/encrypt.go` | Implemented, tested — covers all write paths including Sieve fileinto (maildancer#53) |
| `EncryptMessage()` / `DecryptMessage()` | `msgstore/encrypting_delivery.go` | Implemented |
| `EncryptingDeliveryAgent` | `msgstore/encrypting_delivery.go` | Implemented, tested — `Deliver()` only; not used by the pipeline |
| `DecryptingStore` interface | `msgstore/store.go` | Defined |
| Decrypting store (folder-aware, encrypts appends) | `msgstore/decrypting_store.go` | Implemented, tested |
| `EncryptionInfo` | `msgstore/crypto.go` | Defined |
| `Envelope.Encryption` | `msgstore/delivery.go` | Defined |
| `KeyProvider` interface | `auth/agent.go` | Defined; implemented by the passwd backend (`keys/<user>.pub`, raw 32 bytes) |
| `DeriveKeyPair` | `auth/keys.go` | Stub |
| `EncryptionKeyHint` in wire protocol | `internal/mail-session/deliver/deliver.go` + `proto/mailsession/v1` | Acted on by stage 3.5; smtpd does not send it yet |
| fd 3 key reading | `cmd/mail-session/main.go` (`maybeWrapWithDecryptingStore`, marked `── Encryption seam ──`) | Implemented |
| fd 3 pipe creation | session-manager spawn path (`internal/session-manager/manager`, `keyPipe`) | Implemented — Login passes `AuthSession.PrivateKey` to spawn |

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
