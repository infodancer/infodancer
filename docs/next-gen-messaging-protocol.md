# Next-Generation Federated Messaging Protocol
*Working requirements document — in progress*
*Last updated: 2026-02-19*

## Problem Statement

Email is the universal asynchronous communication layer of the internet, but its core protocols (SMTP/IMAP/POP3) are 40+ years old and have fundamental, unsolved problems:

- **Spam**: Delivery cost is externalized to recipients, making bulk abuse economically viable
- **Identity**: Sender identity is DNS-convention-based, not cryptographically proven
- **Encryption/Privacy**: Near-zero E2E adoption; metadata unencrypted even when content is
- **Deliverability**: Large provider cartel controls deliverability for 90%+ of users
- **Mailing lists**: DMARC broke traditional list forwarding
- **Retraction/correction**: No mechanism exists
- **Threading**: Convention only, inconsistently implemented
- **Attachments**: MIME hacks, size limits per-server convention
- **File transfer**: Demanded but poorly served by the protocol

The **adoption problem** is the real wall: any solution requiring simultaneous opt-in has failed historically (PGP since 1991).

## Approach

**New protocol, federated by design, with traditional protocol bridges.**

- Design without legacy constraints; provide SMTP/IMAP/POP3 bridges (lossy but functional)
- Target small hosters as primary deployment unit — the mammals under the dinosaurs' feet
- Federation decentralized by design, no central servers required
- Do not require beating traditional email for adoption; solve the problems, let adoption follow

## Prior Art

### IM2000 (D.J. Bernstein, ~2000)
https://cr.yp.to/im2000.html

Proposed inverting storage responsibility: sender's server stores messages; recipient's server holds references and fetches on demand.

**Key insight retained:** Externalizing delivery cost to recipients is what makes spam economically viable. Inverting this changes the economics fundamentally.

**Critical flaw addressed:** IM2000 requires indefinite sender availability. This protocol addresses it via shared ownership, explicit transit/store separation, and blind caching.

## Key Management

### Model

Domains act as CAs for their own users — signing user public keys, asserting "this key belongs to this user at this domain." Domains may generate and escrow keys for non-technical user onboarding, but SHOULD allow users to generate their own private keys and submit only the public key for signing. Mandatory escrow domains must declare this publicly.

### Key Discovery and Fingerprint Caching

- Keys published by domain, signed by domain signing key
- Fingerprint caching is **mandatory and automatic** for all correspondents — not opt-in
- Cached fingerprint checked on every communication; change triggers notification **before** sending
- User must acknowledge key changes; may re-verify out-of-band
- Cache timeout measured in years; resuming after timeout gives informational notice, not security alert
- Client monitors its **own** published fingerprint on a schedule independent of send/receive activity; alerts immediately if domain changed it without user action — the user's client is their canary

### Key Rotation

First-class protocol operation: domain publishes revocation of old signature, new signature on new key. Timestamped and auditable. Triggers change notifications at all correspondent caches.

### Escrow Transparency

Mandatory key escrow MUST be declared in domain's published metadata. Senders communicating with escrow-mandatory domain users receive a visible disclosure. With key escrow, the recipient's domain sees everything — unavoidable and disclosed.

### Published Key Record

Key fingerprint, algorithm, public key material, validity period, creation timestamp, domain signature, revocation status, escrow flag.

### Direct User-to-User Trust

Out-of-band fingerprint verification stored locally. First-class path, not a workaround. The protocol is designed so domain trust is never the only option.

## Storage Model

### Ownership

Message ownership is shared and explicit:

- **Sender owns their copy** of every message they send
- **Recipient owns their copy** after explicit acceptance
- Neither party is obligated to retain their copy for any period
- No party can demand another produce a message they have chosen to delete

### Transit vs. Store

Explicit protocol-level separation — visible in the protocol, not an implementation detail.

