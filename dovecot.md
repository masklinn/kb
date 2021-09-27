# Mailbox, IMAP, Dovecot

## Target audience

People who find themselves with a maildir (dump) and try to make heads or tail of it

## Motivation

The internet has fragments of interest but nothing really clear, and the dovecot docs are currently down

## Maildir

The core description is simple enough: at the core, a maildir is a folder representing a mailbox using three sub-folders, `new`, `tmp`, and `cur`:

* `tmp` is the mail server's scratch pad, and more generally equivalent to `/tmp`
* `new` is for messages which haven't yet been given to *mail clients* messages generally go into `new` after having been created in `tmp`
* `cur` is for messages which have been fetched at least once

Unless the mailbox is very high traffic, odds are good `tmp` and `new` will be empty.

Each subdir then contains a bunch of files, one message per files. Most of the name is essentially random and irrelevant so we'll ignore it, the only interesting bit is the *trailer*, which is the *info* segment: a colon (`:`), the character `2`, a comma (`,`), and a bunch of letters. Each letter is a flag.

The original specs defines 6 flags:

* `P`(assed): the message has been resent/bounced/forwarded
* `R`(eplied): the message has been replied to
* `S`(een): the message has been seen
* `T`(rashed): the message has been moved to the trash
* `D`(raft): the message is a draft
* `F`(lagged): no intrinsic meaning

A maildir can have *folders*, these are directories prefixed by `.` and siblings of `cur`. They can be nested but their nesting is logical rather than physical e.g. `.foo.bar` is the folder `foo/bar`.

## IMAP

There are pages and pages of IMAP spec, the only bits of interest currently are the notions of *flags* and *keywords*.

IMAP has *system flags*, these are prefixed by `\` and defined by RFC 2060:

* `\Seen`
* `\Answered`
* `\Flagged`
* `\Deleted`
* `\Draft`
* `\Recent`

5 of those map fine to Maildir flags (Maildir also has `P` while IMAP has `\Recent`).

Keywords are anything attribute which does not start with a `\`. They are server-specified, but a server may allow clients to define new keywords by returning `\*` as one of the `PERMANENTFLAGS`.

Following RFC 5788 (but also inheriting the choices from RFC 3503), all `$`-prefixed keywords are also reserved for [official registration](https://www.iana.org/assignments/imap-jmap-keywords/imap-jmap-keywords.xhtml#Alexey_Melnikov). Only non-prefixed keywords can have arbirary semantics.

Note an odd conflict: common use keywords can get registered officially, leading to duplicate behaviour. That is the case of `Junk`/`NotJunk` and `$Junk`/`$NotJunk`, the latter replaces the former after registration, as a result it can be useful to support both.

Also `INBOX` is the primary mailbox (the maildir root), maildir folders would correspond to IMAP mailboxes.

## dovecot

dovecot is an IMAP server which uses maildir but extends it a lot in order to support IMAP semantics (for which the base maildir is insufficient). A dovecot maildir is recognizable because the root directory has *a lot* of `dovecot`-prefixed files at the root (next to the `cur`, `tmp`, and `new` directories).

The extensions [are documented](https://wiki.dovecot.org/MailboxFormat/Maildir) but right now the entire dovecot docs / wiki is down so I figure I'll write down what I understand.

The most relevant bits for reading a dump are:

### `dovecot-keywords`

As a rule, dovecot maildir files have more flags than standard. dovecot uses lowercase letters to denote its extensions, but because clients can define their own keywords *the mapping is dynamic and mailbox-specific*.

`dovecot-keywords` is an ASCII file containing a list (one item per line) of a decimal number (0-25) and an IMAP keyword. The number is the index of a letter (a-z) used to specify the corresponding keyword on the file.

So for instance given
```
0 $NotJunk
1 $Junk
2 NotJunk
3 JunkRecorded
4 Junk
```
a file could be marked `,Sac`, meaning it has been `S`een, and is flagged as `$NotJunk` and `NotJunk`. But given
```
0 $NotJunk
1 NotJunk
2 $Junk
3 JunkRecorded
4 Junk
```
the exact same file would be `,Sab` instead.

### `dovecot-uidlist`

Each IMAP message is supposed to have a unique id, in a format different than maildir filenames. This is where dovecot maintains the mapping. `uidlist` has a line of <whocares> followed by lines of:

* the message id
* `W` and the "virtual size" (RFC822.SIZE)
* possibly other fields
* `:` and the filename, probably excluding the flags
