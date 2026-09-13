# Protocol Threat Model

*Normative design document -- concrete design phase*
*Issue: https://github.com/infodancer/infodancer/issues/5*
*Last updated: 2026-06-11*

This document is the adversarial pass over every operation in the next-generation
messaging protocol. It exists so that the cryptographic design
([protocol-crypto-design.md](protocol-crypto-design.md)) and the wire
specification ([protocol-wire-spec.md](protocol-wire-spec.md)) are written
against named threats rather than vibes.

Requirements in this document are numbered `TM-<AREA>-<n>` and are **normative**:
the crypto design and wire spec MUST satisfy every requirement here or record an
explicit, justified exception. Requirements use RFC 2119 keywords.

Input documents:

- [next-gen-messaging-protocol.md](next-gen-messaging-protocol.md) -- requirements (what the protocol does)
- [protocol-outlines.md](protocol-outlines.md) -- enumerated design options
- sdmp/scmp v1 draft protos -- early sketches, treated as input, not authority

---

## 1. Assets

| ID | Asset | Properties to protect |
|---|---|---|
| AS-1 | Message content | Confidentiality, integrity, authenticity |
| AS-2 | Correspondence metadata (who talks to whom, when, how much) | Confidentiality, within stated limits |
| AS-3 | User identity keys (signing, encryption) | Confidentiality of private halves, binding to identity |
| AS-4 | Domain keys (signing, encryption) | Same |
| AS-5 | Service availability (notification intake, pending tables, transit store, fetch endpoints) | Availability under attack |
| AS-6 | Capability tokens (message IDs) | Unpredictability; possession = fetch right |
| AS-7 | Reputation state | Integrity, resistance to manipulation |
| AS-8 | Stored ciphertext archives | Long-term confidentiality (see A9) |

## 2. Adversary classes

| ID | Adversary | Capabilities |
|---|---|---|
| A1 | Passive network observer | Sees all packets on path (possibly globally); cannot modify |
| A2 | Active network attacker | On-path modify/inject/drop; off-path UDP source spoofing |
| A3 | Spammer / bulk abuser | Economically motivated; registers many cheap domains (sybil); runs compliant or non-compliant servers |
| A4 | Malicious correspondent | Valid protocol participant; valid keys; sends hostile traffic within the protocol |
| A5 | Malicious or compromised domain operator | Full control of a participating server -- the user's own domain, the counterparty's domain, or a third domain |
| A6 | Malicious intermediary | Blind cache, CDN, gateway, or proxy in the delivery path (mostly deferred to v2, modeled anyway) |
| A7 | Resource-exhaustion attacker | Not a participant; floods endpoints; spoofs sources; needs no keys |
| A8 | Coercive adversary | Seizes servers, compels operators, demands stored data |
| A9 | Future quantum adversary | Records ciphertext today, breaks classical KEMs later (harvest-now-decrypt-later) |

Out of scope (recorded in the requirements doc and re-affirmed here):
global traffic-flow analysis (Tor-level anonymity), lookalike domain
registration, endpoint (client device) compromise, and adversaries who can
break the chosen cryptographic primitives classically.

---

## 3. Operation: domain discovery (DNS SRV)

A sending server resolves `_mail._tcp.{domain}` to locate the receiving server.

**Threats**

- T-DISC-1 (A2): DNS response forgery redirects delivery to an attacker server.
  Consequence is bounded: the attacker receives notifications and envelopes
  encrypted to the *real* domain key, which it cannot decrypt -- unless key
  discovery is also subverted (see section 4). Availability loss is real.
- T-DISC-2 (A2): SRV record suppression downgrades delivery to the SMTP bridge,
  stripping the new protocol's properties (downgrade attack).
- T-DISC-3 (A5): a domain serves different SRV answers to different queriers to
  partition its traffic. Low impact; noted.

**Requirements**

- **TM-DISC-1**: Servers MUST validate DNSSEC where the zone is signed and
  SHOULD record (and expose in logs/metrics) whether discovery was
  DNSSEC-authenticated. Absence of DNSSEC MUST NOT block delivery (deployment
  reality), but the trust level recorded for the exchange MUST reflect it.
