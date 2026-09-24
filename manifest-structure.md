# How an iPhone backup is laid out

> Part of [a set of notes](README.md) on Apple's message format. Documents the
> format, not any particular program.
>
> Last checked: **2026-07-29**, on Windows, against backups from iOS 11 to 26.

## Where backups live on Windows

| Path | Written by |
|---|---|
| `%USERPROFILE%\Apple\MobileSync\Backup` | Apple Devices, the Microsoft Store app |
| `%APPDATA%\Apple Computer\MobileSync\Backup` | Classic iTunes for Windows |

Both roots can exist at once, and **the same device can appear under both**,
identified by the same folder name. That happens after a migration from iTunes to
Apple Devices, or when an interrupted backup leaves a partial folder in one root
while a complete older one sits in the other. Anything scanning for backups
should collect candidates from both roots and then decide per device, rather than
taking the first hit and stopping.

On macOS the equivalent is `~/Library/Application Support/MobileSync/Backup`.

## Inside a device folder

The folder name is the device identifier, forty hexadecimal characters. Newer
identifiers are sometimes written with a hyphen after the eighth character; both
forms name the same device.

Four files keep their real names:

| File | Contents |
|---|---|
| `Manifest.db` | SQLite index of every file in the backup: domain, relative path, size, flags, and per-file metadata |
| `Manifest.plist` | Properties of the backup as a whole. Carries `IsEncrypted`, and on an encrypted backup the `BackupKeyBag` blob |
| `Info.plist` | Device description: model, name, iOS version, phone number, last backup date. Readable even on an encrypted backup |
| `Status.plist` | How the transfer ended, including `SnapshotState` |

Everything else is a two character folder holding forty character files.

## The file naming rule

Backup files are renamed to the **SHA1 of `"<domain>-<relative path>"`**, hex
encoded.

The messages database is at `Library/SMS/sms.db` in `HomeDomain`, so:

```
SHA1("HomeDomain-Library/SMS/sms.db")
  = 3d0d7e5fb2ce288813306e4d4636395e047a3d28
```

That value is identical in every iTunes-style backup ever made, which is why it
appears in every article on this subject. Verified against a real backup on
2026-07-29.

### Where the file then sits

Check all of these, most standard first:

```
<backup>/<first two chars of hash>/<hash>     iOS 10 and later, sharded
<backup>/<hash>                               pre iOS 10, flat
<backup>/Snapshot/<first two chars>/<hash>    some tooling and recent iOS
<backup>/Snapshot/<hash>
```

The `Snapshot` subfolder is the live backup tree for backups created by some
third party tooling and by recent iOS. Assuming a single layout is the most
common reason a lookup fails on somebody else's machine.

### The `~/` trap

Attachment paths stored inside `sms.db` begin with the on-device home shorthand:

```
~/Library/SMS/Attachments/ab/11/GUID/IMG_1.heic
```

The hash is computed over the path **without** that prefix. Strip the leading
`~/` before hashing, or you will produce names that are not in the backup and
conclude that attachments are missing when they are not.

The prefix is only half of it: attachments are in **`MediaDomain`**, not the
`HomeDomain` that holds `sms.db`. Both corrections are needed and each fails the
same silent way. See [`attachments.md`](attachments.md).

### A missing file is a normal result

A lookup that finds nothing usually means the file was never backed up. This is
common for attachments the user had already deleted on the phone. The database
row survives with the file name, size and type, so the record of what was sent
remains even though the bytes are gone. Report that difference rather than
hiding it.

## Manifest.db

An ordinary SQLite database listing every file with its domain and original
path. It is the index that turns "the messages database" into a forty character
file name without anyone needing to know the SHA1 rule, and it is the fastest way
to see what a backup actually contains.

Its `Files` table carries a per-file metadata blob. On an encrypted backup that
blob holds the file's protection class and its own wrapped key. See
[`encrypted-backups.md`](encrypted-backups.md).

On an encrypted backup, `Manifest.db` is itself encrypted. Until the password is
supplied you cannot see what the backup contains at all.

## Telling a finished backup from an abandoned one

`Status.plist` has `SnapshotState`, and `finished` is what you want to see.

**That flag on its own is not trustworthy.** It has been observed being written
at the start of a resumed transfer, minutes before any data arrived, so a folder
can claim to be finished while it is still filling.

A check that holds up: require the flag **and** the presence of a file that
arrives late in the transfer. The messages database is a good choice for exactly
that reason. A folder satisfying only the first is still being written, and
opening it produces a confusing "database missing or corrupted" error that has
nothing to do with the database.

### Nor is an exit code

If you drive a backup from a tool rather than from Apple's own software, the
same caution applies to how the run ended.

Observed on an iPhone 16 running iOS 26.5, with an encrypted backup: the tool
connected, drew a progress bar at 0% for five minutes, and exited with status 0.
On disk it had rewritten `Info.plist` and `Manifest.plist` and nothing else.
`Status.plist` and `Manifest.db` still carried timestamps from twelve days
earlier. A clean exit, a folder that looked like a backup, and not one message
newer than a fortnight in it.

The reason the disguise works is which files get touched when:

| File | When it is written |
|---|---|
| `Info.plist`, `Manifest.plist` | During the opening handshake, **before any data moves** |
| `Manifest.db` | Grows as files actually arrive |
| `Status.plist` | Rewritten when a run ends |

So the two files a handshake writes are exactly the two that cannot tell you
anything, and they are also the two whose freshness is easiest to notice. Check
the modification times of `Status.plist` and `Manifest.db` against when you
started the run; if neither moved, nothing was copied, whatever the exit code
said.

Allow a couple of seconds of tolerance when you compare. Filesystem timestamps
are coarser than a program's clock — FAT rounds to two seconds and network
shares can be worse — so a marker written immediately after the run began can
legitimately stamp fractionally before it.

## What these notes do not cover

- Backup fragmentation across multiple snapshots, beyond the `Snapshot` folder
  itself.
- The full column set of `Manifest.db`'s `Files` table.
- Domains other than `HomeDomain` and `MediaDomain`.
- iCloud backups, which are not files on disk and are not covered here at all.

---

CC BY 4.0, so quote it freely and link back. One of [the notes](README.md) on
Apple's message format, written while building
[ChatExport](https://getchatexport.com).
