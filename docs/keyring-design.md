# Client Keyring and Key-Encryption-Key (KEK) Design

*Design proposal -- NOT decided. Captures the direction agreed in discussion;*
*open decisions are called out inline. Last updated: 2026-06-13.*

## Goal

Let a user generate their keypair(s) on the client and hand the server only an
*encrypted keyring* -- a blob the server stores but cannot read. This decouples
the user's long-term message-encryption key from the password used to log in,
and gives a clean, auditable boundary between three trust postures:

- **off** -- no keypair; mail stored as plaintext.
- **on** -- the server holds an encrypted keyring it *cannot* decrypt. Backup
  and multi-device sync, no recovery. A forgotten unlock secret loses the mail.
- **escrow** -- the server (domain) *can* decrypt, via a domain recovery
  wrap-slot. Recoverable, less private, and *disclosed* per the protocol's
  escrow-transparency rule.

The unifying mechanism is a **key-encryption-key (KEK)**: a random symmetric key
that seals the keyring, itself wrapped to one or more *unlock paths*. The set of
wrap-slots present on the KEK is exactly what distinguishes the three modes.

## Why this fits the protocol (and the constraints it must honor)

This is consonant with commitments already made in
[next-gen-messaging-protocol.md](./next-gen-messaging-protocol.md) and
[protocol-outlines.md](./protocol-outlines.md). It is not a new requirement --
it is the existing one made precise.

1. **Client-side keygen is already the design.** `scmp.KeyService.SubmitPublicKey`
   and the protocol's Key Management section both have the user generate the
   keypair and submit *only the public key* for domain signing. Every key
   surface in `scmp`/`sdmp` (`keys.proto` in both) is public-key-only:
   submit / sign / rotate / revoke / lookup / monitor. **No private key crosses
   to the server in the non-escrow path.** The keyring is private; the published
   records are the signed public artifacts. These two layers stay orthogonal.

2. **The "encrypted blob the server can't read" pattern is already first-class.**
   Filter Rule Storage (C2S Problem 12) defines it: "one canonical, encrypted
   document; the server never reads it -- it stores an opaque blob like any
   other user data," with Option C generalizing to "a small namespace of
   client-encrypted, versioned configuration documents." **The keyring is one
   more entry in that namespace.**

3. **Escrow is the named, disclosed exception.** The `escrowed` /
   `escrow_mandatory` flags exist in both `keys.proto` files. "With key escrow,
   the recipient's domain sees everything -- unavoidable and disclosed."

4. **KEK/DEK wrapping is already idiomatic.** The protocol specifies the same
   primitive at the message layer: "single encrypted payload, per-recipient key
   wrapping distributed separately" (capability model). Wrapping one key to N
   recipients at the keyring layer is the same construction, not a novel one.

