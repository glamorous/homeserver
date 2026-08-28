Running on a Raspberry
======================

Written for a Debian-based OS on a Raspberry board. Other single-board
computers work the same way; only the boot configuration paths differ.

## 1. Install the operating system

Install a Lite image on the SD card or SSD with the Raspberry Pi Imager.

For a headless server, reduce the memory reserved for video. Add to
`/boot/config.txt` (on newer releases `/boot/firmware/config.txt`):

    gpu_mem=16

## 2. Update

    sudo apt-get update
    sudo apt-get upgrade

## 3. Install Docker

    curl -sSL https://get.docker.com | sh
    sudo systemctl enable docker

## 4. Allow your user to run Docker

The Docker client is limited to root and members of the `docker` group:

    sudo usermod -aG docker <your-user>

Log out and back in for the group change to take effect.

## 5. Enable memory cgroups

Container memory limits are silently ignored without this. Check:

    docker info | grep -i cgroup

If a warning about missing memory support appears, edit `/boot/cmdline.txt` (or
`/boot/firmware/cmdline.txt`) and append to the single line:

    cgroup_enable=cpuset cgroup_enable=memory cgroup_memory=1

Reboot afterwards.

## 6. Fix the host's own DNS

When this host runs Pi-hole, it must not resolve through itself — that breaks
name resolution during container restarts. Point the host at an external
resolver.

Add to `/etc/dhcpcd.conf`:

    interface eth0
    static domain_name_servers=1.1.1.1 1.0.0.1
    nohook resolv.conf

Then replace and lock the resolver configuration:

    sudo rm /etc/resolv.conf
    echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf > /dev/null
    echo "nameserver 1.0.0.1" | sudo tee -a /etc/resolv.conf > /dev/null
    sudo chattr +i /etc/resolv.conf
    sudo systemctl restart dhcpcd

## 7. Connect to Portainer

This host is managed from the Portainer instance on another machine. Install
only the agent here, then add the environment in Portainer:

    docker run -d -p 9001:9001 --name portainer_agent --restart=always \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v /var/lib/docker/volumes:/var/lib/docker/volumes \
      portainer/agent:latest

## 8. Data directory

Create the directory you will use as `BASE_DIR`, for example `/docker-data`,
and make sure your user owns it.

## Shared network

Every stack attaches to one external Docker network. Create it once, before
deploying the first stack:

    docker network create homeserver

Nothing creates this for you; deploying a stack without it fails.
