# OS Identity Allocation (uid / gid)

**Status:** Authoritative
**Repos affected:** infodancer/maildancer (auth, session-manager, userctl, the
mail daemons) and homelab (deployment / ansible).

---

## Invariants -- read these before changing anything

We went in circles on where the domain gid lives and it locked a live mailbox
out of IMAP (the password verified; `mail-session` spawned with a gid that could
not traverse the `2750 root:{gid}` data dirs, and the resulting spawn failure
surfaced to the client as `AUTHENTICATIONFAILED`). Six hard rules. If a change
violates one, the change is wrong, not the rule.

1. **OS uid/gid are identity allocation, NOT configuration.** They are exempt
   from the site -> domain -> user override/merge hierarchy that governs
   `config.toml`. A uid or gid is allocated once and is authoritative; it is not
   a value that a lower layer may override. Never put a uid or gid in any file
   that is deep-merged.

2. **There is exactly one home for each, and exactly one code path that writes
   it.**
   - Domain -> gid: the single top-level map `{config}/gid.toml`.
   - User -> uid: the per-domain map `{config}/{domain}/uid.toml`.
   - A single shared package owns reading and writing these maps. `userctl`
     (CLI) and `webadmin` (web UI) are the two entry points, and both go through
     that shared code -- neither touches the files directly. One place manages
     it; two front doors. Nothing else allocates, copies, or caches a uid/gid.
     The daemons read the maps; they never write them.

3. **Allocate once; never reassign a live id.** Reassigning a gid orphans every
   file owned by the old group; reassigning a uid orphans a user's mail. The
   allocator refuses to overwrite an existing entry. Repair means chowning the
   data to the recorded id, never minting a new id to match the data.

4. **The data tree holds mail data and the allocator counter -- never an
   authoritative id record.** `{data}/{domain}/` carries maildirs, per-user
   keyrings, and the shared `.uid-counter`. It must not carry a `gid` of record:
   the data volume is cross-user and is the thing being protected, not the thing
   that defines the protection.

5. **The auth *provider* is configuration; the local provider's identity tables
   are not.** Which authentication/identity backend a domain uses -- local
   passwd-files (the default), LDAP, a database -- is a hierarchical config
   choice: a site-wide default in the top-level `config.toml`, optionally
   overridden per domain. `gid.toml` and `uid.toml` are the identity store *of
   the local passwd-files provider*; they apply only to domains using it. A
   domain configured for LDAP or a database sources its users, uids, gids, and
   credentials from that backend, and these files are not consulted for it.
   The mechanism (which provider) respects the hierarchy; the uid/gid values a
   given provider records do not. Auth configuration -- including LDAP/DB
   connection settings -- always lives in the config tree, never the data tree,
   exactly like `passwd`.

6. **The id space is banded, and the bands are enforced in code**
   (maildancer#197). `[1, 10000)` is reserved for service/system accounts
   (the all-in-one image's fixed 900-906 ids); `[10000, ...)` minus the
   well-known `65532-65535` band (distroless nonroot/nogroup, nobody, the
   16-bit -1) is the allocatable space the shared counter draws from. The
   boundary constants live in maildancer's `internal/idrange` -- the single
   home; do not restate them as literals. Enforcement: the allocator skips
   the excluded band; the identity package refuses to record an
   unallocatable id in the maps; the daemons refuse `handler_uid`/`gid`/
   `groups` values in the allocatable range (an allocated id there would
   run the internet-facing handler as a real mail principal); and
   `credentials.Lookup` refuses to spawn `mail-session` with map values
   outside the allocatable range (a corrupted entry fails the login rather
   than spawning as root or a service account).

If you are about to add a `gid` to `config.toml`, to the postmaster file, or to
`{data}/{domain}/config.toml`, stop -- that is the exact mistake this document
exists to prevent. Re-litigated 2026-06-18 (maildancer#101).

---

## The three concerns, three stores

Mail state splits into three kinds of thing. Only one of them is hierarchical.

| Concern | Store | Hierarchical? | Secret? | Writer |
|---|---|---|---|---|
| **Identity** (uid/gid allocation, local provider) | `gid.toml`, `{domain}/uid.toml` | **No** -- flat, authoritative | No | shared identity pkg (`userctl` + `webadmin`) |
| **Credentials** (password hash, local provider) | `{domain}/passwd` | No -- flat per-domain | Yes | shared identity pkg (`userctl` + `webadmin`) |
| **Configuration** (auth provider + store/dkim/outbound/limits/dns/forwards) | `{domain}/config.toml` (+ site defaults) | **Yes** -- site -> domain -> user merge | No | postmaster / IaC |

The Identity and Credentials rows describe the **local passwd-files provider**.
A domain whose configured provider is LDAP or a database does not use these
files at all -- its identities and credentials live in that backend. The
*choice* of provider is the hierarchical Configuration row.

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

Mode `0640`, owned `webadmin:cfgread` (the webadmin service account writes;
the dedicated cfgread group reads -- see the config-tree permissions section
of mail-security-model.md, maildancer#152). An earlier revision of this doc
mandated `root:root`, which locked the nonroot auth-oidc reader out of the
config tree and crash-looped the IdP; a second revision used `root:65532`.
The ownership is part of the model and is enforced by the perm doctor
(maildancer `internal/admin/perms.go`).
Read directly by the spawn path (`credentials.Lookup`) -- if a domain has no
entry, that is a hard error, not a default.

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

`[auth].type` selects the identity/credential provider -- local passwd-files by
default, or `ldap` / `database`. That selection is hierarchical: a site-wide
default here can be overridden for a single domain. When a non-local provider is
selected for a domain, `gid.toml` / `uid.toml` / `passwd` do not apply to it --
its users, uids, gids, and credentials come from the backend, configured (with
its connection settings) in this same hierarchical config, never in the data
tree.

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

This is the resolution for the local passwd-files provider. A domain configured
for a non-local provider resolves its credential of record (and any uid/gid the
backend supplies) through that provider instead; `gid.toml` / `uid.toml` are not
read for it.

## Allocation and repair (shared identity package; `userctl` + `webadmin`)

A single shared package performs all allocation and repair; `userctl` and
`webadmin` are entry points into it and never write the maps directly. Two
operations matter (for the local passwd-files provider):

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
