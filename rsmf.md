# RSMF, the container eDiscovery platforms ingest

RSMF is the Relativity Short Message Format. It exists because chat is not
email and not documents: a reviewer needs participants, timestamps, reactions,
edits and attachments as *fields*, not as a PDF somebody has to read and re-key.

These notes describe the file as observed while writing one, checked against
Relativity's own published sample rather than against prose. **They are not
Relativity's specification and do not replace it.** Where something here is
inferred from a sample rather than documented, it says so.

> Last checked: **2026-09-22**, against schema `rsmf_schema_2_0_0`.

## The outer file is an email, and that surprises everybody

A `.rsmf` file is **an RFC 5322 message**, not a zip. Open one in a text editor
and you see mail headers.

Its last MIME part is a base64 attachment called **`rsmf.zip`**, and that
archive is where all the meaning lives. Everything before it is a courtesy: a
text part that a platform's importer never reads, but that a person who was
handed the file and has no platform does.

The MIME boundary in Relativity's own generator is the literal string
`RSMFEML`. Nothing requires you to match it, but matching it removes one
variable if a file ever has to be compared against theirs.

## Inside `rsmf.zip`

Two kinds of entry:

| Entry | What it is |
|---|---|
| `rsmf_manifest.json` | The whole conversation as structured data |
| everything else | The attachments the events reference, **named exactly as the manifest names them** |

That last rule is the one that breaks imports. An attachment's `id` in the
manifest is not an opaque identifier: it **is the file name inside the archive,
extension included**. Rename the file and the attachment stops resolving.

Put attachments in as the **original bytes**. Do not convert HEIC, do not
downscale, do not re-encode to make a picture friendlier. A reviewing platform
wants the file the phone stored, and prettifying evidence is exactly the wrong
instinct. (This is the opposite of what a PDF path should do, which is why it
is worth stating.)

## The manifest

Schema version `2.0.0`. Six object shapes, and which fields are required is
the part worth writing down, because the validator is unforgiving and the
error messages are not:

| Object | Required | Optional |
|---|---|---|
| root | `version`, `participants`, `conversations`, `events` | |
| participant | `id` | `display`, `email`, `avatar` |
| conversation | `id`, `platform`, `participants` | `display`, `type` |
| event | `type` | `id`, `participant`, `conversation`, `timestamp`, `body`, `direction`, `deleted`, `reactions`, `attachments`, `edits` |
| edit | `participant` | `timestamp`, `previous`, `new` |
| attachment | `id` | `display`, `size` |

Every id an event refers to has to be declared at the root. A `participant` on
an event or an edit that was never listed in the root `participants` array is
the commonest way to produce a file that looks fine and imports empty.

### `type` is not only `message`

The schema's own second type is **`disclaimer`**: a notice *about the record*
rather than something a person said. If your export needs to state that it is
partial, watermarked or produced under some limit, that statement belongs in a
`disclaimer` event and not smuggled in as a message from a fake participant.

### Edits are the field nothing else has

`edits` is an array, oldest first, and each entry can carry `previous` and
`new`. This is the only common format that has a place for **the wording an
edited message had before it was edited**.

Getting those earlier versions out of `sms.db` in the first place is a separate
problem, and the hard part:
[`message-summary-info.md`](message-summary-info.md) is the blob iOS keeps them
in, and [`attributedBody.md`](attributedBody.md) is how the text comes out of
each one.

### Attachments render inline, conditionally

`display` is what a reviewer sees. A `display` ending in `.png`, `.jpg`,
`.jpeg` or `.gif` renders inline in the viewer; anything else appears as a file
to open. So the extension on `display` is a presentation decision, while the
extension on `id` is a correctness one.

## Size

Relativity recommends **no more than 10,000 events in one file**. This is a
recommendation rather than a hard limit in the schema, and a long conversation
crosses it easily: split by date range rather than truncating, and say in a
`disclaimer` event that you did.

## What this format cannot carry

The honest list, in the spirit of
[`what-cannot-be-recovered.md`](what-cannot-be-recovered.md):

- **Nothing the source did not have.** A WhatsApp "Export chat" file has no
  read receipts, no reactions and no reply threading, so an RSMF built from one
  cannot invent them.
- **No time zone that was never recorded.** If the source carries local times
  with no zone, the timestamps you write are an interpretation, and the file
  should say whose.
- **No proof about the device.** RSMF is a transport container. It says what
  the messages were, not where they came from or that anything is authentic.
  That is what a hash of the exported file and a preserved original are for.

## Reading one without a platform

A `.rsmf` file is designed to be ingested, which means the person who receives
one often cannot open it at all. ChatExport writes the address of a
browser-based viewer into the text part of the message for exactly that reason:
<https://getchatexport.com/tools/rsmf-viewer/>. It runs locally in the page.

---

Written by the author of [ChatExport](https://getchatexport.com), a paid Windows
program that writes this format among four others. The format is Relativity's
and is not ours. Everything above describes the container, which anybody is free
to implement.
