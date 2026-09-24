# Which transport carried the message: iMessage, SMS, MMS and RCS

The `service` column of `message` records how a message travelled. It is one of
the oldest columns in the database and one of the least carefully read, because
for a decade it only ever held two values and code written in that decade
treats it as a boolean.

That stopped being true in iOS 18.

## The values

| Value | Meaning |
|---|---|
| `iMessage` | Apple's own service |
| `SMS` | Carrier messaging |
| `RCS` | Rich Communication Services, iOS 18 and later |

An empty or NULL value means the database did not record a transport, which is
not the same as "none". Old rows leave it blank, and a row that arrived through
an import rather than through the radio has no transport to record. Treat blank
as unknown and say so, rather than defaulting it to `SMS` because the bubble
was green.

A value you do not recognise should be passed through verbatim rather than
dropped. Apple has added one value to this column within the last two years and
there is no reason to assume it has finished. Reporting an unfamiliar string is
a small embarrassment; silently presenting the message as something it was not
is a larger one.

## The green bubble is no longer SMS

Before iOS 18, the colour of a bubble answered the question: blue is iMessage,
green is the carrier. From iOS 18 a green bubble may be either SMS or RCS, and
the phone does not distinguish them visually in any way a screenshot preserves.

This matters more than it sounds. A screenshot of a green bubble used to carry
an implicit claim about how the message arrived. It no longer does, and only
the database still knows. If you are building anything that reproduces a
conversation, this column is now the only place the answer lives.

## It is a property of the message, not of the conversation

The single most common mistake with this column is reading it once per chat.

A conversation falls back. Two people on iMessage lose data coverage, the
thread continues over SMS, and later returns. A conversation with an Android
contact may run over RCS for a year and drop to MMS for one picture the network
would not carry. There is no rule that a `chat` row has one transport, and
plenty of real threads mix all three.

So the honest options are to label each message, or to label none of them. What
does not work is naming the conversation's platform from its first row, or from
its most common one, and letting that stand over messages it does not describe.

If you want to state what a whole thread contained, collect the distinct values
across its messages and list them — "iMessage, RCS" — rather than picking one.

## MMS

MMS does not get a value of its own. An MMS is carrier messaging and appears
here as `SMS`, with its pictures in the `attachment` table exactly like any
other attachment.

> Inferred, not documented: this is what every backup examined here shows, and
> Apple publishes nothing about the column. If you have seen a row where
> `service` holds something else for a multimedia carrier message, that is
> worth an issue.

So there is no reliable way to answer "was this an SMS or an MMS" from this
column alone. What distinguishes them in practice is whether the row has
attachments, which is a different question wearing the same coat.

## RCS is only there if the phone kept it

RCS support arrived in iOS 18, and the rows appear in `sms.db` alongside
everything else. The part worth knowing before you promise anybody a complete
transcript: what reaches the local database is what the phone stored locally.
RCS is a carrier and provider service with its own retention behaviour, and a
backup can only ever contain what was on the device when it was made.

This is the same caveat as everywhere else in these notes rather than a special
weakness of RCS, but it catches people out, because RCS feels like iMessage and
iMessage is reliably local.

---

Checked against the same set as the rest of these notes: backups from iOS 11
through iOS 26.
