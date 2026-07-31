# attributedBody: why the text column is empty and where the message actually is

> Part of [a set of notes](README.md) on Apple's message format. Documents the
> format, not any particular program.
>
> Last checked: **2026-07-29**.

If you read `message.text` from a modern `sms.db` and find that a large share of
rows come back NULL, nothing is broken. Since iOS 16 the text of a message is
frequently not in that column at all. It is in `message.attributedBody`, as a
binary blob.

This is the single most common defect in tools that export iPhone messages. The
symptom is distinctive: **the conversation exports, but a scattering of messages
come out blank**, usually the more recent ones, usually the ones with a link, a
mention or any formatting. Users report it as "some messages are missing" and it
is very hard to diagnose from the outside.

## What the blob is

`attributedBody` holds an `NSAttributedString` serialised in Apple's legacy
**typedstream** format.

That is worth stating precisely, because the wrong assumption costs a day:
typedstream is **not** a binary property list, and it is **not** an
`NSKeyedArchiver` plist. Off-the-shelf plist parsers do not read it. On macOS the
Foundation classes will decode it for you. Everywhere else you are on your own.

## Getting the string out

There are two honest routes.

**A full typedstream decoder** is the correct answer and a real piece of work.
If you write one, you get the attribute runs as well, which is how you would
recover which part of the text was a link or a mention.

**A targeted read** is what most implementations do, and in practice it is
reliable, because the layout immediately around the message string is stable
across the releases where this matters.

The observed shape is:

```
... "NSString" 0x01 0x94 0x84 0x01 0x2B <length> <utf8 bytes> ...
```

Reading it:

1. Find the ASCII bytes `NSString` in the blob.
2. Scan forward a short window, five or six bytes, for the byte `0x2B`. That is
   the typedstream tag meaning "an inline string follows". Do not hardcode the
   gap: it varies slightly between iOS versions, which is exactly the kind of
   thing that works on your phone and fails on somebody else's.
3. Read the length that follows. It is a single byte if small. A leading `0x81`
   means a two byte little-endian length follows, and `0x82` means a four byte
   little-endian length.
4. Read that many bytes and decode as UTF-8.

Three details that decide whether this works on real data:

- **There is usually more than one `NSString`.** The typedstream carries class
  names as part of the object graph. Try each occurrence in turn and take the
  first that yields a decodable string, rather than assuming the first is the
  right one.
- **Edited messages and messages with attributes shift the layout.** A sensible
  fallback is to take the longest printable run in the blob that is not a
  typedstream class name artefact. It is not pretty, and it recovers text that
  would otherwise be lost.
- **Never let this throw.** A blob you cannot decode should produce a null and a
  logged note, not an exception. A single odd message must not end an export of
  a hundred thousand. Check for the Unicode replacement character `U+FFFD` in
  your result as a signal that you decoded the wrong bytes rather than
  succeeding.

## Which messages are affected

There is no reliable rule that lets you skip the check. `text` and
`attributedBody` are not mutually exclusive, and both can be populated.

The workable order is:

1. If `text` is non-empty, use it.
2. Otherwise, if `attributedBody` is non-empty, extract from it.
3. Otherwise the message genuinely has no text, which is normal for a message
   that carried only an attachment.

Anything that reads only step 1 will look correct on a test conversation from
2019 and lose messages on any recent thread.

## How to tell whether your implementation is wrong

Export a long modern conversation and count messages with no text. Then compare
against the number of rows where `text IS NULL AND attributedBody IS NOT NULL`:

```sql
SELECT COUNT(*) FROM message
WHERE text IS NULL AND attributedBody IS NOT NULL;
```

On an iOS 16 or later database, that number is usually substantial. If your
export has that many blank lines, this is why.

## Why this matters beyond tidiness

If the export is going to be read by somebody else, a silently dropped message is
worse than a visibly broken one. A conversation with three blank gaps invites the
question of what was removed and by whom, and the honest answer, "a parsing bug",
is not an answer anybody wants to give afterwards.

---

CC BY 4.0, so quote it freely and link back. One of [five files](README.md) on
Apple's message format, written while building
[ChatExport](https://getchatexport.com).
