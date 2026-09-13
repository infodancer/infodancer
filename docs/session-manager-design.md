# Session Manager Design

## Problem

The current architecture spawns mail-session locally via `SysProcAttr.Credential`
for uid/gid isolation. This works on a single host but prevents network-separated
deployments where protocol handlers (smtpd, imapd, pop3d) run on different hosts
from the mail store.

Additionally, the dispatcher logic -- auth signal handling, credential lookup,
mail-session lifecycle management -- is duplicated across all three daemons.

## Decision

Introduce a **session manager** service that centralizes authentication and
per-user mail-session process management. Protocol handlers become pure protocol
translators with no auth or process spawning logic.

Per-user uid isolation is preserved: the session manager spawns individual
mail-session processes with `SysProcAttr.Credential`, exactly as the current
dispatchers do.

## Architecture

```
                         network (gRPC + mTLS)
smtpd ──┐                                      ┌── mail-session (uid=alice)
imapd ──┼──────────────────► session-manager ───┤       unix socket
pop3d ──┘                    (auth + spawn)     └── mail-session (uid=bob)
scmp  ──┘                                              unix socket
```

### Single-host deployment (backward compatible)

Protocol handlers can still spawn mail-session locally when no session manager
is configured. The session manager is optional -- single-host deployments work
exactly as they do today.

### Multi-host deployment

Protocol handlers connect to a remote session manager over gRPC with mTLS.
The session manager runs on the mail storage host(s) alongside the maildirs.

## Session Manager Responsibilities

1. **Authentication** -- accepts credentials from protocol handlers, validates
   against the auth backend (passwd, LDAP, etc.)
2. **Credential lookup** -- resolves uid, gid, basePath, store type from
   per-domain config and passwd files
3. **Process spawning** -- starts mail-session with the correct uid/gid via
   `SysProcAttr.Credential`
4. **Session multiplexing** -- routes gRPC RPCs from protocol handlers to the
   correct per-user mail-session over local unix sockets
5. **Lifecycle management** -- tracks active sessions, idle reaping, connection
   counting
6. **Delivery routing** -- for SMTP delivery, spawns mail-session in oneshot
   mode with the recipient's uid/gid

## gRPC Service Design

### SessionService (new, on the session manager)

```protobuf
service SessionService {
  // Authenticate and establish a session. Returns a session token.
  rpc Login(LoginRequest) returns (LoginResponse);

  // End a session. Triggers mail-session cleanup.
  rpc Logout(LogoutRequest) returns (LogoutResponse);
}
```

### Proxied services

The session manager proxies MailboxService, FolderService, WatchService, and
DeliveryService RPCs to the appropriate per-user mail-session. Each proxied
RPC includes the session token (via gRPC metadata) so the manager can route
to the correct backend.

