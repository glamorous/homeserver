ai
==

Open WebUI: a browser chat interface for a model server running on the host.

## The model server is not in this stack

It runs natively on the host, deliberately. Containers on macOS cannot reach the
GPU, so a containerised model server falls back to the CPU and becomes several
times slower. Install it as a desktop application and enable its option to
listen on the network, then point `OLLAMA_BASE_URL` at
`http://host.docker.internal:11434`.

Inside a container, `localhost` is the container itself. The `extra_hosts` entry
in the compose file is what makes `host.docker.internal` resolve.

## First run

The first account you create becomes the administrator. Models can be pulled
from the interface under Settings -> Models, so the command line is never
required.

## Access and exposure

Open WebUI has its own authentication; the model API behind it has none at all.
Anyone who can reach that port can run and delete models. Keep the model server
on the local network only, reach it remotely through your VPN, and never forward
either port from the internet.
