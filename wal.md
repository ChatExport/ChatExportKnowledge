# The WAL beside `sms.db`, and why reading a backup can lose the newest messages

> Part of [a set of notes](README.md) on Apple's message format. Documents the
> format, not any particular program.
>
> Last checked against backups from iOS 11 to 26.

If you take one thing from these notes, take this one. It is the failure that
produces a transcript which looks complete, contains no error of any kind, and
is missing the last few days of the conversation.

## What happens

iOS runs its SQLite databases in **WAL mode** — write-ahead logging — and the
backup process copies the files as they are. So a backup very often contains
two files where you expected one:

```
Library/SMS/sms.db
Library/SMS/sms.db-wal
```

Each is stored under its own SHA1 name, by the rule in
[`manifest-structure.md`](manifest-structure.md):

```
SHA1("HomeDomain-Library/SMS/sms.db")
  = 3d0d7e5fb2ce288813306e4d4636395e047a3d28
SHA1("HomeDomain-Library/SMS/sms.db-wal")
  = cd47480f213dba9bc38ee792775d17e3f5a73a59
```

Both values are constant in every iTunes-style backup, and neither resembles
the other, because a hash of two similar strings is two unrelated names.

And there is the problem. **SQLite can only replay a write-ahead log that sits
next to its database, under the matching file name.** In the hashed backup
layout the two never meet: they are two forty-character names in two different
two-character folders, with nothing to connect them but the rule you used to
find them.

Locate `sms.db`, open it where it lies, and SQLite sees a database with no WAL
to apply. Every transaction still sitting in that log is invisible. Not
corrupted, not reported, simply absent — and because the WAL holds the most
recent writes, what goes missing is the newest end of the conversation.
Potentially days of it.

Nothing in the result looks wrong. The database opens, the queries run, the
messages are all there in the sense that every row returned is genuine. The
transcript just stops earlier than the phone does, and only somebody who knows
what the last message was will ever notice.

## Spotting a WAL-mode database

Byte 19 of the SQLite file header is the **read format version**: `2` means
WAL, `1` means the older rollback journal. Twenty bytes off the front of the
file answer it, with no SQLite call and no risk of touching the file:

```python
with open(db_path, "rb") as f:
    wal_mode = f.read(20)[19] == 2
```

Worth checking rather than assuming, in both directions. Not every database in
every backup is in WAL mode, and a backup can carry a `-wal` file that has
already been checkpointed and holds nothing.

## The second problem: opening one writes into the backup

Here is the part that makes the obvious fix — put the two files together and
open it — less obvious than it sounds.

**Opening a WAL-mode database creates `-wal` and `-shm` files beside it, and
they remain after the connection closes.** This happens even when the
connection is strictly read-only, because a read-only reader still needs the
shared memory wal-index to find its way around the log.

So the naive fix writes new files into the user's backup folder. If you are
reading somebody's backup for any purpose where it matters that you did not
alter it — evidence, forensics, or simply a promise you made in your own
documentation — you have just broken that, in a way that shows up in a folder
listing with a timestamp on it.

SQLite has an answer, the `immutable=1` URI parameter, which tells it to skip
the sidecars entirely. It is not always reachable: URI filenames must be
enabled both at compile time and at open time, and a binding compiled without
them takes the URI as a literal file name, so the open simply fails. Check
before relying on it.

## What works

Never open a WAL-mode database where it lies.

1. Locate the database and its `-wal` in the backup, by their two hashes.
2. Copy both into a scratch folder of your own, **under real, paired names** —
   `sms.db` and `sms.db-wal` — because pairing by name is the whole point.
3. Checkpoint the **copy**: `PRAGMA wal_checkpoint(TRUNCATE)`. Never the
   original. For many people that backup is the only copy of these messages in
   existence, and a checkpoint is a write.
4. Optionally `PRAGMA journal_mode = DELETE` on the copy afterwards, which
   leaves it in rollback-journal mode so every later read of it is inert.
5. Read the copy.

Name the scratch folder after something that changes when the backup does —
the size and modification time of the source files work well — so a refreshed
backup cannot be served from a stale snapshot. That bug has the same shape as
the one this whole note is about: correct-looking output, silently out of date.

## It is not only `sms.db`

The same applies to every SQLite database in a backup you intend to read,
including `Library/AddressBook/AddressBook.sqlitedb` in `HomeDomain`
(`31bb7ba8914766d4ba40d6dfb6113c8b614be442`), which is where the names that
turn a phone number into a person come from. It has a `-wal` too, and contacts
added shortly before the backup are exactly the ones that go missing.

---

CC BY 4.0, so quote it freely and link back. One of [the notes](README.md) on
Apple's message format, written while building
[ChatExport](https://getchatexport.com).