- **TM-DISC-2**: Downgrade to SMTP MUST be sticky-checked: once a sending
  server has successfully used the new protocol with a domain, it MUST treat
  subsequent SRV disappearance as suspicious -- retry discovery across the
  transit window and alert rather than silently falling back, for a
  configurable period (default 7 days). This is TOFU pinning of *protocol
  support*, parallel to MTA-STS's role for SMTP.
- **TM-DISC-3**: Transport security for every TCP/QUIC channel MUST NOT depend
  on discovery integrity (i.e., server identity is verified at the TLS layer
  against WebPKI and/or pinned domain-key bindings, never "whatever host SRV
  said").

## 4. Operation: key publication and discovery

Domains publish a domain key; domains sign user key records; senders fetch
recipient user keys before first contact.

**Threats**

- T-KEY-1 (A5, split view): the recipient's domain serves the true key record
  to the recipient's own canary checks and a substituted key to everyone else,
  enabling silent interception of inbound mail (with escrow-style plaintext
  access, undisclosed).
- T-KEY-2 (A2): MITM on key fetch substitutes keys at first contact (TOFU
  window).
- T-KEY-3 (A5): malicious domain rotates a user's key without consent
  (hostile takeover of a mailbox identity).
- T-KEY-4 (A1/A5): key lookups leak correspondence intent -- asking Bob's
  domain for Bob's key reveals that someone is about to write to Bob, before
  any message exists.
- T-KEY-5 (A3): key endpoint used for address harvesting (does this user
  exist?).

**Requirements**

- **TM-KEY-1**: Fingerprint caching at correspondents is mandatory (already in
  the requirements doc). Key *changes* MUST be surfaced to the sending user
  before send; key *additions under rotation with valid overlap* (see crypto
  design) MAY proceed without ceremony.
- **TM-KEY-2**: The self-canary MUST support vantage diversity: a client
  checking its own published key record MUST be able to perform the check via
  a path the domain cannot distinguish from a third-party query (another
  domain's key service, a proxy, or an anonymous fetch). A canary check that
  is identifiable as the user's own client is advisory only and MUST NOT be
  presented to the user as proof of consistency.
- **TM-KEY-3**: Key records MUST be individually signed by the domain key and
  MUST embed: the user address, validity window, creation time, revocation
  status, escrow flag, and a strictly monotonic per-user sequence number.
  Clients MUST treat a sequence-number regression, an unsigned change, or an
  escrow flag flip as an alarm-grade event.
- **TM-KEY-4**: Cross-signing: a user key record SHOULD carry the *user's own
  signature* over its replacement at rotation time (old user key signs new
  user key). A domain-initiated rotation without the user countersignature is
  distinguishable on the wire and MUST be displayed differently by clients
  ("domain reset this user's key" vs "user rotated their key"). This converts
  T-KEY-3 from silent to visible.
- **TM-KEY-5**: Key existence responses MUST be uniform: the key service
  response for a nonexistent user MUST be indistinguishable in timing and
  shape from a policy refusal, and SHOULD be rate-limited per source. (Same
  philosophy as no-rejection notifications; mitigates T-KEY-5.)
- **TM-KEY-6**: Domain key bootstrap MUST NOT depend solely on the HTTPS/WebPKI
  layer: a fingerprint of the domain signing key MUST be publishable in DNS
  (TXT) so that a fetched key bundle can be cross-checked against a second
  channel. DNSSEC strengthens this when present (see TM-DISC-1).
- T-KEY-4 is accepted as residual in v1 and MUST be documented honestly: key
  discovery reveals lookup interest to the recipient's domain. The v2 privacy
  proxy reduces it.

## 5. Operation: message submission (C2S)

Client encrypts payload, builds and signs the envelope, submits to its own
server for transit storage and notification.

**Threats**

- T-SUBMIT-1 (A4): transit store as free hosting -- a "sender" parks arbitrary
  blobs (warez, CSAM, exfil staging) in transit storage and hands out the
  capability URL; the operator is now hosting it. No size limits "by design"
  makes this worse.
- T-SUBMIT-2 (A4): submission floods -- a local user (or compromised account)
  fills the transit store.
- T-SUBMIT-3 (A4): malformed envelopes -- client-signed garbage that the
  server relays and that detonates in recipient parsers.
- T-SUBMIT-4 (A5): the sender's own server tampers with the envelope or
  payload before relay.

**Requirements**

- **TM-SUBMIT-1**: Per-user transit quotas (bytes, message count, and maximum
  transit expiry) are a NORMATIVE part of the C2S protocol surface, with
  first-class error signaling -- not an operator afterthought. Servers MUST
  enforce a maximum accepted payload size and a maximum expiry; both are
  operator-configurable with protocol-recommended defaults.
- **TM-SUBMIT-2**: The server MUST validate envelope well-formedness (field
  presence, signature verifies against the submitting user's current key,
  declared sizes/hashes match the uploaded payload) before accepting transit
  responsibility. The server cannot validate plaintext (it has none); it can
  and MUST validate everything it can see.
- **TM-SUBMIT-3**: Envelope and payload integrity MUST be end-to-end: the
  recipient verifies the *sender's* signature and the payload hash, so
  tampering by either domain (T-SUBMIT-4) is detected at the recipient. The
  sender's server MUST NOT be able to alter any signed field undetected.
- **TM-SUBMIT-4**: Capability URLs alone MUST NOT make the operator a public
  file host in practice: fetch endpoints MUST support operator policy hooks
  (per-source rate limits, fetch-count anomaly detection per message ID,
  takedown by message ID). See also TM-FETCH-4.

## 6. Operation: notification (S2S, UDP)

Sender's server emits a fire-and-forget encrypted UDP datagram to the
recipient's server; silence is the only response; retransmission on backoff
until fetch or expiry.

This is the largest unauthenticated attack surface in the protocol, and the
requirements here are the strictest.

**Threats**

- T-NOTIF-1 (A7): CPU exhaustion -- each datagram forces an asymmetric
  decryption attempt; attacker sprays garbage.
- T-NOTIF-2 (A7): state exhaustion -- each accepted notification creates a
  pending-table entry; attacker sprays well-formed-looking notifications with
  random routing identifiers.
- T-NOTIF-3 (A7 via A2 spoofing): reflective amplification -- if processing a
  notification triggers any outbound query (key fetch for signature
  verification, SRV lookup, message fetch), a spoofed-source datagram converts
  the receiver into an attack reflector against a victim, with amplification.
- T-NOTIF-4 (A3): notification spam -- high-volume real notifications from
  throwaway domains, hoping for auto-pull.
- T-NOTIF-5 (A2): notification suppression -- on-path drop of notifications
  silently delays mail until retransmission lands; total suppression = silent
  expiry, which the protocol treats as normal (no bounce). Availability risk
  inherent to the model.
- T-NOTIF-6 (A4): forged sender-domain claims inside otherwise valid
  notifications, causing the recipient to fetch from (and be rebuffed by) the
  claimed domain -- a second-order reflector aimed at the claimed domain's
  fetch endpoint.

**Requirements**

- **TM-NOTIF-1 (validation order)**: The wire spec MUST define a NORMATIVE
  processing order for inbound notifications, cheapest checks first, and an
  implementation MUST be able to drop a packet at each stage without
  performing any later stage:
  1. length and version sanity (single datagram, fixed bounds);
  2. per-source-IP and global token-bucket rate limits;
  3. trial decryption (the only unavoidable asymmetric operation);
  4. routing identifier known? (unknown -> drop silently, no state);
  5. per-claimed-sender-domain quota on pending entries;
  6. pending-table insert with bounded size and defined eviction.
- **TM-NOTIF-2 (no synchronous fan-out)**: Processing a notification MUST NOT
  trigger any synchronous outbound network operation. Signature verification
  against a domain whose key is not already cached MUST be deferred (entry
  marked unverified, verified lazily or at pull-decision time, when it can be
  batched and rate-limited). DNS/SRV/key lookups on the notification hot path
  are prohibited. This single rule removes T-NOTIF-3's amplification.
- **TM-NOTIF-3 (bounded state)**: Pending tables MUST have: a global cap, a
  per-claimed-sender-domain cap, and a per-recipient cap; entries MUST expire
  no later than the notification's declared transit expiry (clamped to an
  operator maximum); eviction under pressure MUST prefer unverified entries
  from low-reputation or unknown domains. Table occupancy MUST be observable
  (metrics) so operators see pressure.
- **TM-NOTIF-4 (drop unroutable)**: A notification whose routing identifier
  does not correspond to a registered recipient MUST be dropped silently and
  MUST NOT create state. (This requirement forces the addressing design to
  make routing identifiers registerable -- see section 10 and the crypto
  design's routing-tag construction.)
- **TM-NOTIF-5 (datagram discipline)**: A notification MUST fit in a single
  unfragmented datagram; the wire spec MUST set a byte budget compatible with
  conservative path MTU (target <= 1200 bytes). No fragmentation handling, no
  reassembly state.
- **TM-NOTIF-6 (fetch-side backpressure)**: Fetch attempts derived from
  notifications MUST be rate-limited per claimed sender domain with negative
  caching: repeated 404/410 from a domain rapidly backs off further fetches
  attributed to it (mitigates T-NOTIF-6).
- **TM-NOTIF-7 (retransmission discipline)**: Retransmission schedules are
  sender-side protocol constants (exponential backoff with jitter, defined
  base and ceiling, hard stop at expiry). A compliant sender MUST NOT exceed
  them; a receiver MAY penalize (rate-limit, deprioritize) sources that do.
- T-NOTIF-5 (suppression) is accepted as residual: the model already treats
  expiry-without-fetch as normal. Senders observe non-fetch and MAY alert
  users for high-importance messages. No protocol change.

## 7. Operation: message fetch (S2S / direct)

Recipient's server (or client, directly) fetches ciphertext by message ID from
the sender's transit store over HTTPS.

**Threats**

- T-FETCH-1 (A7): fetch flood against the transit store (bandwidth burn);
  capability tokens are static, so a leaked ID can be hammered or CDN-bypassed.
- T-FETCH-2 (A4/A6): capability leakage -- anyone holding the ID gets the
  ciphertext. Ciphertext is E2E-encrypted, so confidentiality holds, but
  acquisition of ciphertext enables offline attack forever (ties to A9) and
  reveals envelope-to-domain metadata to whoever the *envelope* is decryptable
  by (only the destination domain -- acceptable).
- T-FETCH-3 (A5): fetch timing tells the sender's server exactly when (and
  from where) the recipient pulled -- a built-in read-receipt to the sender's
  *server*, independent of the protocol's read-receipt flag.
- T-FETCH-4 (A4): transit-store-as-CDN abuse (see T-SUBMIT-1) exercised at the
  fetch side: one message ID fetched a million times.

**Requirements**

- **TM-FETCH-1**: Message IDs MUST be >= 256 bits from a CSPRNG (capability
  unguessability), and fetch URLs MUST NOT embed any other secret. Servers
  MUST respond identically (status, shape, timing class) for "never existed",
  "expired", and "not yours to know" -- a single uniform not-available
  response, with one exception: an explicit retraction MAY be signaled
  distinctly to authenticated recipient domains (see section 9).
- **TM-FETCH-2**: Transit stores MUST support per-message fetch accounting and
  operator alerting on anomalous fetch counts; MUST support immediate
  takedown (delete = subsequent uniform not-available); SHOULD support
  optional fetch-count or fetch-window caps set at submission time
  (mitigates T-FETCH-4 -- a normal message needs a handful of fetches, not
  a million).
- **TM-FETCH-3**: Range requests are supported for large payloads, but the
  *complete-payload hash* in the envelope remains the integrity root; clients
  MUST verify the full hash after reassembly before treating content as
  authentic.
- T-FETCH-3 is accepted as residual in v1 and MUST be stated honestly in user
  documentation: pull timing is visible to the sender's server. The v2
  privacy proxy and recipient-server batching/prefetch policies reduce it.
  Recipient servers SHOULD support randomized fetch delay as a cheap partial
  mitigation.

## 8. Operation: acceptance

Acceptance is the explicit recipient action that transfers ownership; before
it, the sender may retract; after it, the recipient owns their copy.

**Threats**

- T-ACCEPT-1 (A4): sender retracts immediately after the recipient's *client*
  has fetched but before acceptance is recorded -- ambiguity about who owns
  what (race).
- T-ACCEPT-2 (A5): recipient's server fabricates or suppresses acceptance
  state toward the client (multi-device divergence).
- T-ACCEPT-3 (A3): auto-pull policies convert acceptance into a rubber stamp,
  collapsing the economic model (the equilibrium problem).

**Requirements**

- **TM-ACCEPT-1**: The state machine in the wire spec MUST define the race:
  ownership transfers at successful fetch-completion *by or on behalf of the
  recipient*, not at the recipient's later "accept" UX action. Retraction
  after fetch is a request, per the requirements doc. "Acceptance" as a
  protocol event = completed authenticated fetch by the recipient domain or
  recipient client; servers MUST record it durably (it gates retraction
  semantics).
- **TM-ACCEPT-2**: Default pull policy is NORMATIVE, not advisory, because the
  spam economics depend on it: auto-pull for senders with established
  positive history; deferred (notify-user, pull-on-request) for unknown
  senders. Implementations MAY let users tighten or loosen, but the *shipped
  default* MUST be the two-tier policy. (Addresses T-ACCEPT-3 / the
  equilibrium problem head-on.)
- **TM-ACCEPT-3**: Acceptance state is per-user server state exposed over C2S
  with a sequence number / changelog so multiple devices converge (ties to
  multi-device sync; see wire spec C2S section).

## 9. Operation: retraction

**Threats**

- T-RETRACT-1 (A4): retraction abuse for gaslighting (send abusive content,
  retract before the victim can preserve it). Pre-acceptance retraction is
  guaranteed-effective by design, which *helps* the abuser here.
- T-RETRACT-2 (A4): forged retractions deleting someone else's pending mail.

**Requirements**

- **TM-RETRACT-1**: A retraction MUST identify its target by message ID in a
  dedicated field (NOT by overloading thread_id), and MUST be signed by the
  same sender key (or a valid successor under rotation) that signed the
  target. Anything else is invalid and ignored.
- **TM-RETRACT-2**: Pre-acceptance retraction removes the *pending entry* and
  the transit blob. Recipients' clients MUST be able to display that
  something was retracted (sender domain + timestamp, no content) -- the
  notification already existed; erasing the *fact* of it is not promised, and
  promising it would aid T-RETRACT-1. The requirements doc's privacy goals do
  not require pretending the notification never happened.
- **TM-RETRACT-3**: Post-acceptance retraction is delivered as a normal
  message flagged retraction-request; honoring it is entirely client policy.
  (Restates the requirements doc; no silent deletion of owned copies, ever.)

## 10. Addressing and routing privacy (hashed recipients)

The requirements doc specifies pairwise recipient hashes with a three-phase
cascade, and the receiving domain as "blind store." The threat model has to be
honest about what this achieves against which adversary.

**Analysis**

- The Phase 1 hash `KDF(sender_pub, recipient_pub, ...)` is computable by
  *anyone who holds both published keys*. Against a targeted "is Alice writing
  to Bob?" question, it is a confirmation oracle, not protection -- for the
  receiving domain and for any observer who obtains the hash. Its honest value
  is against *bulk, untargeted* metadata collection and casual logging.
- Routing requires somebody to map hash -> user. Server-side registration
  (the only operationally sane v1 option) means the recipient's domain can
  already enumerate its own users' keys and compute candidate hashes; the
  domain is NOT blind in v1 and the spec must not claim it is.
- **First-contact paradox**: a recipient cannot register (or poll for) a
  Phase 1 hash for a sender they do not yet know exists. Either the server
  accepts and stores *unroutable* hashes (colliding with TM-NOTIF-4 and
  re-opening T-NOTIF-2), or first contact needs a routing identifier the
  server can recognize that any sender can compute. The unregistered-hash
  bucket also gives every domain user visibility into pending unknown hashes,
  leaking cross-user metadata.

**Requirements**

- **TM-ROUTE-1**: v1 uses a **registered routing tag** as the notification
  routing identifier: a stable per-user value derived from the recipient's
  published key (exact construction in the crypto design), registered with the
  recipient's server at enrollment, computable by any sender from the
  recipient's published key record. Unknown tags are dropped (satisfies
  TM-NOTIF-4). The pairwise-hash cascade remains in the *envelope* (which only
  the destination domain decrypts) as the forward-compatible v2 path to blind
  routing + privacy proxy.
