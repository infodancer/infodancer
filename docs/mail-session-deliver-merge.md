# Merging mail-deliver into mail-session

## Status

**Implemented** -- PR [mail-session#14](https://github.com/infodancer/mail-session/pull/14).
Supersedes the original pipe-command proposal with a gRPC service design.

## Problem

mail-deliver and mail-session are separate binaries that share the same
privilege model (uid=user, gid=domain), the same dependencies (msgstore,
auth/domain, rspamd), and the same filesystem access. They exist as two
repos, two binaries, and two wire protocols. This doubles build, deploy,
and version-sync overhead for code that operates in the same security
context.

Additionally, smtpd's `ExecDeliveryAgent` uses `CombinedOutput()` and
never parses mail-deliver's JSON response -- forwarding redirects are
silently lost.

## Design: gRPC on Unix Socket

Instead of extending the pipe protocol, mail-session now supports a
protobuf/gRPC service mode that unlocks horizontal scaling and makes
IMAP/POP3/SMTP protocol translators rather than core services.

### Architecture

```
scmp (native gRPC) ──┐
imapd (IMAP→gRPC) ───┤
pop3d (POP3→gRPC) ───┼──► mail-session (gRPC on unix socket)
smtpd (SMTP→gRPC) ───┘         │
                           ┌────┴────┐
                           │ maildir │
                           └─────────┘
```

### Operating Modes

mail-session supports three operating modes via `--mode`:

- **`pipe`** (default): Legacy stdin/stdout pipe protocol. Backward compatible.
- **`daemon`**: Long-lived gRPC server on unix socket. For IMAP/POP3 sessions.
  Default idle timeout: 30 minutes.
- **`oneshot`**: Single-delivery gRPC server. For SMTP delivery, exits after
  idle timeout. Default: 60 seconds.

### gRPC Services

Four protobuf services in `proto/mailsession/v1/`, aligned with scmp's split:

**MailboxService** -- message retrieval and management:
- `List(folder)` / `Stat(folder)` -- stateless, folder in every request
- `Fetch(folder, uid)` -- server-streaming, 64KB chunks
- `FetchHeaders(folder, uid)` -- headers + optional body lines
- `Append(stream)` -- client-streaming: metadata + body chunks
- `Copy` / `Move` / `SetFlags` / `Expunge` / `Rescan` / `UIDValidity`
- `Delete` / `Undelete` / `Commit` -- POP3 path

**FolderService** -- folder CRUD (mirrors scmp's FolderService):
- `ListFolders` / `CreateFolder` / `DeleteFolder` / `RenameFolder`

**DeliveryService** -- inbound delivery (replaces mail-deliver):
- `Deliver(stream)` -- client-streaming: metadata + body chunks
- Response: `DELIVERED | REJECTED | REDIRECTED` + temporary + reason + addresses
- Runs the full 5-stage pipeline: forwarding, size, spam, sieve, maildir

**WatchService** -- server-streaming notifications (for IMAP IDLE):
- `Watch(folder)` -- pushes NewMessages, Expunged, FlagsChanged events

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Stateless RPCs (folder per request) | gRPC-idiomatic, scales for future multi-user service |
| Client-streaming upload, server-streaming download | Avoids 4MB default gRPC limit |
| Mutex serialization in gRPC server | session.Session is not thread-safe; ops are short + I/O-bound |
| Unix domain socket + 0600 perms | Same security boundary as pipe protocol |
| `READY\n` on stdout | Simple readiness signal; parent reads before connecting |
| Pipe protocol kept as default | Zero-risk migration; callers opt in with config flag |
| Separate check and learn rspamd clients | Different APIs; merged into internal/deliver and internal/rspamd |

### Client Library

`client/` is a public Go package that callers import:

- `Client` implements `msgstore.MessageStore` and `msgstore.FolderStore` --
  drop-in replacement for SubprocessStore
- `DeliveryClient` provides structured delivery results including redirect
  addresses (fixes the smtpd bug)

### Delivery Pipeline

Ported from mail-deliver's `internal/deliver/deliver.go`:

1. **Forwarding resolution** -- 1-hop limit via `forwarded` flag
2. **Per-domain size check** -- domain config can set tighter limit
3. **Spam check** -- rspamd /checkv2 with 3-level config merge
   (global → domain spam.toml → user spam.toml)
4. **Sieve script** -- parsed but not yet executed (fail-safe)
5. **Maildir delivery** -- via `dom.DeliveryAgent.Deliver()`

Path traversal protection on recipient addresses. Encryption seam preserved.

## Security Model Impact

No change to security boundaries:

| Property | Before | After |
|----------|--------|-------|
| Delivery privilege | uid=recipient, gid=domain | uid=recipient, gid=domain |
| Session privilege | uid=user, gid=domain | uid=user, gid=domain |
| Privilege drop | Caller sets SysProcAttr | Caller sets SysProcAttr |
| Binary count | 2 (mail-deliver + mail-session) | 1 (mail-session) |
| Protocol | JSON (deliver) + pipe (session) | gRPC (both) + pipe (legacy) |
| Spam enforcement | In mail-deliver after trust boundary | In mail-session after trust boundary |

## Migration Plan

### Phase 1-3: Done (mail-session PR #14)
- Proto definitions, gRPC server, delivery pipeline port, client library
- Pipe protocol preserved as default

### Phase 4: Caller Migration (separate repos)
- **imapd**: `GrpcStore` using `client.Client`, config `mail_session_mode = "grpc"`
- **smtpd**: `GrpcDeliveryAgent` using `client.DeliveryClient`, config `delivery_mode = "grpc"`
- **pop3d**: Similar to imapd (lower priority)

### Phase 5: Cleanup
- Archive mail-deliver repo (read-only)
- Remove ExecDeliveryAgent from smtpd
- Remove pipe protocol from mail-session
- Update mail-security-model.md
