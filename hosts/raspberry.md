Running on a Raspberry
======================

Written for a Debian-based OS on a Raspberry board. Other single-board
computers work the same way; only the boot configuration paths differ.

This is normally a secondary host: it runs the Portainer agent and is managed
entirely from the primary. See step 7.

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

A board like this is normally a secondary host: it runs the
[Portainer](https://www.portainer.io) agent and nothing else, and every stack on
it is deployed from the primary. You never log in to Portainer here, never clone
this repository here, and never run Compose by hand here.

    docker run -d -p 9001:9001 --name portainer_agent --restart=always \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v /var/lib/docker/volumes:/var/lib/docker/volumes \
      portainer/agent:latest

Then, on the primary host: Environments -> Add environment -> Docker Standalone
-> Agent, and enter `<this-host>:9001`. Keep the agent version equal to the
Portainer server version; a mismatch shows up as unexplained API errors.

From that point on the host appears as an environment, and deploying a stack to
it is the same procedure as for any other environment.

## 8. Shared network

Every stack attaches to one external Docker network. Create it once, before
deploying the first stack:

    docker network create homeserver

Nothing creates this for you; deploying a stack without it fails.

## 9. Data directory

Create the directory you will use as `BASE_DIR`, for example `/docker-data`,
and make sure your user owns it.

## Locally running models

Only relevant if this host runs the `ai` stack. Install
[Ollama](https://ollama.com) natively, not in a container:

    curl -fsSL https://ollama.com/install.sh | sh

The installer sets up a systemd service. To let containers and other machines
reach it, override the bind address:

    sudo systemctl edit ollama

Add:

    [Service]
    Environment="OLLAMA_HOST=0.0.0.0:11434"

Then reload and verify:

    sudo systemctl daemon-reload && sudo systemctl restart ollama
    curl http://localhost:11434/api/tags

**Be realistic about the hardware.** A single-board computer has no usable GPU
and limited memory, so inference falls back to the CPU. Models of a few billion
parameters will run, slowly; anything larger will not fit. Run the `ai` stack on
a machine with real memory and point `OLLAMA_BASE_URL` at that instead.

Note that the API has no authentication whatsoever: binding to `0.0.0.0` exposes
model management to everyone who can reach the port. Keep it on the local
network.
