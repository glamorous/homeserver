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

## Do not point a client at this host's own directory

The repository lives under `${BASE_DIR}` like everything else, so the client on
this same machine must exclude it. Without that, every run copies the backups
into the backups.

## Initialising

Once, from any client:

    restic -r rest:http://<user>:<password>@<host>:<port>/ init

The clients will not create the repository themselves; that is on purpose, so a
typo in an address cannot quietly start a second, empty one.
