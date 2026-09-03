monitoring
==========

[Uptime Kuma](https://uptime.kuma.pet) for availability checks,
[Speedtest Tracker](https://docs.speedtest-tracker.dev) for throughput history,
and [Diun](https://crazymax.dev/diun) for notifications when a newer container
image is published.

Run this on more than one host, so each instance can watch the other. A monitor
that dies together with the thing it monitors reports nothing.

## Speedtest Tracker

`SPEEDTEST_TRACKER_APP_KEY` must be generated once, before the first start:

    echo -n 'base64:'; openssl rand -base64 32

Changing it later makes existing encrypted values unreadable.

## Diun

Diun only reports; it never updates anything. It reads the local Docker socket,
so it sees the containers on its own host only — which is why it belongs in a
stack you deploy everywhere rather than in one central place.

### Watch everything, not just what is labelled

The Docker provider watches nothing by default: without
`DIUN_PROVIDERS_DOCKER_WATCHBYDEFAULT` it only considers containers carrying
the label `diun.enable=true`. That fails quietly — Diun starts, reports
`Found 1 image(s) to analyze`, and you conclude it is working while it is only
watching itself. With the setting on, every running container is watched and a
single container can still opt out with `diun.enable=false`.

Its state lives under `${BASE_DIR}`. If that directory is empty on start, every
watched image counts as newly seen and you get one burst of notifications.

A `:latest` tag does not update by itself. Diun tells you an update exists; you
still decide when to pull it.

### Say which host reported

Every notification ends with the container's hostname, which by default is the
container id — a random string that changes whenever the container is
recreated. Set `HOST_NAME` per host so the message names the machine instead.

## Health checks

Not every service has one, and that is deliberate. Uptime Kuma, Pi-hole,
Node-RED and Open WebUI ship their own. InfluxDB and Speedtest Tracker get one
here, each hitting an endpoint the image can actually reach — both carry
`curl`.

DuckDNS, Diun and cloudflared have none on purpose. The first two have no
listening service, so the only testable thing is whether the process still
exists, which proves nothing about whether the DNS record is current or the
registry was reachable. A green tick that means nothing is worse than none at
all. The cloudflared image contains no shell, so `CMD-SHELL` and `curl` are
both unavailable; monitor it from outside instead, by adding
`--metrics 0.0.0.0:20241` to its command and pointing a check at `/ready`.
