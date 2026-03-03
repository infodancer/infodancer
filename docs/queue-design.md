# Mail Queue Design
*Last updated: 2026-03-03*

## Overview

A unified on-disk queue that supports three delivery modes — SMTP smarthost,
SMTP direct, and new-protocol retrieve-from-sender — without forking the queue
structure. A separate delivery binary handles all three modes; the queue manager
drives retry and expiry.

## Directory Structure

```
queue/
  msg/{tld}/{domain}/{msgid}                   # message body (immutable, opaque bytes)
  env/{tld}/{domain}/{localpart}@{msgid}.{n}   # one envelope file per recipient
```

Bodies are keyed by **sender domain**; envelopes are keyed by **recipient domain**.
Both split first by TLD, then by the registrable domain portion.

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

`MSGID` is a cryptographically random identifier, minimum 128 bits, base62 or
hex encoded. Sequential or predictable IDs are not acceptable.

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

## Delivery Binary: `remote-deliver`

A standalone command-line binary. Can be invoked by the queue manager or by
hand for a single envelope:

```sh
remote-deliver queue/env/com/gmail/alice@abc123def456789.001
```

It reads the envelope, locates the body via MSGID, and delivers. DNS lookup
order per recipient domain:

1. **SRV** `_mail._tcp.{recipient_domain}` → found: send new-protocol
   notification (store-and-notify model). Done.
2. No SRV → **MX** lookup → SMTP delivery to MX host.
3. No MX → **A** record → SMTP delivery to port 25.

New-protocol and SMTP are co-delivery systems, not a fork in queue structure.
On every retry attempt, DNS is queried fresh; a domain that gains an SRV record
between retries will automatically receive new-protocol delivery without any
queue reconfiguration.

On success: exits 0. Caller deletes the envelope file.
On temporary failure: exits with a temp-fail code. Caller updates env mtime
(triggering backoff).
On permanent failure: exits with a perm-fail code. Caller deletes the envelope.

## Queue Manager: `queue-manager`

Drives the retry loop. Responsibilities:

- Scan `env/` for envelopes whose backoff interval has elapsed.
- Batch envelopes by recipient domain: when multiple envelopes under the same
  domain directory are ready, invoke `remote-deliver` for each within a single
  delivery session where the protocol supports batching (SMTP pipelines multiple
  recipients per connection; new-protocol sends one notification per message).
- On TTL expiry: invoke `remote-deliver` for final SMTP attempt, then delete
  envelope regardless of outcome.
- Sweep `msg/` for expired bodies and delete them.
- Rate-limit per destination domain: count in-flight envelopes per domain
  directory; pause new attempts when the limit is reached.

The queue manager does not read message content. It operates only on envelopes
and filesystem metadata.

## VERP

Variable Envelope Return Path is computed at queue-inject time (by smtpd) and
stored in the envelope's `SENDER` field. Format:

```
bounces+{localpart}={recipient_domain}@{bounce_handling_domain}
```

Each envelope carries its own VERP sender. No generation logic in `remote-deliver`.

## Encryption Compatibility

Message bodies are written once by smtpd and treated as opaque bytes thereafter.
When at-rest encryption is implemented:

- smtpd encrypts the payload before writing to `msg/`.
- `remote-deliver` reads and transmits bytes without decryption.
- The queue manager never reads body content.

No changes to queue structure or delivery binary logic are required when
encryption is enabled.

## Implementation Repositories

| Repo | Purpose |
|------|---------|
| `infodancer/remote-deliver` | Delivery binary: DNS resolution, SMTP, new-protocol notification |
| `infodancer/queue-manager`  | Queue driver: retry scheduling, TTL expiry, batching, rate limiting |

`remote-deliver` has no dependency on `queue-manager`. The queue manager
invokes `remote-deliver` as a subprocess, or `remote-deliver` is called
directly for manual delivery or testing.
