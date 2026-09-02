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

Every URL carries an admin password, so how you address a Pi-hole decides
whether that password crosses a wire.

The Pi-hole on the same host is reachable by container name over the shared
network. That traffic never leaves the machine, so plain HTTP is right for it:
encrypting it would buy nothing and cost a dependency.

A Pi-hole on another host is different — its password does cross the network,
so use its HTTPS port. Pi-hole serves its own certificate there, which no CA
vouches for, so `NEBULA_SYNC_SKIP_TLS` has to be on. That is weaker than a real
certificate and still far better than sending the password in the clear.

A reverse proxy does not help here. It would sit on the host this runs on, so
the hop it cannot encrypt is exactly the one that leaves the machine.

## Knowing it still works

A container check is the wrong signal here. This process keeps running on its
own schedule, so it stays up while every sync fails, and the monitor stays
green while the spare quietly drifts.

Use the webhooks instead, pointed at a push monitor: an ordinary check asks "did
it answer", a push monitor asks "did it report in", which is the question that
matters for something that only acts once an hour. Point both the success and
failure URL at the same monitor and all three failure modes surface. A failed
sync marks itself down through `?status=down`. A sync that stopped running, a
removed container or a dead host produce no ping at all, and the monitor falls
over once its window passes. Give that window some slack over the interval in
`CRON`, so a single slow run is not an alert.

Address the monitor over the shared network rather than through a reverse
proxy. This is the one thing that tells you the rest is broken, so it should
depend on as little as possible — a proxy or a public name would make the
alarm rely on the very DNS this stack maintains.

## Gravity

A full sync copies the adlist configuration, not the blocklist built from it.
Without `RUN_GRAVITY` the replica agrees on paper and still blocks the old set,
which is the kind of difference you only discover when you need the spare.