| | Messages-in-Transit | Messages-in-Store |
|---|---|---|
| Purpose | Reliability buffer | User's mailbox |
| Ownership | Infrastructural | User (sender or recipient) |
| Expiry | Well-defined, mandatory | User-controlled |

Transit is not a mailbox. Messages in transit expire on a defined schedule. No messages sit in queues indefinitely.

### Acceptance as Explicit Protocol Action

Recipient server receiving a notification ≠ acceptance. Acceptance is an explicit pull action. Until acceptance:
- Message exists in sender's transit store
- Sender may retract it
- Recipient has no copy

After acceptance:
- Recipient has their own copy
- Sender may delete their transit copy
- Retraction becomes a request (retraction notice delivered), not a guarantee

### Spam Economics

Sender bears storage cost for every message in transit until recipients accept or transit expires. Sending at scale means storing at scale. Messages never pulled still incur full transit storage cost. Economics are maximally punishing for bulk abuse.

## Message Format

### Payload

Servers treat message payloads as **opaque bytes**. The protocol does not define or require a specific message body format. MIME type declared in the envelope is sender-asserted and client-validated — not protocol-verified. Valid types include `message/822` for bridge compatibility with existing email, or any other format including future formats. No protocol changes required to introduce new payload types.

### Envelope

Protocol-defined. Servers operate only on envelopes. **Encrypted to the receiving domain** — intermediate servers see only the destination domain, nothing else.

**Required envelope fields:**

| Field | Description |
|---|---|
| Destination domain | Outer routing target — only field visible to intermediate servers |
| Sender | Cryptographically bound sender identity (visible to destination domain only) |
| Recipient | Hashed recipient address (see Metadata Privacy) |
| Message ID | Globally unique, cryptographically random, protocol-generated |
| Thread ID | Parent message ID if reply — protocol-level threading |
| Payload MIME type | Declared content type (hint only, client-validated) |
| Payload size | Byte count of encrypted payload |
| Payload hash | Hash of encrypted payload — integrity verification without decryption |
| Timestamp | Sender-attested |
| Envelope signature | Sender's signature over all envelope fields |
| Transit expiry | Per message ID — different messages may carry different expiry values |
| Flags | retraction, read-receipt-requested, escrow-disclosed |

### Integrity

Three independent mechanisms:
- **Payload hash**: integrity of content without decryption
- **Envelope signature**: sender authentication
- **Transport encryption**: confidentiality

## Routing and Caching

### Blind Intermediate Servers

Intermediate servers are **blind caches** — they store and forward opaque encrypted blobs. They see only the destination domain. They cannot read sender identity, recipient address, or content without the destination domain's decryption keys. Intermediate servers store only messages from or to known clients.

### Recipient Pull

A recipient retrieves a message by either:
1. **Via own domain**: recipient's server fetches from sender or cache and serves to client
2. **Direct fetch**: client fetches directly from sender's server using message ID as capability token

Recipient chooses per message or per configured policy.

### High-Latency / Intermittent Links

The blind cache model handles high-latency or high-cost links (satellite, interplanetary, air-gap-adjacent) without protocol modification. Gateways on each side batch and cache opaque blobs; no decryption keys required. Earth-to-Mars communication falls out naturally from the model.

### Per-Sender Pull Preferences

| Preference | Behavior |
|---|---|
| Pull immediately | Server fetches automatically on notification |
| Pull when requested | Notification recorded; user or client initiates fetch (default) |
| Ignore notifications | Notification discarded without recording |

Ignore notifications is an effective unsubscribe — the sender cannot distinguish between ignored, undelivered, and not-yet-pulled.

## Message Notification

### Mechanism

Notification packets are **encrypted to the receiving domain's public key**. A network observer sees only that an encrypted packet was sent to the receiving domain's server. Minimum necessary information principle applies; exact field set deferred to detailed protocol design.

Receiving server maintains a pending reference table: hash → pending message ID, sender domain, timestamps.

### No Rejection at Notification Time

