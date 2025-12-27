HomeServer
============
Docker compose repo for home server with a couple of handy services.
Included services:
- [Heimdall](https://github.com/linuxserver/Heimdall): Application dashboard and launcher
- [Pi-hole](https://pi-hole.net): A black hole for Internet advertisements
- [InfluxDB](https://www.influxdata.com): Database that stores and queries any type of time series data
- [Node-RED](https://nodered.org): Browser-based flow editing
- [DuckDNS](https://www.duckdns.org): free service which will point a DNS (sub domains of duckdns.org) to an IP of your choice

## Prerequisites

When using on a "normal" computer, you should make sure that:
- docker is installed
- docker is started at runtime

When running this on a Raspberry Pi, you can follow the steps below, otherwise, go to [step5](#5-install-portainer).

### 1. Install Raspbian Lite
Install Raspbian Lite on the SD-card/SSD-drive through [Raspberry Pi Imager](https://www.raspberrypi.com/software/)

Edit `/boot/config.txt` and add this line:

	gpu_mem=16

### 2. Boot up your device and update
After booting your device, run the basic updates/upgrades:

	sudo apt-get update
	sudo apt-get upgrade

### 3. Install Docker
Start the Docker installer

	curl -sSL https://get.docker.com | sh

Set Docker to auto-start

	sudo systemctl enable docker

### 4. Enable Docker client
The Docker client can only be used by root or members of the docker group. Add pi or your equivalent user to the docker group:

	sudo usermod -aG docker pihome

### 5. Install Portainer
Install portainer volume

	docker volume create portainer_data

Install portainer

	docker run -d -p 8000:8000 -p 9443:9443 --name portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v /my-computer/my-folder/docker-data/portainer/data:/data portainer/portainer-ce:lts

## Restore backups from configs, ...



## Install containers through portainer

1. Go to your portainer instance: https://YOUR-SERVER-IP:9443
2. Go to stacks
3. Click "Add stack"
   - Add a name "homeserver"
   - Choose repository as build method
     - Repository URL: https://github.com/glamorous/homeserver
     - Repository reference: refs/heads/master
     - GitOps updates: true
     - Fetch interval: 5m
   - Environment variables
       - Upload the secrets.env and adjust where needed (only first time, otherwise manual adding)
4. Deploy the stack

## Setting up InfluxDB (first time only!)

1. Go to http://YOUR-SERVER-IP:8086 and follow onboarding
2. Choose a username and password
3. Choose "home" as your organisation name
4. Choose "homeassistant" as your bucket name
5. Click "Quick start"

### Create token(s) for applications

Instead of username and password, we need tokens for applications that will use InfluxDB (such as Home assistant)
1. Click "API tokens" on first menu item
2. Click "Generate API token" - "Custom API token"
3. Click a name, for example "homeassistant" and choose the correct rights for the selected bucket
   - Home assistant will need write access!
4. Copy the token so you can add it to your config of the application

#### Configure HomeAssistant for usage with InfluxDB

Open HomeAssistant `configuration.yaml` file and add:

```
influxdb:
  api_version: 2
  ssl: false
  host: localhost
  port: 8086
  token: !secret INFLUXDB_TOKEN
  organization: !secret INFLUXDB_ORG
  bucket: !secret INFLUXDB_BUCKET
```

Open HomeAssistant `secrets.yaml` file and add:

```
INFLUXDB_TOKEN: YOUR_API_TOKEN_FROM_PREVIOUS_STEP
INFLUXDB_ORG: home
INFLUXDB_BUCKET: homeassistant
```

## Connecting Home Assistant with Node-Red

### Home Assistant

1. Create "node-red" user with an administrator role in Home Assistant
2. Login with new user
3. Create a long-lived access token

### Node-RED

1. Click Menu -> Manage Palette -> Install
2. Search for node-red-contrib-home-assistant-websocket and install
3. Configure/Add Home Assistant server by selecting node and "Add a server" en fill on details and live access token.

## Enabling Memory support (only for Raspberry Pi)

Verify that memory support is enabled on the Raspberry Pi:

```
docker info | grep -i cgroup
```

If you can see "WARNING" and the message that there is no memory support, you should enable it by adding opening `/boot/cmdline.txt` (or somewhere else located such as `/boot/firmware/cmdline.txt`) by

```
sudo nano /boot/cmdline.txt
```

add the following options to the end of the line

```
cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1
```

After setting it, you should reboot the device by `sudo reboot`.

## Setting DNS for host device (only for Raspberry Pi)

```
sudo nano /etc/dhcpcd.conf
```

Inside the config file, at the end of the file, add

```
interface eth0
static domain_name_servers=1.1.1.1 1.0.0.1
nohook resolv.conf
```

Remove `/etc/resolv.conf` and add manually the domain name servers

```
sudo rm /etc/resolv.conf
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf > /dev/null
echo "nameserver 1.0.0.1" | sudo tee -a /etc/resolv.conf > /dev/null
```

Lock the file

```
sudo chattr +i /etc/resolv.conf
```

Restart the service

```
sudo systemctl restart dhcpcd
```
