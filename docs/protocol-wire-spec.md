# Protocol Wire Specification (v1)

*Normative design document -- concrete design phase*
*Issue: https://github.com/infodancer/infodancer/issues/5*
*Last updated: 2026-06-11*

This document specifies the v1 wire protocol for SDMP (server-to-server) and
the v1 shape of SCMP (client-to-server). It satisfies the `TM-*` requirements
assigned to it by [protocol-threat-model.md](protocol-threat-model.md) and
carries the constructions of
[protocol-crypto-design.md](protocol-crypto-design.md) (`CD-*`). Scope
boundaries are in [protocol-v1-scope.md](protocol-v1-scope.md).

Wire-level decisions are numbered `WS-n`. RFC 2119 keywords are normative.

---

## 1. Transport map (WS-1)

**Decision.** v1 SDMP needs **no gRPC at all**. The S2S surface is exactly
two transports:

| Channel | Transport | Operations |
|---|---|---|
| Notification | UDP, single datagram | notify, retract-notify |
| Everything else S2S | HTTPS (TLS 1.3+) | blob fetch, envelope fetch, key fetch |

SCMP (C2S) remains gRPC over TLS (bidirectional streaming fits the client
workload; web clients go through their own backends).

**Rationale.** The draft protos sketched gRPC `FetchService` and `KeyService`
with mTLS, alongside an unauthenticated HTTP blob fetch. Examining what v1
actually requires: blob fetch is capability-authorized (TM-FETCH-1), envelope
and key fetches return data that is either encrypted to the requester's
counterparty or public and signed -- none of it needs transport-level client
authentication. The gRPC services earned their keep only for v2 features
(responsibility transfer, reputation queries), which are deferred. Removing
mTLS client-certificate plumbing from v1 removes its bootstrapping
circularity (verifying a client cert against a key record you fetch via the
same infrastructure), keeps every S2S read CDN-compatible and curl-debuggable,
and shrinks the v1 implementation by a service layer. When v2 adds
domain-authenticated RPCs, the crypto design's key-record infrastructure is
already in place to bind client certificates to domain identity keys.

Server TLS identity: WebPKI certificate for the SRV target name (TM-DISC-3 --
never "whatever host SRV said" without certificate verification). Domain key
records MAY carry SPKI pins for their endpoints; clients with a cached key
record SHOULD honor them.

## 2. Discovery (WS-2)

Per the standing decision: DNS SRV.

```
_mail._tcp.example.com.  IN SRV 10 5 8443  msgs.example.com.   ; HTTPS S2S
_mail._udp.example.com.  IN SRV 10 5 8444  msgs.example.com.   ; notifications
```

- `_mail._tcp` locates the HTTPS S2S endpoint. `_mail._udp` locates the
  notification listener. A domain MAY point both at one host; the ports are
  independent.
- No SRV records -> SMTP bridge fallback (MX lookup), per the standing
  decision.
- **Downgrade stickiness (TM-DISC-2):** a sending server that has
  successfully spoken SDMP to a domain MUST remember that fact for a
  configurable window (default 7 days, refreshed on success). Within the
  window, SRV resolution failure or disappearance MUST NOT silently fall back
  to SMTP: the sender retries SDMP across the transit window and surfaces the
  anomaly (log/metric/operator alert). Operators MAY configure fail-open
  after the window.
- **Domain key fingerprint in DNS (TM-KEY-6):**

```
_sdmpkey.example.com. IN TXT "v=SDMP1; fp=<base64url(SHA-256(domain identity key))>"
```

  Fetched key bundles MUST be cross-checked against this fingerprint when the
  record exists. DNSSEC presence is recorded in the exchange's trust level
  (TM-DISC-1).

## 3. Versioning (WS-3)

**Decision.** Three version surfaces, no negotiation:

- Notifications: leading magic + version byte (section 4). Unknown version ->
  drop at stage 1.
- HTTPS: path-versioned (`/sdmp/v1/...`). Standard ALPN (h2); no custom ALPN
  (it would break CDN fronting).
- Cryptographic suites: one-byte suite ID inside every signed/encrypted
  structure (CD-1); unknown suite -> reject, never negotiate down (TM-X-3).