The session manager proxies all RPCs rather than brokering direct connections.
Direct connect would require per-session network endpoints (port-per-user
doesn't scale) or multiplexing (which is just proxying with extra steps).
Proxying keeps a single network endpoint, centralizes auth/logging/lifecycle,
and gRPC streaming handles the throughput efficiently. If the manager ever
becomes a bottleneck, the fix is sharding by user across multiple instances.

## mTLS for Inter-Host Auth

Protocol handler → session manager connections use mTLS:

- The session manager has a CA that issues client certificates to authorized
  protocol handlers
- Protocol handlers present their client cert on connection
- The session manager verifies the cert against its CA before accepting any RPCs
- This authenticates the *service* (this is a legitimate protocol handler), not
  the *user* (that's what Login() does)

For single-host deployments using unix sockets, mTLS is unnecessary -- socket
file permissions provide access control.

## Process Lifecycle

### Session reuse

When a protocol handler calls Login() for a user who already has an active
mail-session (e.g. multiple IMAP connections), the session manager reuses the
existing mail-session process rather than spawning a new one. Reference counting
tracks how many protocol handler connections are using each session.

### Idle reaping

When the last connection to a mail-session disconnects, an idle timer starts.
If no new connection arrives before the timer fires, the session manager sends
SIGTERM to the mail-session process and cleans up the unix socket.

Default idle timeout: 5 minutes (configurable). This keeps sessions warm for
users who reconnect frequently (e.g. IMAP clients that poll) without leaking
processes for abandoned sessions.

### Delivery (oneshot)

SMTP delivery doesn't go through Login(). Instead, the session manager exposes
the DeliveryService directly. For each delivery, it:

1. Looks up the recipient's uid/gid
2. Spawns mail-session in oneshot mode with those credentials
3. Proxies the Deliver RPC
4. Reaps the process after the RPC completes

## What Moves Where

### Out of protocol handlers (smtpd, imapd, pop3d)

- Authentication (AuthRouter, auth agent setup)
- Credential lookup (lookupCredentials, lookupMailSessionParams)
- mail-session process spawning (spawnGrpcMailSession, dispatchGrpcSession)
- Domain provider setup (for auth routing)

### Into session manager

- All of the above
- Session registry (map of active sessions by user)
- Connection reference counting
- Idle reaper

### Stays in protocol handlers

- Network protocol handling (SMTP, IMAP, POP3 conversations)
- TLS termination for client connections
- Protocol-specific logic (SMTP relay policy, IMAP capability negotiation)
- gRPC client to session manager

## Migration Path

### Phase 1: Extract session manager binary

- New repo: `infodancer/session-manager`
- Consolidate dispatcher logic from smtpd, imapd, pop3d
- Session manager listens on a unix socket (same-host only)
- Protocol handlers configured with `session_manager_socket` instead of
  `mail_session` path
- All three daemons stop spawning mail-session directly

### Phase 2: Network transport

- Add mTLS support to session manager
- Protocol handlers configured with `session_manager_address` (host:port)
  and client certificate paths
- Session manager verifies client certs
- Test multi-host deployments

### Phase 3: Session reuse

- Session registry with reference counting
- Idle reaper with configurable timeout
- Connection migration (client reconnects reuse existing session)

### Phase 4: scmp native client

- scmp connects directly to the session manager (it's already gRPC-native)
- No protocol translation needed -- scmp speaks the same gRPC services

## Security Properties

| Property | How preserved |
|----------|--------------|
| Per-user uid isolation | Session manager spawns mail-session with SysProcAttr.Credential |
| No network process touches mail | Protocol handlers connect to session manager, not to maildirs |
| Auth separated from mail access | Session manager has auth backends; mail-session has maildir access |
| Credential exposure minimized | Protocol handlers never see passwd files; session manager validates and discards |
| Inter-host trust | mTLS ensures only authorized protocol handlers can connect |
| Blast radius of compromise | Protocol handler: no mail access. Session manager: can spawn sessions but doesn't hold mail data. Individual mail-session: one user only. |

## High Availability

The session manager is stateless -- its only state is an in-memory session
registry (which mail-session processes are running, who's connected). This
makes HA straightforward: there is no distributed state to coordinate.

### Process failure

Systemd restart. Session manager comes back with an empty registry. Protocol
handlers reconnect, re-authenticate, get new mail-session instances. SMTP
deliveries tempfail and retry per protocol. For IMAP and POP3 the
re-authentication is performed by the per-connection protocol handler using
its retained credential, so the mail client itself never reconnects -- see
session-recovery-design.md (maildancer#179). No data loss. Same failure
model as any stateless reverse proxy.

### Storage availability

The session manager is pinned to the storage host because mail-session needs
local (or NFS-mounted) filesystem access with uid/gid permissions. Three
deployment models, in order of complexity:

**Domain sharding (no replication)**

```
session-manager-1 (host-a) ── domains: example.com
session-manager-2 (host-b) ── domains: example.org
```

Protocol handlers route by domain. If host-a dies, example.com users are down
until recovery. Simple, scales linearly. Good enough for most self-hosted
deployments with RAID/ZFS underneath.

**Active-passive with shared NFS storage**

```
session-manager (active, host-a)  ──┐
                                    ├── NFS mount ── maildirs
session-manager (standby, host-b) ──┘
```

Maildir is NFS-safe by design: one file per message, no shared index files, no
file locking. Delivery uses `rename()` from `tmp/` to `new/`, which is atomic
on NFS. mail-session's operations -- `readdir()`, `open()/read()`,
`rename()`, `unlink()` -- are all NFS-safe. The only concern is `readdir()`
latency on large mailboxes over NFS, which is a performance issue (mitigated by
session reuse and caching), not a correctness issue.

Failover via keepalived or DNS. The standby session manager starts up and
spawns fresh sessions -- no state to transfer.

**Active-passive with block replication**

DRBD or ZFS send/recv for storage replication. Same failover model as NFS but
with local-disk performance. More operational complexity.

### What not to build

Application-level mail replication (Dovecot dsync style) is extremely complex
and unnecessary. Storage redundancy belongs in the infrastructure layer. The
session manager's statelessness means the application layer doesn't need to
participate in failover -- it just starts up and serves.

## Open Questions

- **Delivery batching**: should the session manager pool oneshot mail-session
  instances for high-volume delivery, or is spawn-per-delivery sufficient?
