# DSN Bounce Generation
*Last updated: 2026-03-11*

## Problem

When a queued message exhausts its TTL and the final delivery attempt fails,
queue-manager silently deletes the envelope. The authenticated local user who
submitted the message receives no indication that delivery failed.

This is a data loss scenario from the sender's perspective.

## Scope

The outbound queue is used exclusively for authenticated submission. Inbound
mail is delivered directly to local mailboxes and never enters the queue.
Every message in the queue has a known local sender. DSNs generated from this
queue are therefore always delivered to a local mailbox — there is no
backscatter risk.

## Decision

queue-manager generates RFC 3464 Delivery Status Notifications on permanent
delivery failure at TTL expiry and delivers them to the sender's local mailbox
via session-manager's DeliveryService gRPC endpoint.

## Design

### mail-remote result reporting

mail-remote currently exits with a status code (0 = success, 75 = temp fail,
69 = perm fail) but does not report per-recipient failure details. DSN
generation requires the SMTP diagnostic code, enhanced status code, and remote
MTA identity.

**Stdout pipe**: mail-remote writes a JSON array of per-recipient results to
stdout before exiting. queue-manager captures stdout via pipe (`cmd.Stdout`
set to a buffer). slog diagnostic output goes to stderr as usual.

```json
[
  {
    "envelope": "/queue/env/com/gmail/alice@abc123.0.delivering",
    "status": "perm_fail",
    "smtp_code": 550,
    "diagnostic": "RCPT TO <alice@gmail.com>: 550 5.1.1 The email account does not exist."
  }
]
```

| Field         | Type   | Description |
|---------------|--------|-------------|
| `envelope`    | string | Envelope file path (as passed to mail-remote) |
| `status`      | string | Outcome: `"delivered"`, `"perm_fail"`, or `"temp_fail"` |
| `smtp_code`   | int    | Three-digit SMTP reply code; 0 if no SMTP response (connection failure) |
| `diagnostic`  | string | Full error string including SMTP reply text |

On success, `status` is `"delivered"` and `smtp_code` is 250. On connection
failure (no SMTP reply available), `smtp_code` is 0 and `diagnostic` contains
the Go error string.

If nothing reads stdout (standalone CLI use), the output is harmlessly
discarded. queue-manager gracefully handles missing or malformed JSON by
falling back to exit-code-only logging.

### DSN generation

queue-manager generates DSNs following RFC 3464. The bounce message is a
`multipart/report; report-type=delivery-status` MIME message with three parts:

```
Content-Type: multipart/report; report-type=delivery-status; boundary=...

--boundary
Content-Type: text/plain; charset=utf-8

[human-readable explanation]

--boundary
Content-Type: message/delivery-status

[per-message and per-recipient fields]

--boundary
Content-Type: text/rfc822-headers

[original message headers]

--boundary--
```

#### Part 1: human-readable explanation (Go text/template)

The human-readable part is rendered from an embedded `text/template`. This
keeps the prose editable without touching generation code.

Template variables:

| Variable          | Type     | Description |
|-------------------|----------|-------------|
| `.Recipient`      | string   | Original recipient address |
| `.Domain`         | string   | Recipient domain |
| `.StatusCode`     | string   | Enhanced status code (e.g., `5.1.1`) |
| `.SMTPCode`       | int      | Three-digit SMTP code |
| `.Diagnostic`     | string   | SMTP reply text |
| `.RemoteMTA`      | string   | MX hostname that rejected |
| `.MessageID`      | string   | Original Message-ID |
| `.QueuedAt`       | string   | When the message entered the queue (envelope `created` field) |
| `.ExpiredAt`      | string   | When TTL expired |
| `.Hostname`       | string   | Reporting MTA hostname |

Default template:

```
This is an automatically generated Delivery Status Notification.

Delivery to the following recipient has failed permanently:

    {{.Recipient}}

Technical details:

    Remote server: {{.RemoteMTA}}
    Response:      {{.Diagnostic}}

The message (ID: {{.MessageID}}) was queued on {{.QueuedAt}} and all delivery
attempts have been exhausted (TTL expired {{.ExpiredAt}}).

No further delivery attempts will be made. If this message was important,
consider contacting the recipient through another channel to verify their
address.

-- {{.Hostname}} mail delivery system
```

The template file is embedded via `//go:embed` from
`internal/dsn/templates/bounce.text.tmpl`. Custom templates can be loaded from
a config-specified path, falling back to the embedded default.

#### Part 2: message/delivery-status (programmatic)

This part is generated from structured data, not templated. The RFC defines
the exact field names and format.

Per-message fields:

```
Reporting-MTA: dns; mail.example.com
Arrival-Date: Thu, 04 Mar 2026 10:00:00 +0000       (from envelope "created" field)
```

Per-recipient fields (one block per failed recipient):

```
Final-Recipient: rfc822; alice@gmail.com
Action: failed
Status: 5.1.1
Diagnostic-Code: smtp; 550 5.1.1 The email account does not exist.
Remote-MTA: dns; gmail-smtp-in.l.google.com
Last-Attempt-Date: Thu, 11 Mar 2026 10:00:00 +0000
```

#### Part 3: original message headers

`text/rfc822-headers` containing the headers from the original message body.
queue-manager reads headers from the body file up to the first blank line.
The body content is not included (it may be encrypted, and headers are
sufficient for identification).

### Why not HTML

DSNs are diagnostic messages, not correspondence. Plain text is:

- Required by RFC 3464 (the `text/plain` part is mandatory).
- Sufficient for the content (addresses, error codes, timestamps).
- What MUAs with DSN-aware rendering expect and parse.