A future v2 listener answers `/sdmp/v2/...` and version-2 datagrams on the
same ports. SRV records do not encode versions.

## 4. Notification datagram (WS-4)

### 4.1 Outer layout

Total datagram length MUST be in [96, 1200] bytes (TM-NOTIF-5). All
multi-byte integers big-endian.

```
offset  size  field
0       4     magic = "SDMP" (0x53 0x44 0x4D 0x50)
4       1     version = 0x01
5       1     suite  = 0x01 (SDMP-V1)
6       32    HPKE enc (X25519 ephemeral share; CD-8)
38      var   HPKE ciphertext (inner notification || 16-byte AEAD tag)
```

There is **no cleartext destination-domain field** (the draft proto's
`destination_domain` is removed): the destination IP already routes the
datagram, and the field only fed observers. A network observer (A1) sees
source, destination, size, and timing -- nothing else.

HPKE: base mode, DHKEM(X25519, HKDF-SHA256), HKDF-SHA256, ChaCha20-Poly1305,
`info = "sdmp/v1/notification-enc"`, recipient key = the X25519 component of
the destination domain's encryption key (CD-8). Receivers MUST retain the
previous domain encryption key through its grace window so rotation does not
drop notifications (CD-6).

### 4.2 Inner notification (protobuf, encrypted)

```
message NotificationV1 {
  bytes  routing_tag      = 1;  // 16 bytes (CD-5); REQUIRED
  bytes  message_id       = 2;  // 32 bytes (CD-11); REQUIRED
  string sender_domain    = 3;  // <= 255 bytes; REQUIRED
  int64  timestamp        = 4;  // Unix seconds, sender-attested
  int64  transit_expiry   = 5;  // Unix seconds; REQUIRED
  string mime_type_hint   = 6;  // <= 127 bytes; optional
  uint64 payload_size     = 7;
  bytes  retracts         = 8;  // 32 bytes; this notification RETRACTS that
                                // message ID (TM-RETRACT-1). When set, fields
                                // 6-7 are absent.
  bytes  signed_bytes     = 15; // serialized NotificationCoreV1 (fields above)
  bytes  signature        = 16; // Ed25519 by sender domain identity key,
                                // label "sdmp/v1/notification" (CD-10)
}
```

Per CD-10, the signature covers exact carried bytes: fields 1-8 are
serialized once into `signed_bytes`, signed, and the receiver verifies
`signed_bytes` then parses it. (Fields 1-8 shown above for readability; on
the wire only `signed_bytes` + `signature` appear.)

Per TM-ROUTE-3, the inner notification carries sender *domain* only -- never
sender user identity, which travels only in the envelope.

### 4.3 Receive pipeline (normative; TM-NOTIF-1/2/3/4)

Implementations MUST process inbound datagrams in this order and MUST be able
to drop at each stage without executing later stages:

1. **Sanity:** length in [96, 1200]; magic; version; suite. Else drop.
2. **Rate limits:** token buckets per source IP and global. Defaults in
   section 9. Exhausted -> drop.
3. **Trial decrypt:** one X25519 HPKE open. Failure -> drop. (The only
   asymmetric operation an unauthenticated packet can force.)
4. **Inner sanity:** parse; field bounds; `transit_expiry` in
   (now, now + max_expiry]; `timestamp` within +-72h skew window. Else drop.
5. **Routing:** `routing_tag` registered? No -> silent drop, zero state
   (TM-NOTIF-4).
6. **Quotas:** per-claimed-sender-domain pending cap; per-recipient pending
   cap; global pending cap. Over -> drop (evict-or-drop policy below).
7. **Dedupe:** existing pending entry for (routing_tag, message_id) ->
   refresh seen-time, do not duplicate.
8. **Insert:** pending entry {routing_tag, message_id, sender_domain,
   timestamp, expiry, hint, size, verified=false}. Entry expires at
   `transit_expiry` (clamped to operator max).

Signature verification is **lazy** (TM-NOTIF-2): verify immediately iff the
claimed domain's identity key is already cached; otherwise verify at
pull-decision time, where key fetches are batched and rate-limited. No
synchronous network I/O anywhere in stages 1-8.

