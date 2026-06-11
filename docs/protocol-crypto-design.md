# Protocol Cryptographic Design

*Normative design document -- concrete design phase*
*Issue: https://github.com/infodancer/infodancer/issues/5*
*Last updated: 2026-06-11*

This document fixes the cryptographic architecture for the next-generation
messaging protocol: algorithms, key hierarchy, encryption constructions,
signing rules, identifiers, and rotation. It satisfies the `TM-*` requirements
assigned to it by [protocol-threat-model.md](protocol-threat-model.md)
(notably TM-KEY-1..6, TM-ROUTE-1/3, TM-DEVICE-2/3, TM-X-1..4).

Decisions are numbered `CD-n`. Each records the decision, the rationale, and
the rejected alternatives. These are recommendations until the PR merges;
after merge they are design baseline, changed only by superseding document.

---

## CD-1: Algorithm suite

**Decision.** v1 defines exactly one REQUIRED suite, `SDMP-V1`:

| Role | Algorithm | Notes |
|---|---|---|
| Signatures (domain + user) | Ed25519 | RFC 8032 |
| KEM, payload + envelope | Hybrid X25519 + ML-KEM-768 | X-Wing (draft-connolly-cfrg-xwing) as the HPKE KEM; see CD-7 fallback note |
| KEM, notifications only | DHKEM(X25519, HKDF-SHA256) | Justified exception to TM-X-4 -- see CD-8 |
| Public-key encryption framework | HPKE, base mode | RFC 9180 |
| AEAD | ChaCha20-Poly1305 | Constant-time everywhere, no AES-NI dependency |
| Hash | SHA-256 | Payload hash, fingerprints |
| KDF | HKDF-SHA-256 | Routing tags, recipient hashes, HPKE internals |
| Password hashing (C2S server-side) | Argon2id | Matches existing auth stack |

Every signed or encrypted structure carries a one-byte suite ID (`0x01` =
SDMP-V1). Verifiers MUST reject unknown suite IDs (TM-X-3): agility exists for
migration, not negotiation.

**Rationale.** Ed25519 and X25519 are already the house algorithms (DKIM
Ed25519 keys, NaCl box in msgstore), are small, fast, and misuse-resistant.
ML-KEM-768 hybrid satisfies TM-X-4 (harvest-now-decrypt-later) at the layer
where it matters -- long-lived content ciphertext. HPKE rather than raw NaCl
box because it standardizes ephemeral-key encryption, KDF labeling, and
multi-recipient export, and has multiple audited implementations.
ChaCha20-Poly1305 over AES-GCM: identical security margin, no timing
hazards on machines without AES hardware -- and small hosters run on
everything.

**Rejected.**
- *PQ signatures (ML-DSA) in v1*: signature forgery requires the quantum
  computer to exist at attack time; there is no retroactive harm, and ML-DSA
  keys/signatures are large enough to hurt the notification byte budget.
  Suite agility covers the future migration.
- *BLAKE3*: faster, but SHA-256 is universal, hardware-accelerated, and the
  performance delta is irrelevant at mail message sizes.
- *Raw NaCl box (current msgstore primitive)*: no PQ path, no standard
  multi-recipient construction, static-static DH complicates sender forward
  secrecy. msgstore's at-rest encryption is unaffected by this document; it
  may migrate later.

## CD-2: Key hierarchy

**Decision.** Four key types, two per principal class:

```
Domain identity key   Ed25519     signs: user key records, notifications,
                                  receipts, service bindings, migration notices
Domain encryption key X-Wing      receives: envelopes (and X25519 half
                                  receives notifications)
User identity key     Ed25519     signs: envelopes, retractions, rotation
                                  statements, device authorizations
User encryption key   X-Wing      receives: payload CEK wraps
```

Identity keys are long-lived (years). Encryption keys rotate on a short cycle
(CD-6). Routing tags derive from the *identity* key (CD-5), so routing
survives encryption-key rotation.

Device keys exist only for C2S authentication (passkeys / device tokens per
TM-DEVICE-1) and for enrollment-record signatures (TM-DEVICE-2). Devices do
NOT have per-device message keys in v1 -- see CD-9.

**Rationale.** Separating signing from encryption is table stakes (different
lifetimes, different compromise impact, enables PQ-hybrid on the KEM side
only). Deriving routing identity from the stable key keeps the addressing
layer rotation-proof.

**Rejected.** *Single combined key with Ed25519->X25519 conversion* (the DKIM
dual-use trick from encryption-design.md): acceptable for a domain's outbound
queue, wrong for a protocol where encryption keys must rotate frequently and
hybrid-PQ while identity stays stable.

## CD-3: Message encryption construction

**Decision.** Per-message content encryption key (CEK), HPKE-wrapped per
recipient:

