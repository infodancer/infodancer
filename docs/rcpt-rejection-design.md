# RCPT Rejection Mode Design
*Last updated: 2026-03-05*

## Problem

smtpd currently rejects unknown recipients at RCPT TO time with `550 User
unknown`. This leaks information: an attacker can enumerate valid addresses by
sending RCPT TO commands and observing which ones return 250 vs 550. Dictionary
attacks against a domain produce a confirmed list of valid mailboxes.

Additionally, rejecting at RCPT TO means the message body is never received for
unknown recipients, so there is no opportunity to feed it to rspamd for Bayes
training. Messages sent exclusively to nonexistent addresses are — with near
certainty — spam, and represent a free, high-confidence training signal that is
currently discarded.

## Design

Add a configurable rejection mode that controls when smtpd rejects messages to
unknown local recipients. Two modes:

| Mode | Behavior |
|------|----------|
| `rcpt` | Reject at RCPT TO (current behavior). Fast, saves bandwidth. Reveals which addresses exist. |
| `data` | Accept RCPT TO with `250 OK` regardless of recipient validity. After DATA, reject the message if all recipients are invalid. |

### Configuration

Global default in `config.toml`:

```toml
[smtpd]
# When to reject messages to unknown recipients.
# "rcpt" = reject at RCPT TO (default, current behavior)
# "data" = accept RCPT TO, reject after DATA (hides valid addresses)
recipient_rejection = "rcpt"
```

Per-domain override in `domains/{domain}/config.toml`:

```toml
[smtpd]
recipient_rejection = "data"
```

The per-domain setting overrides the global default. This allows high-value
domains to use `data` mode while low-value or catch-all domains use `rcpt` mode.

### SMTP Session Flow

#### `rcpt` mode (unchanged)

```
RCPT TO:<bogus@example.com>  →  550 User unknown
```

#### `data` mode

```
RCPT TO:<bogus@example.com>  →  250 OK
RCPT TO:<valid@example.com>  →  250 OK
DATA
[message body]
.
```

After the final `.`, smtpd checks the single recipient:

1. **Recipient invalid** → auto-learn as spam (if conditions met), then
   `550 User unknown`
2. **Recipient valid** → normal delivery

### Spamtrap Auto-Learning

When `data` mode is active and the message has at least one invalid recipient
(who has never existed), smtpd feeds the message body to rspamd as spam training
before rejecting or delivering. This provides automatic Bayes training from
messages that are almost certainly spam.

#### Conditions for auto-learn

All of the following must be true:

