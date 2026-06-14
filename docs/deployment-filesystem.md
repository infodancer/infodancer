# Deployment Filesystem Layout

This document describes the recommended filesystem layout for infodancer
mail server deployments, including the optional separation of read-only
configuration from writable data.

## Single-Tree Layout (Simple)

The simplest deployment co-locates config and data under one root:

```
/srv/mail/domains/
├── config.toml                         # system-wide defaults
├── postmaster                          # postmaster → uid:gid mappings
├── infodancer.net/
│   ├── config.toml                     # domain config (auth, dkim, outbound)
│   ├── passwd                          # user credentials
│   ├── keys/                           # DKIM and encryption keys
│   │   ├── dkim.key
│   │   └── dkim.pub
│   ├── outbound-password               # smarthost secret (mode 0600)
│   └── users/                          # maildir data
│       └── alice/
│           └── Maildir/
```

All daemons receive a single `domains_path` pointing to this root. This
works for development, testing, and simple deployments where the domain
tree is on one filesystem.

## Split Layout (Recommended for Production)

Production deployments benefit from separating read-only configuration
from writable runtime data into distinct filesystem trees:

```
Config tree (read-only):                Data tree (writable):
/etc/infodancer/                        /opt/infodancer/domains/
├── config.toml                         ├── infodancer.net/
├── domains/                            │   └── users/
│   ├── config.toml                     │       └── alice/
│   ├── postmaster                      │           └── Maildir/
│   ├── infodancer.net/                 └── otherdomain.com/
│   │   ├── config.toml                     └── users/
│   │   ├── passwd                              └── bob/
│   │   ├── keys/                                   └── Maildir/
│   │   │   ├── dkim.key
│   │   │   └── dkim.pub
│   │   └── outbound-password
│   └── otherdomain.com/
│       ├── config.toml
│       ├── passwd
│       └── keys/
```

Daemons receive two paths:

```toml
[smtpd]
domains_path = "/etc/infodancer/domains"        # config tree
domains_data_path = "/opt/infodancer/domains"   # data tree
```

queue-manager's `domain_config_path` points to the config tree:

```toml
[queue-manager]
domain_config_path = "/etc/infodancer/domains"
```

### Why separate config from data

**Immutable config at runtime.** Mounting the config tree read-only
prevents a compromised process from modifying domain configuration,
credential files, or DKIM keys. A vulnerability in a network-facing
daemon cannot alter auth settings or outbound routing to redirect mail.

**Distinct backup strategies.** Config changes infrequently and should be
version-controlled (Ansible, git). Mail data changes constantly and needs
incremental backup (borgmatic, restic). Mixing them in one tree means
either backing up everything at mail-data frequency (wasteful) or risking
stale config backups.

**Separate storage characteristics.** Config is small and read-heavy --
it can live on the root filesystem or a fast SSD. Mail data grows
unboundedly and benefits from larger, cheaper storage with its own
monitoring and quota management.

**Container volume isolation.** With separate trees, the config mount is
`:ro` and the data mount is `:rw`. This enforces the read-only property
at the container runtime level, independent of filesystem permissions.

**Secret isolation.** Password files, DKIM private keys, and credential
databases live in the config tree where access is tightly controlled.
The data tree holds only maildir content under per-user uid ownership.

### Docker Compose example

```yaml
services:
  smtpd:
    volumes:
      - ./config:/etc/infodancer:ro
      - ./domains:/opt/infodancer/domains
      - ./certs:/etc/letsencrypt:ro

  session-manager:
    volumes:
      - ./config:/etc/infodancer:ro
      - ./domains:/opt/infodancer/domains

  queue-manager:
    volumes:
      - ./config:/etc/infodancer:ro
      - ./queue:/var/spool/queue
```

Note: queue-manager needs the config tree (for domain outbound config and
password files) but does not need the data tree. It reads the queue spool
directly.

## Path Resolution

Per-domain config files reference relative paths that resolve from the
domain directory within the config tree:

```toml
# /etc/infodancer/domains/infodancer.net/config.toml
[auth]
credential_backend = "passwd"       # → /etc/infodancer/domains/infodancer.net/passwd
key_backend = "keys"                # → /etc/infodancer/domains/infodancer.net/keys/

[dkim]
private_key = "keys/dkim.key"       # → /etc/infodancer/domains/infodancer.net/keys/dkim.key

[outbound]
password_file = "outbound-password" # → /etc/infodancer/domains/infodancer.net/outbound-password
```

The `msgstore.base_path` in per-domain config points to the data tree
since that is where maildirs are written:

```toml
[msgstore]
base_path = "/opt/infodancer/domains/infodancer.net/users"
```

## File Permissions

See [mail-security-model.md](mail-security-model.md) for the full uid/gid
model and directory permission scheme.

Key points for the split layout:

| Path | Owner | Mode | Notes |
|------|-------|------|-------|
| Config tree root | `root:root` | `755` | Read-only mount in containers |
| Domain config dir | `root:{domain-gid}` | `2750` | Setgid for group inheritance |
| `passwd` | `root:{domain-gid}` | `640` | Daemons read via domain gid |
| `keys/dkim.key` | `root:{domain-gid}` | `640` | smtpd reads for signing |
| `outbound-password` | `root:{domain-gid}` | `640` | queue-manager reads for smarthost auth |
| Data tree root | `root:root` | `711` | |
| Domain data dir | `root:{domain-gid}` | `2750` | |
| User maildir | `{user-uid}:{domain-gid}` | `700` | Only user uid can access |