1. Sender's client generates a fresh 256-bit CEK per message.
2. Payload is encrypted under the CEK with chunked AEAD (CD-4).
3. For each recipient, the CEK is wrapped via HPKE base mode (single-shot,
   ephemeral KEM share) to that recipient's current user encryption key, with
   `info = "sdmp/v1/cek-wrap" || suite_id`.
4. Sender authenticity comes from the Ed25519 envelope signature (CD-10),
   NOT from HPKE auth mode.

Each recipient receives their own envelope containing their own CEK wrap; the
encrypted payload blob is shared (single blob, one message ID, CDN-friendly --
the multi-recipient efficiency the requirements doc asks for).

**Rationale.** HPKE base mode means every message uses a fresh ephemeral KEM
share: compromise of the *sender's* long-term keys never exposes past content
(sender-side forward secrecy is total). Recipient-side exposure is bounded by
encryption-key rotation (CD-6). Signature-based authenticity is verifiable by
any holder of the envelope -- needed for mailing-list countersigning later --
at the cost of deniability, which email semantics already gave up (DKIM).

**Rejected.**
- *HPKE auth mode (static-static)*: gives sender authentication but ties
  content secrecy to the sender's static key (worse forward secrecy) and
  authenticates to the recipient only (breaks list countersigning).
- *Signal-style double ratchet*: superb forward secrecy, but fights the
  async/store-and-forward model -- messages fetched years later, multiple
  devices, server-held ciphertext. Wrong tool for mail; bounded-lifetime keys
  (CD-6) capture most of the value at a fraction of the complexity.

## CD-4: Payload format -- chunked AEAD

**Decision.** The encrypted payload is a chunked AEAD stream (STREAM
construction, as used by age):

- Chunk size 64 KiB; each chunk sealed with ChaCha20-Poly1305 under the CEK.
- Nonce = 11-byte big-endian counter || 1-byte final-chunk flag.
- The envelope's `payload_hash` is SHA-256 over the complete *encrypted*
  stream: integrity verification without decryption (servers), exactly as the
  requirements doc specifies.
- Recipients verify both the outer hash (TM-FETCH-3: after reassembly, before
  trusting content) and every AEAD tag during decryption; AEAD failure
  anywhere = message invalid, no partial plaintext released.

**Rationale.** Mail payloads include arbitrarily large files by design (file
transfer is "a message with a non-message/822 MIME type"); single-shot AEAD
would require buffering entire payloads in memory on both ends and forbids
ranged fetch. STREAM prevents chunk reordering/truncation attacks that naive
chunking invites.

**Rejected.** *Single-shot AEAD over the whole payload*: fine at 10 KB,
unusable at 10 GB.

## CD-5: Routing tags and recipient hashes (TM-ROUTE-1)

**Decision.** Two identifiers, two jobs:

**Routing tag** (in the notification, visible to the recipient's domain after
notification decryption):

```
routing_tag = HKDF-SHA-256(
    ikm  = recipient user identity public key,
    salt = SHA-256(recipient_domain),
    info = "sdmp/v1/routing-tag"
)[0:16]
```

Stable per (user, domain). Computable by any sender from the recipient's
published key record. Registered with the recipient's server at enrollment;
notifications bearing unregistered tags are dropped without state
(TM-NOTIF-4). Derived from the identity key, so it survives encryption-key
rotation; it changes on domain migration (salt) and on identity-key rotation
(rare, ceremonial).

**Pairwise recipient hash** (in the envelope, visible only to the destination
domain, carried for v2 blind-routing forward compatibility):

- Phase 1 (first contact):
  `HKDF-SHA-256(ikm = sender_identity_pub || recipient_identity_pub, salt = SHA-256(recipient_domain), info = "sdmp/v1/recipient-hash-p1")[0:16]`
- Phase 2 (negotiated): same construction with `ikm = shared_random` agreed
  in-band during first exchange, `info = ".../recipient-hash-p2"`.
- Phase 3 (published single-use tokens): deferred to v2 with blind routing.

**Rationale.** This resolves the first-contact paradox identified in the
threat model (section 10): the server can route first-contact mail because the
tag is sender-computable yet server-registered, and the DoS surface stays
closed because unknown tags create no state. The pairwise hash continues to
ride in the envelope so the v2 blind mode needs no envelope format change.
Privacy claims are the honest ones of TM-ROUTE-2: v1 hides recipient identity
from network observers (who see only ciphertext), not from the recipient's own
domain.

**Rejected.**
- *Pairwise hash as the routing identifier* (the original sketch): unroutable
  at first contact; forces an unbounded unknown-hash bucket that reopens
  T-NOTIF-2 and leaks pending metadata across the domain's users.
