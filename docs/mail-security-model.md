# Mail Server Security Model

This document defines the privilege and process separation model for the
infodancer mail server stack (smtpd, pop3d, imapd, mail-deliver, mail-session).

## Design Principles

- No network-facing process ever holds filesystem access to mail data
- No single process ever holds credentials for more than one user simultaneously
- Privilege is acquired at the last possible moment and held for the shortest possible time
- Each process does one job with the minimum privilege required for that job
- The trust boundary between processes is explicit and versioned

This model is inspired by qmail's process separation architecture.

## Process Hierarchy

### SMTP delivery

```
smtpd (nonroot, binds no privileged ports)
  │
  └── forks smtpd --protocol-handler (nonroot, handles SMTP conversation)
        │
        │  validated envelope + message over pipe (see Pipe Protocol below)
        │
        └── smtpd (parent dispatcher) forks mail-deliver (uid=recipient, gid=domain)
              │
              ├── rspamd/spam check (per-user/per-domain config)
              ├── sieve filtering
              └── writes maildir, exits
```

### POP3 / IMAP retrieval

```
pop3d / imapd (nonroot, binds no privileged ports)
  │
  └── forks pop3d --protocol-handler / imapd --protocol-handler (nonroot)
        │
        │  auth success signal + authenticated user identity over pipe
        │
        └── parent dispatcher forks mail-session (uid=user, gid=domain)
              │
              └── long-lived process, handles LIST/RETR/DELE or IMAP commands
                  over pipe back to protocol handler for the session duration
```

### Port binding

All listeners run as nonroot (container uid 65532). Port mapping is handled
entirely by Docker — the container listens on unprivileged ports and the host
maps them to the standard mail ports (25, 465, 587, 110, 995, 143, 993).
No `CAP_NET_BIND_SERVICE` or root is required.

## Uid/Gid Model

### Allocation

- Uids and gids are allocated from a global monotonic counter managed by webadmin
- The counter is stored in the data root and updated atomically (write + rename)
- Uids and gids are never reused after deletion
- There is no separation of ranges between domain gids and user uids — the
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

The domain gid is not stored per-user — it is read from the domain's
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
domain gid. User maildirs are `700` — only the user uid can read them.

## Pipe Protocol

The pipe between the protocol handler and the parent dispatcher is a simple
line-oriented text protocol. This is a trust boundary; the dispatcher must
treat all input as untrusted.

### Envelope format (smtpd-protocol → dispatcher)

```
ENVELOPE 1\r\n
FROM:<sender@example.com>\r\n
TO:<recipient@domain.com>\r\n
TO:<recipient2@domain2.com>\r\n
END\r\n
<raw RFC 5321 message bytes>
<EOF>
```

- `ENVELOPE` line includes a version number (currently `1`)
- Multiple `TO` lines for multiple recipients
- The dispatcher performs uid/gid lookup for each recipient — the protocol
  handler only validates that recipients exist (for 550 responses during SMTP),
  it never sees uids
- The raw message follows immediately after `END\r\n` and is terminated by EOF

### Auth signal format (pop3d/imapd-protocol → dispatcher)

```
AUTH 1\r\n
USER:<localpart@domain>\r\n
END\r\n
```

- Sent once, after the protocol handler has successfully authenticated the user
- The dispatcher looks up the uid/gid for the authenticated user and forks
  `mail-session` with those credentials
- The protocol handler then communicates with `mail-session` for the remainder
  of the session via the dispatcher-managed pipes

## Process Responsibilities

### smtpd / pop3d / imapd (listener)

- Bind sockets (unprivileged ports via Docker mapping)
- Fork protocol handler subprocesses (`--protocol-handler` subcommand)
- Receive completed envelopes or auth signals from protocol handlers over pipe
- Look up recipient/user uid and domain gid from domain config and passwd
- Fork `mail-deliver` or `mail-session` with the correct credentials
- Never touch mail data directly

### smtpd --protocol-handler / pop3d --protocol-handler / imapd --protocol-handler

- Handle the network conversation with the remote client
- Validate recipient existence (SMTP 550) and relay policy
- Do NOT resolve or handle uids — pass only addresses to the dispatcher
- Do NOT access mail data directly
- Write completed envelope to parent over pipe; read delivery result

### mail-deliver (own repo: infodancer/mail-deliver)

- Spawned by dispatcher as `uid=recipient-user, gid=domain`
- Receives envelope and raw message on stdin
- Loads per-user and per-domain spam/AV configuration
- Runs rspamd check with that configuration
- Runs user's sieve script (if present)
- Writes message to user's maildir
- Exits 0 on success, non-zero on failure with a reason code on stderr
- Never opens a network connection except to rspamd

### mail-session (own repo: infodancer/mail-session)

- Spawned by dispatcher as `uid=user, gid=domain` after successful auth
- Long-lived for the duration of the authenticated session
- Communicates with the protocol handler over pipes managed by the dispatcher
- Handles maildir operations: list, fetch, delete, flag
- Implements the storage side of the retrieval protocol
- Exits when the session ends or the pipe closes

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

Spam checking and content filtering occur inside `mail-deliver`, after privilege
drop to the recipient uid. This placement:

- Allows per-user spam configuration (threshold, allowlists, blocklists)
- Allows per-domain spam configuration as a fallback
- Ensures a compromised protocol handler cannot suppress spam scanning
- Keeps `mail-deliver` as the single policy enforcement point for inbound mail

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
| Sieve script escapes to filesystem | Sieve runs inside mail-deliver already dropped to user uid; can only write to that user's maildir |
| Spam filter bypass via protocol manipulation | Spam check runs in mail-deliver after the protocol boundary; protocol handler cannot influence it |
| Uid reuse after user deletion | Monotonic counter never decrements; deleted uids are never reassigned |

## Repository Map

| Repo | Role |
|------|------|
| `infodancer/smtpd` | SMTP listener + protocol handler subcommand |
| `infodancer/pop3d` | POP3 listener + protocol handler subcommand |
| `infodancer/imapd` | IMAP listener + protocol handler subcommand |
| `infodancer/mail-deliver` | Delivery agent: spam, sieve, maildir write |
| `infodancer/mail-session` | Retrieval agent: maildir read for authenticated sessions |
| `infodancer/webadmin` | Admin UI: domain/user management, uid allocation |
| `infodancer/auth` | Authentication: passwd backend, argon2id hashing |
| `infodancer/msgstore` | Storage abstraction (used by mail-session) |

## Implementation Notes

- The `--protocol-handler` subcommand pattern means one binary per daemon,
  two execution modes. The listener detects it was invoked with
  `--protocol-handler` and enters the protocol handler code path directly.
- Uid/gid are set on spawned processes via `syscall.SysProcAttr.Credential`
  in Go — this sets uid/gid on the child before any code runs.
- The monotonic uid counter file must be updated atomically: write to a temp
  file, then `os.Rename` — rename is atomic on Linux.
- The passwd file format adds a `uid` field to the existing
  `username:hash:mailbox` format. Existing entries without a uid field are
  treated as not yet migrated; webadmin assigns uids on next edit.
