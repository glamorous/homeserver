auth
====

[authentik](https://goauthentik.io) as the single place where accounts live, so
a household has one password instead of one per service.

Four containers: the server, a worker, PostgreSQL and Redis. Budget around a
gigabyte of memory. Everything persists under `${BASE_DIR}/authentik`.

## First run

1. Deploy the stack and give it a minute; the server waits for the database
2. Open `http://<host>:${AUTHENTIK_PORT_HTTP}/if/flow/initial-setup/`
3. Create the first administrator — this is your break-glass account
4. Point a proxy host at it, then use the hostname from then on

## Two ways to connect a service, and they are not equal

**OpenID Connect**, when the service supports it. The application learns who
you are, keeps its own roles, and every family member gets a real account
inside it. Use this wherever it exists.

**Forward authentication** at the reverse proxy, for services that have no
concept of accounts. authentik decides *whether* someone gets in; the service
still sees a single anonymous session. Everyone admitted shares the same view,
so grant it through a group and only to people you would have handed the
password to anyway.

The difference matters for a household. Behind forward auth, a family member is
not a user of that service — they are simply past its door.

## The dashboard is the launcher

Every application registered here carries a launch URL, and the provider shows
each person only the ones their group allows. That is a dashboard already, and
one filtered by who is looking — which a separate dashboard listing everything
to everyone cannot be. Applications that need no gate at all can be registered
as a tile with only a launch URL and no provider.

Point the bare domain at this stack so old bookmarks land on it. Note what that
implies: an administrator who belongs to no group sees an empty launcher, which
is correct and still surprising the first time.

## Forward authentication behind Nginx Proxy Manager

Per protected host, in the Advanced tab. Three details are not interchangeable
with what a generic guide will tell you:

    proxy_buffers 8 16k;
    proxy_buffer_size 32k;
    set $ak_http_host $http_host;
    if ($ak_http_host = "") {
        set $ak_http_host $host;
    }

    location /outpost.goauthentik.io {
        proxy_pass              http://<host-ip>:<authentik-port>/outpost.goauthentik.io;
        proxy_set_header        Host $ak_http_host;
        proxy_set_header        X-Original-URL $scheme://$http_host$request_uri;
        add_header              Set-Cookie $auth_cookie;
        auth_request_set        $auth_cookie $upstream_http_set_cookie;
        proxy_pass_request_body off;
        proxy_set_header        Content-Length "";
    }

    location @goauthentik_proxy_signin {
        internal;
        add_header Set-Cookie $auth_cookie;
        return 302 /outpost.goauthentik.io/start?rd=$request_uri;
    }

    auth_request     /outpost.goauthentik.io/auth/nginx;
    error_page       401 = @goauthentik_proxy_signin;
    auth_request_set $auth_cookie $upstream_http_set_cookie;
    add_header       Set-Cookie $auth_cookie;
    auth_request_set $authentik_username $upstream_http_x_authentik_username;
    proxy_set_header X-authentik-username $authentik_username;

**No `location /`.** Nginx Proxy Manager generates one; a second refuses to
load. The `auth_request` directives sit at server level and are inherited.

**Address the outpost by IP, not by container name.** Nginx resolves a name in
`proxy_pass` when it loads its configuration, so a name makes the whole proxy
refuse to reload whenever this stack happens to be down — taking every
unrelated site with it.

**Use single-application providers, not domain level.** Domain level means one
application and therefore one access policy for every host under the domain,
which makes it impossible to admit the household to one service and not
another. People still sign in once: the session is shared, so the second host
lets them through silently.

Check before enabling it on a host that nothing machine-driven uses that
hostname. Forward authentication answers an API client with a redirect to a
login page, which it cannot follow.

## Keep a local administrator everywhere

Every service that gets an identity provider should keep one local account that
does not depend on it. When authentik is down, its database is broken, or a
misconfigured flow locks you out, that account is the way back in. This is also
why the reverse proxy's own admin port stays reachable without the proxy: an
authentication loop that can only be fixed through the thing it broke is not a
loop you want to discover at night.

## Not given the Docker socket

The official example mounts it into the worker so authentik can run outposts as
containers it manages. The embedded outpost in the server covers proxy
providers on a single host, and handing an internet-facing service control over
the Docker daemon is a poor trade for that convenience.
