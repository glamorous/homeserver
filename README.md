HomeServer
==========

Docker Compose stacks for a self-hosted home server, deployed through Portainer
from this repository.

Each stack under `stacks/` is a self-contained unit with its own compose file,
documentation and environment files. Stacks are host-agnostic: a host simply
deploys the stacks it needs and ignores the rest.

## Stacks

| Stack | Services | Purpose |
|---|---|---|
| `proxy` | [Nginx Proxy Manager](https://nginxproxymanager.com) | Reverse proxy and TLS certificates |
| `tunnel` | [cloudflared](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) | Outbound tunnel for external access |
| `dns` | [Pi-hole](https://pi-hole.net), [DuckDNS](https://www.duckdns.org) | Network-wide DNS filtering and dynamic DNS |
| `dns-sync` | [nebula-sync](https://github.com/lovelaze/nebula-sync) | Copy one resolver's configuration to the others |
| `apps` | [Heimdall](https://github.com/linuxserver/Heimdall), [Node-RED](https://nodered.org), [InfluxDB](https://www.influxdata.com) | Dashboard, flow automation, time series storage |
| `monitoring` | [Uptime Kuma](https://uptime.kuma.pet), [Speedtest Tracker](https://docs.speedtest-tracker.dev), [Diun](https://crazymax.dev/diun) | Availability, throughput and image update alerts |
| `ai` | [Open WebUI](https://docs.openwebui.com) | Chat interface for a locally running model |
| `auth` | [authentik](https://goauthentik.io) | One account per person, shared across services |

## Deployment roles

Not every host runs every stack. A typical two-host setup:

| Stack | Primary host | Secondary host |
|---|:---:|:---:|
| `proxy` | yes | no |
| `tunnel` | yes | no |
| `dns` | yes | yes |
| `dns-sync` | yes | no |
| `apps` | yes | yes |
| `monitoring` | yes | yes |
| `ai` | yes | no |
| `auth` | yes | no |

Running `dns` and `monitoring` on both hosts is deliberate: two resolvers survive
one host going down, and two monitoring instances can watch each other.

`dns-sync` is the opposite: exactly one host runs it, whichever holds the
configuration you maintain. It writes to the other resolvers, so a second
instance would either duplicate that work or undo it.

## One Portainer, many hosts

Every stack on every host is managed from a single [Portainer](https://www.portainer.io)
instance. Only one host runs Portainer itself; the others run nothing but its
agent and are added there as extra environments.

| Role | Runs | Manages |
|---|---|---|
| Primary | Portainer server | Its own stacks and those of every secondary |
| Secondary | Portainer agent only | Nothing locally; it is driven from the primary |

A secondary host therefore needs no Portainer login, no repository clone and no
manual Compose commands. You add its environment once, then deploy stacks to it
from the same interface as everything else. The platform guides cover both
roles.

Pick the most reliable machine as primary: when it is down you cannot deploy
anywhere, although already running containers keep going untouched.

## Prerequisites

Docker must be installed and start automatically. See the guide for your
platform:

- [hosts/macos.md](hosts/macos.md)
- [hosts/raspberry.md](hosts/raspberry.md)

### Shared network

Containers reach each other by name across stacks through one external network.
It is **not** created automatically: `external: true` means Compose expects it to
already exist, and deploying without it fails with an unknown network error.

Create it once per host, before the first stack:

    docker network create homeserver

It survives redeploying and removing stacks, so this is a one-time step per
host. It is listed as a step in both platform guides.

## Deploying a stack in Portainer

1. Stacks -> Add stack -> Repository
2. Repository URL: this repository
3. Repository reference: the branch you deploy from
4. Compose path: `stacks/<name>/compose.yaml`
5. GitOps updates: enable, with a webhook or a fetch interval
6. Environment variables: load `stacks/<name>/secrets.env` and replace every
   value with the real one. They are stored in Portainer, never in this
   repository.
7. Deploy

Repeat per stack and per host. Each combination is its own Portainer stack.

## Environment variables

Each stack splits its configuration in two:

| File | In Git | Read by | Contains |
|---|---|---|---|
| `.env` | yes, with real values | Compose, automatically | Ports, timezone, uid/gid — nothing sensitive |
| `secrets.env` | yes, with placeholders | nobody | Template to load into Portainer |

`secrets.env` is never read at deploy time. It exists so you can load it into a
Portainer stack in one go and then replace each placeholder with the real value.
Those live in Portainer's database from then on.

Anything host-specific belongs in `secrets.env` rather than `.env`, `BASE_DIR`
above all: it differs per machine and should not be published here.

## Where data lives

Every service writes to a bind mount under `${BASE_DIR}`, a path you choose per
host. Nothing of value lives inside a container, so redeploying is safe and the
data directory is what your backup needs to cover.