- *Plaintext localpart in the notification*: the tag costs nothing and keeps
  the v2 upgrade path and log hygiene (TM-X-6).

## CD-6: Key rotation and forward secrecy (TM-KEY-3/4, TM-DEVICE-3)

**Decision.**

- User encryption keys rotate on a 90-day default cycle with overlapping
  validity: new key published at day 0, old key accepted for *sending* until
  day 14 (grace for senders with cached records), retained for *decryption*
  per user retention policy.
- Senders MUST encrypt to the newest valid encryption key in the recipient's
  record; encrypting to a key inside its grace window is valid; to an expired
  key, invalid (uniform fetch-side rejection... the message simply will not
  decrypt; clients warn at compose time).
- Forward secrecy bound: destroying an old encryption private key makes all
  ciphertext encrypted to it permanently unreadable. The client offers this as
  the retention policy ("messages older than N are unreadable even under
  future compromise") -- the user chooses the FS/archive tradeoff explicitly.
- Key records (TM-KEY-3) carry: address, suite ID, both public keys, validity
  window, created-at, revoked flag, escrow flag, **per-user monotonic sequence
  number**, domain signature over the exact record bytes (TM-X-1).
- Rotation statements (TM-KEY-4): the *user's previous identity key* signs the
  new record (`info` label `sdmp/v1/key-rotation`). Records lacking the user
  countersignature are valid but wire-distinguishable as domain-initiated
  resets; clients display them as such, alarm-grade for established
  correspondents.
- Identity-key rotation is rare and ceremonial: requires user
  countersignature or, for genuine key loss, a domain reset that correspondent
  clients surface loudly (this is the unavoidable trust-your-domain case;
  direct out-of-band verification is the escape hatch, per the requirements
  doc).
- Device revocation (TM-DEVICE-3): revoking a device SHOULD trigger an
  immediate user-encryption-key rotation by default; a revoked device that
  kept the old private key can read only ciphertext it already obtained, never
  new mail.

**Rationale.** Rotation-as-FS gives the protocol a tunable forward-secrecy
window with zero ratchet machinery, composes with HPKE's per-message
ephemerals (CD-3), and reuses the rotation infrastructure the requirements doc
already made first-class. Monotonic sequence numbers plus user
countersignatures convert silent key substitution (T-KEY-1/3) into loud,
wire-visible events.

**Rejected.** *No rotation (PGP-style eternal keys)*: maximizes
harvest-now-decrypt-later exposure and makes device revocation meaningless.

## CD-7: X-Wing contingency

**Decision.** If X-Wing has not reached RFC (or interop-stable final draft) by
implementation time, SDMP-V1 uses the same construction pinned to
draft-connolly-cfrg-xwing-06 semantics under our own suite label -- the suite
ID already isolates us from upstream drift, and the construction (X25519 and
ML-KEM-768 combined via SHA3-256 with a fixed label) is independently
analyzed. Either way the wire format is unchanged.

## CD-8: Notification encryption -- the X25519-only exception

**Decision.** Notifications are encrypted with HPKE using
DHKEM(X25519, HKDF-SHA256) to the X25519 component of the domain encryption
key -- *not* the hybrid KEM. This is a recorded exception to TM-X-4.

**Rationale.** ML-KEM-768 ciphertext alone is 1088 bytes; the hybrid KEM share
would consume the entire single-datagram budget (TM-NOTIF-5, <= 1200 bytes)
before any payload. The notification plaintext is short-lived routing
metadata: an adversary who decrypts a recorded notification decades from now
learns a routing tag, a message ID for a blob that expired years earlier, a
sender domain, and timestamps. Message *content* confidentiality never
depends on notification confidentiality. The metadata exposure is real but
bounded, time-eroded, and identical in kind to RR-3's accepted residual.

**Rejected.**
- *Hybrid KEM in notifications*: does not fit the datagram (T-NOTIF
  requirements take precedence; fragmentation is a worse security hole than
  classical-only metadata encryption).
- *ML-KEM-512*: still 768 bytes of ciphertext, weaker margin, and the
  metadata-vs-content analysis makes the complexity unjustified.

## CD-9: Multi-device model

**Decision.** One user encryption key, shared across the user's devices via
an enrollment ceremony; no per-device message keys in v1.

- Enrollment: a new device generates its C2S credential and displays a
  short-lived enrollment code/QR; an *existing* device authorizes it and
  transfers the current encryption private key (and retained old keys per
  policy) wrapped to the new device's ephemeral enrollment key. The
  authorizing device signs the enrollment record (TM-DEVICE-2); the server
  stores and serves the device list but cannot forge an authorization
  signature.
- First device: generated at account creation (or by the domain, escrow-style,
  for managed onboarding -- disclosed per the escrow rules).
- Recovery with all devices lost: domain reset (loud, per CD-6) or
  user-held offline backup of the identity key (recommended client feature:
  printable/exportable key backup at enrollment).

**Rationale.** Per-device published keys leak device count to every
correspondent, complicate the key record, and multiply CEK wraps; private-key
sync via authenticated device-to-device transfer is the model users already
understand from Signal/WhatsApp. The threat model's required guarantees
(visible device list, non-forgeable authorization, revocation-triggers-
rotation) are what actually defend users; per-device crypto adds cost without
adding those guarantees.

**Rejected.** *iMessage-style per-device fanout*: leaks device count, bloats
envelopes, and still requires a device-list trust mechanism -- all pain, no
extra guarantee given TM-DEVICE-2.

## CD-10: Signing rules (TM-X-1/2)

**Decision.**

- Sign-the-bytes, carry-the-bytes, everywhere: every signed structure is
  serialized once by its creator; the exact bytes travel (as a `bytes` field)
  with the signature beside them; verifiers verify the carried bytes and only
  then parse. Re-serialization before verification is prohibited. The SDMP
  `Envelope` proto's embedded `envelope_signature` field (signature inside the
  signed message) is superseded by this rule -- envelopes travel as
  `envelope_bytes` + `signature`, the pattern SCMP's `SubmitMessageRequest`
  already uses.
