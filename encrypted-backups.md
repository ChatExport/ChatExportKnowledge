# Encrypted iPhone backups: keybag, key derivation, per-file keys

> Part of [a set of notes](README.md) on Apple's message format. Documents the
> format, not any particular program.
>
> Last checked: **2026-07-29**.
>
> **Scope.** This describes how to open a backup with its password. It is not a
> guide to opening one without it, and there is no such guide to write: the
> derivation is deliberately expensive and there is no escrow.

## What changes when a backup is encrypted

The folder layout does not change. What changes is:

1. `Manifest.plist` sets `IsEncrypted` and carries a `BackupKeyBag` blob.
2. `Manifest.db`, the index of the whole backup, is encrypted.
3. Every file has its own key, wrapped so it can only be unwrapped through the
   keybag.

`Info.plist` stays readable, which is how a device model and iOS version can be
shown before any password is entered.

An encrypted backup also **contains more** than an unencrypted one. Apple only
includes certain categories, saved passwords, Health data and Wi-Fi settings
among them, when the backup is encrypted.

## The keybag

`BackupKeyBag` is a sequence of TLV records: a four character ASCII tag, a four
byte **big-endian** length, then the value.

Header tags:

| Tag | Meaning |
|---|---|
| `TYPE` | Keybag type. `1` is a backup keybag |
| `SALT` | Salt for the classic derivation round |
| `ITER` | Iteration count for that round |
| `DPSL` | Salt for the added SHA-256 round. Present from iOS 10.2 |
| `DPIC` | Iteration count for that round. Present from iOS 10.2 |

After the header come per-class entries. The keybag's **own** `UUID` record comes
first, and each subsequent `UUID` starts a new class entry. Tags that belong to a
class entry rather than the header:

| Tag | Meaning |
|---|---|
| `CLAS` | Protection class number, 1 to 11 |
| `WRAP` | Bitmask. Bit 1 device wrapped, bit 2 passcode wrapped |
| `WPKY` | The wrapped class key. 40 bytes for a 256 bit key |
| `KTYP`, `PBKY`, `SEED`, `WKEY` | Other per-class fields |

Entries whose `WRAP` has the passcode bit set are the ones a password can open.

A truncated trailing record should be ignored rather than treated as fatal. A
damaged keybag ought to surface later as "wrong password or cannot decrypt",
not as a crash.

## Password to key

Two shapes, depending on the age of the backup.

**iOS 10.2 and later**, when `DPSL` and `DPIC` are present:

```
hardened = PBKDF2-HMAC-SHA256(password, DPSL, DPIC, 32 bytes)
key      = PBKDF2-HMAC-SHA1(hardened, SALT, ITER, 32 bytes)
```

**Older backups**, with no `DPSL`:

```
key = PBKDF2-HMAC-SHA1(password, SALT, ITER, 32 bytes)
```

Note the order and the algorithms: the SHA-256 round is a pre-round feeding the
classic SHA-1 round, not a replacement for it.

**Cost.** Real backups use on the order of ten million iterations. That is
seconds of solid CPU for one attempt. It is the security property, not an
inefficiency: barely noticeable once, ruinous at scale. Do the derivation off
your main thread, or whatever UI you have will freeze for the duration.

## Unwrapping

Class keys are protected with **AES Key Wrap, RFC 3394**, using the derived key
as the key-encryption key. The RFC's default initial value `A6A6A6A6A6A6A6A6`
doubles as an integrity check on unwrap.

That check has a useful consequence: **a wrong password fails cleanly** rather
than yielding plausible garbage. You learn immediately, instead of after
exporting a thousand corrupted messages.

## Per-file keys

`Manifest.db`'s `Files` table stores a metadata blob per file, an
`NSKeyedArchiver` object commonly called `MBFile`. What you need from it:

- the protection class,
- the wrapped per-file key,
- the real, unpadded size.

The wrapped key blob has the shape:

```
<4 byte little-endian protection class><wrapped key>
```

Note the endianness difference from the keybag's big-endian TLV lengths. The
same shape is used by `Manifest.plist`'s `ManifestKey`, which is how
`Manifest.db` itself is decrypted.

Unwrap the per-file key with the class key, then decrypt.

## File decryption

**AES-256-CBC, zero initialisation vector, cipher padding disabled.**

The ciphertext is padded up to the block size, so the plaintext must be
truncated to the real length taken from the file metadata. Leaving cipher
padding enabled, or trusting the ciphertext length, gives you trailing bytes that
corrupt a SQLite database in ways that are unpleasant to diagnose.

Order of operations for a single file:

1. Read `Manifest.plist`, confirm `IsEncrypted`, take `BackupKeyBag` and
   `ManifestKey`.
2. Parse the keybag, derive the key from the password, unwrap the class keys.
3. Unwrap `ManifestKey` and decrypt `Manifest.db`.
4. Look up the file, read its class and wrapped key from its metadata.
5. Unwrap that key with the matching class key.
6. Decrypt AES-256-CBC with a zero IV, no padding, truncate to the recorded size.

## What cannot be done

There is no escrow copy, no vendor override, and no recovery process anywhere
that opens an encrypted backup without its password. If the device still exists
and works, the route is to set a new backup password on the device and take a
fresh backup. If the device is gone and the password with it, the backup is
noise.

Recovery services advertising otherwise are describing either a different
problem or a different product.

## What these notes do not cover

- Protection class semantics beyond "some classes are passcode wrapped".
- Keybags other than the backup keybag.
- The full `MBFile` object graph.
- Any timing or side channel consideration, which is out of scope for reading
  your own backup.

---

CC BY 4.0, so quote it freely and link back. One of [five files](README.md) on
Apple's message format, written while building
[ChatExport](https://getchatexport.com).
