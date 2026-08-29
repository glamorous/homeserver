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

Its state lives under `${BASE_DIR}`. If that directory is empty on start, every
watched image counts as newly seen and you get one burst of notifications.

A `:latest` tag does not update by itself. Diun tells you an update exists; you
still decide when to pull it.