`retracts` handling: after stage 5, a retraction notification MUST be
verified (cached key, else deferred-verify before acting), MUST match an
existing pending entry from the same sender domain, and the signature MUST
verify under the same domain key that the original entry's signature
verifies under. On success: remove the pending entry, record the retraction
fact (sender domain + timestamp, no content) per TM-RETRACT-2.

Eviction under global pressure (TM-NOTIF-3): unverified entries from
unknown-reputation domains first, then oldest-expiry-first. Occupancy is a
required metric.

### 4.4 Retransmission (TM-NOTIF-7)

Sender schedule per (message, recipient): initial send, then exponential
backoff -- base 60 s, factor 2, +-20% jitter, ceiling 6 h -- until the
envelope is fetched (section 6) or transit expiry passes. Hard stop at
expiry; no notification after expiry. Receivers MAY penalize sources that
materially exceed this schedule.

## 5. HTTPS S2S resources (WS-5)

All under `https://{srv_target}:{srv_port}/sdmp/v1/`. All responses uniform
per TM-X-5: the not-available response (404, fixed body, fixed shape) is
identical for never-existed, expired, retracted, deleted, and
policy-refused. TM-FETCH-1's optional distinct retraction signal is NOT used
on this surface (retraction reaches recipients via `retracts` notifications
instead, which composes with uniform 404 here).

| Resource | Method | Auth | Cacheable |
|---|---|---|---|
| `/message/{message_id}` | GET | capability (the ID) | yes -- immutable |
| `/envelope/{message_id}/{destination_domain}` | GET | none (encrypted to that domain) | no |
| `/key/domain` | GET | none (signed record) | yes, short TTL |
| `/key/user/{address}` | GET | none (signed record) | yes, short TTL |

- `{message_id}` is base64url, 43 chars (32 bytes). Malformed -> uniform 404.
- **Blob** (`/message/{id}`): `application/octet-stream`, the chunked-AEAD
  encrypted payload (CD-4). Immutable; `Cache-Control: public, max-age=
  <until expiry>`. Range requests supported (TM-FETCH-3: client verifies the
  full payload hash after reassembly). Per-message fetch accounting, anomaly
  alerting, optional fetch caps, and instant takedown are required
  (TM-FETCH-2, TM-SUBMIT-4).
- **Envelope**: one envelope per (message, destination domain), containing
  the per-recipient CEK wraps for that domain's recipients (CD-3) inside the
  domain-encrypted envelope body. `Cache-Control: no-store`. Fetching it is
  the **delivered** event for that destination (section 6).
- **Keys**: the signed key records of CD-6/TM-KEY-3. Existence-uniform
  (TM-KEY-5): unknown user -> a response indistinguishable in shape and
  timing class from a policy refusal; rate-limited per source. `max-age`
  default 300 s; clients honor `Cache-Control` but MUST respect validity
  windows and sequence numbers over HTTP caching.

Envelope wire form (per CD-10 sign-the-bytes):

```
message EnvelopeV1 {
  bytes envelope_bytes = 1;  // serialized EnvelopeCoreV1, exact signed bytes
  bytes signature      = 2;  // Ed25519, user identity key, "sdmp/v1/envelope"
}
```

`EnvelopeCoreV1` carries the requirements doc's field set with these v1
adjustments: `recipient_hash` is the pairwise hash (CD-5, envelope-only);
`retracts` is a dedicated field (TM-RETRACT-1), not a flags overload of
`thread_id`; a `bridge_provenance` submessage (section 8) is present iff the
BRIDGED flag is set; `destination_domain` remains (it is inside
domain-encrypted content here, not observer-visible). The whole EnvelopeV1 is
then encrypted to the destination domain's hybrid key (CD-1) for transport
and storage.

## 6. Message lifecycle (WS-6)

Sender-side state per (message, recipient destination domain):

```
SUBMITTED -> STORED -> NOTIFYING -> DELIVERED        (envelope fetched)
                          |    \-> RETRACTED          (sender retracts pre-delivery)
                          \------> EXPIRED            (transit expiry, no fetch)
```

