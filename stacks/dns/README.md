dns
===

Pi-hole for network-wide DNS filtering, and DuckDNS to keep a hostname pointed
at a changing public IP address.

Run this on more than one host. Two resolvers mean a single host going down
degrades resolution instead of breaking it.

## Listening mode

`PIHOLE_LISTENING_MODE` decides which queries Pi-hole answers:

| Value | Behaviour |
|---|---|
| `local` | Only sources on a directly attached subnet |
| `single` | One interface, no source filtering |
| `all` | All interfaces, no source filtering |

`local` breaks in a bridged container: clients on your LAN are not in the
container's subnet, so their queries are refused. Use `single` when the
container is attached to one network — the default here. Switch to `all` only
when the container joins more than one network, because `single` then binds to
just one of them.

`all` makes Pi-hole answer anyone who can reach port 53. On a LAN that is fine.
Never expose port 53 to the internet: that turns it into an open resolver,
usable for amplification attacks.

## Two Pi-holes do not synchronise

Blocklists, whitelists and local DNS records drift apart on their own, and then
clients get different answers depending on which resolver they happened to pick.
If you run more than one, add a synchronisation tool that pushes configuration
from one designated source to the others.

Which instance is "primary" is not a Pi-hole setting. It only matters in two
places: the order your router hands out resolvers over DHCP, and the direction
you configure in that synchronisation tool.

## Host DNS

A host running Pi-hole must not resolve through itself. See the platform guide
for your host.
