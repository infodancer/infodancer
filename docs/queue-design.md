# Mail Queue Design
*Last updated: 2026-03-04*

## Overview

A unified on-disk queue that supports three delivery modes — SMTP smarthost,
SMTP direct, and new-protocol retrieve-from-sender — without forking the queue
structure. A separate delivery binary handles all three modes; the queue manager
drives retry and expiry.

## Directory Structure

```
queue/
  msg/{tld}/{domain}/{msgid-hex}                   # message body (immutable, opaque bytes)
  env/{tld}/{domain}/{localpart}@{msgid-hex}.{n}   # one envelope file per recipient
```

Bodies are keyed by **sender domain**; envelopes are keyed by **recipient domain**.
Both split first by TLD, then by the registrable domain portion.

**Filesystem paths use only the hex left-hand part of the msgid** (see
[Message-ID Format](#message-id-format) below). The full RFC 5322 msgid
(`hex@sender-domain`) lives in the envelope `MSGID` field and the
`Message-ID:` header written into the body file.

### Examples

```
msg/com/example/abc123def456789              # body sent from example.com
env/com/gmail/alice@abc123def456789.001      # envelope → alice@gmail.com
env/com/gmail/bob@abc123def456789.002        # envelope → bob@gmail.com
env/co/example.uk/carol@abc123def456789.003  # envelope → carol@example.co.uk
```

For multi-level TLDs, the rightmost label is the TLD directory; everything to
the left of it is the domain directory (`example.co.uk` → `uk/example.co/`).

## File Formats

### Body file

Raw message bytes. Immutable once written. When encryption is enabled, this
will be an opaque encrypted blob; the queue and delivery layers never read the
content, only pass bytes.

### Envelope file

Plain text, one field per line:

```
TTL 2026-03-07T10:00:00Z
SENDER bounces+alice=gmail.com@origin.example.com
RECIPIENT alice@gmail.com
MSGID abc123def456789
```

`SENDER` is the VERP-encoded `MAIL FROM` address, computed at queue-inject time
and stored per-envelope. No generation needed at delivery time.

`TTL` is the absolute expiry timestamp. One TTL per message; all envelopes for
a given `MSGID` carry the same value.

`MSGID` is the full RFC 5322 message identifier: `{hex}@{sender-domain}`.
The hex part is 128 bits of cryptographically random data. Sequential or
predictable IDs are not acceptable. See [Message-ID Format](#message-id-format)
for the rationale for the sender-domain component.

## TTL and Cleanup

TTL is the **only** determinant of when queue entries are deleted. There is no
reference counting.

- Message bodies stay in `msg/` until their TTL passes.
- Envelope files stay in `env/` until successfully delivered or TTL passes.

On **successful delivery**: the envelope file is deleted. The body remains
until TTL — no early cleanup, no ref counting.

On **TTL expiry** (detected by the queue manager on any pass):
1. The delivery binary is invoked one final time via SMTP (safety net, regardless
   of the message's configured delivery mode).
2. The envelope file is deleted whether or not that final attempt succeeded.
3. Expired body files are swept separately; the queue manager deletes any body
   whose TTL has passed.

Future versions may replace the on-disk queue with an explicitly ephemeral
system (e.g., NATS JetStream); the TTL-only cleanup model is compatible with
that transition.

## Backoff via mtime

The queue manager uses filesystem timestamps as delivery timing hints — no
explicit retry counter or next-retry timestamp is stored in the envelope.

- **`msg/` mtime**: set at queue-inject time; represents message age.
- **`env/` mtime**: updated by the delivery binary on each attempt; represents
  time of last delivery attempt.

The queue manager computes the next eligible delivery time as:

```
next_attempt = env_mtime + backoff_interval(now - env_mtime)
```

where `backoff_interval` doubles with age (e.g., 5m → 10m → 20m → 1h → 4h),
capped at a configured maximum. No state beyond the mtime is needed.

## Delivery Binary: `mail-remote`

A standalone command-line binary. Named to parallel `mail-deliver` (local
delivery). Can be invoked by the queue manager or by hand.

```sh
mail-remote <body-file> <envelope-file> [envelope-file ...]
```

The body file is the first argument; one or more envelope files follow. This
allows the queue manager to batch all ready recipients for a given domain into a
single invocation — one body read, one SMTP connection, multiple `RCPT TO`.
It also enables clean manual single-recipient delivery for testing or redelivery.

```sh
# Single recipient
mail-remote queue/msg/com/example/abc123 queue/env/com/gmail/alice@abc123.001

# Batch: all ready envelopes for gmail.com
mail-remote queue/msg/com/example/abc123 \
    queue/env/com/gmail/alice@abc123.001 \
    queue/env/com/gmail/bob@abc123.002
```

`mail-remote` reads the first envelope to determine the recipient domain, performs
DNS resolution once for that domain, and delivers to all supplied recipients in
one session where the protocol permits (SMTP RCPT TO batching; new-protocol
sends one notification per message regardless of recipient count). All envelopes
must share the same recipient domain — the queue manager enforces this grouping.

DNS lookup order per recipient domain:

1. **SRV** `_mail._tcp.{recipient_domain}` → found: send new-protocol
   notification (store-and-notify model). Done.
2. No SRV → **MX** lookup → SMTP delivery to MX host.
3. No MX → **A** record → SMTP delivery to port 25.

New-protocol and SMTP are co-delivery systems, not a fork in queue structure.
On every retry attempt, DNS is queried fresh; a domain that gains an SRV record
between retries will automatically receive new-protocol delivery without any
queue reconfiguration.

Exit codes (per envelope):
- 0: success — caller deletes the envelope file.
- Temp-fail: caller updates env mtime (triggering backoff), retries later.
- Perm-fail: caller deletes the envelope; no further attempts.

## Queue Manager: `queue-manager`

Drives the retry loop. Responsibilities:

- Scan `env/` for envelopes whose backoff interval has elapsed.
- Batch envelopes by recipient domain: when multiple envelopes under the same
  domain directory are ready, invoke `mail-remote` with the body and all ready
  envelopes in one call (SMTP batches multiple RCPT TO per connection;
  new-protocol sends one notification per message regardless of recipient count).
- On TTL expiry: invoke `mail-remote` for final SMTP attempt, then delete
  envelope regardless of outcome.
- Sweep `msg/` for expired bodies and delete them.
- Rate-limit per destination domain: count in-flight envelopes per domain
  directory; pause new attempts when the limit is reached.

The queue manager does not read message content. It operates only on envelopes
and filesystem metadata.

## Message-ID Format

Message identifiers follow RFC 5322 §3.6.4: `{hex}@{sender-domain}`.

### Why sender-domain, not smtpd-hostname

Using the sender's domain (e.g. `example.com`) rather than the mail server
hostname (e.g. `mail.infodancer.net`) in the Message-ID domain serves two
goals:

1. **Authentication.** The Message-ID ties the message to the sender's domain,
   not to whoever is currently hosting their mail. This parallels how DKIM
   uses `d=example.com` regardless of which server processed the message.

2. **New-protocol retrieval.** In the store-and-notify delivery model, the
   recipient's server fetches the body by looking up the domain in the
   Message-ID. Using the sender's domain means the lookup resolves via the
   sender domain's DNS, which is under the sender's administrative control.

### Filesystem split

The full msgid (`hex@sender-domain`) is used in the `MSGID` envelope field
and the `Message-ID:` header injected into the body at queue-inject time.
Filesystem paths use only the hex part to avoid confusing filename parsers
that use `@` as a field separator:

```
Message-ID header:   <abc123def456@example.com>
MSGID envelope field: abc123def456@example.com
Body file path:       msg/com/example/abc123def456
Envelope filename:    alice@abc123def456.001
```

### SRV-based retrieval and multi-server deployments

The recipient server retrieves the body by resolving
`_mail._tcp.{sender-domain} SRV` and connecting to the result.

Unlike inbound MX records (where any listed host can independently accept a
new message), a retrieval SRV should resolve to **one canonical retrieval
endpoint** per domain — either a single server, or a hostname behind a load
balancer or cluster VIP. A specific body exists on exactly one server;
multiple SRV targets without coordination would force the recipient to try
all of them to find it.

For large deployments that run multiple outbound servers, acceptable
approaches are:

- A load-balancer VIP as the single SRV target, with sticky routing by
  msgid hash behind it.
- Consistent hashing on the hex part of the msgid to route retrieval
  requests deterministically to the correct node.

For most deployments the SRV target is simply `mail.example.com`, which is
also the smtpd hostname — the distinction only matters at scale.

## VERP

Variable Envelope Return Path is computed at queue-inject time (by smtpd) and
stored in the envelope's `SENDER` field. Format:

```
bounces+{localpart}={recipient_domain}@{bounce_handling_domain}
```

Each envelope carries its own VERP sender. No generation logic in `mail-remote`.

## Encryption Compatibility

Message bodies are written once by smtpd and treated as opaque bytes thereafter.
When at-rest encryption is implemented:

- smtpd encrypts the payload before writing to `msg/`.
- `mail-remote` reads and transmits bytes without decryption.
- The queue manager never reads body content.

No changes to queue structure or delivery binary logic are required when
encryption is enabled.

## Implementation Repositories

| Repo | Purpose |
|------|---------|
| `infodancer/mail-remote`    | Delivery binary: DNS resolution, SMTP, new-protocol notification |
| `infodancer/queue-manager`  | Queue driver: retry scheduling, TTL expiry, batching, rate limiting |

`mail-remote` has no dependency on `queue-manager`. The queue manager invokes
`mail-remote` as a subprocess; `mail-remote` can also be called directly for
manual delivery or testing.
