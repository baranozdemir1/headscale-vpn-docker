# Headscale VPN Docker

Self-hosted VPN Docker Compose stack for running [Headscale](https://headscale.net/), [Headplane](https://headplane.net/), and a containerized Tailscale exit node. It is designed for Dokploy, homelab, and VPS deployments where you want a private WireGuard mesh VPN with a self-hosted Tailscale-compatible control server.

Use this repository when you want:

- A Docker Compose Headscale deployment.
- Headplane as a web UI for managing users, nodes, and settings.
- A Tailscale exit node container for routing client internet traffic through your VPS.
- A Dokploy-friendly layout using persistent `../files` config mounts.

The current compose file runs three services:

- `headscale`: Headscale control server on port `8080`.
- `headplane`: Headplane web UI on port `3000`.
- `tailscale-exit-node`: Tailscale client container advertising itself as an exit node.

Persistent data is kept in Docker named volumes:

- `headscale-data`
- `headplane-data`
- `tailscale-exit-state`

## Requirements

- Docker and Docker Compose.
- DNS records for your Headscale and Headplane domains.
- A reverse proxy or Dokploy domains pointing to:
  - `headscale:8080` for Headscale.
  - `headplane:3000` for Headplane.
- Config files mounted from Dokploy's persistent `../files` directory.

This repository uses these placeholder domains:

```text
vpn.xxx.com        Headscale public URL
vpn-plane.xxx.com  Headplane public URL
tail.xxx.com       MagicDNS base domain
```

Replace them with your real domains before deployment.

## Environment Variables

The compose file uses environment variables only for the Tailscale exit-node container:

```yaml
environment:
  TS_AUTHKEY: ${TS_AUTHKEY}
  TS_EXTRA_ARGS: --login-server=${HEADSCALE_URL} --advertise-exit-node --accept-routes
  TS_STATE_DIR: /var/lib/tailscale
  TS_USERSPACE: false
```

Create a `.env` file from `.env.example`:

```bash
cp .env.example .env
```

Then set:

```env
TS_AUTHKEY=hskey-auth-your-generated-key
HEADSCALE_URL=https://vpn.xxx.com
```

Important distinction:

- `${VAR}` in `docker-compose.yml` is resolved by Docker Compose before containers start.
- Variables from `.env` are not automatically injected into every container unless the compose file references them or uses `env_file`.
- Mounted config files such as `config.yaml` and `headplane-config.yaml` are not templated by Docker or Dokploy. Do not rely on `${HEADSCALE_URL}` inside those YAML files unless the application itself explicitly supports it.

For production, you may hard-fail when required variables are missing by changing the compose values to:

```yaml
TS_AUTHKEY: "${TS_AUTHKEY:?set TS_AUTHKEY}"
TS_EXTRA_ARGS: "--login-server=${HEADSCALE_URL:?set HEADSCALE_URL} --advertise-exit-node --accept-routes"
```

## Dokploy Notes

Dokploy stores Docker Compose environment variables in a `.env` file next to `docker-compose.yml`. In this stack, that is enough for `TS_AUTHKEY` and `HEADSCALE_URL` because they are referenced directly in `docker-compose.yml`.

Dokploy does not automatically rewrite mounted config YAML files. Keep real values in the files under `../files`, for example:

```yaml
# config.yaml
server_url: https://vpn.xxx.com

dns:
  base_domain: tail.xxx.com
```

```yaml
# headplane-config.yaml
headscale:
  url: http://headscale:8080
  public_url: "https://vpn.xxx.com"
```

The compose file already mounts these paths:

```yaml
- ../files/config.yaml:/etc/headscale/config.yaml:ro
- ../files/headplane-config.yaml:/etc/headplane/config.yaml:ro
```

That layout is intentional for Dokploy because `../files` persists across deployments.

## Local Docker Compose Notes

If you run this repository directly outside Dokploy, either create the expected `../files` directory:

```bash
mkdir -p ../files
cp config.yaml ../files/config.yaml
cp headplane-config.yaml ../files/headplane-config.yaml
```

Or change the volume mounts to local files:

```yaml
- ./config.yaml:/etc/headscale/config.yaml:ro
- ./headplane-config.yaml:/etc/headplane/config.yaml:ro
```

Check the resolved compose config before deploying:

```bash
docker compose config
```

If `HEADSCALE_URL` is missing, `TS_EXTRA_ARGS` will resolve to:

```text
--login-server= --advertise-exit-node --accept-routes
```

That is wrong; set `HEADSCALE_URL` first.

## Configure Headscale

Edit `config.yaml` and replace the placeholder domains:

```yaml
server_url: https://vpn.xxx.com

dns:
  magic_dns: true
  base_domain: tail.xxx.com
```

Headscale does not template arbitrary `${VAR}` values in this YAML file. Use real values.

## Configure Headplane

Generate a new cookie secret:

```bash
openssl rand -hex 32
```

Put it in `headplane-config.yaml`:

```yaml
server:
  cookie_secret: "paste-generated-secret-here"
```

Set the Headscale URLs:

```yaml
headscale:
  url: http://headscale:8080
  public_url: "https://vpn.xxx.com"
  config_path: "/etc/headscale/config.yaml"
```

`url` is the internal Docker network URL. `public_url` is the browser/client-facing URL.

Headplane can override config values from environment variables, but that feature is disabled unless `HEADPLANE_LOAD_ENV_OVERRIDES=true` is set. The current compose file does not enable this, so keep normal values in `headplane-config.yaml`.

## Install

Start the stack:

```bash
docker compose pull
docker compose up -d
```

Check status and logs:

```bash
docker compose ps
docker logs -f headscale
docker logs -f headplane
docker logs -f tailscale-exit-node
```

Verify Headscale:

```bash
curl https://vpn.xxx.com/health
```

Then open Headplane:

```text
https://vpn-plane.xxx.com
```

## Generate a Headplane API Key

Headplane authenticates to Headscale with a Headscale API key. Generate one inside the running Headscale container:

```bash
docker compose exec headscale \
  headscale apikeys create --expiration 90d
```

Copy the printed API key immediately. Headscale only shows the full key once.

Set it in `headplane-config.yaml`:

```yaml
headscale:
  api_key: "paste-api-key-here"
```

Restart Headplane:

```bash
docker compose restart headplane
```

Useful API key commands:

```bash
docker compose exec headscale headscale apikeys list
docker compose exec headscale headscale apikeys expire --prefix <KEY_PREFIX>
```

## Add a User

Create a Headscale user:

```bash
docker compose exec headscale headscale users create alice
```

List users:

```bash
docker compose exec headscale headscale users list
```

Create a pre-authentication key:

```bash
docker compose exec headscale \
  headscale preauthkeys create --user alice --expiration 24h
```

Use the returned key on a client machine:

```bash
sudo tailscale up \
  --login-server https://vpn.xxx.com \
  --authkey <PREAUTH_KEY>
```

For a reusable provisioning key:

```bash
docker compose exec headscale \
  headscale preauthkeys create \
  --user alice \
  --reusable \
  --expiration 30d
```

## Register a Node Without a Preauth Key

On the client:

```bash
sudo tailscale up --login-server https://vpn.xxx.com
```

The client shows a machine key or registration URL. Register it on the Headscale server:

```bash
docker compose exec headscale \
  headscale nodes register \
  --user alice \
  --key <MACHINE_KEY>
```

List nodes:

```bash
docker compose exec headscale headscale nodes list
```

## Create the Exit Node

The `tailscale-exit-node` service uses:

```yaml
TS_EXTRA_ARGS: --login-server=${HEADSCALE_URL} --advertise-exit-node --accept-routes
```

Create a user and pre-authentication key for the exit node:

```bash
docker compose exec headscale headscale users create exit-node
docker compose exec headscale \
  headscale preauthkeys create \
  --user exit-node \
  --reusable \
  --expiration 365d
```

Put the returned key in `.env`:

```env
TS_AUTHKEY=hskey-auth-your-exit-node-key
HEADSCALE_URL=https://vpn.xxx.com
```

Restart the exit node:

```bash
docker compose up -d tailscale-exit-node
```

Headscale requires double opt-in for exit nodes. The node advertises the exit routes, then Headscale must approve them.

List advertised routes:

```bash
docker compose exec headscale headscale nodes list-routes
```

Find the exit node ID for `vps-exit-node`, then approve the exit route:

```bash
docker compose exec headscale \
  headscale nodes approve-routes \
  --identifier <NODE_ID> \
  --routes 0.0.0.0/0
```

Approving `0.0.0.0/0` is enough for an exit node; Headscale approves the IPv6 `::/0` route too.

On a client, use the exit node:

```bash
sudo tailscale set --exit-node vps-exit-node
```

Allow local LAN access while using the exit node:

```bash
sudo tailscale set --exit-node vps-exit-node --exit-node-allow-lan-access
```

## Security Notes

- Do not commit real `.env`, `TS_AUTHKEY`, API keys, or cookie secrets.
- Keep real Dokploy config files in `../files`, not in a public repository.
- Use short-lived pre-authentication keys unless the node is intentionally automated.
- Rotate the Headscale API key if it is exposed.
- Keep Headscale, Headplane, and Tailscale images pinned and update them intentionally.

## References

- [Dokploy Docker Compose](https://docs.dokploy.com/docs/core/docker-compose)
- [Docker Compose interpolation](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/)
- [Headscale getting started](https://docs.headscale.org/usage/getting-started/)
- [Headscale API keys](https://headscale.net/stable/ref/api/)
- [Headscale routes and exit nodes](https://headscale.net/0.28.0/ref/routes/)
- [Headplane Docker installation](https://headplane.net/install/docker)
- [Headplane configuration](https://headplane.net/configuration/)