- **TM-ROUTE-2**: All published privacy claims MUST be scoped per adversary
  class. v1 normative claims: content and recipient identity are hidden from
  A1/A2/A6 entirely (everything they see is encrypted to the domain); the
  recipient's own domain (A5) sees sender domain, routing tag (= which of its
  users), timing, and sizes -- and, after envelope decryption, sender
  identity. The phrase "genuinely blind" MUST NOT appear in v1 documentation.
- **TM-ROUTE-3**: The notification payload visible *after* domain decryption
  MUST still exclude sender *user* identity (domain only); sender user
  identity travels only in the envelope. (Limits what a compromised
  notification pipeline learns; the envelope requires a separate fetch.)

## 11. Operation: C2S authentication and multi-device

**Threats**

- T-DEVICE-1 (A4/A7): credential stuffing/phishing on C2S auth.
- T-DEVICE-2 (A5): malicious domain enrolls a shadow device for a user
  (equivalent of adding a silent recipient).
- T-DEVICE-3: device loss -- revocation and key continuity.

**Requirements**

- **TM-DEVICE-1**: C2S auth MUST support phishing-resistant credentials
  (passkeys/WebAuthn) alongside password+TOTP; long-lived device tokens are
  distinct from short-lived session tokens; per-device revocation is
  first-class.