- Rejection mode is `data` for the recipient's domain
- At least one recipient address does not exist and has never existed
- rspamd scored the message below the reject threshold (messages already
  rejected by rspamd don't need additional training)
- The message was not sent by an authenticated user (authenticated submission
  is relay, not inbound delivery)

#### Rate limiting

To avoid flooding rspamd's Bayes database with repetitive spam during
dictionary attacks:

- Cap auto-learns at **10 per source IP per hour**
- Deduplicate by rspamd fuzzy hash when available (skip learning if rspamd
  already returned a high fuzzy score, indicating the corpus already contains
  similar messages)

#### "Never existed" check

The auto-learn signal depends on distinguishing "address never existed" from
"address was deleted." A deleted account might still receive legitimate replies
to old threads.

For the initial implementation, the check is simple: `UserExists()` returns
false. This is sufficient because:

- The auth backend (passwd file) has no concept of historical users
- Deleted users are removed from the passwd file entirely
- If historical user tracking is added later, the auto-learn check can be
  refined to exclude recently deleted accounts

### Interaction with Existing Features

#### Single-recipient enforcement

smtpd enforces exactly one recipient per message (`452 One recipient at a
time` after the first RCPT TO). This is a permanent constraint, not subject
to future relaxation. It simplifies `data` mode considerably: every message
has exactly one recipient, so the post-DATA decision is binary — the
recipient is either valid (deliver) or invalid (auto-learn + reject). There
is no mixed valid/invalid case and no silent-drop scenario.

#### Spam checking

The rspamd scan happens at DATA time regardless of rejection mode. In `data`
mode, the scan result is available before the rejection decision:

1. rspamd scans the message
2. If rspamd says reject → reject (regardless of recipient validity)
3. If recipient is invalid → auto-learn as spam (if conditions met),
   then reject
4. Otherwise → deliver normally

#### Authenticated submission

`data` mode only affects inbound delivery. Authenticated senders submitting
mail for relay are not subject to recipient validation at all (remote
recipients are queued, not validated locally). No change needed.

#### Metrics

Add a counter for `recipient_rejection_mode` outcomes:

- `rcpt_rejected_at_rcpt` — rejected during RCPT TO (rcpt mode)
- `rcpt_rejected_at_data` — rejected after DATA (data mode)
- `spamtrap_autolearn_total` — messages auto-learned as spam
- `spamtrap_autolearn_skipped_rate_limit` — skipped due to rate limit
- `spamtrap_autolearn_skipped_already_rejected` — skipped because rspamd
  already rejected

## Implementation Plan

1. Add `RecipientRejection` field to smtpd config and domain config structs
2. In `Rcpt()`: when mode is `data` and domain provider says user unknown,
   accept with 250 but mark the recipient as `pendingValidation`
3. In `Data()`: after rspamd scan, check pending recipients. Reject if all
   invalid; auto-learn if conditions met.
4. Add rate-limit state (per-IP counter with hourly expiry, in-memory map
   with periodic cleanup)
5. Add metrics counters
6. Tests: RCPT-mode unchanged behavior, DATA-mode rejection, auto-learn
   trigger, rate limiting, mixed valid/invalid recipients

## Mandatory Addresses and Forwarding

### RFC-required addresses

RFC 5321 §4.5.1 requires every domain to accept mail for `postmaster@`.
RFC 2142 additionally recommends `abuse@` for all internet-connected domains.
These two addresses are the practical minimum for a mail server.

### Interaction with spamtrap auto-learning

The recipient validation check (`authRouter.UserExists()`) considers an
address existing if it matches **any** of:

1. A real mailbox in the passwd file
2. An explicit forward rule (user-level, domain-level, or system default)
3. A catchall (`*`) forward rule

This means forwarded addresses and catch-all targets are **never** fed to
the spamtrap learner — they are treated as valid recipients. This is correct:
mail that would actually be forwarded and delivered is not a reliable spam
signal.

### Catch-all domains defeat spamtrap

A domain with a catchall forward (`*:someone@domain`) treats every possible
address as existing. The spamtrap auto-learn will never fire for that domain.
Once real user addresses are provisioned, the catchall should be removed to
enable spamtrap learning.

### Recommended domain setup

Every locally-served domain should have at minimum:

```
# In domains/{domain}/forwards
postmaster:admin@example.com
abuse:admin@example.com
```

These forwards ensure RFC compliance without a catchall. All other
addresses that don't match a mailbox or explicit forward are considered
nonexistent — eligible for spamtrap auto-learning in `data` mode.

The **webadmin setup wizard** should create these forwards by default when
a new domain is added, prompting the admin to choose the target mailbox.
The webadmin dashboard should warn if any locally-served domain is missing
`postmaster` or `abuse` destinations.

## Security Considerations

- **data mode accepts more bytes from spammers.** The tradeoff is deliberate:
  we accept the bandwidth cost in exchange for address enumeration defense and
  free training data. The rspamd scan still applies, so reject-level spam is
  still rejected — just after DATA instead of before.
- **Auto-learn poisoning.** A sophisticated attacker could send carefully
  crafted "clean-looking" messages to nonexistent addresses to pollute Bayes
  toward classifying legitimate patterns as spam. The rate limit and
  rspamd-score-below-reject-threshold guard mitigate this. Monitoring the
  `spamtrap_autolearn_total` metric over time will surface anomalies.
- **Single-recipient enforcement eliminates backscatter.** Because there is
  always exactly one recipient, the post-DATA rejection is unambiguous — the
  sending MTA receives the `550` directly and handles it. No bounce generation
  is ever needed.
