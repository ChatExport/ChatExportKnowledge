# Attachments: the rows, the paths, and the character that is not a character

> Part of [a set of notes](README.md) on Apple's message format. Documents the
> format, not any particular program.
>
> Last checked against backups from iOS 11 to 26.

Photos, videos, voice notes, PDFs and everything else sent in a conversation
live in the `attachment` table, tied to messages through
`message_attachment_join`. The table is small and the joins are obvious. What
is not obvious is everything below.

## The columns

| Column | Meaning |
|---|---|
| `ROWID` | Local id, used by `message_attachment_join` |
| `filename` | Where the file sat **on the device**, beginning with `~/`. May be NULL |
| `transfer_name` | The human-readable name the file travelled under, e.g. `IMG_4471.HEIC` |
| `mime_type` | As recorded at transfer. May be NULL, and may be wrong |
| `total_bytes` | Size in bytes |

`filename` and `transfer_name` are both worth reading, because either can be
absent and they answer different questions. `filename` is where to look for the
bytes; `transfer_name` is what to call the file in front of a reader. Falling
back from one to the other gives a usable name on far more rows than either
alone.

## The path, and the domain that changes under you

A `filename` looks like this:

```
~/Library/SMS/Attachments/ab/11/0A1B2C3D-4E5F-.../IMG_4471.HEIC
```

Two levels of two-hex-character sharding, then one folder per attachment,
named with the attachment's own GUID, then the file.

To find that file inside a backup you need the SHA1 naming rule from
[`manifest-structure.md`](manifest-structure.md), and **two** adjustments, not
one:

1. **Strip the leading `~/`.** The hash is computed over the path without it.
   This one is at least widely known.
2. **The domain is `MediaDomain`, not `HomeDomain`.** The messages database
   lives in `HomeDomain`; its attachments do not.

So, for the path above:

```
SHA1("MediaDomain-Library/SMS/Attachments/ab/11/0A1B2C3D-.../IMG_4471.HEIC")
```

Carrying the domain over from `sms.db` because the attachments belong to the
messages is the mistake worth warning about. It fails silently: every hash is
well-formed, every lookup misses, and the reasonable conclusion is that the
backup contains no attachments at all. It contains all of them.

A lookup that genuinely finds nothing is still a normal result — see
[`manifest-structure.md`](manifest-structure.md), "A missing file is a normal
result". The row survives with the name, size and type after the bytes are
gone, so a record of what was sent outlives the file.

## U+FFFC, the character that is not a character

This one costs people an afternoon.

Where an attachment sits in a message, Apple embeds **U+FFFC OBJECT REPLACEMENT
CHARACTER** into the message's text, in `text` and in `attributedBody` alike.

- A photo with no caption has text consisting of exactly that one character.
- A captioned photo has `caption` followed by it.
- Several attachments put several of them in, at their positions.

Two consequences.

**It is not a blank message.** A row whose text is `"￼"` is a photo, and
treating empty-after-trim as "nothing to show" drops it from the transcript
entirely. The attachment join is where the content is.

**It has no glyph.** Nothing in a normal font stack draws it, so rendering it
through a browser or a PDF engine produces the fallback box — often the
literal letters `OBJ` in a rectangle, scattered through an otherwise clean
document. Strip it before rendering, and take what remains, not what the
stripping leaves behind: `"caption￼"` should become `caption`, and
`"￼"` should become nothing at all rather than an empty string that later
code mistakes for a real blank.

## HEIC is the default, and browsers cannot read it

Photos taken on an iPhone are HEIC, not JPEG, unless the owner changed a
setting most owners never open.

No mainstream browser decodes HEIC, and neither does Chromium, which means
anything rendering these attachments through web technology — an HTML export, a
PDF produced by a headless browser, a preview pane — gets a blank rectangle
rather than an error. It fails quietly, and it fails only on the photos, which
is exactly the way a bug survives to a release.

Converting to JPEG first is the practical answer. `libheif` is the reference
implementation and there are WASM builds of it, so this does not have to mean
a native dependency.

The same caution applies less dramatically elsewhere: `mime_type` is what the
sending side declared, and a file whose extension, declared type and actual
magic bytes disagree is not rare. If you intend to show the file, sniff the
bytes.

## What this note does not cover

- Stickers, and how they differ from attachments proper.
- Thumbnail and preview files stored beside the original on the device.
- The attachment columns that exist and are not listed above, several of which
  appear to be transfer bookkeeping and are not read here.

---

CC BY 4.0, so quote it freely and link back. One of [the notes](README.md) on
Apple's message format, written while building
[ChatExport](https://getchatexport.com).
