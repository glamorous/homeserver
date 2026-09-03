apps
====

[Node-RED](https://nodered.org) for flow automation and
[InfluxDB](https://www.influxdata.com) for time series storage.

Documentation: [Node-RED](https://nodered.org/docs/),
[InfluxDB v2](https://docs.influxdata.com/influxdb/v2/).

A dashboard used to live here. The identity provider lists the applications a
person is allowed to open, which is the same thing filtered by who is looking,
so a separate dashboard that shows everything to everyone had nothing left to
add. See [../auth/README.md](../auth/README.md).

Each service keeps its own state under `${BASE_DIR}`, so the same stack can run
on several hosts with entirely different contents.

## InfluxDB, first run only

1. Open InfluxDB on `INFLUXDB_PORT` and follow the onboarding
2. Choose a username and password
3. Choose an organisation name
4. Choose a bucket name
5. Click "Quick start"

### Application tokens

Applications authenticate with tokens, not with your password:

1. API tokens -> Generate API token -> Custom API token
2. Name it after the application and grant it access to the right bucket only
3. Write access is required for anything sending measurements

### Do not point a scraper at your data bucket

InfluxDB can scrape Prometheus metrics, including its own at `/metrics`. If such
a scraper writes into the bucket that holds your real measurements, that bucket
fills up with internal bookkeeping — hundreds of kilobytes every few seconds,
tens of gigabytes over a year. Give scrapers a dedicated bucket with a short
retention, or leave them off.

### Retention and granularity

A bucket with infinite retention grows forever. For long-term history, keep raw
data for a bounded period and use a downsampling task to aggregate older points
into a second bucket at a coarser interval. That is built in and needs no extra
tooling.

## Connecting a home automation platform to InfluxDB

Add to its configuration:

    influxdb:
      api_version: 2
      ssl: false
      host: influxdb
      port: 8086
      token: !secret INFLUXDB_TOKEN
      organization: !secret INFLUXDB_ORG
      bucket: !secret INFLUXDB_BUCKET

Use the container name as host when both run on the same Docker network,
otherwise the host address and published port.

## Connecting Node-RED to a home automation platform

1. Create a dedicated user with an administrator role on that platform
2. Sign in as that user and create a long-lived access token
3. In Node-RED: Menu -> Manage Palette -> Install, and add
   `node-red-contrib-home-assistant-websocket`
4. Add a server in a node's configuration and paste the token

Credentials are stored encrypted in `flows_cred.json`, using a secret kept
elsewhere in `/data`. Moving flows to another host means copying the whole data
directory, not just the flow files, or the stored credentials cannot be
decrypted.
