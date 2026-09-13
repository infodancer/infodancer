# JMAP Evaluation
*Decision record*
*Last updated: 2026-09-13*

## Decision

**Deferred, not rejected.** maildancer does not implement JMAP now. There is no `jmapd`. The prerequisite work JMAP shares with IMAP is done first because IMAP needs it anyway, and JMAP is reconsidered when the revisit triggers below fire.

When JMAP is built, it is a **bridge in the same class as IMAP**: a server-side protocol over server-parsed mail, with the same workarounds for whole-message encryption. It is not the next-gen client protocol -- that is SCMP.

## What JMAP is

JMAP (https://jmap.io/) is a stateless HTTPS/JSON client-to-server protocol for mail, intended as an IMAP replacement. Its main design points:

- Batched method calls in one request, with back-references between calls.
- First-class synchronization: every object type carries a `state` string and a `Foo/changes` method, where IMAP needed CONDSTORE/QRESYNC added later.
- Email ids are immutable and independent of mailbox; a message can belong to several mailboxes at once (labels).
- Push via EventSource, WebSocket, or Web Push -- no long-lived per-mailbox connection.
- Submission, vacation response, and Sieve management in the same API, so a webmail client needs nothing else.

### Standards status (2026-09)

| Spec | Status |
|---|---|
| Core | RFC 8620 |
| Mail (incl. submission, vacation) | RFC 8621 |
| WebSocket transport | RFC 8887 |
| MDN | RFC 9007 |
| S/MIME signature verification | RFC 9219 |
| Blob management | RFC 9404 |
| Quotas | RFC 9425 |
| JSContact / Contacts | RFC 9553 / RFC 9610 |
| Sieve scripts | RFC 9661 |
| Sharing | RFC 9670 |
| VAPID push | RFC 9749 |
| Calendars, JSCalendar 2.0 | Drafts |

### Ecosystem (2026-09)

- **Servers:** Stalwart (Rust, AGPL, implements every extension), Cyrus IMAP, Apache James, tmail-backend. atmail is Go but proprietary. There is **no open-source Go JMAP server and no Go server-side library**; the Go libraries (rockorager/go-jmap, pr0ton11/jmap-go) are clients.
- **Clients:** aerc, meli, Ltt.rs, Sterna Mail, Boogie, Allodia, Mailtemi, Parula. Apple Mail, Outlook, the Gmail apps, and K-9/Thunderbird for Android do not speak JMAP.
- **Thunderbird** has committed to JMAP (Thundermail is built on Stalwart) but has not shipped desktop support. This is the event most likely to change the client picture.

## Why deferred

### 1. The deployable audience is small

Client support is limited to terminal clients, a handful of mobile apps, and proprietary Apple-platform clients. A `jmapd` today would serve almost no users, and would be built without a Go server library to start from.

### 2. It needs the same sync state IMAP lacks

JMAP requires `state` and `/changes` for Email, Mailbox, and Thread. msgstore tracks UIDVALIDITY and a uidlist and nothing else: no modification sequence, no change log. imapd does not advertise CONDSTORE or QRESYNC for the same reason. A per-folder modseq and change log in msgstore is the shared prerequisite, and IMAP benefits from it immediately, so it is tracked separately (maildancer#253) and done first.

### 3. Token authentication cannot unseal the mailbox key

JMAP clients authenticate with bearer tokens. The user's private key is sealed under their password (see [encryption-design.md](encryption-design.md)), and sessions authenticated without a password -- OAUTHBEARER today -- are served encrypted blobs raw. JMAP over OAuth against an encrypted mailbox returns ciphertext. HTTP Basic works, but because JMAP requests are stateless, the unsealed key has to be bound to something that outlives the request, and a token that unseals a mailbox is a retained credential. This is the same problem [session-recovery-design.md](session-recovery-design.md) addressed for IMAP IDLE, and OAUTHBEARER on IMAP already hits it. It needs one answer that covers every protocol, not a JMAP-specific one; that design is maildancer#254 (session-scoped token wrap slots in the keyring, with optional folder scope for agent identities).

### 4. The mail model assumes a server-side index

`Email/query` with sort, `Thread/get`, `SearchSnippet/get`, and the parsed header and `bodyStructure` properties are designed around an indexed server. Whole-message encryption forecloses a plaintext index, headers included. Under that model, query sorts are scan-with-decrypt, threading requires decrypting References headers (or storing thread ids, which leaks the thread structure the model protects), and the encrypted header cache (maildancer#62) moves from a performance item to a requirement.

This is the same cost IMAP already pays -- SEARCH is brute-force scan-with-decrypt, and SORT and THREAD (not yet advertised) would be too -- and maildancer is committed to carrying those workarounds for IMAP. JMAP inherits them rather than adding new ones. The difference is in who absorbs the cost: IMAP clients cache aggressively and tolerate a slow unindexed server, while JMAP clients are designed as thin views over server queries and will expose the latency more directly.

### 5. SCMP is the modern client protocol

SCMP is client-encrypts, server-opaque. JMAP is server-parses-MIME. Investing in JMAP as *the* modern client path would compete with SCMP rather than bridge to it, and "modern protocol over a server that holds plaintext" is a space Stalwart already occupies. As a bridge, JMAP does not conflict with SCMP any more than IMAP does.

## Architectural fit when built

The process model does not obstruct JMAP. A `jmapd` is a protocol daemon in the same position as imapd: network-facing, no storage or auth imports (depguard rules apply unchanged), authentication and every mailbox operation routed through session-manager to a per-user mail-session. What it needs from below:

- msgstore modseq/change log (prerequisite; maildancer#253).
- Stable message ids independent of folder. The Maildir unique filename base is a candidate; per-folder UIDs are not.
- The token-to-key design from item 3 (maildancer#254), and folder-scope enforcement (maildancer#255, deferred) if JMAP sessions may carry agent tokens.
- The encrypted header cache (maildancer#62) for acceptable query and thread performance.

## Revisit triggers

- Thunderbird ships JMAP support, or another mainstream desktop or mobile client does.
- A Go JMAP server library of usable quality appears.
- The msgstore change log lands and the token-to-key question is decided, leaving JMAP as mostly protocol-layer work.
- A webmail client for maildancer is scheduled -- JMAP is the natural API for one.