- **DELIVERED** = completed GET of `/envelope/{id}/{destination_domain}`.
  The envelope fetch names the destination domain; a blob fetch alone is NOT
  a delivery signal (it may be CDN warming or replay). This implements
  TM-ACCEPT-1's fetch-completion ownership transfer at domain granularity,
  and preserves the requirements doc's inference rule: envelope fetch =
  delivered; expiry without fetch = inferred failure, no bounce.
- Retraction is guaranteed-effective only in NOTIFYING; after DELIVERED it is
  a retraction-request message (TM-RETRACT-3).
- The transit blob is deletable when every recipient domain is DELIVERED,
  RETRACTED, or EXPIRED. (Operators MAY keep it until expiry for
  late re-fetch; recipients own their copies after fetch either way.)

Recipient-side state per pending entry:

```
PENDING -> FETCHING -> ACCEPTED      (envelope + blob fetched, hashes verify)
   |   \-> RETRACTED                 (retracts notification)
   \-----> EXPIRED                   (entry expiry, never pulled)
```

The pull decision out of PENDING follows the normative default policy
(TM-ACCEPT-2): auto-pull for established-positive senders, deferred
notify-user for unknown senders, per-sender user overrides on top. Fetches
out of PENDING are rate-limited per claimed sender domain with negative-
result backoff (TM-NOTIF-6): repeated uniform-404s from a domain push that
domain's pending entries toward eviction and throttle further fetches.

ACCEPTED is durable per-user server state exposed to clients through the C2S
event log (TM-ACCEPT-3, section 7).

## 7. SCMP (C2S) v1 surface (WS-7)

gRPC over TLS 1.3, services versioned as `scmp.v1`. This section fixes the
service inventory and the contested design choices; full message-level proto
work happens in the scmp repo against this document.

- **AuthService**: passkey/WebAuthn registration and assertion, password+TOTP
  fallback, short-lived session tokens, long-lived device tokens, per-device
  revocation (TM-DEVICE-1). Device enrollment per CD-9: enrollment records
  signed by the authorizing device, device list readable by all of the
  user's devices (TM-DEVICE-2).
- **MessageService**: one-shot submit (envelope_bytes + signature + payload
  stream) and reserve-then-upload (CD-11); list-pending; fetch; accept;
  delete; folder operations. Submission validation per TM-SUBMIT-2 (server
  verifies signature against the submitting user's current key, size/hash
  consistency, quota) with **structured quota errors** (quota kind, limit,
  usage -- TM-SUBMIT-1). The submit response includes the delivery class per
  recipient: `sdmp`, `bridge-smtp` (TM-BRIDGE-3 -- clients MUST surface
  downgrades at composition), or `rejected`.
- **SyncService** (multi-device backbone): per-user append-only event log --
  message-accepted, flags-changed, folder-changed, message-deleted,
  policy-changed, key-rotated, device-enrolled/revoked -- with monotonic
  sequence numbers; clients resume from their last-seen sequence
  (protocol-outlines C2S Problem 4 option C / Problem 8 option B). Server
  push of new events over a server-streaming RPC replaces polling.
- **KeyService**: the full operation list from protocol-outlines Problem 9
  (submit-for-signing, retrieve-signed, revoke, rotate with old-key
  countersignature per CD-6, inbound key discovery proxying the S2S key
  resources, canary checks). Canary checks MUST support the
  vantage-diversity requirement (TM-KEY-2): the client can request its own
  record via an arbitrary third-party domain's key resource and compare.
- **PolicyService**: per-sender pull preferences (the very common case gets
  a dedicated API -- Problem 10 option C) plus a validated policy document
  for the long tail (retention, quotas display, floor behavior).

Routing-tag registration (CD-5) is server-side automatic at enrollment, not a
client operation.

## 8. Bridge marking (WS-8; TM-BRIDGE-1/2/3)

Inbound SMTP bridge: the bridge synthesizes the envelope, sets flag
`ENVELOPE_FLAG_BRIDGED`, and attaches:

```
message BridgeProvenanceV1 {
  bytes  provenance_bytes = 1;  // serialized BridgeProvenanceCoreV1:
                                //   original RFC 5322 From,
                                //   SPF/DKIM/DMARC results,
                                //   receiving bridge host, received time
  bytes  signature        = 2;  // bridge domain identity key,
                                //   "sdmp/v1/bridge-provenance"
}
```

