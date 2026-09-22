# How iOS stores messages, and how to read a backup

Notes on Apple's on-device message database and on the layout of an iPhone
backup. They document **Apple's format**, not any particular program.

Written because working this out took a long time and none of it is secret,
only scattered. If you are building something that reads `sms.db`, this should
save you the weeks.

> Last checked: **2026-07-29**, against backups from iOS 11 through iOS 26.
> Where a claim is inferred from observed data rather than documented, it says
> so.

## Contents

| File | Subject |
|---|---|
| `README.md` | The message database: tables, columns, what changed between iOS versions, and Apple time |
| [`attributedBody.md`](attributedBody.md) | Why the `text` column is often NULL since iOS 16, and how to get the string out |
| [`manifest-structure.md`](manifest-structure.md) | Backup folder layout, the SHA1 file naming rule, `Manifest.db` |
| [`encrypted-backups.md`](encrypted-backups.md) | Keybag, key derivation, per-file keys |
| [`what-cannot-be-recovered.md`](what-cannot-be-recovered.md) | The honest list of what a backup does not contain |
| [`rsmf.md`](rsmf.md) | The RSMF container: the outer email, the manifest schema, what it cannot carry |

---

## The database

Messages live in a SQLite database at `Library/SMS/sms.db` in `HomeDomain`.
Historically it was called `chat.db` on macOS and the two names are used
interchangeably in most writing about this. The layout is the same.

The tables that matter:

| Table | What it holds |
|---|---|
| `message` | One row per message, plus one row per reaction and per system event |
| `chat` | One row per conversation, 1:1 or group |
| `handle` | One row per counterpart address: a phone number or an email |
| `attachment` | One row per file sent or received |
| `chat_message_join` | Which messages belong to which conversation |
| `chat_handle_join` | Which participants belong to which conversation |
| `message_attachment_join` | Which attachments belong to which message |
| `chat_recoverable_message_join` | iOS 16 and later: messages in Recently Deleted |

The join tables are the first thing that surprises people. A message does not
carry a conversation id: you reach it through `chat_message_join`. A group chat's
participants are not a column: they are rows in `chat_handle_join`.

### Columns of `message` worth knowing

| Column | Meaning |
|---|---|
| `ROWID` | Local integer id, used by the join tables |
| `guid` | Stable identifier, and what replies and reactions point at |
| `text` | The message text. **Often NULL from iOS 16 onwards.** See [`attributedBody.md`](attributedBody.md) |
| `attributedBody` | Blob holding the text when `text` is NULL |
| `handle_id` | Counterpart, into `handle.ROWID`. 0 on some outgoing rows |
| `is_from_me` | 1 outgoing, 0 incoming |
| `date` | When the message was sent. Apple time, see below |
| `date_delivered` | When it reached the other device. 0 or NULL when never confirmed |
| `date_read` | Read timestamp. Asymmetric, see below |
| `date_edited` | iOS 16 and later. 0 or NULL means never edited |
| `date_retracted` | iOS 16 and later. Non-zero means the sender unsent it |
| `associated_message_guid` | For a reaction: the target, as `p:0/GUID` or `bp:GUID` |
| `associated_message_type` | 0 normal. 2000 to 2005 add a reaction. 3000 to 3005 remove one |
| `thread_originator_guid` | iOS 14 and later. The message this one replies to |
| `message_summary_info` | Blob. Edit history and retraction details |
| `item_type` | 0 for a real message. Other values are system events such as a group rename |
| `service` | `iMessage` or `SMS` |

Two consequences of that table which catch most implementations:

**Reactions are messages.** A tapback is a row in `message` with a non-zero
`associated_message_type` pointing at another message's GUID. Count rows
naively and every conversation is inflated. Render rows naively and every
"Liked a message" appears as its own line in the transcript.

**System events are messages too.** `item_type` other than 0 marks things like
someone being added to a group or a group being renamed. They are worth showing,
but not as if somebody said them.

### `date_read` is not symmetrical

The same column means two different things depending on direction.

- On an **outgoing** row, it is set only when the recipient has read receipts
  enabled. An empty value is not evidence the message went unread. It usually
  means the other person never turned the feature on.
- On an **incoming** row, iOS records when the owner of this device read it,
  regardless of anyone's settings.

Presenting both as "read at" is a misreading. It is the single most common error
in tools that export this data.

---

## Apple time