An `alternative` HTML part would add template complexity, a second template to
maintain, and MIME nesting depth — for no functional benefit. If a future need
arises (e.g., branded bounce pages for multi-tenant hosting), it can be added
as an optional `multipart/alternative` wrapping part 1, without changing the
rest of the structure.

### DSN delivery via session-manager

The generated DSN is delivered to the original sender's local mailbox via
session-manager's `DeliveryService.Deliver` gRPC endpoint.

```
queue-manager                       session-manager
     │                                    │
     │ Login(user, domain, password)      │
     │ ──────────────────────────────────►│
     │ ◄────────────── session_token ─────│
     │                                    │
     │ Deliver(metadata + body chunks)    │
     │   metadata:                        │
     │     sender: ""  (null sender)      │
     │     recipient: user@domain         │
     │     client_ip: "127.0.0.1"         │
     │ ──────────────────────────────────►│
     │ ◄──────── DELIVER_RESULT_* ────────│
     │                                    │
     │ Logout(session_token)              │
     │ ──────────────────────────────────►│
```

Key details:

- **Null sender**: the DSN uses `MAIL FROM:<>` (empty string in the `sender`
  field). This is mandated by RFC 3461 §5.2.1 to prevent bounce loops.
- **Recipient**: the original submitter, read from the envelope `ORIGIN` field
  (see [Envelope ORIGIN Field](#envelope-origin-field)).
- **Authentication**: queue-manager authenticates to session-manager using a
  service credential (see [Configuration](#configuration)). This is a system
  account, not the sending user's credentials.

If delivery of the DSN itself fails (session-manager down, user mailbox full),
queue-manager logs the failure and discards the DSN. DSNs are never re-queued
— generating a bounce for a bounce is an infinite loop.

### Envelope `origin` field

The DSN recipient is read from the envelope `origin` field — the authenticated
submitter's address before VERP rewriting. This field is part of the JSON
envelope format defined in [queue-design.md](queue-design.md#envelope-file).
Envelopes missing `origin` (pre-migration) skip DSN generation with a log
warning.

### Delivery flow (updated)

DSNs are generated on any permanent delivery failure — both TTL expiry and
mid-queue 5xx rejections. Results are always captured via stdout pipe.

```
queue-manager scan pass
  │
  ├─ envelope TTL expired?
  │    ├─ invoke mail-remote --final (capture stdout)
  │    └─ delete envelope regardless of outcome
  │
  ├─ envelope ready for retry?
  │    └─ invoke mail-remote (capture stdout)
  │
  ├─ parse JSON results from stdout
  ├─ log per-recipient delivery outcomes
  ├─ log domain summary (delivered/perm_fail/temp_fail/deferred)
  ├─ for each recipient with permanent failure (5xx):
  │    ├─ read ORIGIN from envelope
  │    ├─ read original message headers from body file
  │    ├─ render DSN from template + result data
  │    └─ deliver DSN via session-manager DeliveryService
```

For TTL expiry, the envelope is deleted unconditionally (existing behavior).
For mid-queue permanent failures, mail-remote already deletes the envelope
(exit code 69). In both cases, the DSN is generated after mail-remote exits,
using the results captured from stdout.

## Configuration

New fields in the shared TOML config `[queue-manager]` section:

```toml
[queue-manager]
hostname = "mail.example.com"     # Reporting-MTA for DSN headers

[queue-manager.dsn]
enabled = true                    # default: true
bounce_template = ""              # path to custom bounce.text.tmpl; empty = embedded default

[queue-manager.session-manager]
socket = "/run/session-manager/session-manager.sock"   # gRPC endpoint
service_user = "queue-manager"                         # service account for auth
service_domain = "mail.example.com"                    # domain for service account
```

The service account `queue-manager` must exist in the auth backend with
permission to deliver to any local mailbox. This is the same pattern used by
smtpd's delivery path.

## Implementation Plan

### Phase 0: JSON envelope migration (prerequisite, separate work)

- smtpd: write JSON envelopes with `origin` field at queue-inject time
- mail-remote: replace line-based envelope parsing with `json.Unmarshal`
- queue-manager: replace `parseTTL` with `json.Unmarshal` into envelope struct
- No deployed queue to migrate — all repos change simultaneously

### Phase 1: mail-remote result reporting

- Write per-recipient JSON results to stdout (always; harmlessly discarded if unread)
- Define the `recipientResult` JSON struct: envelope, status, smtp_code, diagnostic
- Add `SMTPCode()` helper to extract SMTP reply code from error chain
- queue-manager captures stdout via pipe, logs results, tallies domain summaries

### Phase 2: DSN generation in queue-manager

- `internal/dsn/` package: RFC 3464 message builder, template rendering
- Embedded default template via `//go:embed`
- Read original message headers from body file
- TOML config for hostname and template path

### Phase 3: session-manager delivery

- gRPC client in queue-manager for DeliveryService
- Service account authentication via session-manager Login
- Integration into the delivery flow (both TTL expiry and mid-queue permanent failures)

## Affected Repositories

| Repo | Changes |
|------|---------|
| `infodancer/smtpd`          | JSON envelope writer with `origin` field (Phase 0) |
| `infodancer/mail-remote`    | JSON envelope reader, result file output (Phases 0-1) |
| `infodancer/queue-manager`  | JSON envelope reader, DSN generation, session-manager client (Phases 0-3) |
| `infodancer/infodancer`     | This design doc; queue-design.md envelope format updated |
