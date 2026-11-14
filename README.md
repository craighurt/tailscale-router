# tailscale-router

Documentation for running a Synology NAS as a Tailscale subnet router using the official [`tailscale/tailscale`](https://hub.docker.com/r/tailscale/tailscale) image.

## Scope

This repository is docs-only. It does not build a custom container image.

## Synology setup (DSM 7+ / Container Manager)

### 1) Prerequisites

- Synology NAS with Container Manager installed.
- `/dev/net/tun` available on the host.
- A Tailscale auth key from your existing tailnet.
- Subnet CIDRs you want to advertise (for example `192.168.1.0/24`).
- IP forwarding enabled on the Synology host (DSM no longer allows `sysctls` in Container Manager compose).

> If an auth key has ever been shared in chat, logs, or screenshots, revoke it and create a new one.

### 2) Compose (bridge mode)

Create a project in Container Manager with this `compose.yaml`:

```yaml
services:
  tailscale-router:
    image: tailscale/tailscale:latest
    container_name: tailscale-router
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    volumes:
      - /volume1/docker/tailscale-router/state:/var/lib/tailscale
    environment:
      TS_AUTHKEY: "tskey-auth-REPLACE_ME"
      TS_HOSTNAME: "syno-router"
      TS_USERSPACE: false
      TS_ROUTES: "192.168.1.0/24,192.168.20.0/24"
      TS_STATE_DIR: /var/lib/tailscale
      # Optional for Synology environments where you want to set container UID/GID
      PUID: 1000
      PGUID: 1000
```

### 3) Compose (host mode alternative)

If bridge mode causes routing issues on your NAS, use host networking:

```yaml
services:
  tailscale-router:
    image: tailscale/tailscale:latest
    container_name: tailscale-router
    restart: unless-stopped
    network_mode: host
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    volumes:
      - /volume1/docker/tailscale-router/state:/var/lib/tailscale
    environment:
      TS_AUTHKEY: "tskey-auth-REPLACE_ME"
      TS_HOSTNAME: "syno-router"
      TS_USERSPACE: false
      TS_ROUTES: "192.168.1.0/24,192.168.20.0/24"
      TS_STATE_DIR: /var/lib/tailscale
      # Optional for Synology environments where you want to set container UID/GID
      PUID: 1000
      PGUID: 1000
```

Use either bridge mode or host mode for a single container, not both.

## How joining an existing tailnet works

On first start, the container runs `tailscale up` with your provided auth key and joins the tailnet.

After deployment:

1. Open the Tailscale admin console.
2. Confirm the new device appears with your `TS_HOSTNAME`.
3. Approve subnet routes (and exit-node capability if configured).

Because `/var/lib/tailscale` is bind-mounted to `/volume1/docker/tailscale-router/state`, node identity and config persist across restart/redeploy.

## How to advertise specific routes

Set `TS_ROUTES` to a comma-separated CIDR list:

```yaml
environment:
  TS_ROUTES: "192.168.1.0/24,192.168.20.0/24"
  TS_USERSPACE: false
  TS_STATE_DIR: /var/lib/tailscale
  PUID: 1000
  PGUID: 1000
```

Update the compose file and redeploy, then approve new routes in the Tailscale admin console.

## Optional: advertise as an exit node

Add exit-node advertisement:

```yaml
environment:
  TS_ROUTES: "192.168.1.0/24,192.168.20.0/24"
  TS_EXTRA_ARGS: "--advertise-exit-node"
  TS_USERSPACE: false
  TS_STATE_DIR: /var/lib/tailscale
  PUID: 1000
  PGUID: 1000
```

Keep IP forwarding enabled on the Synology host and approve exit-node use in admin/policy.
For the official image, there is no dedicated `TS_*` environment variable just for exit-node advertisement; use `TS_EXTRA_ARGS`.

## Operational notes

- `PUID`/`PGUID` are optional; include them if they are part of your Synology working setup.
- For route advertisement on Synology, this guide uses `TS_ROUTES` (matching known working config).
- After initial join, you can remove `TS_AUTHKEY` from compose and redeploy if you do not want a reusable key left in config.
- Synology DSM Container Manager may reject compose `sysctls`; enable IP forwarding at the NAS host level instead.
