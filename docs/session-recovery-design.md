# Session Recovery Design (issue #179, later stages)

Status: implemented (maildancer `internal/smclient` + imapd/pop3d wiring;
see the #179 branch history). Covers the remaining scope of maildancer#179 after
the fork-per-connection stages landed (imapd PR #187, pop3d PR #190, smtpd
migration #189): credential retention in protocol handlers, transparent
session recovery across a session-manager restart, and write-RPC replay
semantics. Supersedes the "protocol handlers reconnect, re-authenticate"
sentence in session-manager-design.md's HA section for IMAP and POP3: the
*handler* now re-authenticates on the client's behalf; the client never
reconnects.

## Problem

session-manager holds all session state in memory: the token maps
(`internal/session-manager/manager/manager.go`, `byToken`/`byUser`) are never
persisted, and every mail-session child dies with the manager (graceful
`Close` SIGTERMs them; Pdeathsig covers ungraceful death, maildancer#178). A
restarted session-manager rebinds the same unix socket with empty maps, so:

- Every previously issued session token yields `codes.Unauthenticated` on the
  next proxied RPC.
- While the manager is down, RPCs yield transport-level `codes.Unavailable`.
- Re-establishing a session requires a full `Login`, because login is what
  unseals the user's private key from their password
  (`auth/passwd.Authenticate` -> `loadKeys`); with at-rest encryption on,
  no token or cached state can substitute for the password.

Today neither daemon retains the password past the initial `Login` call, no
code inspects gRPC status codes, and no retry exists anywhere. The observed
failure mode for IMAP is the worst kind: silent. IDLE keepalives and rescans
log their errors and continue (`internal/imapd/backend/session.go`,
`runIdleKeepalive`), so an IDLE'ing client hangs indefinitely, receiving no
new-mail notifications and no error. pop3d is worse in a different way: QUIT
answers `+OK` before the deferred Delete/Expunge RPCs run, so a restart at
commit time silently loses deletions the client believes succeeded.

## Why credential retention is now acceptable

The #126/#179 design pass rejected retaining passwords (option A) when imapd
was a single goroutine-per-connection process: one memory-disclosure bug
would have leaked every IDLE'ing user's password at once. That objection was
to the *shared* process, not to retention as such.

With fork-per-connection landed everywhere, the retaining process is a
subprocess scoped to exactly one client connection:

- It already handled that user's plaintext password during authentication;
  retention extends the lifetime within the same process, it does not widen
  the set of processes that see the credential.
- A memory-disclosure bug leaks one user's credential, not all of them --
  the same blast radius the connection already has.
- With `handler_uid` configured, the handler runs under a dedicated
  unprivileged uid, bounding what else a compromised handler can read.

What retention must NOT do:

- The unsealed private key never enters a protocol handler. It exists only
  in session-manager (transiently, zeroed after the fd-3 key envelope is
  written to mail-session) and in mail-session. Retaining the password, not
  the key, preserves this.
- No credential is ever written to disk, logged, or included in a metrics
  report.

## Design

### 1. Shared session-manager client (`internal/smclient`)

imapd's and pop3d's session-manager clients are parallel copies today
(`internal/imapd/backend/smclient.go`, 347 lines, full mailbox surface;
`internal/pop3d/pop3/smclient.go`, 180 lines, subset; identical dial, Login,
token-metadata, and TLS logic). The recovery engine must not be written
twice, so consolidation comes first:

- New package `internal/smclient` owning: dial (lazy `grpc.NewClient`, unix
  or mTLS), `Login`/`Logout`, the `session-token` metadata helper, and the
  full RPC surface (pop3d simply uses a subset). Both daemons' config types
  map onto one `smclient.Config`.
- The `sessionManagerStore` adapters stay per-daemon (they adapt to each
  daemon's store interfaces) but shrink to thin wrappers.

This mirrors how connfork was extracted: prove on one daemon, move the
other, delete the copies.

### 2. Credential retention

- The shared client gains a per-session credential slot, set at `Login`:
  the exact credential the client presented (password for LOGIN/USER+PASS/
  AUTH PLAIN). Stored as `[]byte`, zeroed on connection close and after any
  final recovery failure. It lives only in the handler subprocess and only
  for the lifetime of its one connection.
- Retention is unconditional. A config knob would buy nothing: the process
  already held the password at auth time, and turning retention off just
  reintroduces the silent-hang failure mode. (Revisit if a deployment ever
  needs a compliance story for "passwords in RSS.")
- mlock/madvise hardening is deliberately out of scope: Go's GC can copy
  memory regardless, the process is short-lived and single-user, and the
  meaningful boundaries (per-connection process, dedicated uid, no swap on
  mail hosts) are already in place. Zeroing on close is best-effort hygiene,
  not a security boundary.
- Token-based auth (OAUTHBEARER, where supported) retains the bearer token
  instead; recovery then works only while the token remains valid, and a
  recovery failure drops the connection exactly as a rejected password does.
  Note the encryption design already excludes password-less sessions from
  key unsealing, so nothing is lost relative to today.

### 3. Detection and re-login

The recovery engine lives in `internal/smclient` and wraps every RPC:

Trigger set (from the actual server behavior):

| Signal | Meaning | Action |
|--------|---------|--------|
| `codes.Unavailable` | manager down or restarting; RPC not executed by the proxy | retry the *connection* with backoff; then re-login when it answers |
| `codes.Unauthenticated` on a data RPC, when a previously valid token exists | manager restarted; token map is empty | re-login immediately |
| `codes.Unauthenticated` from `Login` itself | credential rejected (password changed, account disabled) | fatal: no retry, drop the connection |

Mechanics:

- **Single-flight re-login.** One recovery at a time per session (mutex);
  concurrent RPCs block on the in-progress recovery and then proceed with
  the new token. Each successful re-login swaps the token atomically.
- **Backoff and deadline.** Exponential backoff from 250ms capped at 15s,
  total recovery deadline configurable (`session_recovery_deadline`,
  default 2m). Past the deadline the session is declared dead: IMAP sends
  an untagged BYE, POP3 `-ERR [SYS/TEMP]`, the handler exits, the client
  reconnects and re-authenticates -- i.e. we degrade to option D behavior
  instead of hanging.
- **Continuity check after re-login (IMAP).** Re-login spawns a fresh
  mail-session; the previous session's state is gone. Before resuming, the
  engine re-fetches `UIDValidity` for the selected folder. Unchanged
  UIDVALIDITY (the normal case -- it is persisted per folder) means UIDs
  remain valid and the session resumes transparently, followed by a
  `Rescan` so anything delivered during the outage is queued as untagged
  EXISTS. A changed UIDVALIDITY means transparent resumption would violate
  IMAP semantics: send BYE and let the client resync -- correct, and
  vanishingly rare.
- **Metrics.** The handler collector gains recovery counters
  (`<ns>_session_recoveries_total{result=ok|auth_failed|deadline}`),
  reported over the existing fd-4 pipe.

### 4. Write-RPC replay: at-most-once

Decision: **reads retry, writes never auto-retry.**

- Idempotent reads -- `List`, `Stat`, `Fetch`, `FetchHeaders`,
  `SearchContent`, `Rescan`, `UIDValidity`, `ListFolders` -- are retried
  automatically after a successful recovery. The client never sees the
  restart on a read path.
- Writes -- `Append`, `SetFlags`, `Expunge`, `Copy`, `Move`, `Delete`,
  folder create/rename/delete -- are never replayed by the engine. If a
  write was in flight when the failure hit, recovery still runs (so the
  session is healthy afterward), but the in-flight command returns failure
  to the mail client: a tagged `NO [UNAVAILABLE]` for IMAP, `-ERR` for
  POP3. Mail clients already handle NO/-ERR by resubmitting under user
  control; a duplicated APPEND (duplicate message) or double EXPUNGE
  (sequence-number corruption) is strictly worse than a visible failure.
- The tempting refinement -- "retry writes on `Unavailable` because the
  request never reached the server" -- is rejected: gRPC does not guarantee
  that `Unavailable` means not-executed (the request can die after delivery),
  and a wrong guess on Append duplicates mail. If restart-window write
  failures ever become an operational annoyance, the correct extension is
  idempotency keys (client-generated UUID on Append, deduplicated in
  mail-session), recorded here as future work, not built now.

### 5. IDLE recovery path

IMAP IDLE today is Redis pub/sub + a keepalive `UIDValidity` RPC every
`session_keepalive` (default 5m) + `Rescan` on notification; there is no
long-lived gRPC stream to the manager. Recovery slots in naturally:

- The keepalive and the notification-triggered `Rescan` stop dropping
  errors: any trigger-set error starts recovery. After recovery, the
  continuity check + `Rescan` queue whatever arrived during the outage.
- Redis is a separate transport and keeps delivering notifications during a
  manager restart, so recovery is usually triggered within seconds of new
  mail arriving -- well inside the acceptance window of one keepalive
  interval.

### 6. pop3d commit ordering (required fix)

pop3d currently replies `+OK` to QUIT before running the deferred
Delete/Expunge RPCs and swallows their errors (`handler.go` commit loop).
That is wrong independently of recovery -- RFC 1939's UPDATE state completes
before the positive response -- and it makes deletion loss invisible. The
pop3d stage reorders commit before the QUIT response and maps a failed
commit (after one recovery attempt) to `-ERR [SYS/TEMP]`, letting the client
keep its DELE state and retry.

### 7. Out of scope: smtpd

smtpd's delivery path needs no recovery engine. Delivery uses
`DeliveryService` without `Login` (no token, no unsealed key on the
submission path), and SMTP has retry built into the protocol: a
session-manager restart mid-delivery surfaces as a 4xx tempfail and the
sender requeues. Adding transparent recovery there would buy nothing.

## Security model delta

| Property | Before | After |
|----------|--------|-------|
| Plaintext password lifetime in handler | duration of the Login call | lifetime of the one client connection, zeroed on close |
| Processes holding a given user's password | handler (transiently), session-manager (transiently) | same set; handler holds it longer |
| Processes holding multiple users' credentials | none | none (unchanged) |
| Unsealed key locations | session-manager (transient), mail-session | unchanged; never the handler |
| Failure of a handler process | leaks at most its one session | unchanged |

`mail-security-model.md` gains a "credential lifetime" paragraph stating the
above when the implementation lands (acceptance criterion from #126).

## Staging

1. **smclient extraction.** Merge the two clients into `internal/smclient`
   with no behavior change; both daemons onto it. (Also collapses the
   duplicated TLS/dial/token code.)
2. **Recovery engine + imapd.** Retention, detection, single-flight
   re-login, read-retry, IDLE wiring, recovery metrics. Acceptance test:
   integration harness that runs a real session-manager binary, kills and
   restarts it mid-IDLE, and asserts new mail appears without the IMAP
   client reconnecting (same build-the-real-binary pattern as the
   fork-per-connection tests).
3. **pop3d.** Same engine; commit-ordering fix; `-ERR` mapping.
4. **Docs.** mail-security-model credential-lifetime paragraph;
   session-manager-design HA cross-reference; close out #179 acceptance
   criteria (manual Thunderbird IDLE-across-restart check).

## Open questions

- Recovery deadline default: 2m is a guess; the right bound is "long enough
  for a systemd restart + auth backend warmup, short enough that a wedged
  manager doesn't strand thousands of half-dead connections." Revisit
  against real restart timings.
- Should recovery counters distinguish `Unavailable`-path recoveries
  (manager was down) from `Unauthenticated`-path ones (manager restarted)?
  Cheap to add; useful for diagnosing flapping.
- Append idempotency keys (future work trigger): revisit if write failures
  during restarts generate user-visible complaints.