The receiving server provides **no rejection response** to notifications. Silence is the only response. This eliminates address harvesting, infrastructure mapping, backscatter, and NDR spam.

### Retransmission

Sending server retransmits on exponential backoff until message is fetched or expires. Protocol-defined maximum interval. No retransmission after expiry.

### Responsibility Transfer

If an intermediate gateway accepts a message, it accepts notification responsibility. Original sender's obligation ends. Gateway failure results in silent expiry — no bounce, no NDR.

### Delivery Inference

Fetch = delivered. Expiry without fetch = inferred failure. No explicit failure notification generated.

## Spam and Reputation

### Pull-Time Reputation Check

**Sender reputation is evaluated at pull time, not send time.**

| | Traditional SMTP | This Protocol |
|---|---|---|
| Decision point | Delivery time | Pull time |
| Information available | Minimal — campaign just started | Full — campaign may be identified |
| False positive risk | High | Lower — reputation has time to stabilize |
| Spammer certainty | Knows immediately if delivered | Never knows if message will be pulled |

The window between send and pull is exactly the window reputation systems need to identify campaigns.

### Reputation Sources

Federated — no single authority. Recipient servers and clients choose which sources to consult and how to weight them.

### Already-Accepted Messages

Reputation affects pull-time decisions only. Messages already accepted are not subject to retroactive reputation decisions — recipient's copy is their property.

### Msgcoin: Voluntary Delivery Cost Accounting

A voluntary, protocol-native accounting mechanism. Not a real currency — a signal extraction and reputation instrument.

**Mechanism:**
1. Recipients declare a cost (in msgcoin) for receiving a message
2. Senders offer what they are willing to pay to have the message read
3. **Refund is automatic** — reading triggers automatic refund via client; no user action required
4. Recipients **veto** by: deleting unread (no refund, weak negative signal) or marking spam (explicit negative signal, potential penalty reversed to sender)

**Domain-level ledger:** Each domain maintains an append-only signed ledger (tamper-evident, no global consensus or proof-of-work required) tracking reputation balances for domains it interacts with.

**Economic properties:**
- Read + no veto → automatic refund → sender's effective cost approaches zero for wanted messages
- Delete unread → weak negative signal
- Mark spam → strong negative signal, potential penalty
- Wealthy senders cannot sustain broad spam — balance depletes, floor reached
- High-value messages paying above declared cost is the intended use case

**Floor behavior and time decay:**
Floor behavior is recipient-defined policy: hard block, increased cost threshold, reduced acceptance rate, or quarantine. Reputation balances **decay toward neutral over time** in the absence of new signals. History is preserved in the permanent ledger; effective reputation reflects recent behavior weighted by time.

## Protocol Architecture

### Core Principle

Traditional email conflates server-to-server and client-to-server communication. This protocol defines two explicit, separate, full-featured protocols.

### Server-to-Server Protocol

Handles all inter-domain communication:
- Message notification (encrypted UDP signals)
- Message fetch
- Domain key publication and discovery
- Reputation queries (federated, opt-in)
- Gateway-to-gateway bulk transfer
- Responsibility transfer
- Domain-to-domain authentication

Clients never speak this protocol.

### Client-to-Server Protocol

Handles all communication between user's client and their own domain server:
- User authentication
- Message composition and submission
- Message retrieval from store
- Folder and mailbox management
- Key management
- Push notification to client
- Policy configuration (pull preferences, reputation thresholds, retention policies, hash address management)

Client has exactly one server relationship — its own domain.

### Client Model

**New native clients** encrypt before handoff — server receives ciphertext only, never holds plaintext (except on escrow-mandatory domains). Full feature access.

**Legacy gateway clients** connect via IMAP/SMTP/POP3 gateway. Gateway encrypts immediately on receipt before writing to disk. Plaintext exists only briefly in memory. Gateways SHOULD encrypt immediately on receipt and MUST NOT persist plaintext.

