# Outbound Transport Routing
*Last updated: 2026-03-12*

## Problem

queue-manager currently passes a single global `--smarthost` flag to every
mail-remote invocation. There is no way to vary the delivery transport by
sender domain. This is needed for deployments where different sending domains
have different outbound requirements — for example, one domain delivers
directly via MX while another relays through AWS SES.

## Design Principle

**The sender domain determines the outbound transport, not the recipient.**

This matches how the stack already works for DKIM: smtpd looks up the sender
domain's key at queue-inject time and signs accordingly. Outbound transport is
the same class of per-domain policy.

## Where the Decision Lives

queue-manager owns delivery policy. It already owns retry scheduling, rate
limiting, TTL expiry, and DSN generation. Adding transport routing here means:

- **One place** regardless of submission protocol (SMTP, scmp, sdmp).
- **Changeable at runtime** — updating a domain's config affects the next
  queue scan, including already-queued messages on retry.
- **smtpd stays focused** on receiving, authentication, and queue injection.
- **mail-remote stays simple** — it already accepts `--smarthost` and
  `--smarthost-user` flags; no changes needed.

## Configuration

### Per-domain outbound config

A new `[outbound]` section in the existing per-domain config file
(`{domain-base}/{domain}/config.toml`):

```toml
[outbound]
strategy = "smarthost"                              # "direct" | "smarthost"
smarthost = "email-smtp.us-east-1.amazonaws.com:587"
smarthost_user = "AKIAIOSFODNN7EXAMPLE"
password_file = "outbound-password"                 # relative to domain dir
```

| Field            | Description |
|------------------|-------------|
| `strategy`       | `"direct"` for MX delivery, `"smarthost"` for relay. Default: `"direct"`. |
| `smarthost`      | Relay address in `host:port` form. Required when strategy is `"smarthost"`. |
| `smarthost_user` | SMTP AUTH username. Required when strategy is `"smarthost"`. |
| `password_file`  | Path to a file containing the SMTP AUTH password (one line, trimmed). Relative paths resolve from the domain directory. Required when strategy is `"smarthost"`. |

### System-wide default

The system-wide `config.toml` (`{domain-base}/config.toml`) can set a default
`[outbound]` section that applies to all domains without their own override.
Per-domain config merges over the system default using the existing
`mergeConfig` pattern.

### Example: mixed direct + SES

System default (`/srv/mail/domains/config.toml`):
```toml
[outbound]
strategy = "direct"
```

Domain override (`/srv/mail/domains/infodancer.net/config.toml`):
```toml
[dkim]
selector = "default"
private_key = "dkim.key"

[outbound]
strategy = "direct"
```

Domain override (`/srv/mail/domains/otherdomain.com/config.toml`):
```toml
[dkim]
selector = "ses"
private_key = "dkim-ses.key"

[outbound]
strategy = "smarthost"
smarthost = "email-smtp.us-east-1.amazonaws.com:587"
smarthost_user = "AKIAIOSFODNN7EXAMPLE"
password_file = "ses-password"
```

### queue-manager TOML addition

queue-manager needs to know where domain configs live:

```toml
[queue-manager]
domain_config_path = "/srv/mail/domains"
```

This is the same base path used by smtpd's `FilesystemDomainProvider`.

## Implementation

### Data flow

```
queue-manager                          mail-remote
     |                                      |
     |  1. Scan env/{tld}/{domain}/         |
     |  2. Resolve body path:               |
     |     msg/{tld}/{sender-domain}/{msgid}|
     |  3. Extract sender domain from       |
     |     body path                        |
     |  4. Load {domain-base}/{sender}/     |
     |     config.toml → [outbound]         |
     |  5. If strategy == "smarthost":      |
     |     write JSON config to stdin      |
     |  6. Invoke mail-remote               |
     |     ─────────────────────────────────>
     |                                      |
     |  (mail-remote delivers using         |
     |   whatever flags it received)        |
```

### Sender domain extraction

queue-manager already resolves the body path via glob:
`msg/{tld}/{sender-domain}/{msgid}`. The sender domain is embedded in the
path. Extract it:

```go
// bodyPath example: /var/spool/queue/msg/com/example/abc123def456
// sender domain: example.com
func senderDomainFromBodyPath(queueDir, bodyPath string) string {
    // Strip msg/ prefix, parse {tld}/{domain}/{msgid}
    rel := strings.TrimPrefix(bodyPath, filepath.Join(queueDir, "msg")+"/")
    parts := strings.SplitN(rel, "/", 3) // [tld, domain, msgid]
    if len(parts) < 3 {
        return ""
    }
    return parts[1] + "." + parts[0] // domain.tld
}
```

For multi-level TLDs (e.g., `uk/example.co/msgid` → `example.co.uk`), the
domain directory already contains the full registrable domain minus the TLD,
matching the existing queue layout convention.

### Config loading

queue-manager reads domain config on demand. No need for a full
`DomainProvider` — only the `[outbound]` section is needed:

```go
type OutboundConfig struct {
    Strategy     string `toml:"strategy"`      // "direct" | "smarthost"
    Smarthost    string `toml:"smarthost"`
    SmarthostUser string `toml:"smarthost_user"`
    PasswordFile string `toml:"password_file"`
}
```

Loading:
1. Read `{domain-base}/config.toml` → system default `[outbound]`
2. Read `{domain-base}/{sender-domain}/config.toml` → domain override
3. Merge (domain overrides system default)
4. If `strategy` is empty after merge, default to `"direct"`

