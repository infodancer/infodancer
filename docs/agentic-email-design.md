# Agentic Email: Delegated Authority and Content Trust

*Design proposal -- NOT decided. Captures a direction explored in discussion;*
*open decisions are called out inline. Last updated: 2026-06-13.*

## The reframe

The market answer to "email for AI agents" is currently *give the agent its own
address*. That is the thin part. An address is a routing label; the value is the
**delegation, scoping, and provenance** behind it -- and that is the part the
agent-email startups mostly hand-wave.

This doc argues that the two primitives worth building are:

1. **Attenuable, verifiable, revocable delegated authority** -- a capability
   model, so an agent holds exactly the access a human granted it and no more,
   and a recipient can verify under whose authority an agent acted.
2. **A content-trust boundary** -- because an agent reading a mailbox is
   ingesting attacker-controlled input, and email is the number-one prompt-
   injection vector. The server is the right place to make incoming content safe
   to consume.

Everything else (push, memory, disposable addresses, quotas) is secondary and
rides on substrate that already exists in this stack. The through-line: the bets
this suite already made -- privilege separation, per-domain keys, SCMP
capabilities, federated OIDC identity, SDMP notifications, the client-encrypted
document namespace -- are *exactly* the primitives an agent-trustworthy mail
system needs. The agent-email companies are building an addressing layer on top
of commodity mail; the scarce, hard layer is the authority-and-trust substrate
underneath it.

## Why this fits the existing substrate

Like the keyring proposal, this is not a new set of requirements -- it is mostly
existing commitments pointed at a new consumer.

1. **Capabilities are already the message-layer model.** The protocol specifies
   "single encrypted payload, per-recipient key wrapping distributed separately"
   ([next-gen-messaging-protocol.md](./next-gen-messaging-protocol.md)).
   Wrapping a *scoped slice of mailbox access* to an agent is the same
   construction as wrapping a message key to a recipient. The capability is not a
   bolt-on; it is the access-control dual of the encryption model.

2. **Federated identity already issues and verifies tokens.** auth-oidc is a leaf
   IdP per owned domain ([oidc-federation-design.md](./oidc-federation-design.md),
   [federated-identity-stack.md](./federated-identity-stack.md)). Minting and
   verifying attenuated, signed capability tokens -- and delegation credentials
   binding `agent acts-for human, scope=...` -- is an identity-layer job this
   stack is already shaped for.

3. **The client-encrypted document namespace already exists.** Filter Rule
   Storage (C2S Problem 12 in [protocol-outlines.md](./protocol-outlines.md))
   generalizes to "a small namespace of client-encrypted, versioned
   configuration documents." Thread-scoped agent memory is one more entry in that
   namespace.

4. **Push already exists at the federation layer.** SDMP carries UDP/protobuf
   notifications ([next-gen-messaging-protocol.md](./next-gen-messaging-protocol.md)).
   Agent event subscriptions are a client-facing subscription layer over the same
   idea.

5. **Privilege separation is the right mental model for an agent.** The suite
   already runs delivery and retrieval as constrained uid/gid processes that hold
   credentials for exactly one user at a time
   ([mail-security-model.md](./mail-security-model.md)). An agent is another
   constrained principal -- a capability is its credential, and it should be
   scoped at least as tightly as a mail-session.

## The primitives

Each gap below names the server primitive and the existing piece it rides on.

### 1. Capabilities: attenuable, scoped, revocable authority

**Gap.** Email auth is all-or-nothing. A password or OAuth token grants read of
*everything* and send *as you*. Even Gmail's OAuth scopes are coarse
(read / modify / send) -- never per-correspondent, per-folder, per-time-window,
or rate-bounded. An agent does not need that, and granting it is the entire risk.

**Primitive.** A macaroon-style capability: a signed bearer token with caveats
the holder can *further narrow but never broaden* before delegating onward.
Caveats span at least:

- scope: folders/labels, read vs send vs annotate vs act
- correspondent allowlist / denylist for send
- time bounds (message age on read; token expiry)
- rate and recipient caps (see blast-radius below)
- a "propose-only above N recipients" gate (see human-in-the-loop)

Attenuation matters: an orchestrator agent holding a broad capability can mint a
narrower one for a sub-agent without server round-trips, and the server verifies
the chain. Revocation is by short expiry plus a revocation list keyed on a
capability id; never long-lived bearer secrets.

**Rides on.** SCMP capability / key-wrapping model; auth-oidc as issuer/verifier.

