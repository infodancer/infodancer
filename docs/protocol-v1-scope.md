# Protocol v1 Scope

*Normative design document -- concrete design phase*
*Issue: https://github.com/infodancer/infodancer/issues/5*
*Last updated: 2026-06-11*

This document draws the v1 line. Everything in section 1 is in scope for the
first implementable version; everything in section 2 is deliberately deferred,
with the reason and the condition under which it comes back. The requirements
doc ([next-gen-messaging-protocol.md](next-gen-messaging-protocol.md))
remains the statement of where the protocol is ultimately going; nothing here
removes a requirement, only sequences it.

The test for v1 inclusion: **does the core loop -- submit, notify, pull,
accept, retract -- need it to be correct, secure, and honest about its
properties?** Features that broaden the loop wait; features that protect it
do not.

## 1. In scope for v1

- The core message loop: C2S submit, S2S notify (UDP), S2S fetch (HTTPS),
  acceptance, pre-delivery retraction, retraction-request messages.
- The full cryptographic architecture of
  [protocol-crypto-design.md](protocol-crypto-design.md): SDMP-V1 suite,
  hybrid-PQ payload encryption, key hierarchy, rotation with overlap,
  routing tags, sign-the-bytes.
- Key publication, discovery, rotation, revocation; DNS fingerprint
  cross-check; fingerprint caching; self-canary with vantage diversity.
- The anti-abuse machinery of the threat model: normative notification
  pipeline, bounded pending tables, quotas, uniform responses, normative
  default pull policy, local-only reputation tiers (unknown / established).
- Multi-device: enrollment ceremony, device list, revocation-triggers-
  rotation, event-log sync.
- SMTP bridges, both directions, with signed bridge provenance and
  composition-time downgrade disclosure.
- Domain migration notices (signed by old domain + user key). Kept in v1
  because it is a single message type riding existing machinery, and "you
  can leave your provider without losing your identity" is an adoption
  argument the first release should be able to make honestly.
- File transfer (inherent: it is just a payload) and reference messages
  (an envelope flag plus client behavior).
- Threading (envelope field, client behavior).
- Prometheus-grade observability per TM-X-6.

## 2. Deferred, with re-entry criteria

| # | Deferred item | Why | Comes back when |
|---|---|---|---|
| D1 | **Msgcoin ledger synchronization** | No sybil resistance at protocol layer (T-REP-1); bilateral ledger reconciliation and dispute semantics are a protocol's worth of unsolved complexity; the storage-cost inversion plus TM-ACCEPT-2 defaults already carry the v1 spam economics. v1 keeps *local* reputation only. | A worked design exists for sybil-resistant balance bootstrap and refund verification, validated against real v1 traffic data. |
| D2 | **Reputation publication / web-of-trust** | Already marked future work in the requirements doc; depends on D1 semantics. | D1, plus demonstrated demand from multi-domain operators. |
| D3 | **Blind routing (pairwise-hash routing) + blind-server mode** | First-contact paradox and pending-table DoS (threat model section 10); v1 routing tags keep the wire format forward-compatible. | v2 design for the unknown-hash bucket that satisfies TM-NOTIF-2/3/4; privacy proxy (D4) available to make the claim meaningful. |
| D4 | **Privacy proxy** | Reduces RR-1/RR-2/RR-3 residuals; not needed for loop correctness. | v2; design alongside D3 since the claims compose. |
| D5 | **Blind-cache intermediaries / gateways + responsibility transfer (TransferService)** | Whole subsystem (transfer receipts, gateway expiry, high-latency links) serving topologies v1 deployments will not have. Direct domain-to-domain only in v1. | First real demand (satellite/intermittent deployment, or store-and-forward relay operators). Proto is parked, not deleted. |
| D6 | **Mailing lists (managed and manifest models)** | Both models ride on the core loop plus list-server software; nothing in v1 blocks them. Ad-hoc lists (a user sending to multiple recipients on their own reputation) work in v1 by construction. | Core loop stable; list server is its own design doc against the v1 spec. |
| D7 | **TLD-signs-domain trust chain** | Requires registry cooperation nobody has yet; v1 trust levels are DNS-authenticated baseline + direct out-of-band verification. The key-record format already carries the chain slot. | A cooperating TLD or registrar exists. |
| D8 | **Read receipts** | Flag exists in the envelope; semantics (consent model, privacy) deserve their own pass. v1 servers/clients ignore the flag. | Client UX design phase. |
| D9 | **Phase 3 published single-use hash tokens** | Belongs to the blind-routing privacy tier (D3/D4). | With D3. |
| D10 | **Notification source cookies (anti-spoofing round trip)** | Speculative at v1 scale; rate limits + trial-decrypt cost cover the modeled threat; datagram version byte reserves the upgrade path. | Deployment evidence of spoofed-source floods overwhelming stage 2/3 of the receive pipeline. |
| D11 | **QUIC / HTTP-3 fetch surface** | Optimization, not capability; path-versioned HTTPS makes it a drop-in later. | Reference implementation benchmarks justify it. |
| D12 | **ML-DSA (post-quantum) signatures** | No harvest-now risk for signatures; size conflicts with the datagram budget (CD-1). Suite agility reserved. | NIST/industry settlement plus a notification framing answer (likely: PQ signatures on TCP-carried structures only). |

## 3. Consequences accepted by this scope

- v1's privacy claim against the user's own domain is the honest, limited one
  of TM-ROUTE-2 -- D3/D4 are what upgrade it. Documentation and marketing
  MUST track the v1 claim, not the v2 ambition.
- v1 spam defense rests on storage-cost inversion, the normative two-tier
  pull default, pending-table quotas, and local reputation -- not on msgcoin.
  If those prove insufficient in practice, that evidence feeds D1's design
  rather than rushing it.
- v1 federation is direct domain-to-domain only; high-latency and relay
  topologies wait for D5.

## 4. Implementation sequencing (reference implementation)

The reference implementation lands in **maildancer** as new daemons that
import the external protocol repos (per maildancer's CLAUDE.md, the
dependency arrow is maildancer -> sdmp/scmp, never the reverse):

1. **sdmp/scmp repos**: reconcile protos to the wire spec (WS-10 lists the
   exact changes); add the notification binary framing and crypto primitives
   (suite, HPKE wrapper, sign-the-bytes helpers) as importable packages with
   test vectors.
2. **maildancer `cmd/sdmpd`**: notification listener (receive pipeline,
   pending tables) + HTTPS S2S resources + transit store, wired to the
   existing queue/delivery architecture behind session-manager, honoring the
   depguard boundaries.
3. **maildancer `cmd/scmpd`**: C2S services against session-manager, reusing
   the auth stack (passkeys are new; the OIDC work is adjacent, not blocking).
4. **Bridges**: inbound SMTP-to-SDMP in the smtpd delivery path (bridge
   provenance signing); outbound via the existing queue-manager transport
   routing.
5. **messagedancer**: native client work begins once scmpd exists to talk to.

Each step gets its own issues in the owning repos; this document is the
cross-repo sequencing reference.

---

*Companions: [protocol-threat-model.md](protocol-threat-model.md),
[protocol-crypto-design.md](protocol-crypto-design.md),
[protocol-wire-spec.md](protocol-wire-spec.md).*