Legacy gateways are provided for adoption, not as the intended end state.

## Domain-to-Domain Authentication

### Trust Chain

```
TLD (e.g. .invalid)
  └── signs example.invalid domain key        [optional]
        └── signs bob's user key
```

Maps the existing DNS delegation hierarchy into messaging trust without requiring a new root authority.

### Trust Levels

| Level | Mechanism | Strength |
|---|---|---|
| Full chain | TLD signs domain; domain signs user | Strong — verifiable without trusting intermediates |
| DNS-authenticated | Domain controls DNS records and publishes key material | Baseline — equivalent to DKIM/DMARC trust today |
| Direct | Out-of-band fingerprint verification | Strongest — independent of all infrastructure |

TLD signing is **optional**. Chain depth is a visible trust signal in clients, not a binary allow/block decision.

### Future Extension

Root zone signing (`root → TLD → domain → user`) requires ICANN coordination. Out of scope for initial design.

## Metadata Privacy

### Layered Encryption

Without key escrow, message delivery uses explicit encryption layers:

1. **Payload layer**: Sender encrypts content directly to recipient's public key
2. **Envelope layer**: Wrapped in delivery envelope, encrypted to receiving domain's key
3. **Submission**: Sender's server sees destination domain and encrypted blob only — never plaintext
4. **Receiving domain**: Decrypts envelope to find hashed recipient address — not actual identity

### Client-Side Encryption

New-style clients encrypt before handing off to their own server. The sender's domain is blind — it sees a destination domain and an encrypted blob, nothing more.

**Key escrow exception:** Escrow-mandatory domains necessarily access plaintext. Disclosed in domain metadata.

**Legacy gateway clients:** Gateway encrypts immediately on receipt; plaintext exists only briefly in memory.

### Hashed Recipient Addressing

The recipient field in the delivery envelope contains a **hash derived from user identity plus a per-sender identifier**. The receiving domain sees an opaque hash — no mapping to a user account. The domain is a blind store.

### Bootstrap Hash Cascade

**Phase 1 — First contact:**
```
recipient_hash = KDF(Alice_pubkey, Bob_pubkey, "recipient-address", recipient_domain)
```
Computable from published key material. No prior coordination required.

**Phase 2 — Automatic upgrade (negotiated in first exchange):**
```
recipient_hash = KDF(shared_random, "recipient-address", recipient_domain)
```
Clients negotiate random string in-band, switch automatically. No user action required.

**Phase 3 — Optional published tokens:**
Bob publishes single-use hash tokens in key material for maximum-privacy first contact. Optional protocol support.

All three phases look identical at the domain's hash table.

### Polling and Fetch

1. Sending domain sends notification encrypted to receiving domain: "message for [hash]"
2. Receiving domain stores: hash → pending message ID, sender domain
3. User polls their domain (or privacy proxy): "anything for [hash]?"
4. Domain returns matching message IDs
5. User fetches directly from sender domain or via privacy proxy

**Privacy proxy:** User connects to proxy which fetches on their behalf. Receiving domain sees a proxy connection, not the user. Combined with hashed addressing: opaque hash, unreadable content, unidentified connection. Genuinely blind.

### Residual Metadata

- **Traffic patterns**: Timing and frequency of connections. Out of scope — Tor-level anonymity is a different design space.
- **Sender's domain**: Sees plaintext at composition time for legacy clients only.
- **Proxy trust**: User trust decision.

## File Transfer

File transfer requires no special protocol support. A file is a message with a non-`message/822` MIME type. No size limits. No MIME encoding overhead.

### Reference Messages

Senders transmit a reference message containing message IDs of associated file transfers — providing human context, grouping, and sender intent. Recipient client displays transfer metadata without fetching content.

### Recipient Options

1. Fetch directly
2. Defer
3. Delegate to server — optimal for large files and mobile clients
4. Ignore — let transfer expire

