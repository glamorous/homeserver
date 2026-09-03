backup-store
============

[rest-server](https://github.com/restic/rest-server) holds the repository that
every host writes its snapshots into. Exactly one machine runs this — pick the
one with the disk and, ideally, the one whose whole-disk backup then carries the
repository off the premises.

## Access

`--private-repos` keeps each client to its own path, so one host cannot read or
overwrite another's snapshots. Accounts live in `.htpasswd` inside the
repository directory; create it with any bcrypt-capable tool:

    htpasswd -B -c /path/to/backup/repo/.htpasswd <user>

Deliberately not `--append-only`. It would stop a compromised client from
erasing history, which is worth something, but it also stops the clients from
pruning — and a repository that can never forget grows until the disk does not.
Turn it on if you also arrange to prune by hand.

Confidentiality does not depend on any of this: restic encrypts before it
sends, so the repository is unreadable without the password even to whoever
holds the disk.

## Put the repository on another disk

`BACKUP_REPO_PATH` is deliberately not derived from `${BASE_DIR}`. A repository
on the same disk as the data it protects survives a deleted file and a bad
upgrade, but not the failure that actually loses everything at once. Point it at
a second disk, and preferably one that is copied onward from there, so the chain
does not end on this machine.

If that path is on a removable volume, a run while it is unmounted fails rather
than silently starting an empty repository — restic only creates one when told
to `init`. The check below is what turns that failure into something you hear
about.

Wherever it ends up, the client on this same host must exclude it, or every run
copies the backups into the backups.

## Checking

A repository can be verified, which is the one thing a folder of copied files
cannot offer. `restic check` on its own confirms the structure; reading a slice
of the pack files is what catches a disk corrupting data underneath you, so
`RESTIC_CHECK_ARGS` asks for a percentage rather than metadata alone.

The result goes to a push monitor on both outcomes. That is on purpose: a check
that fails tells you something is wrong, and a check that stops running tells
you nothing at all unless something is expecting to hear from it.

Note that the scheduling variables in this image are mutually exclusive — one
container runs one job — which is why the check is a separate service.

## Initialising

Once, from any client:

    restic -r rest:http://<user>:<password>@<host>:<port>/ init

The clients will not create the repository themselves; that is on purpose, so a
typo in an address cannot quietly start a second, empty one.