- **TM-DEVICE-2**: Device enrollment MUST require authorization by an existing
  enrolled device (or initial-account ceremony). The enrolled-device list
  MUST be visible to all of the user's devices, server-attested but
  client-verifiable: each enrollment record is signed by the *authorizing
  device's* key, so a domain-injected device shows as "authorized by: domain
  override" -- alarm-grade in clients (parallel to TM-KEY-4). Escrow-mandatory
  domains are the disclosed exception.
- **TM-DEVICE-3**: The crypto design MUST specify what a lost device can and
  cannot decrypt after revocation, and what a stolen *server* learns
  (it stores ciphertext; C2S auth secrets MUST be stored only in
  phishing-resistant / hashed forms).

## 12. Operation: SMTP bridges

Inbound: legacy SMTP -> bridge -> protocol delivery. Outbound: protocol ->
bridge -> SMTP. The adoption story leans on these, so they get a real threat
treatment, not a paragraph.

**Threats**

- T-BRIDGE-1: bridge holds plaintext (inbound SMTP has no E2E); it is the
  single most attractive compromise target in a deployment.
- T-BRIDGE-2: bridged messages have synthesized envelopes -- sender identity
  is asserted by the bridge, only as strong as SPF/DKIM/DMARC results it
  validated.
- T-BRIDGE-3 (A4): attacker routes through the bridge deliberately to acquire
  the weaker-provenance path while displaying like native mail.