5. **It answers two open C2S problems.** Multi-device sync (Problem 8) and the
   device-onboarding pain noted in Problem 2 ("getting the cert onto new devices
   requires a separate ceremony"). The encrypted keyring is that ceremony's
   payload.

### The decision that governs everything: what unlocks the KEK

There are two different things called "keyring on the server," with **opposite**
trust postures. What separates them is solely *what can unwrap the KEK*:

| | Server can decrypt? | Protocol status |
|---|---|---|
| Encrypted keyring backup/sync | No | Not escrow; same posture as filter-rule storage. No disclosure. |
| Escrow / recovery | Yes | Escrow; mandatory published `escrowed` disclosure. |

So escrow stops being a vague property and becomes a concrete, auditable
question: **is a domain wrap-slot present on this keyring's KEK?** That maps
one-to-one onto the published `escrowed` flag and onto the maildancer
`encryption_mode` off/on/escrow setting.

## The keyring

The keyring is a **set**, not a single key, because rotation and archive access
require keeping historical private keys:

- `RotateKey` replaces the *published* key, but old mail was encrypted to the
  old key -- the old private key must remain available to read it.
- The list-archive model in the protocol explicitly "retains historic private
  keys, not plaintext."

A single sealed-key file cannot represent this; a keyring document can. Each
entry is a `(keypair, purpose, validity, fingerprint, status)` tuple.

### On-disk / on-wire structure (sketch -- not final)

```
Keyring (cleartext only inside the client / inside mail-session after unwrap):
  version
  entries[]:
    key_id            (fingerprint of the public key)
    algorithm         ("X25519" for box; "Ed25519" for signing)
    purpose           (encryption | signing)
    public_key
    private_key       (32 bytes)
    created, valid_from, valid_until
    status            (active | rotated | revoked)

Sealed keyring (what the server stores -- opaque to it):
  version
  kek_wrapped_blob    = secretbox/AEAD(keyring, KEK)   # one DEK-style seal
  wrap_slots[]:                                         # how the KEK is unlocked
    slot_type         (device | passphrase | escrow)
    slot_id           (device fingerprint, or "primary", or domain key id)
    wrapped_kek       = seal(KEK, slot_key)
  doc_version         (monotonic counter for compare-and-swap, per Problem 12A)
```

Server sees: `kek_wrapped_blob`, the *shape* of `wrap_slots` (how many, of what
type), and `doc_version`. It sees no key material it can use unless an `escrow`
slot is present and the server holds that slot's key.

**Open question:** how much wrap-slot metadata to expose. Slot *count and type*
leak a little (e.g. "this user has 3 devices and an escrow slot"). Fully opaque
is possible (encrypt the slot table too, with a fixed bootstrap) at the cost of
self-service device management. Lean toward visible slots for `on`, since the
escrow slot is *meant* to be disclosed anyway.

## Wrap slots: how the KEK is unlocked

Priority order, with the security rationale.

### 1. Device public key (primary, recommended)

Wrap the KEK under each enrolled device's public key. Enrolling a new device =
an existing device wraps the KEK to the new device's public key -- an explicit,
user-approved ceremony matching the protocol's fingerprint-acknowledgement ethos
and Problem 2 Option D (per-device model).

- The server never sees any unlock secret.
- **Per-device revocation is free:** drop a wrap-slot.
- Strictly stronger than anything a password-derived direct seal can express.

### 2. Passphrase slot (optional, for recovery -- with a sharp edge)

A passphrase wrap-slot covers "I lost my only device." But the unlock secret
**must be decoupled from the auth secret**, or backup silently degrades to
de-facto escrow:

- C2S auth (Problem 2) allows password, client-cert, and passkey. Today
  maildancer derives the key seal from the **login password** (argon2id in
  `auth/keyseal`).
- The server *sees the login password* at auth time (Problem 2 Option A,
  "password transmitted over TLS"). If the same secret authenticates *and*
  unwraps the keyring, a compromised server harvests it at login -- so "server
  can't read the keyring" becomes "can't read it except during login."

Mitigations, best first:

- **aPAKE (OPAQUE):** the server verifies the password without ever seeing it;
  the client retains the password locally to derive the keyring KEK wrap-key.
  Principled, but real work and not present anywhere in the stack today.
- **Domain-separated KDF (interim):** `auth_verifier = KDF(pw, "auth")`,
  `keyring_wrap_key = KDF(pw, "keyring")`; the server receives only the former.
  Weaker (the server still sees a value it could grind), but closes the
  same-secret footgun cheaply. Acceptable for a v1 password slot.

### 3. Escrow slot (per domain, disclosed)

An additional wrap of the KEK to a **domain recovery key**, present only when the
domain's mode is `escrow`. Its presence is what flips the published `escrowed`
flag. The domain recovery private key must itself be held safely -- admin-held
and password/HSM-sealed, **not** sitting beside the DKIM key.

## Mode -> wrap-slot mapping

| Domain mode | Wrap slots on the KEK | Server can decrypt? | `escrowed` flag |
|---|---|---|---|
| off | (no keyring) | n/a | n/a |
| on | device(s) and/or passphrase | no | false |
| escrow | device/passphrase **+ domain recovery** | yes | true |

This is the same `encryption_mode` already added to maildancer's `DomainConfig`
(off/on today; escrow reserved). The keyring model is what gives `escrow` a
concrete implementation.

## Relationship to the published key lifecycle

Keep these two planes orthogonal:

- **Public plane (already specified):** `scmp.KeyService` submit/sign/rotate/
  revoke/lookup/monitor; `sdmp.KeyService` domain/user key discovery. Signed,
  published, fingerprint-monitored.
- **Private plane (this doc):** the sealed keyring the server stores but cannot
  read (unless escrow). Rotation on the public plane appends a new active entry
  to the keyring and marks the prior one `rotated`; nothing is deleted, so old
  mail stays readable.

A keyring update and a key rotation are different operations that often co-occur:
rotate publishes the new public key *and* appends its private half to the keyring
(re-sealed, doc_version bumped).

## Maildancer migration (today -> this)

The `auth/keyseal` consolidation (maildancer#67) is the seam this sits behind.
Incremental path:

1. `auth/keyseal` becomes the **KEK layer**: it seals/opens a *keyring document*,
   not a bare 32-byte key. The current single-key `.key` file is the degenerate
   one-entry, one-passphrase-slot case -- a clean migration target.
2. The **fd-3 hand-off to mail-session is unchanged**: session-manager still
   passes the unwrapped private key(s) over the inherited descriptor for
   decrypt-on-retrieval. Only the at-rest representation changes.
3. **Legacy IMAP/POP (password auth)** keep a passphrase wrap-slot -- those
   clients have nothing but a password, and the weaker posture is acceptable
   because the legacy gateway is "for adoption, not the intended end state."
   Native SCMP clients get device-key wrapping.
4. The reserved `escrow` mode becomes "add a domain wrap-slot," surfacing as the
   protocol `escrowed` disclosure.

## Open decisions (to settle before implementation)

1. **Primary wrap mechanism:** device-public-key (recommended) vs passphrase.
   The protocol's own auth model pushes toward device-key as primary.
2. **Recovery story:** device-key-only (accept loss risk) vs add a passphrase
   recovery slot (and then aPAKE vs domain-separated KDF) vs escrow + disclosure.
   No free lunch; the keyring format should permit all three as slots so the
   choice is per-domain / per-user, not baked in.
3. **Wrap-slot metadata visibility:** visible slot table (enables self-service
   device management) vs fully opaque (hides device count). Probably visible for
   `on`, since the escrow slot is disclosed regardless.
4. **AEAD choice for the keyring seal:** NaCl secretbox (consistent with the rest
   of the stack) vs an AES-GCM / XChaCha20-Poly1305 with explicit associated
   data binding `doc_version` (prevents rollback of the sealed blob).
5. **OPAQUE adoption:** worth it, but scoped as its own effort; not a v1 blocker.

## Honest caveats

- Device-key wrapping needs a recovery answer or a lost sole device loses the
  keyring. The passphrase slot that fixes this reintroduces the password
  problem. The menu is (device-wrap + accept loss), (device-wrap + passphrase
  recovery via aPAKE), or (escrow + disclosure) -- chosen per domain.
- OPAQUE is real, unbuilt work. The domain-separated-KDF interim is the
  reasonable v1 for a password slot.
- None of this is blocked by current code; it is a format and protocol-design
  move sitting behind the `auth/keyseal` seam.

---

*See also: [encryption-design.md](./encryption-design.md) (at-rest model and the*
*off/on/escrow `encryption_mode`), [next-gen-messaging-protocol.md](./next-gen-messaging-protocol.md)*
*(Key Management), [protocol-outlines.md](./protocol-outlines.md) (C2S Problems*
*2, 8, 12).*
