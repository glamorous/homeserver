dns-sync
========

[nebula-sync](https://github.com/lovelaze/nebula-sync) copies one Pi-hole's
configuration to the others, so a second resolver is a real spare rather than a
machine that merely answers.

Running two resolvers only helps if they answer the same way. Otherwise a
device that fails over to the spare suddenly sees different blocking, and a
rule you added an hour ago is missing.

## Run it on exactly one host

This is the reason it is a stack of its own rather than part of `dns`, which
every resolver host deploys. Two instances would push to the same replicas,
doubling the work for nothing, and if they were configured with different
primaries they would overwrite each other's results in turn.

Which host it runs on does not matter, and it need not be the Portainer primary.
Put it on the machine holding the configuration you actually maintain.

## Direction is destructive

Replicas are overwritten. Before switching a primary, check that it is a
superset of what the replicas hold — anything that exists only on a replica is
gone after the first run. The Pi-hole API answers this without logging in to
each interface:

    curl -sk 'https://<host>:<port>/api/domains' -H "X-FTL-SID: <session>"

## Addressing

The Pi-hole on the same host is reachable by container name over the shared
network, which keeps its password off the wire. A Pi-hole on another host needs
its address and published port.

Note that Pi-hole's own HTTPS port serves a self-signed certificate, so
`NEBULA_SYNC_SKIP_TLS` has to stay on for any replica reached that way.

## Gravity

A full sync copies the adlist configuration, not the blocklist built from it.
Without `RUN_GRAVITY` the replica agrees on paper and still blocks the old set,
which is the kind of difference you only discover when you need the spare.
