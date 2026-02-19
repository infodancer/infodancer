# Protocol Outlines: C2S and S2S
*Editorial discussion document — problems and options, not decisions*
*Last updated: 2026-02-19*

These outlines enumerate the design problems each protocol must solve and the leading options for each. This is not a specification — it is a set of open questions intended to drive editorial discussion before the specification phase begins.

The two protocol split is established in the main requirements document:
- **S2S (Server-to-Server)**: All inter-domain communication. Clients never speak this protocol.
- **C2S (Client-to-Server)**: All communication between a user's client and their own domain server. Client has exactly one server relationship — its own domain.

---

## S2S Protocol

### Problem 1: Domain Discovery

How does a sending server locate the receiving server for a domain?

This is the most fundamental problem for federation adoption and the most politically fraught. See the extended discussion at the end of this document: [IM2000 Prime-Number MX vs. DNS SRV vs. Alternatives](#domain-discovery-extended).

**Options:**

- **A. DNS SRV records** — `_msgs._tcp.example.com SRV 0 5 1234 server.example.com`. Well-specified (RFC 2782), already used by XMPP/SIP federation. Explicit service name, port, weight, target. Adoption friction: many DNS operators and provisioning tools handle SRV poorly for email-adjacent services. SMTP deliberately avoided SRV; the email world has a long memory.

- **B. Well-known HTTPS endpoint** — `https://example.com/.well-known/messaging` returns a JSON document with server address, port, and key material. Zero DNS changes required; works with any DNS that supports A records. Adds HTTP dependency to bootstrapping. CDN/proxy complications. Hosting providers could redirect this transparently. Easiest to implement, easiest to adopt.

- **C. DNS TXT record** — `_messaging.example.com TXT "v=msg1 server=msgs.example.com port=1234"`. Same family as SPF/DKIM/DMARC; familiar to email administrators. TXT records are overloaded but widely supported. No RFC to reference; protocol would define the format.

- **D. IM2000 prime-number MX** — Signal support via prime MX priority values; fall back to SMTP if not recognized. No DNS changes required for publishers; discovery is implicit. See extended discussion below.

- **E. Hybrid** — DNS SRV preferred; fall back to well-known URL; fall back to SMTP bridge. Maximizes compatibility at cost of discovery complexity.

---

### Problem 2: Notification Transport

Notifications are specified as encrypted UDP. The details matter for reliability, encryption, and NAT traversal.

**Options:**

- **A. Raw UDP** — Minimal, lightweight. Sender retransmits on backoff. No connection state. NAT traversal handled by addressing receiving server's public IP directly. No built-in encryption at transport layer — encryption is at the payload layer per the requirements doc.

- **B. QUIC** — Reliable UDP with built-in TLS 1.3. Handles retransmission internally. Multiplexed streams. NAT traversal friendlier. More complex to implement. May be overkill if notification is truly fire-and-forget.

- **C. UDP notifications + separate TCP/QUIC for fetch** — Notifications are one-way UDP fire-and-forget; message fetch uses a reliable request/response channel. Matches the asymmetric nature: notifications are inherently lossy by design, fetches must not be.

- **D. All TLS/TCP** — Simpler implementation, more universally supported, no UDP firewall issues. Heavier per notification. Loses the lightweight notification model.

**Note:** The requirements doc specifies "no rejection response to notifications." This rules out anything that requires a handshake before notification delivery.

---

### Problem 3: Domain-to-Domain Authentication

How does a receiving server verify who is sending a notification or responding to a fetch request?

**Options:**

- **A. TLS client certificates tied to domain keys** — Domain presents its signing key as a TLS client certificate. Server verifies certificate against published domain key. Strong binding between transport identity and messaging identity.

- **B. Signed notification payloads only** — Notifications are signed by sender domain key; transport-layer identity is not verified. Simpler, works over UDP. Notification forgery is detectable but doesn't prevent amplification abuse.

- **C. Challenge-response on fetch** — Sender's transit server issues a signed challenge when a fetch arrives; requestor must sign with their domain key. Fetch channel authenticated; notification channel authenticated by signature only.

- **D. HMAC tokens derived from shared domain reputation** — Domains that have exchanged messages generate a shared secret for cheaper verification. Cold-start requires option A or B. Reduces cryptographic overhead for established relationships.

- **E. None at notification layer** — Spam filtering and msgcoin balance handle abuse. Notification authenticity is verified at payload layer. Simplest, relies entirely on economic disincentives. Appropriate if the economics are trusted to work.

---

### Problem 4: Message Fetch Protocol

How does a recipient server (or client directly) fetch a message by ID from a sender server?

**Options:**

- **A. HTTPS REST** — `GET /message/{id}` with domain authentication header. Simple, proxiable, CDN-cacheable (message ID as capability token fits perfectly). Requires HTTPS infrastructure on every S2S server. High-latency overhead for small messages but acceptable for large payloads.

- **B. Custom binary protocol over TLS** — Minimizes overhead, purpose-built. Higher implementation burden, no off-the-shelf tooling.

- **C. QUIC streams** — One connection, multiple concurrent fetches over multiplexed streams. Good for batch fetching. QUIC library required.

- **D. HTTPS with range requests** — Allows partial fetch and CDN support trivially. Useful for large file-type messages.

**Note:** The capability model in the requirements doc (message IDs as capability tokens, CDN-compatible) strongly favors HTTPS/REST. This is worth evaluating carefully.

---

### Problem 5: Transit Storage Access Control

Which servers are permitted to retrieve a given message from transit storage?

**Options:**

- **A. Capability token only** — Knowing the message ID is sufficient to retrieve ciphertext. No authentication required. Any server can cache and serve any blob it has. Maximum cacheability.

- **B. Message ID + requester domain authentication** — Sender verifies requester is the intended recipient's server. Limits who can fetch. Recipient domain must be known at send time.

- **C. Two-phase** — Anyone can retrieve ciphertext; only authorized parties can retrieve the content decryption key. Separates delivery from decryption access.

---

### Problem 6: Responsibility Transfer

When a gateway accepts a message and assumes notification responsibility, how is the handoff formalized?

**Options:**

- **A. Signed acknowledgment** — Gateway signs a receipt with its domain key: "I have accepted responsibility for message ID X for recipient domain Y." Sender can verify and purge from their transit queue.

- **B. Implicit transfer** — Sender infers transfer when gateway takes over retransmission. No explicit protocol message. Simpler but harder to audit.

- **C. Transfer receipt + expiry** — Gateway issues receipt and declares the expiry it will honor, which may differ from original sender's expiry. Recipient domain now has visibility into gateway-declared delivery window.

---

### Problem 7: Notification Minimum Field Set

What is the minimum information in a notification packet? (Requirements doc defers this.)

**Options:**

- **A. Message ID + sender domain only** — Recipient domain looks up message by ID. Minimal exposure. Recipient hash must be in the encrypted payload.

- **B. Message ID + sender domain + recipient hash** — Allows receiving server to route to correct user's pending queue without decrypting. Recipient hash is already opaque per the hashed addressing model.

- **C. Add MIME type hint** — Lets recipient server make pull/discard decisions without fetching. Useful for file-type messages. Slightly more metadata exposed.

- **D. Fully opaque** — Just "you have pending mail from someone, check by pulling a challenge token." Maximum privacy. Maximum complexity. Likely impractical for first version.

---

### Problem 8: Key and Policy Publication

Where do domains publish their public keys and policy metadata?

**Options:**

- **A. DNS TXT records** — Consistent with DKIM/DMARC. Good tooling. Size limits constrain what can be published. Key material may exceed TXT record limits; fingerprint + pointer to full record may be needed.

- **B. Well-known HTTPS endpoint** — `/.well-known/messaging-keys`. No size limit. Adds HTTP dependency. CDN-cacheable. Easy to update without DNS propagation delay.

- **C. Combined** — DNS TXT contains fingerprint and well-known URL; HTTPS contains full key material. DNS fingerprint allows verification without trusting the HTTPS response.

- **D. Dedicated key server** — Separate domain (e.g., `keys.example.com`) serves key material. Consistent with existing key server models. Another service to operate.

---

### Problem 9: Protocol Versioning

How do servers negotiate protocol version?

**Options:**

- **A. Version in DNS/well-known record** — Advertised at discovery time. Client knows before connecting which versions server supports.

- **B. TLS ALPN extension** — Protocol version negotiated during TLS handshake. Well-established for HTTP/2, HTTP/3. Requires registered ALPN identifiers.

- **C. Version field in notification/fetch headers** — Version declared per-message. Allows gradual rollout within a deployed server. Complex.

- **D. Separate DNS records per version** — `_msgs-v1._tcp.example.com`, `_msgs-v2._tcp.example.com`. Clean separation. DNS record proliferation.

---

### Problem 10: Msgcoin Ledger Synchronization

Domain-level ledgers are append-only signed chains. How do two domains share ledger state?

**Options:**

- **A. Pull on interaction** — When domain A fetches a message from domain B, it also pulls domain B's latest ledger state relevant to the interaction. Lazy synchronization.

- **B. Periodic gossip** — Domains periodically push ledger summaries to known correspondents. More timely but adds overhead.

- **C. No synchronization** — Each domain maintains its own local ledger for its outbound behavior only. No consensus required. Reputation is purely local. Simplest; loses cross-domain reputation visibility entirely.

- **D. On-demand query** — Domain B can query domain A's published ledger state via S2S request. Domain A decides what to publish.

---

## C2S Protocol

### Problem 1: Transport

What protocol carries C2S traffic?

**Options:**

- **A. HTTPS/REST + WebSocket for push** — Familiar, proxiable, works through corporate firewalls. Push via WebSocket. Most tooling exists. Versioning via URL path. REST semantics may be awkward for streaming message retrieval.

- **B. gRPC (HTTP/2 + protobuf)** — Bidirectional streaming, binary encoding, strong schema definition, auto-generated clients. Requires gRPC tooling on both sides. Excellent fit for the operation types. Less human-debuggable.

- **C. QUIC with multiplexed streams** — One connection, independent streams per operation type (notifications, fetch, key management). Low latency, NAT-friendly. New library dependency. Best long-term for native clients; worst for web clients today.

- **D. Custom TLS binary** — Maximum control over encoding. Highest implementation burden. No off-the-shelf tooling.

**Practical note:** Web clients want HTTPS/REST. Native desktop/mobile clients want something more efficient. These may want different answers. A versioned HTTPS API serves both, with an optional binary/gRPC binding for native clients.

---

### Problem 2: Client Authentication

How does a client prove it is an authorized user of the domain?

**Options:**

- **A. Username + password → session token** — Familiar, universally supported. Password transmitted over TLS. TOTP/hardware key as second factor. Session tokens rotated on use.

- **B. Client TLS certificate** — User's key pair used as TLS client cert. Cryptographically strong, no password. Distribution problem: getting the cert onto new devices requires a separate ceremony. Fits well with the key management model already in the protocol.

- **C. Passkey / WebAuthn** — Phishing-resistant, no shared secret. Platform-native UX on modern devices. Requires browser or platform support. Not universally available on all client types (headless, CLI).

- **D. Session token + device token** — Server issues a long-lived device token and short-lived session tokens. Device registration is a distinct ceremony. Revocation is per-device.

- **E. OAuth2 / OIDC for third-party clients** — Domain acts as authorization server. Third-party clients request scoped tokens. Standard model for mail clients that aren't the canonical client. Overhead for single-domain deployments.

**Note:** Legacy gateway clients using IMAP/SMTP will authenticate via existing mechanisms — C2S auth only applies to native new-protocol clients.

---

### Problem 3: Message Submission

How does a client hand off an outbound message to its server?

**Options:**

- **A. Single-shot encrypted submission** — Client submits complete envelope + encrypted payload in one request. Server validates envelope fields and queues for S2S delivery. Simple. Full message must be buffered before submission.

- **B. Streaming upload** — Client streams payload to server, server writes to transit store. Better for large file-type messages. Requires the server to handle partial uploads.

- **C. Two-phase: reserve ID, then upload** — Client requests a message ID from server first, then uploads payload separately. Allows client to include the message ID in linked messages (reference messages for file transfer) before upload completes. More round trips.

- **D. Multipart** — Envelope and payload submitted as multipart; server splits them. Familiar from HTTP. Awkward for binary payloads.

---

### Problem 4: Message Retrieval

How does a client list and fetch messages from the server's store?

**Options:**

- **A. Paged list + per-message fetch** — `GET /messages?folder=inbox&after={cursor}` returns a page of message summaries; client fetches full payload per message. Simple, cacheable. N+1 pattern for loading a full folder.

- **B. Batch fetch** — Client requests a list of message IDs and gets all payloads in one response. More efficient for initial sync. Large responses.

- **C. Delta sync / changelog** — Server maintains a numbered event log (message received, message deleted, flags changed). Client stores its last-seen sequence number and requests changes since then. Multi-device consistency falls out naturally. More server state to maintain.

- **D. IMAP-like sync model** — UIDs, sequence numbers, FETCH operations, conditional requests. Proven for mail workload. Complex to implement correctly. Good bridge compatibility story.

---

### Problem 5: Folder and Mailbox Organization

How are stored messages organized?

**Options:**

- **A. Hierarchical folders, server-side** — Server stores folder membership. Compatible with IMAP bridge. Server must maintain folder state per user.

- **B. Tags/labels, server-side** — Messages have sets of tags. More flexible than folders. Server maintains tag index. "Inbox" is a tag.

- **C. Client-side only** — Server stores a flat collection of messages with metadata. Clients organize locally. Zero server complexity. Bad for multi-device: each device has different organization. Not compatible with IMAP bridge.

- **D. Folders as first-class + labels as extension** — Folder hierarchy for compatibility; label support added on top. More complex but covers both use cases.

---

### Problem 6: Server-Push Notification to Client

How does the server alert a client that a new message has arrived or a notification has been received?

**Options:**

- **A. WebSocket long-poll** — Client maintains a persistent WebSocket; server pushes notification objects. Works through most firewalls. Requires server to maintain per-client connection state.

- **B. Server-Sent Events (SSE)** — HTTP/2-native unidirectional streaming from server to client. Simpler than WebSocket. Read-only push; client uses separate requests for commands.

- **C. Client polling** — Client polls on a schedule. Simplest server implementation. Not real-time. Fine for batch email behavior; bad for IM-like latency expectations.

- **D. OS push (APNs, FCM)** — Native mobile push. Requires third-party intermediary (Apple, Google). Privacy implications. Client device must be registered with third party.

- **E. QUIC streams** — Multiplexed bidirectional streams on a persistent QUIC connection. Best long-term for native clients. Not available from browser JS today.

---

### Problem 7: Hash Address Management

The client generates and tracks hashed recipient addresses for privacy. The server needs to know which hashes to watch for.

**Options:**

- **A. Client registers hashes with server** — Client tells server: "watch for notifications addressed to this hash." Server maintains a hash table per user. Server knows which hashes belong to which user.

- **B. Blind server** — Server stores all incoming notification hashes. Client periodically polls: "anything for hash X?" Server returns matching message IDs. Server never associates hashes with users. Slower lookup, more private. Must handle hash table growth.

- **C. Per-sender registration** — Client registers a specific (sender, hash) pair. More granular. Server can correlate sender with hash for that sender. Acceptable tradeoff if sender identity is known at receipt time.

- **D. Privacy proxy model** — Server knows nothing about hashes. Proxy handles the hash→user lookup. Maximum privacy. Requires operating or trusting a proxy.

---

### Problem 8: Multi-Device Sync

User has multiple clients on multiple devices. All should see consistent state.

**Options:**

- **A. Server-side state, clients are views** — All state (messages, folder membership, flags, read status) lives on server. Clients request state on connect. Simplest model. Requires persistent server state per user.

- **B. Event log** — Server maintains a per-user append-only event log. Clients store their last-seen sequence. On reconnect, client requests events since last sequence. Eventual consistency. Works well offline.

- **C. Per-device encrypted copies** — Server stores encrypted message per device. Each device has its own "mailbox." Adding a new device requires re-delivering. POP3 model. No cross-device sync of read/delete state.

- **D. Hybrid: server state + client-cached encrypted copies** — Server holds structural state (UIDs, flags, folder membership); clients hold encrypted payloads locally. New device gets structural state from server, re-fetches payloads as needed.

---

### Problem 9: Key Management Operations

Client manages keys through the C2S interface.

**Operations to support:**
- Submit a user-generated public key for domain signing
- Retrieve domain-signed key for distribution
- Publish revocation
- Rotate key (combined revocation + new key)
- Retrieve another user's signed key (inbound key discovery)
- Monitor own published fingerprint (canary)

**Options:**

- **A. Dedicated key management API** — Separate endpoint group for all key operations. Clean separation from messaging. Clients that only do key management need only implement this subset.

- **B. Integrated into message protocol** — Key operations are messages to a special system address. Reuses message delivery infrastructure. Awkward for operations that need immediate consistency (revocation).

- **C. Separate key management protocol** — Entirely out of band. Not recommended — splits the client implementation needlessly.

---

### Problem 10: Policy Configuration

Client configures per-sender pull preferences, reputation thresholds, retention policies, and rate-limit floor behavior.

**Options:**

- **A. Structured settings API** — Typed endpoints per policy domain. Easy to validate, document, and version. More endpoints to implement.

- **B. Policy document** — Client submits a structured JSON/CBOR policy document. Server validates schema. Flexible, easy to extend. Full document replaced or patched on update.

- **C. Per-sender preference API** — Pull preference is set per-sender-domain with a sensible default. Policy document covers everything else. Hybrid approach reflecting the fact that per-sender preferences will be very common.

---

### Problem 11: Client Capability Negotiation

Native full clients, legacy gateway clients, and minimal clients need different capability sets.

**Options:**

- **A. Capability flags on connect** — Server advertises capabilities; client requests subset. Client declares its type (native, gateway, minimal). Established model (IMAP CAPABILITY, ESMTP extensions).

- **B. API versioning** — v1 endpoints for baseline; v2 for full feature set. Clients use the version they support. Clean but leads to version fragmentation.

- **C. Feature flags in auth response** — Server includes feature list in authentication response. Simple, one round trip.

---

## Domain Discovery: Extended Discussion

### The Problem

Every federating messaging protocol must solve: given a domain name like `example.com`, how does a sending server find the actual server IP and port to talk to?

Email solved this with MX records — simple, works, but MX is email-specific and carries SMTP semantics. A new protocol needs its own discovery mechanism. This decision has major adoption implications.

---

### IM2000: The Prime-Number MX Approach

**Reference:** https://cr.yp.to/im2000.html

DJB proposed that IM2000 servers signal support via MX record priority values. Specifically: an MX record with a prime number priority (2, 3, 5, 7, 11...) signals IM2000 support; standard SMTP falls back to composite-priority MX records.

**Mechanics:**
- Domain publishes MX records as usual for SMTP
- IM2000-capable server adds or modifies an MX record to use a prime priority
- Sending server tries IM2000 on prime-priority targets before falling back to SMTP
- No new DNS record types required
- Existing DNS infrastructure handles it unchanged

**Arguments for:**
- Zero DNS operator changes — works with any DNS infrastructure that supports MX (which is all of them)
- Transparent fallback to SMTP is automatic and deterministic
- No coordination with DNS registrars, hosting providers, or DNS tooling vendors
- Fastest possible adoption path for early deployers
- DJB's track record (qmail, djbdns) suggests the approach is deployable in practice

**Arguments against:**
- Semantics are implicit convention, not protocol — nothing formally assigns prime vs. composite to IM2000 vs. SMTP
- MX is already overloaded; mixing IM2000 semantics into it creates a maintenance burden
- Prime priority signals nothing about which IM2000 version, what port, or what capabilities
- A third protocol wanting similar treatment cannot use the same trick
- The approach is undocumented in any RFC; discovery depends on knowing the convention
- Priority field is 16 bits; in practice, operators use small round numbers (10, 20, 100). Accidentally landing on a prime (2, 3, 5, 7, 11...) creates silent false positives
- Not extensible: adding metadata (server address, port, capabilities) requires additional tricks

**Assessment:** Clever and deployable, but fragile by design. Appropriate for a one-man project (DJB running his own infrastructure) or a guerrilla deployment. Problematic as a formal protocol standard where multiple implementations need to agree.

---

### DNS SRV Records

**Reference:** RFC 2782 (January 2000)

SRV records provide explicit service discovery: `_service._proto.domain SRV priority weight port target`.

**Example for this protocol:**
```
_msgs._tcp.example.com.   IN SRV  10 5 1234 msgs.example.com.
```

**Arguments for:**
- Well-specified, documented, long-established
- Explicit: service name, port, weight, target are all directly stated
- Already deployed for XMPP (`_xmpp-server._tcp`), SIP, and others
- Extensible: add more SRV records for new versions or services
- No ambiguity — `_msgs._tcp.example.com` unambiguously means "messaging service for this domain"
- Separate from MX — no inherited SMTP semantics

**Arguments against:**
- Adoption friction is real. Many DNS control panels either don't support SRV or require manual zone file editing. Shared hosting often doesn't expose SRV at all.
- Email administrators are not familiar with SRV (it's a voice/chat record to them)
- SMTP explicitly chose MX over SRV in its history — there is institutional resistance
- DNS propagation delay affects service changes (same as MX, but worth noting)
- Many DNS libraries and utilities don't implement SRV lookup correctly
- Self-hosting with common providers (Cloudflare, Route53, Namecheap) has variable SRV support

**Adoption data point:** XMPP adopted SRV in 2002. Over two decades later, SRV support in DNS control panels is still uneven. XMPP federation adoption remained low. Correlation is not causation, but the difficulty of SRV provisioning contributed to XMPP's deployment friction.

---

### Well-Known HTTPS

**Mechanics:**
```
GET https://example.com/.well-known/messaging
```
Returns JSON:
```json
{
  "version": "1",
  "s2s_endpoint": "msgs.example.com:1234",
  "key_endpoint": "https://msgs.example.com/.well-known/messaging-keys",
  "capabilities": ["v1", "notifications-udp", "fetch-https"]
}
```

**Arguments for:**
- No DNS configuration changes required — works with any DNS that supports A records
- Can carry capabilities, version, and multiple endpoints in one request
- HTTPS is universally supported by hosting providers
- Familiar deployment model (same family as `/.well-known/acme-challenge`, WebFinger, etc.)
- Easy to redirect, proxy, and CDN-cache
- Can be updated without DNS propagation delays

**Arguments against:**
- Adds HTTP dependency to bootstrapping — S2S server now needs to make an HTTP request before sending anything
- If `example.com` is down, bootstrapping fails even if `msgs.example.com` is healthy
- Corporate firewalls and proxies may interfere with the HTTPS request
- TLS certificate for `example.com` must be valid — conflates web hosting with messaging infrastructure
- Chicken-and-egg: verifying the HTTPS response requires trusting the TLS cert, which is a different trust chain than the messaging key chain

**Note:** The HTTPS and DNS TXT/SRV approaches are composable — DNS carries a fingerprint and pointer, HTTPS carries full metadata. This hybrid addresses the chicken-and-egg problem.

---

### Summary Table

| Approach | DNS Changes | Deployment Ease | Expressiveness | Precedent |
|---|---|---|---|---|
| Prime-number MX | None | Trivial | Very low | IM2000 only |
| DNS SRV | New record type | Moderate | Good | XMPP, SIP |
| DNS TXT | New record | Easy | Moderate | SPF, DMARC |
| Well-known HTTPS | None | Easy | High | WebFinger, ACME |
| Hybrid DNS TXT + HTTPS | New record | Moderate | High | Novel |

### Editorial Position (Deferred)

No decision made. The choice should be driven by:
1. Deployment experience of small hosters (primary target audience)
2. Whether the initial reference implementation targets technical users or general adoption
3. Whether we prioritize zero-infrastructure-change adoption or long-term protocol cleanliness

The prime-number MX approach is probably not the right answer for a formal protocol standard, but it merits acknowledgment as the approach taken by the closest prior art.

The well-known HTTPS approach has the best adoption profile but adds infrastructure coupling. DNS TXT has good prior art in the email space. SRV is technically cleanest but historically problematic for email adoption.

**Recommend prototyping with well-known HTTPS** for the reference implementation (zero friction for early adopters) while specifying DNS SRV as the formal mechanism. Implementations SHOULD support both.

---

*See also: [next-gen-messaging-protocol.md](./next-gen-messaging-protocol.md)*