A BRIDGED envelope is signed by the *bridge's* service identity, never a user
identity key; clients MUST render bridged provenance distinctly, and a
non-BRIDGED envelope claiming a local user's identity but arriving via the
bridge MUST be rejected at submission (TM-BRIDGE-1). Bridges encrypt
immediately on receipt; no plaintext spool (TM-BRIDGE-2; testable: the
bridge's queue directory contains only ciphertext).

Outbound bridging (protocol -> SMTP) follows the existing
outbound-transport-routing and queue designs; the composition-time downgrade
disclosure is in section 7's submit response.

## 9. Limits and defaults (WS-9)

Operator-tunable; these are the protocol-recommended defaults
(TM-SUBMIT-1, TM-NOTIF-3, TM-FETCH-2):

| Parameter | Default |
|---|---|
| Max notification datagram | 1200 bytes (hard, normative) |
| Notification rate, per source IP | 50/s sustained, 200 burst |
| Notification rate, global | sized to deployment; required metric |
| Pending table, global cap | 100,000 entries |
| Pending table, per claimed sender domain | 1,000 entries |
| Pending table, per recipient | 1,000 entries |
| Timestamp skew window | +-72 h |
| Transit expiry: default / operator max | 7 days / 30 days |
| Retransmission | 60 s base, x2, +-20% jitter, 6 h cap |
| Max payload size (submission default) | 1 GiB |
| Per-user transit quota | 5 GiB, 10,000 messages |
| Fetches per claimed sender domain | 10/s, backoff on uniform-404 |
| Negative-result backoff | 15 min, x2 per repeat, 24 h cap |
| Key record HTTP max-age | 300 s |
| SDMP-support stickiness window (TM-DISC-2) | 7 days |

## 10. Draft-proto reconciliation (WS-10)

Changes required in the sdmp/scmp repos to match this spec (follow-up
implementation issues, one per repo):

**sdmp:**
1. `Envelope`: embedded `envelope_signature` replaced by the
   `envelope_bytes` + `signature` carrier (CD-10); `retracts` becomes a
   dedicated field; `recipient_hash` semantics pinned to CD-5 pairwise hash;
   add `bridge_provenance`.
2. `Notification`/`EncryptedNotification`: replaced by the section 4 layout --
   cleartext `destination_domain` removed, `routing_tag` added, `retracts`
   added, sign-the-bytes carrier, fixed outer binary framing (magic/version/
   suite/enc) instead of an outer proto.
3. `FetchService`, `KeyService` (gRPC): removed from v1; replaced by the
   HTTPS resources of section 5.
4. `TransferService`: deferred to v2 (see scope doc); proto parked.
5. Notification flag enum: dropped (retraction is the `retracts` field;
   reference/escrow flags live in the envelope).

**scmp:**
1. Add AuthService passkey flows, device tokens, enrollment records (CD-9).
2. Add SyncService event log; recast folder/flag state changes as events.
3. `SubmitMessageRequest`: already sign-the-bytes-shaped; add payload
   streaming/reserve flow, structured quota errors, per-recipient delivery
   class in the response.
4. KeyService: add rotation countersignature and canary-with-vantage ops.

## 11. Open questions (tracked, non-blocking)

1. **QUIC**: HTTP/3 for the fetch surface is attractive (it is still
   path-versioned HTTPS); revisit once reference implementation exists.
   Nothing here precludes it.
2. **Notification source cookies**: if real deployments show rate limiting
   alone is insufficient against spoofed-source floods (stage 2-3 pressure),
   a stateless cookie echo (DTLS-style) can be added in a v1.1 datagram
   version without breaking the version byte. Deliberately not in v1:
   it adds a round trip and the threat is speculative at v1 scale.
3. **Multi-recipient envelope batching**: one envelope per destination domain
   carries all that domain's recipient wraps; whether very large recipient
   counts need pagination is deferred until list support (v2) forces it.

---

*Upstream: [protocol-threat-model.md](protocol-threat-model.md),
[protocol-crypto-design.md](protocol-crypto-design.md). Scope:
[protocol-v1-scope.md](protocol-v1-scope.md).*
