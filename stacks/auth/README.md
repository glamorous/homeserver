auth
====

[authentik](https://goauthentik.io) as the single place where accounts live, so
a household has one password instead of one per service.

Four containers: the server, a worker, PostgreSQL and Redis. Budget around a
gigabyte of memory. Everything persists under `${BASE_DIR}/authentik`.

## First run

1. Deploy the stack and give it a minute; the server waits for the database
2. Open `http://<host>:${AUTHENTIK_PORT_HTTP}/if/flow/initial-setup/`
3. Create the first administrator — this is your break-glass account
4. Point a proxy host at it, then use the hostname from then on

## Two ways to connect a service, and they are not equal

**OpenID Connect**, when the service supports it. The application learns who
you are, keeps its own roles, and every family member gets a real account
inside it. Use this wherever it exists.

**Forward authentication** at the reverse proxy, for services that have no
concept of accounts. authentik decides *whether* someone gets in; the service
still sees a single anonymous session. Everyone admitted shares the same view,
so grant it through a group and only to people you would have handed the
password to anyway.

The difference matters for a household. Behind forward auth, a family member is
not a user of that service — they are simply past its door.

## Keep a local administrator everywhere

Every service that gets an identity provider should keep one local account that
does not depend on it. When authentik is down, its database is broken, or a
misconfigured flow locks you out, that account is the way back in. This is also
why the reverse proxy's own admin port stays reachable without the proxy: an
authentication loop that can only be fixed through the thing it broke is not a
loop you want to discover at night.

## Not given the Docker socket

The official example mounts it into the worker so authentik can run outposts as
containers it manages. The embedded outpost in the server covers proxy
providers on a single host, and handing an internet-facing service control over
the Docker daemon is a poor trade for that convenience.