- T-BRIDGE-4: outbound bridging silently strips E2E (protocol-native sender
  believes in protocol guarantees; SMTP recipient gets none).

**Requirements**

- **TM-BRIDGE-1**: Bridged messages are a distinct security class. The
  envelope MUST carry a bridge-origin marker plus a summary of the legacy
  authentication results (SPF/DKIM/DMARC outcomes), signed by the bridge.
  Clients MUST render bridged provenance differently from native provenance.
  A native-looking message MUST NOT be constructible via the bridge
  (the marker is signed into the envelope; TM-SUBMIT-3 applies).
- **TM-BRIDGE-2**: Bridges MUST encrypt immediately on receipt and MUST NOT
  persist plaintext (restates requirements doc; here it is normative and
  testable: no plaintext spool files, crash-safe via encrypt-then-queue).
- **TM-BRIDGE-3**: Outbound downgrade MUST be visible at composition time:
  the C2S submission response indicates bridge delivery, and clients MUST
  surface it ("this recipient gets legacy email -- no end-to-end
  encryption"). Silent downgrade is prohibited; users MAY configure
  fail-closed per recipient or per message.

## 13. Reputation (v1: local-only)

Msgcoin ledger synchronization is out of v1 scope (see
[protocol-v1-scope.md](protocol-v1-scope.md)); v1 reputation is each domain's
local scoring of counterparty domains.

**Threats**

- T-REP-1 (A3, sybil): cheap fresh domains arrive with neutral reputation;
  per-domain penalties are shed by re-registration. No per-domain reputation
  scheme fixes this alone; the storage-cost inversion and deferred-pull
  default (TM-ACCEPT-2) are the structural defenses for unknown senders.
- T-REP-2 (A4): reputation poisoning -- inducing a victim domain's users to
  generate negative signals against a competitor (e.g., spoofing... not
  possible: sender identity is signature-bound). Residual: social engineering
  only.
- T-REP-3: reputation state as a privacy leak if ever published (defer; v2
  web-of-trust work).

**Requirements**

- **TM-REP-1**: New/unknown domains MUST start at "unknown", not "good";
  unknown maps to the deferred-pull tier of TM-ACCEPT-2 and to the stricter
  pending-table quotas of TM-NOTIF-3. Reputation upgrades require positive
  history (accepted + non-vetoed messages over time).
- **TM-REP-2**: All reputation effects MUST key off cryptographically bound
  identities (domain signatures), never off claimed strings alone.

## 14. Cross-cutting requirements

- **TM-X-1 (sign-the-bytes)**: Every signature in the protocol is computed
  over an exact, transmitted byte string, which is carried verbatim alongside
  the signature. Re-serialization before verification is prohibited
  (protobuf encoding is not canonical). Signed structures travel as
  `bytes` + signature.
- **TM-X-2 (domain separation)**: Every signature and every KDF invocation
  uses a distinct context label (e.g. `sdmp/v1/envelope`,
  `sdmp/v1/notification`, `sdmp/v1/routing-tag`). Cross-context signature or
  hash reuse MUST fail verification structurally, not by luck.
- **TM-X-3 (algorithm agility with discipline)**: Every signed/encrypted
  structure carries an algorithm-suite identifier; v1 defines exactly one
  REQUIRED suite (see crypto design) and verifiers MUST reject unknown suites
  rather than negotiate downward. Agility is for *migration*, not for runtime
  negotiation games.
- **TM-X-4 (harvest-now-decrypt-later)**: A9 is in scope for payload
  confidentiality: the v1 payload encryption MUST be hybrid post-quantum
  (classical + ML-KEM). Signatures MAY remain classical in v1 (forgery
  requires the quantum computer to exist at attack time; no retroactive harm),
  with agility per TM-X-3 for later ML-DSA adoption.
- **TM-X-5 (uniform errors)**: Wherever the protocol's anti-abuse stance is
  "silence" or "uniform response" (notifications, fetch not-available, key
  existence), implementations MUST NOT add distinguishable debug/verbose
  responses in production builds.
- **TM-X-6 (observability without leakage)**: Metrics and logs MUST be
  domain-level aggregates consistent with the existing stack's
  privacy-respecting observability stance; routing tags, message IDs, and
  hashes are treated as sensitive in logs (truncated/salted).

## 15. Residual risk register (accepted, v1)

| ID | Residual risk | Why accepted | Revisit |
|---|---|---|---|
| RR-1 | Pull-timing visible to sender's server (T-FETCH-3) | Inherent to sender-stores/recipient-pulls | v2 privacy proxy |
| RR-2 | Key-lookup intent leak (T-KEY-4) | Inherent to recipient-domain key discovery | v2 privacy proxy |
| RR-3 | Recipient's own domain sees routing metadata (section 10) | Blind routing deferred; honesty required by TM-ROUTE-2 | v2 blind mode |
| RR-4 | Sybil domains reset reputation (T-REP-1) | No protocol fix exists; structural defenses via defaults | continuous |
| RR-5 | Notification suppression = silent delay/expiry (T-NOTIF-5) | Model treats expiry as normal; no bounce by design | -- |
| RR-6 | Traffic analysis of timing/sizes | Out of scope per requirements doc | -- |
| RR-7 | Retracted-notification *fact* remains visible (TM-RETRACT-2) | Anti-gaslighting beats cosmetic erasure | -- |

---

*Downstream documents: [protocol-crypto-design.md](protocol-crypto-design.md)
(satisfies TM-KEY-*, TM-ROUTE-1, TM-X-1..4, TM-DEVICE-3),
[protocol-wire-spec.md](protocol-wire-spec.md) (satisfies TM-DISC-*,
TM-NOTIF-*, TM-FETCH-*, TM-ACCEPT-*, TM-RETRACT-*, TM-BRIDGE-*),
[protocol-v1-scope.md](protocol-v1-scope.md) (records deferrals).*