**Open question.** Macaroons (HMAC caveat chain, offline attenuation, no
asymmetric verification by third parties) vs signed tokens / verifiable
credentials (third-party-verifiable, heavier). Attenuation favors macaroons;
cross-domain provenance (primitive 2) favors signatures. Likely both, at
different layers.

### 2. Delegated provenance: DKIM-for-delegation

**Gap.** DKIM proves the *domain* signed a message. It says nothing about which
principal, under what authority. A recipient (and a recipient's agent) needs to
distinguish: a human sent this; an agent sent this autonomously; an agent sent
this under a delegation the human signed, with scope attached.

**Primitive.** A verifiable delegation credential carried in the message, binding
agent identity + delegation chain + scope, signed up to the human's key or the
domain key. Think of it as a signed assertion "`agent@` acts for `human@`,
scope=`send-on-behalf:thread-123`, valid-until=...", verifiable by any recipient
that trusts the domain. A recipient agent can then decide how much weight to give
the message.

**Rides on.** Per-domain key infrastructure; the same signing primitives as DKIM
and the published key lifecycle.

**Open question.** Header-carried credential vs a body MIME part vs an out-of-band
lookup keyed by a message identifier. Header is simplest and survives existing
transport; size limits and folding are the cost.

### 3. The content-trust boundary (the prompt-injection wart)

This is the security-differentiated primitive and the one the large providers are
least likely to build, because it requires being opinionated about agent safety.

**Gap.** An agent reading a mailbox ingests attacker-controlled content:
instructions hidden in white-on-white text, zero-width characters, image alt
text, base64 parts, quoted-reply tails. Current AI email integrations dump raw
bodies straight into model context with essentially no defense. Letting an agent
autonomously *act* on incoming mail today is, bluntly, a security incident
waiting to happen.

**Primitive.** The server is the right place to intervene because it holds the
raw MIME and already computes most of the needed signals.

- **Per-message trust tiering**: authenticated + known correspondent; SPF/DKIM/
  DMARC-pass but stranger; first-contact; failing. Surfaced so an agent can gate
  "may this content influence actions" (read-only vs actionable).
- **An agent-safe rendering** distinct from the human view: hidden / zero-width /
  offscreen text stripped, quoted sections separated and labeled, steganographic
  patterns flagged, imperative "instructions to the reader" quarantined.
- **A hard data/instruction marking**: deliver body content to the agent tagged
  as *data, never instructions*, so a compliant agent runtime treats it as
  inert.

**Rides on.** Existing SPF/DKIM/DMARC verification, greylisting, and correspondent
history -- the trust tier is mostly assembling signals already computed. The
agent-safe renderer is new work but bounded.

**Open question.** How much canonicalization the server should do vs how much it
should only *annotate* and leave to the client/agent. Over-rewriting risks
breaking the human view or losing evidence; under-annotating leaves the agent
exposed. Leaning toward annotate-and-quarantine over destructive rewrite, with
the safe rendering offered as an additional view, not a replacement.

### 4. Push subscriptions for ephemeral agents

**Gap.** IMAP IDLE is connection-bound and assumes a long-lived process holding
full credentials -- the exact anti-pattern for serverless/ephemeral agents.

**Primitive.** An agent registers a filter + a signed delivery endpoint + a
verification secret and receives structured events pushed to it. The subscription
carries only the capability it needs, so the agent never holds a standing
full-mailbox session.

**Rides on.** SDMP UDP/protobuf notifications, as a client-facing subscription
layer.

**Open question.** Delivery transport (signed webhook vs a long-poll/stream vs
SDMP-native) and the retry/replay story for an endpoint that is down.

### 5. Thread-scoped agent memory

**Gap.** Agents bolt on external databases to remember "what did I already do
about this thread." The mailbox is the natural home for thread-scoped state.

**Primitive.** A per-thread (or per-account) key-value annotation store -- an
"agent scratchpad attached to a thread" -- client-encrypted, opaque to the
server, versioned for compare-and-swap.

**Rides on.** The client-encrypted, versioned document namespace (C2S Problem 12),
the same pattern as the keyring and filter rules.

**Open question.** Thread identity across forwards/subject changes; garbage
collection when a thread dies; whether memory is per-agent or shared across an
account's agents.

### 6. Disposable, capability-bearing addresses

**Gap.** Sub-addressing (plus-addressing) is the poor man's version: a label, no
authority, trivially stripped or guessed.

**Primitive.** Because the domain is under our control, mint **signed local-parts**
that embed routing + scope + expiry -- a single-use address bound to one task or
correspondent, auto-expiring, routed to a specific agent with a specific
capability, where the local-part itself is a verifiable token. A client cannot do
this; it is server-only territory.

**Rides on.** Domain control + the AuthRouter normalization seam + capability
signing.

**Open question.** Local-part length and opacity (a signed token is not
human-friendly); collision and replay handling; whether these are first-class
mailboxes or pure routing tokens.

### 7. Outbound blast-radius controls

**Gap.** Agents act fast and fan out. A buggy or hijacked agent can send 10k
messages or exfiltrate a mailbox before anyone notices.

**Primitive.** Per-capability send quotas, recipient caps, "human co-sign required
above N recipients," outbound DLP hooks, and anomaly detection on agent send
behavior. Greylisting's outbound cousin, scoped per capability rather than per
sender.

**Rides on.** The existing Redis + Prometheus path; the capability id is the
natural quota key.

**Open question.** Where the co-sign gate lives (a pending-action object the
human's client approves, tied to releasing a held message) and how anomaly
thresholds are set without nuisance-blocking legitimate bursts.

## Honest caveats

- **Addressing is the easy part.** The "agents get their own email addresses"
  framing sells the routing label and skips the authority, provenance, and
  content-trust work that is actually hard and actually scarce. If we build here,
  build the substrate, not the vanity address.

- **Email is a bad agent-to-agent transport.** Much "agentic email" is people
  using email as a clumsy API. For agent-to-agent, that is the wrong tool -- use
  SCMP or an actual protocol. Email's value for agents is precisely the *human*
  boundary: agent-to-human and agent-to-external-human, where the existing email
  world is the constraint you cannot escape. Aim the work there.

- **Encryption and cloud AI are in genuine tension.** If you want server-opaque
  *and* agent processing, the agent must run client-side with the keys -- the
  "MCP client speaks SCMP, local models" path. You cannot have a cloud LLM
  process mail the server cannot read. The encrypted path therefore implies
  edge/local inference, or explicit per-message decryption with consent. This is
  a fork that shapes the whole product, not a footnote. See the at-rest threat
  model in maildancer's encryption docs and [keyring-design.md](./keyring-design.md):
  at-rest encryption protects against disk/backup compromise, not a live server,
  and only a crypto-capable client gets true opacity.

- **Autonomy is the liability, not the feature.** Until the content-trust boundary
  (3) and capabilities (1) exist, "just let the agent handle my email" is
  irresponsible. The defensible default is propose-and-confirm, with autonomy
  permitted only inside tightly scoped capabilities.

## Recommended sequencing

If this becomes real, order the work to front-load the defensible parts and build
on existing substrate:

1. **Capabilities** -- the keystone; everything else gates on it.
2. **Content-trust tiering + agent-safe rendering** -- the security
   differentiator, mostly assembling signals already computed.
3. **Delegated-provenance signatures** -- once capabilities exist, sign them into
   outbound mail.
4. **Push subscriptions, thread-scoped memory, disposable addresses, blast-radius
   controls** -- all ride existing substrate (SDMP notifications, the encrypted-
   document namespace, domain control, Redis/Prometheus).

## Open decisions (to settle before implementation)

1. **Capability token format**: macaroons (offline attenuation) vs signed VCs
   (third-party verifiable) vs both at different layers.
2. **Provenance carriage**: header vs MIME part vs out-of-band lookup.
3. **Content-trust posture**: annotate-and-quarantine vs destructive
   canonicalization; safe view as addition vs replacement.
4. **Inference location**: cloud (server sees plaintext, no opacity) vs edge/local
   (opacity, capability-limited models) -- and whether the product offers both
   with an explicit, disclosed trade.
5. **Agent identity model**: distinct addresses per agent vs a single human
   identity with capability-scoped sub-principals. (Distinct addresses are the
   market default; sub-principals may be the cleaner authority model.)
6. **Standardization ambition**: keep these proprietary to the SCMP path, or
   propose the delegation-provenance and content-trust pieces as something the
   wider ecosystem could adopt.

---

*See also: [next-gen-messaging-protocol.md](./next-gen-messaging-protocol.md)*
*(capability model, key wrapping, notifications), [protocol-outlines.md](./protocol-outlines.md)*
*(C2S problems, the client-encrypted document namespace),*
*[oidc-federation-design.md](./oidc-federation-design.md) and*
*[federated-identity-stack.md](./federated-identity-stack.md) (token issuance,*
*delegation), [mail-security-model.md](./mail-security-model.md) (privilege*
*separation), [keyring-design.md](./keyring-design.md) (at-rest model and the*
*encryption-vs-AI tension).*
