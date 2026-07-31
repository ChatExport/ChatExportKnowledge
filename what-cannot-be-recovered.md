# What cannot be recovered from an iPhone backup

> Part of [a set of notes](README.md) on how iOS stores messages and how a
> backup is laid out. It documents Apple's format, not any particular program.
>
> Last checked against iOS 11 through 26 behaviour: 2026-07-29.

Most writing about iPhone message recovery is written by people selling
recovery. This file is the opposite list: the cases where the answer is no, and
what is actually left when it is.

It is worth reading before you spend a weekend on a case that cannot work, and
it is worth quoting when someone asks you whether a message can be "got back".

---

## The rule that explains most of it

**A backup is a copy of one moment, not a history of the phone.**

Everything below follows from that single sentence. If a message was not on the
device when the backup ran, no amount of reading that backup will produce it,
because it was never written into the file. There is no deleted-items area of a
backup, no free space to carve, and no journal of what the phone used to hold.

Two consequences people find counterintuitive:

- An **older** backup is often more useful than a new one. It predates the
  deletion.
- Making a **fresh** backup to "recover" something you deleted yesterday
  usually destroys your last chance, if it overwrites the older backup that
  still contained it.

---

## The table

| Situation | Recoverable | What is left |
|---|---|---|
| Message present on the phone when the backup ran | Yes | Everything: text, time, attachments, receipts |
| Message deleted before the backup ran | **No** | Nothing at all. It is not in the file |
| Message deleted after an older backup was made | Yes, from that older backup | The older backup predates the deletion |
| Message deleted within roughly the last 30 days, iOS 16 and later | Usually | Full text and time, plus the date it was deleted. See below |
| Message deleted more than roughly 30 days ago | **No** | The row is purged from the device, so it is not in any backup made afterwards |
| Message the sender unsent | **Text: no** | The fact that something was sent at that minute and then withdrawn, and by whom |
| Earlier wording of an edited message | Usually, iOS 16 and later | Sometimes only the fact of the edit. See below |
| Phone lost, sold or broken, backup still on the computer | Yes | The backup is the source. The phone is not needed |
| Attachment deleted on the phone before the backup | **No** | The attachment row survives with its file name, size and type. The file itself is absent |
| Messages only in iCloud, no local backup | **Not from a backup** | Nothing to read. Make a local backup first |
| WhatsApp, Signal, Messenger, Telegram | **Not from here** | Separate databases in separate app containers, with separate rules |
| Encrypted backup whose password is lost | **No** | Nothing. There is no escrow and no recovery path |

---

## Recently Deleted, and what "roughly 30 days" means

iOS 16 added a Recently Deleted folder for Messages. A deleted message is not
erased immediately: it is moved out of the live thread and kept for a limited
period, then purged.

In the database this shows up as a separate join table,
`chat_recoverable_message_join`, rather than a flag on the message row. A row
reachable only through that table is one that the phone would not show you in
Messages today. The deletion date is recorded alongside it, which is why a
document produced from such a row can state when it was deleted rather than
presenting it as an ordinary message.

Three honest qualifications:

1. **The window is approximate.** Apple describes it as up to 30 days, and it is
   not a precise countdown you can rely on to the hour. Treat it as "roughly a
   month, and do not gamble on day 29".
2. **It does not exist before iOS 16.** On an older backup these rows are simply
   absent, and their absence is not a fault in whatever is reading the file.
3. **Once purged, it is gone from every later backup.** A backup made in
   February cannot contain something the phone purged in January.

Anything presented as "recovered" from this store should be labelled as such in
whatever document it ends up in. It is a message the phone no longer shows, and
presenting it as an ordinary line of a conversation misdescribes it.

---

## Unsent messages: the text really is gone

"Undo Send", also iOS 16 and later, is not a local hide. The text is removed
from both devices. Nobody has it: not the sender, not the recipient, not Apple,
not a forensic examiner, and not the person reading the backup.

What survives is the record: a row with a retraction timestamp, marking that a
message existed at that minute and was withdrawn. That is a fact worth keeping,
and it is the reason to record the event rather than skip the line and leave a
silent gap in the sequence.

If a tool shows you the text of an unsent message, something is wrong with your
understanding of what you are looking at. The likely explanation is that the
message was unsent on one device but the backup predates the sync.

---

## Edited messages, and the case people get wrong

iOS 16 and later keeps earlier versions of an edited message in a per-message
summary blob (`message_summary_info`). That is how the previous wording of a
message can be shown under the current one.

The honest part: **an edit flag with no history attached is a normal outcome,
not a fault.** iOS sometimes records that a message was edited without keeping
what it said before. Anything reading that data should say "edited" and stop,
rather than implying that the absence of a previous version means there was
never one.

---

## Read receipts are not symmetrical, and this trips people up

Both directions store a "read" timestamp, and the two mean different things.

- **On a message you sent**, a read timestamp exists only if the other person
  has read receipts turned on. No timestamp does not mean unread. It usually
  means they never enabled the feature.
- **On a message you received**, iOS records when you read it, locally, whether
  or not anybody has receipts enabled.

Presenting the two identically is a misreading of the data. "No read receipt on
an outgoing message" is not evidence that it went unread, and it should never be
described as if it were.

Delivery is a separate field again, and its absence means only that the device
never recorded a delivery confirmation.

---

## Attachments: the row survives the file

Attachment metadata lives in the database. The bytes live as separate files
inside the backup, found by a hashed name derived from the on-device path.

If the user deleted the photo on the phone, or the file was never included in
the backup, the lookup returns nothing while the row remains. The result is a
record that says what was sent, its file name, size and type, and the time,
without the image itself.

That is not a failure to report. A conversation showing that a photo was sent at
19:04, with its name and size, is a different and weaker thing from the photo,
and the difference should be visible rather than papered over.

---

## iCloud is not a backup you can read

Messages in iCloud is a sync service. It keeps devices in step with each other.
It is not a file you can open, and it is not the same as a local backup.

If a conversation exists only in iCloud and there is no local backup, there is
nothing on the computer to read. The fix is to make a local backup first, which
pulls the current state of the phone down into a file. That still does not
recover anything deleted before you did it.

---

## Encrypted backups: no password, no data

An encrypted iPhone backup is encrypted properly. There is no escrow copy, no
vendor override and no support process that unlocks it. A forgotten password
means the backup is a folder of noise.

This cuts both ways and is worth stating plainly: it is the reason an encrypted
backup is the safer thing to have sitting on a shared computer for months, and
the reason to write the password down somewhere you will still have it.

---

## What this file does not cover

- **Physical forensic acquisition.** Some of what is impossible from a backup is
  possible from a device image obtained by an examiner with the right tools and
  the right authority. That is a different discipline with different rules, and
  nothing here should be read as a claim about its limits.
- **Anything about admissibility.** This is a description of what data exists.
  Whether a given document is accepted anywhere is a legal question, and this
  file does not answer it.
- **Deleted-row carving inside the SQLite file.** Freed pages can sometimes hold
  fragments after a delete. It is unreliable, it is not what any of the above
  describes, and treating it as a recovery route sets an expectation that will
  usually disappoint.

---

CC BY 4.0, so quote it freely and link back. One of [five files](README.md) on
Apple's message format, written while building
[ChatExport](https://getchatexport.com).
