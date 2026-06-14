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

**How:** The pipeline looks up the recipient's public key via the domain auth agent's `auth.KeyProvider`, keyed by the recipient's base localpart (matching the key backend's per-username `.pub` files). When a key exists, the message is encrypted once with `msgstore.EncryptMessage` (NaCl box; the blob is self-describing: ephemeral public key || nonce || ciphertext) and every write path stores the same encrypted bytes -- the stage-4 keep path (with `Envelope.Encryption` metadata set), Sieve `fileinto` via `DeliverToFolder`, the imap4flags `AppendToFolder` variant, and the local copy from `redirect :copy`. **Key presence is the gate (maildancer#65):** a recipient with a key gets ciphertext; a recipient with no key gets plaintext. There is no per-delivery signal -- the retired `EncryptionKeyHint` is gone.

Encrypting in the pipeline rather than by wrapping `dom.DeliveryAgent` was a deliberate choice: by the time any storage call runs, only ciphertext exists, so no delivery path can bypass the seam — the earlier design (wrap the delivery agent with `msgstore.EncryptingDeliveryAgent`) left Sieve's direct `FolderStore` writes uncovered (maildancer#53). `EncryptingDeliveryAgent` still exists for other callers but the pipeline does not use it.

**Fail-closed:** a recipient who *has* a key but for whom encryption is unsatisfiable (key backend read error, corrupt or wrong-length key, encryption failure) temp-fails the delivery -- never a silent plaintext fallback. A recipient with no key is not an error; that is the plaintext case. The SMTP reason stays generic; the detail goes to the server log.

**Why here:** mail-session oneshot delivery already runs as the recipient's uid (spawned by session-manager with `SysProcAttr.Credential`), has access to the domain config, and is the natural privilege boundary for per-recipient operations.

**Activation is per-domain via key provisioning (maildancer#65).** Because the gate is key presence, a user's mail is encrypted as soon as they have a keypair. The `encryption_mode` field in a domain's `config.toml` -- `off` (default) or `on` -- controls whether `userctl user add` / webadmin provision a keypair for new users. `on` therefore turns on at-rest encryption for that domain's users; `off` leaves them plaintext. The mode governs provisioning, not the runtime gate, so setting a domain back to `off` does not strip existing keys or downgrade stored mail. A future `escrow` mode (recoverable via an admin recovery key) is reserved but unimplemented.

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

### Server-side search under whole-message encryption

Whole-message encryption (headers included — headers *are* content; From/To/Subject/Message-ID/refs are the social graph, topic, and thread structure the at-rest model exists to protect) forecloses a plaintext server-side search index. The accepted property, which holds on both the server and any client: **content search needs no decrypted cache or plaintext index anywhere.**

Predicates are evaluated message-by-message — retrieve, decrypt in memory, test substrings, discard the plaintext. Nothing decrypted is persisted and nothing is indexed. `msgstore.SearchContentStore`, the `ContentSearcher` interface, and `MatchMessageContent` (`msgstore/search.go`) are the shared implementation behind both the local path and the mail-session gRPC handler.

For imapd/pop3d the evaluation runs **inside mail-session** and returns only per-message match booleans (plus the header block when a header or sent-date predicate needs it); message bodies never cross the session-manager gRPC boundary (maildancer#61, implemented).

The cost is honest: brute-force scan-with-decrypt (case-insensitive octet containment), O(folder) per search, no index — the same work a non-indexed IMAP server does, minus the index. There is no graceful escape hatch for very large mailboxes; semantic/indexed search is a client concern, and an SCMP client (or the IMAP/SCMP bridge) holds plaintext locally and can index there. Two items preserve the model without changing the no-cache property: an encrypted-at-rest header-index cache decrypted transiently in-session (maildancer#62, performance only), and — foreclosed — a naive vector index, since embeddings stored at rest leak the plaintext under inversion (maildancer#63).

### Client-side reuse

The same threat model applies to a native SCMP client's local mailbox: a stolen laptop is the client-side equivalent of a stolen mail-store disk, so the client wants its local store encrypted at rest too. The natural construction is the same one used here — the client views its local mailbox through a mail-session-style component that holds the user's key and decrypts on retrieval. mail-session is therefore a reusable at-rest-decryption layer, not a server-only one; SCMP/messagedancer are expected to consume it (or an equivalent) for local storage rather than re-implementing the decrypt-at-retrieval boundary.

## Key Model

### Derivation

Password-derived keys: a user's password plus a per-user salt produces an X25519 key pair via a KDF (Argon2id or HKDF — TBD). The function signature is defined in `auth/keys.go`:

```go
func DeriveKeyPair(password, username string, salt []byte) (pub, priv []byte, err error)
```

Currently a stub. The output is two opaque 32-byte slices compatible with `nacl/box`.

**What is actually implemented is the keyring model, not direct derivation:** the `auth/passwd` backend stores a sealed private key per user (`keys/<user>.key`: salt || nonce || secretbox under an Argon2id password-derived key) alongside the raw 32-byte public key (`keys/<user>.pub`). `Agent.Authenticate` unseals the private key at login and returns it in `AuthSession.PrivateKey`. `DeriveKeyPair` (direct password-to-key derivation, no stored key file) remains a stub and an open alternative.

**Provisioning:** `internal/admin/keys.GenerateKeypair` writes the sealed `.key` and raw `.pub` in exactly the format `auth/passwd` reads (a cross-package round-trip test pins the shared Argon2id parameters against drift). It is driven by `userctl user key create` / `user add --gen-keys` and the webadmin key UI. Because the key is sealed under the user's password, changing the password must re-seal it: `admin.ChangePassword` (current password known) re-seals the same keypair, `admin.ResetPasswordRegenKeys` (admin reset, current password unknown) regenerates it — a bare `ResetPassword` on a keyed user is refused, since it would orphan the key and lock the user out (`auth/passwd` treats an unsealable key as a hard authentication failure). See maildancer#58 / PR #59.

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

The delivery envelope carries no encryption signal. mail-session decides to encrypt purely on whether the recipient has a public key (see the activation note above). The `encryption_key_hint` field on the `DeliveryMetadata` protobuf was retired in maildancer#65 -- field 6 is now `reserved` -- and `DeliverRequest` no longer carries it.

### Storage (Envelope)

`msgstore.Envelope.Encryption *EncryptionInfo` carries per-recipient encryption metadata after delivery. The store writes this alongside the encrypted blob.

## Existing Implementation

The following is already implemented in the maildancer monorepo:

| Component | File | Status |
|-----------|------|--------|
| Pipeline encryption stage 3.5 (`maybeEncrypt`) | `internal/mail-session/deliver/encrypt.go` | Implemented, tested, **active** -- gated on recipient key presence (maildancer#65); covers all write paths including Sieve fileinto (maildancer#53) |
| Per-domain `encryption_mode` (off/on) | `auth/domain/config.go`, `internal/admin` | Implemented, tested -- `on` provisions a keypair at user creation, activating at-rest encryption for the domain (maildancer#65) |
| `EncryptMessage()` / `DecryptMessage()` | `msgstore/encrypting_delivery.go` | Implemented |
| `EncryptingDeliveryAgent` | `msgstore/encrypting_delivery.go` | Implemented, tested — `Deliver()` only; not used by the pipeline |
| `DecryptingStore` interface | `msgstore/store.go` | Defined |
| Decrypting store (folder-aware, encrypts appends) | `msgstore/decrypting_store.go` | Implemented, tested |
| `EncryptionInfo` | `msgstore/crypto.go` | Defined |
| `Envelope.Encryption` | `msgstore/delivery.go` | Defined |
| `KeyProvider` interface | `auth/agent.go` | Defined; implemented by the passwd backend (`keys/<user>.pub`, raw 32 bytes) |
| `DeriveKeyPair` | `auth/keys.go` | Stub |
| `EncryptionKeyHint` in wire protocol | (removed) | Retired in maildancer#65 -- gate is recipient key presence; `DeliverMetadata` field 6 reserved |
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
- **Key rotation:** ~~When a user changes their password, how are existing encrypted messages re-encrypted?~~ **Resolved for password changes (maildancer#59):** the password seals the private key, not the messages, so a password change re-seals the *same* X25519 keypair and existing mail needs no re-encryption (`admin.ChangePassword`). Still open: rotating the message-encryption *keypair itself* (as opposed to its password seal) — there is no mechanism to re-encrypt a mailbox from an old keypair to a new one, so today regenerating the keypair (`admin.ResetPasswordRegenKeys`, the admin-reset path) abandons access to old mail rather than migrating it. A batch re-encrypt or lazy-on-access migration would be needed to make true keypair rotation non-destructive.
- **Admin recovery (the reserved `escrow` mode):** There is no escrow today, so under `encryption_mode = "on"` a forgotten password means permanent data loss -- which is why `off` is the default. A recoverable `escrow` mode is reserved in the config but unimplemented. It is the substantive remaining work: the current blob is sealed to a single recipient key (`box.Seal` with one ephemeral sender), so recovery-by-second-key needs a format change (a random per-message DEK wrapped to both the user's and a domain recovery key, or a parallel escrow blob) plus safe custody of the domain recovery private key (admin-held and itself password/HSM-sealed, not sitting beside the DKIM key). Escrow deliberately weakens the "server holds no key that decrypts user mail" property, so it must stay opt-in per domain.
- **Multiple device keys:** Should a user be able to have multiple key pairs (one per device) with the message encrypted to all of them?
- **Header preservation:** IMAP SEARCH and SORT require access to parsed headers. If messages are encrypted as opaque blobs, server-side search is impossible. Options: encrypt body only, store headers in plaintext alongside the encrypted blob, or require client-side search.
- **KDF choice:** Argon2id (memory-hard, widely recommended) vs HKDF (fast, deterministic). Argon2id is preferred for password-derived keys.
