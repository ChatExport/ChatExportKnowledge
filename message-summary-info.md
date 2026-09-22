# `message_summary_info`, and the words an edited message used to say

iOS 16 gave Messages an Edit button and an Undo Send button. Both leave
`message.text` holding the **current** wording, so from that column alone an
edited message can be marked `(edited)` and nothing more.

The superseded words live in exactly one place: the `message_summary_info`
column, next to the message they belong to. This note is how to get them out.

> Last checked: **2026-09-22**. Apple documents none of this. The layout below
> is observed from real backups through a decoder that runs in production, and
> any of it can change in an iOS release.

## Where it is and what it is

Column `message_summary_info` on the `message` table in `sms.db`. A BLOB, and
**its first six bytes are the ASCII string `bplist`**: this is a binary
property list, not a typedstream. That matters, because the blob one column
over, `attributedBody`, *is* a typedstream and needs a completely different
decoder. Check the magic before you parse, so a surprise costs you nothing.

It is NULL on every message that was never edited or unsent, which is almost
all of them.

## The keys

All three are optional and any of them can be absent.

| Key | Name | What it holds |
|---|---|---|
| `ec` | edit contents | The superseded wordings. The whole point of this note |
| `rp` | retracted parts | Part indices the sender unsent with Undo Send |
| `otr` | original text range | Not needed to rebuild a transcript |

### `ec`, and the shape that catches people out

```
ec = { "0": [ { d: <date>, t: <attributedBody blob> }, ... ],
       "1": [ ... ] }
```

Four things about it:

1. **The keys are part indices, as strings.** A message can have several parts
   and each keeps its own history.
2. **Each array entry is one SUPERSEDED version, oldest first.** The current
   wording is not in here. It is in `message.text`, or in `attributedBody` when
   `text` is NULL, which since iOS 16 is most of the time. See
   [`attributedBody.md`](attributedBody.md).
3. **`t` is an `attributedBody` blob in the very same typedstream format the
   `message` column uses.** Whatever you already wrote to decode that column
   applies here unchanged. This is the one piece of good news in the whole blob.
4. **`d` is when that wording was replaced**, not when the message was sent.

### The date has two representations, and you must accept both

`d` arrives **either as a plist date, already absolute, or as a raw Apple
timestamp**, depending on the iOS build that wrote the backup.

Bet on one and every edit written by the other kind of build lands on
2001-01-01, silently, in a document nobody re-checks. Accept a `Date` as it
comes and convert a number through the Apple epoch, the same conversion the
`date` column needs (see the README for the 2001 epoch and the nanosecond
question).

### Sort the part keys numerically

Dictionary key order out of a plist parser is insertion order, and a lexical
sort puts `"10"` before `"2"`. On a message with more than nine parts that
reads the history out of order. Sort the keys as numbers, falling back to a
string compare for anything that is not one.

## `rp`: the parts that were unsent

A non-empty `rp` array means the sender used Undo Send on at least one part.

**The words are not here, and they are not anywhere.** Undo Send removes them
from both devices. What survives is the fact that something was sent at that
minute and then withdrawn, and that fact is worth printing: a transcript that
silently drops the row hides an event, and one that invents the text is worse.

## Failure behaviour worth copying

This blob is best-effort by nature, and the useful discipline is that **a
message you cannot fully decode still exports with its current text**:

- Check the `bplist` magic before handing the bytes to a parser.
- Drop an entry that has neither text nor a date. It carries no information,
  and keeping it renders as a blank "previously:" line.
- A blob that will not parse costs that one message its history. It must not
  abort the conversation. The person on the other end usually has a real need
  and exactly one backup.

## What this blob will not tell you

- **When the message was first sent.** That is `message.date`.
- **Anything about edits made before iOS 16.** The feature did not exist, and
  neither did the column's contents.
- **Anything about the other party's copy.** This is what this phone recorded.
- **The words of an unsent part.** See above. There is no trick for this and
  guesses about it do not belong in these notes.

---

Written by the author of [ChatExport](https://getchatexport.com), which prints
these earlier versions under the current wording when the option is on. The
format is Apple's and is not ours. Corrections are welcome and the convention
for them is in the [README](README.md).
