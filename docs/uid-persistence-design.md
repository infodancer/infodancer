# Persistent IMAP UIDs and UIDValidity

## Problem

imapd assigns IMAP UIDs positionally (`uid = index + 1`) from the in-memory
message list. UIDs shift when messages are deleted, but UIDVALIDITY never
changes (it's an FNV-32a hash of the folder name). This violates RFC 9051
§2.3.1.1 and causes:

- **Duplicate messages** in Thunderbird when sessions reconnect (UIDs shift,
  UIDVALIDITY stays the same, client trusts stale UIDs)
- **Lost flag state** (message that was UID 4 becomes UID 3 after a deletion;
  client applies UID 3's cached flags to the wrong message)
- **Broken COPY/MOVE responses** (returns computed destination UIDs, not actual
  ones)

## Requirements (RFC 9051 §2.3.1.1)

1. UIDs are unsigned 32-bit integers, strictly ascending within a mailbox
2. UIDs must not be reused within the same UIDVALIDITY
3. UIDs must persist across sessions (SELECT → logout → SELECT must return
   the same UIDs for the same messages)
4. UIDVALIDITY must change if and only if UIDs have been reassigned
5. UIDNEXT must be greater than or equal to all existing UIDs + 1

## Design: msgstore-owned UID map

### Ownership

msgstore owns the UID map. It assigns numeric UIDs to messages during
`List()`/`ListInFolder()` and translates them back to Maildir keys during
`Retrieve()`/`Delete()`. imapd never sees Maildir keys.

### Interface changes

**MessageInfo** gains a numeric UID; the Maildir key becomes internal:

```go
type MessageInfo struct {
    UID          uint32    // Numeric IMAP UID (assigned by uidlist)
    Key          string    // Storage key (Maildir filename); opaque to consumers
    Size         int64
    Flags        []string
    InternalDate time.Time
}
```

All interface methods that currently take `uid string` change to `uid uint32`.
msgstore maps the uint32 back to the Maildir key internally.

```go
// MessageStore
Retrieve(ctx, mailbox string, uid uint32) (io.ReadCloser, error)
Delete(ctx, mailbox string, uid uint32) error

// FolderStore
RetrieveFromFolder(ctx, mailbox, folder string, uid uint32) (io.ReadCloser, error)
DeleteInFolder(ctx, mailbox, folder string, uid uint32) error
SetFlagsInFolder(ctx, mailbox, folder string, uid uint32, flags []string) error
CopyMessage(ctx, mailbox, srcFolder string, uid uint32, destFolder string) (uint32, error)
AppendToFolder(ctx, mailbox, folder string, r io.Reader, flags []string, date time.Time) (uint32, error)
```

`UIDValidity()` signature stays the same but returns a persistent value.

**New method** on FolderStore:

```go
// UIDNext returns the next UID that will be assigned in the folder.
UIDNext(ctx context.Context, mailbox string, folder string) (uint32, error)
```

### File format: `.uidlist`

One file per Maildir directory (INBOX and each `.folder/`), stored alongside
`cur/`, `new/`, `tmp/`.

```
3 V1741824000 N42
1 1710339422.M123456.hostname
2 1710339423.M789012.hostname
...
41 1710340000.M345678.hostname
```

**Header line**: `<version> V<uidvalidity> N<uidnext>`
- Version: file format version (currently `3`, matching Dovecot convention)
- V: UIDValidity (uint32)
- N: UIDNext -- next UID to assign (always > max assigned UID)

**Entry lines**: `<uid> <maildir-key>`
- uid: uint32, strictly ascending
- maildir-key: the Maildir filename key (no flags suffix, no path)
- One line per message currently in the folder

### Operations

#### List / ListInFolder

1. Read `.uidlist` into memory (parse header + entries into a map)
2. Scan `cur/` directory for current Maildir keys
3. Reconcile:
   - **Key in uidlist and cur/**: keep entry, include in results
   - **Key in cur/ but not uidlist**: new message -- assign UIDNext, increment
     UIDNext, append entry
   - **Key in uidlist but not cur/**: externally deleted -- remove entry
4. If any changes occurred, rewrite `.uidlist`
5. Return `[]MessageInfo` with numeric UIDs, sorted by UID ascending

#### Deliver / DeliverToFolder

1. Write message to Maildir (existing `new/` → `cur/` flow)
2. Read `.uidlist`, assign UIDNext to the new key, increment UIDNext
3. Append entry, write file

#### AppendToFolder

Same as Deliver but also sets flags on the Maildir file before writing the
uidlist entry. Returns the assigned UID.

#### CopyMessage

1. Copy the Maildir file to the destination folder (existing logic)
2. Read destination's `.uidlist`, assign UIDNext, write
3. Return the assigned uint32 UID

#### Delete / Expunge

- `Delete()`: soft-delete unchanged (in-memory tracking by uint32 UID)
- `Expunge()`: remove Maildir files, then remove entries from `.uidlist`
  and rewrite. UIDs are never reused -- UIDNext is not decremented.

#### Retrieve / RetrieveFromFolder

1. Read `.uidlist` (or use cached map from prior List call -- see
   Caching below)
2. Look up uint32 UID → Maildir key
3. Open and return the file

#### UIDValidity

Read the V field from `.uidlist` header. If the file doesn't exist yet,
generate a new value (see Bootstrap).

#### UIDNext

Read the N field from `.uidlist` header.

### Bootstrap (first access / migration)

When `.uidlist` does not exist:

1. Generate UIDValidity from current Unix timestamp (uint32, truncated --
   good until 2106)
2. Scan `cur/` for all Maildir keys
3. Sort keys lexicographically (Maildir keys start with a timestamp, so
   this gives chronological order)
4. Assign UIDs 1..N
5. Set UIDNext = N + 1
6. Write `.uidlist`

**One-time impact**: existing IMAP clients will see a UIDVALIDITY change
(hash → timestamp) and re-sync. This is unavoidable and acceptable.

### Recovery (corrupt file)

If `.uidlist` fails to parse:

1. Log a warning
2. Delete the corrupt file
3. Run Bootstrap
4. Clients re-sync (UIDVALIDITY changed)

### Caching

The MaildirStore should cache the parsed uidlist in memory per folder for
the duration of a session (between `List()` and the next `List()` or
`Expunge()`). This avoids re-reading the file for every `Retrieve()` call.

The cache is a `map[uint32]string` (UID → Maildir key) plus the header
fields. Invalidated on any mutating operation (Deliver, Append, Copy,
Expunge).

### Concurrency

Multiple processes may access the same Maildir concurrently (e.g., smtpd
delivering while imapd reads). The uidlist file must be safe under
concurrent access:

- **Writes**: acquire an exclusive flock on `.uidlist.lock` before
  reading, modifying, and rewriting. Release after write completes.
- **Reads**: acquire a shared flock. This prevents reading a half-written
  file.
- **Atomic rewrite**: write to `.uidlist.tmp`, fsync, rename over
  `.uidlist`. This ensures readers never see a partial file even
  without flocking.

In practice, session-manager serializes most access through a single
mail-session process per user, so contention is low. The locking is
defense-in-depth for direct Maildir access (manual delivery, migration
tools).

## Downstream changes

### gRPC protobuf (mail-session)

The `MessageInfo` proto message changes `string uid` to `uint32 uid`.
Add `string key` if needed (probably not -- mail-session uses msgstore
directly, so it never needs to expose the key).

All RPC methods that take `string uid` change to `uint32 uid`:
- `Retrieve`, `Delete`, `DeleteInFolder`, `SetFlags`, `Copy`, `Move`

`AppendToFolder` and `CopyMessage` responses change from `string new_uid`
to `uint32 new_uid`.

Add `UIDNext` RPC (or extend `UIDValidity` response to include both).

### session-manager

Passthrough changes only -- update proto field types in proxy calls.

### imapd

This is where the real cleanup happens. Every `imap.UID(idx + 1)` becomes
`imap.UID(info.UID)`. Every `len(msgs) + 1` UIDNEXT becomes a call to
`folderStore.UIDNext()`.

**Affected files**:

| File | Change |
|------|--------|
| `fetch.go:27` | `uid := imap.UID(info.UID)` |
| `mailbox.go:113` | `UIDNext: imap.UID(uidNext)` via `folderStore.UIDNext()` |
| `mailbox.go:203` | Same for Status |
| `mailbox.go:261` | `UID: imap.UID(appendedUID)` from AppendToFolder return |
| `storeops.go` | Copy/Move use actual returned UIDs |
| `session.go:resolveNumSet` | Look up UIDs from `s.messages[i].UID` instead of `i+1` |
| `search.go:30` | `uid := imap.UID(msg.UID)` |

`resolveNumSet` for the UIDSet branch needs a UID→index lookup map
(built once on Select, stored in Session):

```go
// Built in Select after populating s.messages
s.uidIndex = make(map[imap.UID]int, len(msgs))
for i, m := range msgs {
    s.uidIndex[imap.UID(m.UID)] = i
}
```

For static UID sets, look up each UID in the map. For dynamic ranges,
iterate messages and check `Contains(imap.UID(m.UID))`.

### pop3d

POP3 UIDL responses currently use the Maildir key string. After this
change, `info.UID` is a uint32. Options:

1. Use `strconv.FormatUint(uint64(info.UID), 10)` as the UIDL -- valid
   per RFC 1939, stable across sessions, simpler
2. Use `info.Key` -- preserves current behavior for existing POP3 clients

Option 1 is cleaner. One-time POP3 client re-sync (most POP3 clients
re-download on UIDL change; this is a minor impact since POP3 is not the
primary retrieval protocol).

## Implementation order

1. **msgstore**: Add `.uidlist` read/write/reconcile in maildir
   package. Change `MessageInfo.UID` to uint32, add `Key` field. Update
   all interface signatures. Update tests.

2. **mail-session protobuf**: Update proto definitions, regenerate,
   update mail-session gRPC handlers.

3. **session-manager**: Update proxy passthrough for new field types.

4. **imapd**: Replace all `imap.UID(idx+1)` with `imap.UID(info.UID)`.
   Add `uidIndex` map to Session. Update resolveNumSet. Call
   `UIDNext()` instead of `len(msgs)+1`. Update tests.

5. **pop3d**: Update UIDL to use numeric UID.

6. **Deploy**: Restart services. All IMAP/POP3 clients re-sync once
   (UIDVALIDITY change from hash to timestamp).

## Risks

- **One-time client re-sync**: unavoidable. All connected clients will
  re-download headers when UIDVALIDITY changes. For a small mailbox this
  takes seconds; for large mailboxes it could take minutes.

- **External Maildir tools**: anything that adds/removes files from cur/
  without updating `.uidlist` will trigger reconciliation on the next
  List(). This is handled gracefully (new files get UIDs, missing files
  get removed) but the UIDs won't match what the external tool expected.

- **Interface breakage across repos**: MessageStore and FolderStore are
  used by smtpd (DeliveryAgent), pop3d, imapd, mail-session, and
  session-manager. All must be updated in the same release cycle.
  Version-pin msgstore, update consumers, then tag.
