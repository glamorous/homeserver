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

Lose it and the backups are unreadable. This is encryption, not a login you can
reset, and restic offers no unencrypted mode — so the only decision left to you
is where the key lives.

Encryption earns its place here regardless of where the disk sits. The snapshots
contain the container platform's own database, and that holds every stack's
environment: the admin passwords, the API tokens, the provider secrets. Add the
identity provider's user table and the certificate manager's private keys and
the repository stops being a pile of data and becomes the keys to everything, in
one file, on a disk that will eventually leave the building — to a repair shop,
a cupboard, or a bin.

### Do not keep the key inside the backup

The trap is quiet and it is easy to walk into: the natural place to put the key
is with the other stack variables, and those live in a database that this very
backup copies. Then the machine dies, you reach for the snapshots, and the
password you need to open them is inside them.

Keep it somewhere that is not part of what is being backed up. A password
manager, or the operating system's own credential store **on a machine that
this backup does not cover**. Do the same for the store's login, which is
embedded in the repository URL.

One copy on one laptop is better than the loop above and still a single point of
failure. Treat this like any other key you cannot regenerate.

## Restoring

    restic -r <repository> snapshots
    restic -r <repository> restore <id> --target /somewhere

Snapshots carry the hostname, so the two machines stay apart in one repository.

Test this before you need it. A backup nobody has restored is a hypothesis.
