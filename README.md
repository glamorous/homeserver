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
| `proxy` | Nginx Proxy Manager | Reverse proxy and TLS certificates |
| `tunnel` | cloudflared | Outbound tunnel for external access |
| `dns` | Pi-hole, DuckDNS | Network-wide DNS filtering and dynamic DNS |
| `apps` | Heimdall, Node-RED, InfluxDB | Dashboard, flow automation, time series storage |
| `monitoring` | Uptime Kuma, Speedtest Tracker, Diun | Availability, throughput and image update alerts |
| `ai` | Open WebUI | Chat interface for a locally running model |

## Deployment roles

Not every host runs every stack. A typical two-host setup:

| Stack | Primary host | Secondary host |
|---|:---:|:---:|
| `proxy` | yes | no |
| `tunnel` | yes | no |
| `dns` | yes | yes |
| `apps` | yes | yes |
| `monitoring` | yes | yes |
| `ai` | yes | no |

Running `dns` and `monitoring` on both hosts is deliberate: two resolvers survive
one host going down, and two monitoring instances can watch each other.

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