### Separate Expiry Per Message

Expiry is per message ID. Reference messages and file messages carry independent values.

## Message IDs as Capabilities

Message IDs function as **capability tokens** for retrieving ciphertext. Knowing a message ID is sufficient to fetch the encrypted payload. Knowing the ID plus the decryption key is sufficient to read it. These properties are explicitly separable.

- **Retrieval delegation without content exposure**
- **CDN-compatible caching**
- **Multi-recipient efficiency**: single encrypted payload, per-recipient key wrapping distributed separately
- **Implicit access control**: message IDs distributed only through encrypted envelopes

### Message ID Requirements

Cryptographically random, globally unique, protocol-generated (not sender-chosen), minimum bit length TBD. Sequential or predictable IDs compromise the capability model.

## Encryption at Rest

Because servers treat payloads as opaque bytes and never require plaintext to function, **all messages are encrypted at rest by design** — not as a bolt-on feature.

**Operational consequences:**
- **Untrusted storage**: Blobs may be stored on untrusted third-party infrastructure
- **Backup and replication**: Trivial
- **Legal process**: Server operators have no plaintext to produce
- **Search**: Client-side only, over locally decrypted content

**Exception:** List servers transiently decrypt to distribute. Explicit, disclosed trust boundary.

## Mailing Lists

### Model A: Managed List (Push-Notify)

List server knows its subscribers. Manages distribution explicitly.

**Flow:**
1. Sender composes to list address, encrypts to list's public key, sends normally
2. List server decrypts transiently
3. Constructs per-recipient delivery envelope: original payload with per-recipient key wrapping, original sender's signature preserved, list countersignature added
4. Sends notifications to each subscriber's domain server
5. Subscribers pull per their per-sender preference

**Properties:**
- List cannot forge sender identity — original signature preserved and verifiable
- Joining is a trust decision — list server sees plaintext transiently; disclosed in subscription metadata
- Subscriber identities known to list operator, not to other subscribers or outside observers
- Moderation between decryption and distribution
- Non-subscriber messages silently ignored or held for moderation

**Archives:** List server retains historic private keys, not plaintext. Encrypted blobs are the archive. Key rotation creates natural access boundaries.

### Model B: Manifest List (Poll)

List server publishes a manifest of available messages. Subscribers poll; list never knows who its subscribers are.

**Flow:**
1. Sender composes to list address, encrypts to list's public key, sends normally
2. List server decrypts transiently, approves (if moderated), adds message ID to manifest
3. Subscriber servers poll list address for new manifest entries
4. New message IDs trigger fetches per subscriber pull preference

**Key rotation:**

Each manifest is encrypted to the current list key and contains content decryption keys for referenced messages. Rotation procedure:
1. List generates new key
2. New manifest encrypted with old key, contains new key inside
3. Subscribers following the chain receive new key automatically
4. Subscribers who lapsed receive no new key — natural access expiry

**Unsubscribe:** Stop polling. No server-side state change. No address confirmation leak.

**New subscriber onboarding:** List operator sends current key encrypted to subscriber's public key — a direct message.

**Revocation:** Rotate key at next manifest boundary.

### Pull Preferences Apply to Both Models

Both models use notification — the list has no ability to force delivery. Per-sender pull preferences apply identically.

### Ad-Hoc Lists

A user running local list software is simply a user sending to multiple recipients. The list owner puts their own reputation and domain reputation on the line for bulk sending. Correctly aligned incentives.

## Rate Limiting

### Protocol-Level Backoff

Notification rates governed by a protocol-defined backoff algorithm. Receiving servers MAY enforce rate limits by ignoring notifications at the firewall level. Exact parameters TBD.

Msgcoin balance depletion provides organic economic rate limiting independently of protocol-level enforcement.

## Federation Governance

### Position: Ungovernable by Design

This protocol deliberately provides no formal or informal governance mechanism. No blocklist authority, no defederation process, no appeals body, no coordinating committee.

