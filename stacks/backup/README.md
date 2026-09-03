backup
======

[restic](https://restic.net) copies each host's data directory into one shared
repository held by the `backup-store` stack. Every host that has something worth
keeping runs this; the store runs on one machine only.

## Why not just let the file-level backup handle it

A whole-disk backup of the machine that keeps the data does cover this
directory, but it has two blind spots this stack closes.

It only covers *that* machine. Anything on another host — often the most
irreplaceable data, because it is the one nobody thinks about — is not in it at
all.

And it copies databases while they are open. That yields a crash-consistent
copy, which PostgreSQL and SQLite are built to recover from, so it is usually
fine. Usually is doing a lot of work in that sentence, and you find out which
kind of copy you have on the day you need it. restic at least tells you: `restic
check` verifies a repository, which a folder of copied files cannot do.

## What gets copied

Everything under `${BASE_DIR}`, minus the store's own repository — backing that
up would copy the backups into themselves.

That is deliberately the whole directory rather than a list of services. The
convention throughout these stacks is that anything worth keeping lives under
`${BASE_DIR}`, so a new service is covered the day it is deployed instead of the
day someone remembers to add it here.

## The repository password is the whole thing

Lose it and the backups are unreadable — this is encryption, not a login you can
reset. Keep a copy somewhere that does not depend on the machines being backed
up. A password stored only in the thing you are protecting is not a backup of
anything.

## Restoring

    restic -r <repository> snapshots
    restic -r <repository> restore <id> --target /somewhere

Snapshots carry the hostname, so the two machines stay apart in one repository.

Test this before you need it. A backup nobody has restored is a hypothesis.
