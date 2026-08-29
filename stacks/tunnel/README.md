tunnel
======

[cloudflared](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/): an outbound-only tunnel that publishes selected services without
opening any inbound port on your router.

## Setup

1. Create a tunnel in the Cloudflare Zero Trust dashboard
2. Copy the tunnel token into `CLOUDFLARE_TUNNEL_TOKEN`
3. Define public hostnames in the dashboard, pointing at the proxy or directly
   at a container name on the shared network

Routing lives in Cloudflare, not in this compose file. Adding a hostname needs
no redeploy.

## Overlap to be aware of

A tunnel, a reverse proxy and dynamic DNS can each provide external access. If
cloudflared handles everything reachable from outside, dynamic DNS in the `dns`
stack may be redundant.