Timestamps do not count from the unix epoch. They count from
**2001-01-01T00:00:00Z**, the Apple or Mac absolute time epoch. The offset is
`978307200` seconds.

The unit also changed. Older backups store **seconds**. From roughly iOS 11 the
same columns hold **nanoseconds**.

There is no version field to consult, so the practical approach is to look at the
magnitude. A value above `1e12` cannot be a number of seconds, because that would
be somewhere around the year 33,000. Nanosecond timestamps land around `7e17`.
So `1e12` separates the two cleanly without knowing the iOS version at all.

Worked conversion:

```
raw = 707521234000000000        # nanoseconds, because it exceeds 1e12
seconds = raw / 1e9             # 707521234
unix = seconds + 978307200      # 1685828434
```

Zero, negative and absent values occur in real databases, in draft rows and
damaged rows. Treat them as "no timestamp" rather than as 2001, and certainly
rather than throwing: one bad row should not end a hundred thousand message
export.

---

## Telling the iOS generation apart

Apple ships no reliable schema version number in this database. The workable
approach is to look for **marker columns**, because each generation of features
added a distinctive one.

| Evidence in `message` | Generation |
|---|---|
| `thread_originator_guid` **and** `date_edited` present | iOS 16 and later |
| `attributedBody` or `associated_message_guid` present, `thread_originator_guid` absent | roughly iOS 11 to 15 |
| Neither | older, or an unrecognised layout |

The presence of the table `chat_recoverable_message_join` is a separate signal
and indicates the Recently Deleted store, iOS 16 and later. Check separately
whether it has a `delete_date` column, because it is not always populated.

`PRAGMA table_info('message')` is enough to gather all of this in one query.

**Degrade rather than refuse.** An unrecognised layout is not a good reason to
show the user nothing. Reading what you can identify and reporting what you
could not beats an error message, because the person on the other end usually
has a real need and only one backup.

---

## What these notes do not cover yet

- Attachment storage layout on the device beyond the paths recorded in the
  database.
- The `message_summary_info` blob format in detail, beyond its role in edit
  history.
- Group chat renames and participant changes over time, which are recorded as
  system events but not reconstructed here into a timeline.
- SMS specific fields that differ from iMessage rows.
- Anything about Messages in iCloud, which is a sync service and not a file
  format.

---

## Corrections

Corrections are the reason this is public. Every claim here is either read from
a real backup or marked as inferred, and the inferred ones are where an outside
eye is worth the most.

An issue that can be acted on says three things: which iOS version the backup
came from, what you observed, and how you observed it, such as a
`PRAGMA table_info` dump, a row count or the query you ran. Half the statements
in these files are version dependent by their nature, so "this is wrong" with no
version attached cannot be checked against anything.

A file that is unclear is also worth an issue. These notes exist to save
somebody a fortnight, and a paragraph that has to be read twice is failing at
that.

### Two things an issue will not get an answer to

Worth saying before anybody spends time writing one up.

**Reading a backup that is not yours.** Not a partner's, not an ex-partner's,
not an employee's. It does not change anything that the file is already on your
computer, or that the situation is unfair, or that you are certain what the
messages say. Whether you may read a particular backup is a question about
consent and about the law where you live, and an issue thread is not where it
gets settled.

**A way past a forgotten password.** There is none. That is the subject of
[`encrypted-backups.md`](encrypted-backups.md) rather than an omission from it,
and guesses about it are not going to be hosted here.

Everything else about the format is fair game, from anybody.

## Licence

CC BY 4.0. Quote it, paste it into your own documentation, translate it, build
on it. The one condition is attribution, and a link back to this repository
satisfies it.

## Who wrote this

Krzysztof Kowalski, Bokart, Warsaw. These notes came out of building
[ChatExport](https://getchatexport.com), a Windows program that turns an iPhone
backup into a document you can file with a court or hand to a lawyer.

That program is closed source, and this repository is deliberately not a way
into it. There are no function names, no file names and no design decisions from
it anywhere above. What is described is Apple's format, which is not ours, was
never secret, and is only scattered: half documented, half folklore, and wrong
in enough places that verifying it against real backups took longer than reading
it did. That is the part nobody else needs to repeat.

## Trademarks

Apple, iPhone, iMessage, iTunes and macOS are trademarks of Apple Inc. These
notes are unofficial, they describe a file format as observed from real backups,
and they reproduce no Apple code or documentation. Nothing here is affiliated
with, endorsed by or sponsored by Apple Inc.
