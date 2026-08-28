Running on macOS
================

## Container runtime

Any Docker-compatible runtime works. Two things matter more than which one you
pick:

- **It must survive a reboot unattended.** Most macOS container runtimes are
  user applications that only start once someone logs in graphically. If the
  machine reboots while nobody is there, containers stay down. If this host
  serves DNS or a database, that is a real availability limit — plan for it
  rather than discovering it during an outage.
- **No GPU passthrough.** macOS does not expose Metal to containers. Anything
  that needs the GPU has to run natively on the host, outside Docker.

## Data directory

Pick a path and use it as `BASE_DIR` in every stack, for example a
`docker-data` directory inside your home folder. Each service creates its own
subdirectory there.

If the directory sits somewhere macOS protects, such as Documents or Desktop,
remote shells cannot read it until you grant Full Disk Access to `sshd` under
System Settings -> Privacy & Security. Local access is unaffected.

## Locally running models

The `ai` stack expects a model server running natively on the host, reachable
from containers at `host.docker.internal`. Install it as a normal desktop
application rather than in a container, so it can use the GPU, and enable the
option that exposes it to the network — otherwise it only listens on loopback
and the container cannot reach it.

## Portainer

Install once, outside the stacks, so that it survives redeploying anything:

    docker volume create portainer_data
    docker run -d -p 8000:8000 -p 9443:9443 --name portainer \
      --restart=always \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v <BASE_DIR>/portainer/data:/data \
      portainer/portainer-ce:lts

Using a bind mount for `/data` rather than a named volume keeps Portainer's own
database — including every stack's environment variables — inside the directory
your backup already covers.

## Shared network

Every stack attaches to one external Docker network. Create it once, before
deploying the first stack:

    docker network create homeserver

Nothing creates this for you; deploying a stack without it fails.
