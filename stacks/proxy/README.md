proxy
=====

[Nginx Proxy Manager](https://nginxproxymanager.com): reverse proxy with a web UI, handling TLS certificates for
the services you expose under your own hostnames.

Deploy this on the host that owns your inbound traffic, and on that host only.
Keep it in its own stack: everything else reaches the outside world through it,
so it should never be restarted as a side effect of redeploying something
unrelated.

## First run

1. Open the admin interface on `NGINX_PROXY_MANAGER_PORT_ADMIN`
2. Sign in with the default credentials `admin@example.com` / `changeme`
3. Change them immediately when prompted

## Proxying other stacks

Because every stack joins the shared `homeserver` network, proxy hosts can point
at a container name and its internal port rather than a host IP, for example
`heimdall:80`. That keeps working when host ports change.
