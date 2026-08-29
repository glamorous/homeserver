Running on macOS
================

## Container runtime

Any Docker-compatible runtime works — [Docker Desktop](https://www.docker.com/products/docker-desktop/),
[OrbStack](https://orbstack.dev), [Colima](https://github.com/abiosoft/colima).
Two things matter more than which one you pick:

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

## Portainer

### As the primary host

Install the server once, outside the stacks, so it survives redeploying
anything:

    docker volume create portainer_data
    docker run -d -p 8000:8000 -p 9443:9443 --name portainer \
      --restart=always \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v <BASE_DIR>/portainer/data:/data \
      portainer/portainer-ce:lts

Using a bind mount for `/data` rather than a named volume keeps Portainer's own
database — including every stack's environment variables — inside the directory
your backup already covers.

### As a secondary host

Install only the agent. Everything is then deployed from the primary; nothing
is configured here.

    docker run -d -p 9001:9001 --name portainer_agent --restart=always \
      -v /var/run/docker.sock:/var/run/docker.sock \
      -v /var/lib/docker/volumes:/var/lib/docker/volumes \
      portainer/agent:latest

Then, on the primary: Environments -> Add environment -> Docker Standalone ->
Agent, and enter `<this-host>:9001`. Match the agent version to the server
version to avoid API mismatches.

## Shared network

Every stack attaches to one external Docker network. Create it once, before
deploying the first stack:

    docker network create homeserver

Nothing creates this for you; deploying a stack without it fails.

## Locally running models

The `ai` stack expects [Ollama](https://ollama.com) running natively on this
host, not in a container — a containerised model server on macOS cannot reach
the GPU and falls back to the CPU, several times slower.

Install the desktop application, which starts at login and keeps itself
updated:

    brew install --cask ollama-app

Or download it from [ollama.com/download](https://ollama.com/download). Then:

1. Open the application once and complete the first-run setup
2. In its settings, enable exposing Ollama to the network — without this it
   listens on loopback only and containers cannot reach it
3. Pull a model, either from the Open WebUI interface or with
   `ollama pull <model>`

Verify it answers on the network:

    curl http://localhost:11434/api/tags

The desktop application has no model list in its settings pane; models are
selected from the dropdown in its chat window, or managed from Open WebUI.
