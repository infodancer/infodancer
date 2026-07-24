# Mail Server Security Model

This document defines the privilege and process separation model for the
infodancer mail server stack (smtpd, pop3d, imapd, mail-session).

## Design Principles

- No network-facing process ever holds filesystem access to mail data
- No single process ever holds credentials for more than one user simultaneously
- Privilege is acquired at the last possible moment and held for the shortest possible time
- Each process does one job with the minimum privilege required for that job
- The trust boundary between processes is explicit and versioned

This model is inspired by qmail's process separation architecture.

## Process Hierarchy

All three daemons conform to the fork-per-connection model: smtpd, imapd
(maildancer PR #187), and pop3d (PR #190) each fork a `protocol-handler`
subprocess per accepted connection
(https://github.com/infodancer/maildancer/issues/179), built on the shared
`internal/connfork` dispatcher.

### SMTP delivery

```
smtpd serve (dispatcher, nonroot, binds no privileged ports)
  │
  └── forks smtpd protocol-handler per connection (nonroot, one SMTP session)
        │
        │  delivery via gRPC to session-manager (unix socket from config)
        │
        └── session-manager spawns mail-session --mode=oneshot
              (uid=recipient, gid=domain)
              │
              ├── rspamd/spam check (per-user/per-domain config)
              ├── sieve filtering
              └── writes maildir, exits
```

### POP3 / IMAP retrieval

```
pop3d serve / imapd serve (dispatcher, nonroot, binds no privileged ports)
  │
  └── forks pop3d protocol-handler / imapd protocol-handler per connection
        │
        │  auth + mailbox RPCs via gRPC to session-manager
        │  (unix socket named in the shared config)
        │
        └── session-manager spawns mail-session --mode=daemon
              (uid=user, gid=domain)
              │
              └── long-lived process, handles LIST/RETR/DELE or IMAP commands
                  via gRPC for the session duration; idle-reaped
```

### Port binding

All listeners run as nonroot (container uid 65532). Port mapping is handled
entirely by Docker -- the container listens on unprivileged ports and the host
maps them to the standard mail ports (25, 465, 587, 110, 995, 143, 993).
No `CAP_NET_BIND_SERVICE` or root is required.

## Uid/Gid Model

### Allocation

- Uids and gids are allocated from a global monotonic counter managed by webadmin
- The counter is stored in the data root and updated atomically (write + rename)
- Uids and gids are never reused after deletion
- There is no separation of ranges between domain gids and user uids -- the
  counter is shared and values are large enough that artificial partitioning
  would only add complexity

### Assignment

- Each **domain** is assigned one gid at creation time, stored in the domain's
  `config.toml`
- Each **user** is assigned one uid at creation time, stored in the domain's
  `passwd` file

### passwd file format

```
username:argon2id-hash:mailbox:uid
```

The domain gid is not stored per-user -- it is read from the domain's
`config.toml`. The gid is set on the spawned process by the dispatcher using
the domain config, not from the passwd entry.

### Directory permissions

```
domains/                              drwx--x--x  root:root
domains/{domain}/                     drwxr-s---  root:{domain-gid}
domains/{domain}/users/               drwxr-s---  root:{domain-gid}
domains/{domain}/users/{user}/        drwx------  {user-uid}:{domain-gid}
```

The setgid bit on domain and users directories ensures new files inherit the
domain gid. User maildirs are `700` -- only the user uid can read them.

### Config-tree permissions

The read-only config tree (domain config.toml, passwd, uid/gid maps, forwards,
legacy flat keys) has a second, independent nonroot consumer beyond the mail
stack: **auth-oidc**, the leaf IdP, reads each domain's `config.toml` and
`passwd` as the distroless nonroot uid (65532) over a read-only mount. The
model:

```
config/                               drwxr-s---  webadmin:cfgread
config/gid.toml                       -rw-r-----  webadmin:cfgread
config/{domain}/                      drwxr-s---  webadmin:cfgread
config/{domain}/config.toml           -rw-r-----  webadmin:cfgread
config/{domain}/passwd                -rw-r-----  webadmin:cfgread
config/{domain}/uid.toml              -rw-r-----  webadmin:cfgread
config/{domain}/keys/                 drwxr-s---  webadmin:cfgread  (files 0640)
```

The webadmin service account (uid 905 in the all-in-one image) owns the tree
because it is the writer; owning its writes is what lets it eventually run
unprivileged. cfgread (gid 906) is a dedicated read group: membership is an
explicit grant -- auth-oidc (distroless nonroot 65532) joins via compose
`group_add`, queue-manager's account via image group membership -- and is
deliberately NOT the distroless-nonroot gid, so merely running as distroless
nonroot conveys nothing (maildancer#152; the original #145 model used
root:65532 and is superseded). No world bits, so passwd stays readable only
by the owner, root, and the read group. The setgid bit on the directories
makes every file any writer creates later -- temp+rename saves, root-run
userctl, the id allocator -- inherit the group without cooperation from the
write sites; admin mutations additionally re-assert ownership on success.
Enforced at domain creation and by the fix-perms doctor (maildancer
`internal/admin/perms.go`, issues maildancer#145 + #152).

mail-session (recipient uid) is deliberately **not** granted config-tree
access: it cannot traverse these directories, and its domain loading degrades
to defaults by design. Forwards resolve upstream in root-side session-manager;
per-user keyrings live in the data tree beside the maildir. Do not "fix" a
mail-session config-tree permission error by widening this model.

**Invariant: no unsealed private key material in the config tree.** The tree
is group-readable by design, so anything placed there is readable by every
member of the read group. Private keys under `config/{domain}/keys/` are
acceptable only because they are password-sealed (keyseal); an unsealed key
written there is leaked to the read group the moment it lands. Unsealed
secrets belong in the data tree under a 0700 per-user directory, or outside
both trees entirely.

auth-oidc runs as nonroot deliberately (it is the internet-facing IdP); a
domain it cannot load at startup is served fail-closed rather than aborting
init, and init fails only when no domain loads at all.

## Inter-Process Communication

### gRPC transport (protocol handler ↔ mail-session)

All communication between protocol handlers and mail-session uses protobuf/gRPC
over unix domain sockets. mail-session exposes four gRPC services:

- **MailboxService** -- message retrieval and management (List, Stat, Fetch,
  Append, Copy, Move, SetFlags, Expunge, Rescan, Delete, Undelete, Commit)
- **FolderService** -- folder management (ListFolders, CreateFolder,
  DeleteFolder, RenameFolder)
- **DeliveryService** -- inbound delivery with structured results (replaces
  mail-deliver)
- **WatchService** -- server-streaming notifications for IMAP IDLE

Protocol handlers do not talk to mail-session directly at spawn time: they
dial **session-manager** on the unix socket named in the shared config.
session-manager authenticates the user (auth library), spawns mail-session
under the user's credentials, and proxies the mailbox RPCs for the session
(see session-manager-design.md).

RPCs are stateless -- each request includes the folder name. The gRPC server
calls `sess.Select()` before each folder-scoped operation. IMAP's stateful
SELECT is handled by the imapd protocol translator.

### Authentication (protocol handler → session-manager)

- The protocol handler passes the client's credentials over gRPC
  (`SessionService.Login`) to session-manager and receives a session token
- session-manager performs the actual credential check via the auth library,
  looks up the user's uid and domain gid, and spawns
  `mail-session --mode=daemon` with those credentials
- The protocol handler never sees uids, password hashes, or the mail store;
  a compromised handler holds at most one client's submitted credentials

### fd layout in protocol-handler subprocesses

```
fd 3  TCP socket (from the dispatcher's accept)
fd 4  write-only: metrics report → dispatcher (present when metrics are
      enabled; the handler ships its per-session series here at exit, and
      the dispatcher aggregates them -- maildancer#188)
```

The metrics pipe is one-way by construction: the child holds only the write
end, so nothing can flow back into the possibly-lower-privileged handler, and
the parent bounds its read (64 KiB) so a compromised child cannot drive
unbounded allocation in the dispatcher.

## Process Responsibilities

### smtpd / pop3d / imapd (dispatcher, `<daemon> serve`)

- Bind sockets (unprivileged ports via Docker mapping)
- Fork one protocol-handler subprocess per accepted connection
  (`protocol-handler` subcommand), passing the connection as fd 3 and
  optionally dropping to `handler_uid`/`handler_gid` credentials
- Maintain connection counters and aggregate handler metrics reports (fd 4)
- Never speak the mail protocol, never load TLS private keys, never touch
  mail data directly

### smtpd / pop3d / imapd protocol-handler

- Handle the network conversation with the remote client; terminate TLS
- Serve exactly one session, then exit
- Validate recipient existence (SMTP 550) and relay policy
- Do NOT resolve or handle uids -- authentication and credential lookup live
  in session-manager
- Do NOT access mail data directly
- For SMTP: deliver messages via gRPC DeliveryService through session-manager
- For POP3/IMAP: authenticate and access the mailbox via gRPC through
  session-manager (SessionService, MailboxService, FolderService)

### session-manager

- Owns the auth boundary: authenticates users via the auth library, holds
  the only access to passwd data on the retrieval path
- Spawns mail-session with `SysProcAttr.Credential` (uid=user, gid=domain);
  ref-counts and idle-reaps sessions
- Proxies mailbox/folder/delivery/watch RPCs between protocol handlers and
  mail-session

### mail-session

- Spawned by session-manager as `uid=user, gid=domain`
- Operates in two modes:
  - **daemon** (POP3/IMAP): long-lived, serves gRPC for the authenticated
    session duration; idle-reaped after configurable timeout
  - **oneshot** (SMTP delivery): handles one delivery via gRPC DeliveryService,
    then exits
- Handles maildir operations: list, fetch, delete, flag, append, move, copy
- Handles inbound delivery: rspamd check, sieve filtering, maildir write
- Writes `READY\n` to stdout when the gRPC socket is listening
- Exits when idle timeout fires or the client disconnects

### webadmin (infodancer/webadmin)

- Runs as container nonroot uid for all read operations
- Domain and user creation requires writing to the data directory; these
  operations run as a privileged admin uid (or root) within the container,
  scoped to the specific domain being modified
- Allocates uids and gids via the monotonic counter
- Stores uid in passwd entry and gid in domain config at creation time
- Creates directory trees with correct ownership and permissions at domain/user
  creation time

## Spam and Content Filtering

Spam checking and content filtering occur inside `mail-session` (oneshot
delivery mode), after privilege drop to the recipient uid. This placement:

- Allows per-user spam configuration (threshold, allowlists, blocklists)
- Allows per-domain spam configuration as a fallback
- Ensures a compromised protocol handler cannot suppress spam scanning
- Keeps `mail-session` as the single policy enforcement point for inbound mail

Configuration lookup order:
1. User-level config in `domains/{domain}/users/{user}/spam.toml`
2. Domain-level config in `domains/{domain}/spam.toml`
3. Global defaults in the shared `config.toml`

## Security Properties

| Threat | Mitigation |
|--------|-----------|
| Compromised SMTP protocol handler reads mail | Protocol handler has no filesystem access to mail data |
| Compromised delivery process reads other users' mail | Each delivery runs as the recipient uid; other users' dirs are `700` |
| Cross-domain mail access | Domain dirs are `drwxr-s---` with domain gid; users outside the domain have no gid membership |
| Sieve script escapes to filesystem | Sieve runs inside mail-session already dropped to user uid; can only write to that user's maildir |
| Spam filter bypass via protocol manipulation | Spam check runs in mail-session after the gRPC boundary; protocol handler cannot influence it |
| Uid reuse after user deletion | Monotonic counter never decrements; deleted uids are never reassigned |

## Code Map

All of the below live in the `infodancer/maildancer` monorepo:

| Path | Role |
|------|------|
| `internal/connfork` | Shared fork-per-connection dispatcher: accept, fd-3 handoff, credential drop, metrics report pipe, reaping |
| `internal/procmetrics` | Handler-metrics transport: child report writer, parent-side aggregation |
| `internal/smtpd`, `cmd/smtpd` | SMTP dispatcher + protocol-handler subcommand |
| `internal/pop3d`, `cmd/pop3d` | POP3 dispatcher + protocol-handler subcommand |
| `internal/imapd`, `cmd/imapd` | IMAP dispatcher + protocol-handler subcommand |
| `internal/session-manager` | Auth boundary, mail-session lifecycle, RPC proxy |
| `internal/mail-session` | gRPC mailbox service: delivery, retrieval, folder management |
| `internal/webadmin` | Admin UI: domain/user management, uid allocation |
| `auth` | Authentication: passwd backend, argon2id hashing |
| `msgstore` | Storage abstraction (used by mail-session) |

## Implementation Notes

- The `protocol-handler` subcommand pattern means one binary per daemon,
  two execution modes: `<daemon> serve` runs the dispatcher; the dispatcher
  invokes the same binary as `<daemon> protocol-handler` per connection.
- Uid/gid are set on spawned processes via `syscall.SysProcAttr.Credential`
  in Go -- this sets uid/gid on the child before any code runs.
- The monotonic uid counter file must be updated atomically: write to a temp
  file, then `os.Rename` -- rename is atomic on Linux.
- The passwd file format adds a `uid` field to the existing
  `username:hash:mailbox` format. Existing entries without a uid field are
  treated as not yet migrated; webadmin assigns uids on next edit.