Cache the loaded configs for the duration of one queue scan pass to avoid
re-reading the filesystem for every envelope group. Invalidate between passes
so config changes take effect on the next scan.

### Password handling

Passwords are read from files, not environment variables or TOML, because:
- Secrets don't belong in config files (TOML is readable by anyone with
  filesystem access to the domain dir).
- Per-domain env vars are awkward (`MAIL_REMOTE_PASSWORD_EXAMPLE_COM`).
- File-based secrets integrate well with Docker secrets, systemd credentials,
  and Ansible vaults.

queue-manager reads the password file, serializes the full outbound config
(strategy, hostname, smarthost, user, password) as JSON, and writes it to mail-remote's
stdin. This avoids environment variables, which are visible in
`/proc/pid/environ`.

```go
stdinJSON, _ := json.Marshal(outboundConfig)
cmd.Stdin = bytes.NewReader(stdinJSON)
// mail-remote reads JSON from stdin, writes results to stdout
```

The password file should be mode 0600 or 0640 (group-readable by domain
gid), owned by root.

### Changes to mail-remote

mail-remote reads JSON outbound config from stdin when it's a pipe.
CLI flags `--smarthost`/`--smarthost-user` remain as manual overrides
and take precedence over stdin config. Falls back to `MAIL_REMOTE_PASSWORD`
env var for the password when stdin has none.

Protocol: stdin = JSON config, stdout = JSON results, stderr = slog.

### Changes to `buildArgs` in scheduler.go

The existing `buildArgs` method reads smarthost flags from the global
`Config`. Replace with per-invocation args based on the resolved outbound
config:

```go
func (s *Scheduler) buildArgs(bodyPath string, envPaths []string, final bool, outbound OutboundConfig) []string {
    var args []string
    if s.cfg.ConfigPath != "" {
        args = append(args, "--config", s.cfg.ConfigPath)
    }
    // Outbound config (hostname, smarthost, password) is passed via stdin JSON.
    if final {
        args = append(args, "--final")
    }
    args = append(args, bodyPath)
    args = append(args, envPaths...)
    return args
}
```

The global `--smarthost` and `--smarthost-user` CLI flags on queue-manager
become the fallback when no domain config is found — preserving backward
compatibility.

## Changes to `auth/domain`

Add `OutboundConfig` to `DomainConfig`:

```go
type OutboundConfig struct {
    Strategy      string `toml:"strategy"`
    Smarthost     string `toml:"smarthost"`
    SmarthostUser string `toml:"smarthost_user"`
    PasswordFile  string `toml:"password_file"`
}

type DomainConfig struct {
    Auth     DomainAuthConfig     `toml:"auth"`
    MsgStore DomainMsgStoreConfig `toml:"msgstore"`
    DKIM     DKIMConfig           `toml:"dkim"`
    Outbound OutboundConfig       `toml:"outbound"`
    // ... existing fields
}
```

Update `mergeConfig` to handle Outbound fields.

queue-manager does **not** import `auth/domain` as a library dependency. It
defines its own minimal struct for parsing `[outbound]` from the TOML file.
This avoids pulling auth/domain (and its transitive dependencies) into
queue-manager. The structs are compatible because they parse the same TOML
structure — they just live in different packages.

## Fallback Order

For a given sender domain, the outbound strategy is resolved as:

1. `{domain-base}/{sender-domain}/config.toml` `[outbound]`
2. `{domain-base}/config.toml` `[outbound]` (system default)
3. queue-manager CLI flags (`--smarthost`, `--smarthost-user`)
4. No smarthost → direct MX delivery

This means:
- Existing deployments with only CLI flags continue to work unchanged.
- Adding a `domain_config_path` enables per-domain routing.
- Domains without an `[outbound]` section inherit the system default.
- The CLI flags serve as a last-resort global override.

## Security Considerations

- **Password files** must be mode 0600, owned by the queue-manager user.
  queue-manager should warn (not fail) if permissions are too open.
- **Outbound config via stdin**: smarthost credentials are passed as JSON on
  stdin, not environment variables. Stdin pipes are not visible in
  `/proc/pid/environ` and are cleaned up when the process exits.
- **Config file permissions**: `config.toml` files contain smarthost addresses
  and usernames but not passwords. Standard domain-dir permissions apply.

## Migration

1. Add `[outbound]` to `auth/domain.DomainConfig` and `mergeConfig`.
2. Add `domain_config_path` to queue-manager's TOML config.
3. Add outbound config loading and per-invocation arg building to scheduler.
4. Remove global `--smarthost` / `--smarthost-user` from queue-manager CLI
   (or keep as deprecated fallback).
5. Write per-domain `[outbound]` sections and password files.
6. No changes to mail-remote, smtpd, or envelope format.

## Non-Goals

- **Recipient-based routing**: Not needed. The sender domain determines the
  transport because it controls SPF, DKIM, and IP reputation.
- **Multiple smarthosts per domain**: One smarthost per sending domain is
  sufficient. SES and other relays handle their own HA.
- **Hot-reload of config**: Config is re-read on each queue scan pass
  (every ~1 minute). This is fast enough for operational changes.
- **Envelope format changes**: The envelope carries delivery data (sender,
  recipient, TTL). Transport policy is operational config, not message
  metadata.