This is a principled free speech position. Every federated system that has attempted to build governance mechanisms has recreated central authority in disguise.

### How Bad Actors Are Handled

**Economics do most of the work.** Bad domains deplete msgcoin balances and collapse reputation scores. Self-destructs economically without governance decisions.

**Individual decisions aggregate.** Recipients decide what to pull. Domain operators decide which domains to accept notifications from. Sovereign individual decisions produce de facto isolation without centralized power.

**TLD revocation as last resort.** TLDs can revoke domain registrations at the DNS layer for extreme cases. Not duplicated in the protocol.

### Collateral Damage and Domain Migration

Domain migration is a **first-class protocol operation**:
- Old domain publishes a signed migration notice to known correspondents
- Notice signed by both old domain and user's key — proves identity continuity
- Correspondents' clients update cached addresses and fingerprints automatically
- User's key pair is portable — same cryptographic identity continues at new domain

### What This Protocol Does Not Do

Maintain blocklists, define abusive content, provide appeals processes, coordinate defederation, make editorial decisions.

### Future Work: Federated Reputation Visibility

*TBD — not required for core protocol.*

Opt-in web-of-trust reputation layer:
- **Publication (opt-in)**: Domains publish sender reputation scores, potentially via DNS
- **Consumption (opt-in)**: Domains consume others' published reputation, weighted by trust in publisher
- **Trust derivation**: Derived from direct communication history — high volume + positive msgcoin signals = basis for trusting assessments
- **Automatic trust accumulation**: Configurable threshold of exchange history triggers automatic weighting
- **No central authority**: Graph of bilateral relationships, not a hierarchy

---

## Current Assessment

### Problem Coverage

| Problem | Coverage | Notes |
|---|---|---|
| Spam | ~85% | Cost inversion + pull-time reputation + msgcoin + storage economics |
| Identity | ~80% | TLD→domain→user chain, DNS baseline, direct fingerprint, self-monitoring |
| Encryption/Privacy | ~90% | E2E mandatory, encrypted at rest by design, client-side encryption, hashed addressing, layered envelope, privacy proxy |
| Deliverability cartel | ~90% | Federated, no gatekeeper, small hosters first-class, distributed reputation |
| Mailing lists | ~85% | Two models covering full range, encrypted archives, key rotation solved |
| Retraction/correction | ~85% | Pre-acceptance guaranteed, post-acceptance notice |
| Threading | ~90% | Thread ID in protocol envelope |
| Attachments/file transfer | ~95% | Fully natural, capability model, no size limits, CDN-compatible |
| Adoption | ~40% | Bridges specified, small-hoster targeting. Network effects are social. |
| Federation governance | ~75% | Economics + individual decisions + TLD last resort + ungovernable position |

**Overall: ~85% of identified problems have solid design basis.**

### What's Intentionally Deferred

- Specific cryptographic algorithm choices
- Wire format / binary encoding
- Exact notification packet minimum field set
- Msgcoin ledger implementation details
- Federated reputation visibility (web-of-trust, DNS publication)
- Encrypted client-side search indexes
- Thread tree management details
- Manifest list key rotation edge cases

### What's Probably Unsolvable by Protocol

- Adoption — network effects are social, not technical
- Lookalike domain registration — DNS/registrar problem
- Traffic analysis — Tor-level anonymity is a different design space
- Determined well-resourced actors — economics hurt but don't eliminate

### What's Left to Design

1. **Encryption model specifics** — multi-recipient key wrapping, forward secrecy, algorithm selection
2. **Wire protocol** — binary format, versioning, extension mechanisms
3. **Domain discovery** — ~~how does a sending server find the receiving server for a domain?~~ **Decided: DNS SRV (`_[abbrev]._tcp.domain`). No SRV → fall back to SMTP. See protocol-outlines.md.**
4. **Reference implementation design** — minimal correct implementation targeted at small hosters