- Signature input framing: `sig = Ed25519(sk, label || 0x00 || bytes)` where
  `label` is the ASCII context string. v1 labels:

  | Label | Signer |
  |---|---|
  | `sdmp/v1/envelope` | user identity key |
  | `sdmp/v1/notification` | domain identity key |
  | `sdmp/v1/key-record` | domain identity key |
  | `sdmp/v1/key-rotation` | previous user identity key |
  | `sdmp/v1/receipt` | domain identity key |
  | `sdmp/v1/migration` | old domain identity key + user identity key |
  | `sdmp/v1/device-enrollment` | authorizing device key |
  | `sdmp/v1/bridge-provenance` | bridge's domain identity key |

- All HKDF `info` strings are likewise drawn from a registered label table;
  no two contexts share a label (TM-X-2).

## CD-11: Message IDs (TM-FETCH-1)

**Decision.** 32 bytes from a CSPRNG, generated by the submitting client,
validated and uniqueness-enforced by the sender's server at submission
(duplicate or malformed -> rejected). Fetch scope is (sender domain, message
ID), so global uniqueness follows from per-store uniqueness. Servers MAY also
offer reserve-then-upload (two-phase) for reference-message workflows; a
reserved ID is server-generated by the same rules.

**Rationale.** The requirements doc's "protocol-generated, not sender-chosen"
intent is unguessability, which the generation rule plus server-side
validation delivers; a malicious sender choosing weak IDs weakens only its own
blobs' capability secrecy. Client generation removes a round trip from the
common path.

## CD-12: What each party can decrypt (summary table)

| Artifact | Encrypted to | Readable by |
|---|---|---|
| Payload | CEK; CEK wrapped to recipient user key | Recipient user only (plus escrow domain if flagged) |
| Envelope | Destination domain encryption key (hybrid) | Destination domain |
| Notification | Destination domain X25519 key | Destination domain |
| Transit blob at rest | Already payload-encrypted | Nobody at the storing server |
| C2S traffic | TLS 1.3 | Endpoints |
| S2S fetch/gRPC | TLS 1.3 (+ mTLS where specified) | Endpoints |

A coercive adversary (A8) seizing any server obtains: ciphertext blobs,
envelope plaintext *for that domain's own traffic* (sender identities, sizes,
timing), routing-tag registrations, and pending tables -- and no message
content. This matches the requirements doc's "no plaintext to produce" claim,
now with its metadata caveats stated.

## Open questions (tracked, non-blocking)

1. **Encrypted client-side search indexes** -- deferred with the client
   design; nothing in this document precludes them.
2. **ML-DSA migration timing** -- revisit when notification byte budgets and
   ML-DSA sizes can be reconciled, or accept PQ signatures only on
   TCP-carried structures.
3. **Phase 2 hash negotiation details** -- the in-band shared-random exchange
   piggybacks on first reply; exact field layout belongs to the wire spec's
   envelope extension section (deferred to v2 along with blind routing).
4. **msgstore at-rest migration** -- the existing NaCl-box at-rest scheme
   (encryption-design.md) stays as-is for the legacy stack; convergence on
   HPKE is desirable but not part of this protocol's scope.

---

*Upstream: [protocol-threat-model.md](protocol-threat-model.md).
Downstream: [protocol-wire-spec.md](protocol-wire-spec.md) carries these
constructions on the wire; [protocol-v1-scope.md](protocol-v1-scope.md)
records what is deferred.*
