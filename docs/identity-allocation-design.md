# OS Identity Allocation (uid / gid)

**Status:** Authoritative
**Repos affected:** infodancer/maildancer (auth, session-manager, userctl, the
mail daemons) and homelab (deployment / ansible).

---

## Invariants -- read these before changing anything

We went in circles on where the domain gid lives and it locked a live mailbox
out of IMAP (the password verified; `mail-session` spawned with a gid that could
not traverse the `2750 root:{gid}` data dirs, and the resulting spawn failure
surfaced to the client as `AUTHENTICATIONFAILED`). Four hard rules. If a change
violates one, the change is wrong, not the rule.

1. **OS uid/gid are identity allocation, NOT configuration.** They are exempt
   from the site -> domain -> user override/merge hierarchy that governs
   `config.toml`. A uid or gid is allocated once and is authoritative; it is not
   a value that a lower layer may override. Never put a uid or gid in any file
   that is deep-merged.

2. **There is exactly one home for each, and exactly one writer.**
   - Domain -> gid: the single top-level map `{config}/gid.toml`.
   - User -> uid: the per-domain map `{config}/{domain}/uid.toml`.
   - `userctl` is the **only** thing that writes these maps. Nothing else
     allocates, copies, or caches a uid/gid. The daemons read them; they never
     write them.

3. **Allocate once; never reassign a live id.** Reassigning a gid orphans every
   file owned by the old group; reassigning a uid orphans a user's mail. The
   allocator refuses to overwrite an existing entry. Repair means chowning the
   data to the recorded id, never minting a new id to match the data.

4. **The data tree holds mail data and the allocator counter -- never an
   authoritative id record.** `{data}/{domain}/` carries maildirs, per-user
   keyrings, and the shared `.uid-counter`. It must not carry a `gid` of record:
   the data volume is cross-user and is the thing being protected, not the thing
   that defines the protection.

If you are about to add a `gid` to `config.toml`, to the postmaster file, or to
`{data}/{domain}/config.toml`, stop -- that is the exact mistake this document
exists to prevent. Re-litigated 2026-06-18 (maildancer#101).

---

## The three concerns, three stores

Mail state splits into three kinds of thing. Only one of them is hierarchical.

| Concern | Store | Hierarchical? | Secret? | Writer |
|---|---|---|---|---|
| **Identity** (uid/gid allocation) | `gid.toml`, `{domain}/uid.toml` | **No** -- flat, authoritative | No | `userctl` only |
| **Credentials** (password hash) | `{domain}/passwd` | No -- flat per-domain | Yes | `userctl` only |
| **Configuration** (auth/store/dkim/outbound/limits/dns/forwards) | `{domain}/config.toml` (+ site defaults) | **Yes** -- site -> domain -> user merge | No | postmaster / IaC |

### Identity: `{config}/gid.toml` (domain -> gid)

A single flat map at the config-tree root. Authoritative. Never merged, never
overridden, never templated by IaC.

```toml
# domain = gid. Authoritative OS group allocation for every mail domain.
# Managed ONLY by userctl. Allocate-once; never reassign a live domain.
# Do NOT hand-edit and do NOT render this from ansible.
"matthewjayhunter.com" = 10014
"amyhunter.org"        = 10010
```

Mode `0640`, owned `root:root`. Read directly by the spawn path
(`credentials.Lookup`) -- if a domain has no entry, that is a hard error, not a
default.

### Identity: `{config}/{domain}/uid.toml` (user -> uid)

Per-domain flat map. Same rules.

```toml
# localpart = uid. Authoritative OS user allocation for this domain.
# Managed ONLY by userctl. Allocate-once; never reassign a live user.
"matthew" = 10026
```

### Credentials: `{config}/{domain}/passwd`

`localpart:argon2id-hash:mailbox`. **No uid field** -- identity is `uid.toml`'s
job. Gitignored / managed out-of-band; carries the secret (hash) and the
mailbox-of-record only.

### Configuration: `{config}/{domain}/config.toml`

The only file subject to the merge hierarchy. Holds `[auth]`, `[msgstore]`,
`[dkim]`, `[outbound]`, `[limits]`, `[dns]`, `recipient_rejection`,
`encryption_mode`, and `forwards`. **Carries no gid.** Forwards are configuration
(they are routing policy a domain/user may override) and stay here -- they are
not identity.

---

## Resolution at spawn (`credentials.Lookup`)

session-manager spawns `mail-session` with `SysProcAttr.Credential{Uid, Gid}`.
Those two numbers come from exactly two reads, with no merge and no fallback:

```
gid = gid.toml[domain]                 // hard error if missing
uid = uid.toml[domain][localpart]      // hard error if missing
```

Everything else about the user (store type, base path, forwards) comes from the
hierarchical config and is orthogonal to identity.

There is deliberately **no** config-tree-`Gid` seed, **no** data-tree override,
and **no** postmaster-file override. Those three layered sources were the bug.

## Allocation and repair (`userctl`)

`userctl` is the sole writer. Two operations matter:

- **Allocate** (`domain create`, `user add`): take the next id from
  `{data}/.uid-counter`, record it in `gid.toml` / `uid.toml`, create the data
  dirs `root:{gid}` mode `2750` (domain, users) and `{uid}:{gid}` mode `0700`
  (the user maildir). Refuse if an entry already exists.

- **Reconcile** (`domain fix`): the one repair path. For each domain/user, read
  the authoritative id from the maps and chown the data tree to match
  (recursively, including the maildir leaves). It **reads** ids; it never mints
  a new one to paper over a mismatch. `domain fix` also performs the one-time
  migrations described below.

## Migration (one-time, performed by `domain fix`)

The pre-#101 layout had: uid in the passwd 4th field, gid in
`{config}/{domain}/config.toml` (top-level `gid`) and/or the
`{config}/domains/postmaster` file and/or `{data}/{domain}/config.toml`.
`userctl domain fix` migrates each domain forward:

1. If `gid.toml` has no entry for the domain, adopt the existing authoritative
   gid -- preferring the value that the data dirs are actually owned by, so no
   mass chown is forced -- and write it to `gid.toml`.
2. If `uid.toml` is absent, read each passwd entry's uid field into `uid.toml`,
   then rewrite `passwd` without the uid field.
3. Remove the `gid` key from `config.toml`; remove the per-domain gid role from
   the postmaster file (retire the file, or regenerate it derived from
   `gid.toml` if its system-postmaster identity is still needed); remove
   `{data}/{domain}/config.toml`.
4. Chown the data tree to the `gid.toml` value.

Migration is idempotent: a second run finds the maps populated and only verifies
ownership.

## Deployment (homelab)

ansible renders `{config}/{domain}/config.toml` (configuration) and the `passwd`
secret, and declares which domains/users should exist. It does **not** render
`gid.toml` or `uid.toml` -- those are allocation state owned by `userctl`. After
a deploy that adds a domain or user, `userctl domain fix --all` allocates any
missing ids and reconciles ownership.
