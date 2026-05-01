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
docker compose logs -f headscale
docker compose logs -f headplane
docker compose logs -f tailscale-exit-node
```

Verify Headscale:

```bash
curl https://vpn.xxx.com/health
```

Then open Headplane:

```text
https://vpn-plane.xxx.com
```

## Headscale CLI

The examples below use `docker exec -it` to run the Headscale CLI inside the running Headscale container.

Set a helper variable first:

```bash
HEADSCALE_CONTAINER="$(docker compose ps -q headscale)"
```

Then use:

```bash
docker exec -it "$HEADSCALE_CONTAINER" headscale <command>
```

## Generate a Headplane API Key

Headplane authenticates to Headscale with a Headscale API key. Generate one inside the running Headscale container:

```bash
docker exec -it "$HEADSCALE_CONTAINER" \
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
docker exec -it "$HEADSCALE_CONTAINER" headscale apikeys list
docker exec -it "$HEADSCALE_CONTAINER" headscale apikeys expire --prefix <KEY_PREFIX>
```

## Add a User

Create a Headscale user:

```bash
docker exec -it "$HEADSCALE_CONTAINER" headscale users create alice
```

List users:

```bash
docker exec -it "$HEADSCALE_CONTAINER" headscale users list
```

Create a pre-authentication key:

```bash
docker exec -it "$HEADSCALE_CONTAINER" \
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
docker exec -it "$HEADSCALE_CONTAINER" \
  headscale preauthkeys create \
  --user alice \
  --reusable \
  --expiration 30d
```

## Connect Client Devices

Users connect with the official Tailscale client, but they point the client at your Headscale server instead of Tailscale's hosted control plane.

Install Tailscale on each device:

- Linux: install from the Tailscale package repository for your distribution.
- macOS: install the Tailscale app, or use the standalone CLI.
- Windows: install the Tailscale app.
- Android: install Tailscale from Google Play or F-Droid.
- iOS/iPadOS: install Tailscale from the App Store.

The easiest flow is to create a Headscale user and a pre-authentication key first:

```bash
docker exec -it "$HEADSCALE_CONTAINER" headscale users create alice
docker exec -it "$HEADSCALE_CONTAINER" \
  headscale preauthkeys create --user alice --expiration 24h
```

Then connect a Linux server, VPS, Raspberry Pi, or other CLI-based client:

```bash
sudo tailscale up \
  --login-server https://vpn.xxx.com \
  --authkey <PREAUTH_KEY>
```

For better shell history hygiene, pass the key through an environment variable:

```bash
export TS_AUTHKEY=<PREAUTH_KEY>
sudo tailscale up \
  --login-server https://vpn.xxx.com \
  --authkey "$TS_AUTHKEY"
unset TS_AUTHKEY
```

After connecting, verify the device:

```bash
tailscale status
tailscale ip
```

On the Headscale server, confirm that the node exists:

```bash
docker exec -it "$HEADSCALE_CONTAINER" headscale nodes list
```

### Interactive Login

You can also connect without a pre-authentication key:

```bash
sudo tailscale up --login-server https://vpn.xxx.com
```

The client prints a registration URL or machine key. Register it on the Headscale server:

```bash
docker exec -it "$HEADSCALE_CONTAINER" \
  headscale nodes register \
  --user alice \
  --key <MACHINE_KEY>
```

### macOS and Windows

If you use the Tailscale CLI:

```bash
tailscale login --login-server https://vpn.xxx.com
```

or, with a pre-authentication key:

```bash
tailscale up \
  --login-server https://vpn.xxx.com \
  --authkey <PREAUTH_KEY>
```

On Windows, run the command from PowerShell as Administrator if the normal terminal lacks permission.

### Android

Open the Tailscale app and use an alternate server:

1. Open the account/settings menu.
2. Choose **Use an alternate server**.
3. Enter your Headscale URL, for example `https://vpn.xxx.com`.
4. If you are using a pre-authentication key, choose **Use an auth key** and paste the key generated by Headscale.
5. Once Headscale registers the node, the device connects to your private tailnet.

### iOS and iPadOS

Open the Tailscale app and choose the custom/alternate coordination server option. Enter:

```text
https://vpn.xxx.com
```

Then complete the login/registration flow. If using pre-authentication keys, use the app's auth-key flow when available. Mobile client UI changes over time, but the important bit is always the same: the client must use your Headscale URL as the login/control server.

## Create the Exit Node

The `tailscale-exit-node` service uses:

```yaml
TS_EXTRA_ARGS: --login-server=${HEADSCALE_URL} --advertise-exit-node --accept-routes
```

Create a user and pre-authentication key for the exit node:

```bash
docker exec -it "$HEADSCALE_CONTAINER" headscale users create exit-node
docker exec -it "$HEADSCALE_CONTAINER" \
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
docker exec -it "$HEADSCALE_CONTAINER" headscale nodes list-routes
```

Find the exit node ID for `vps-exit-node`, then approve the exit route:

```bash
docker exec -it "$HEADSCALE_CONTAINER" \
  headscale nodes approve-routes \
  --identifier <NODE_ID> \
  --routes 0.0.0.0/0
```

Approving `0.0.0.0/0` is enough for an exit node; Headscale approves the IPv6 `::/0` route too.

Use the exit node on a client:

```bash
sudo tailscale set --exit-node vps-exit-node
```

Allow local LAN access while using the exit node:

```bash
sudo tailscale set --exit-node vps-exit-node --exit-node-allow-lan-access
```

To stop using the exit node:

```bash
sudo tailscale set --exit-node=
```

In the desktop and mobile apps, choose `vps-exit-node` from the app's exit-node menu.

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
- [Tailscale install](https://tailscale.com/docs/install)
- [Tailscale custom control server](https://tailscale.com/kb/1507/custom-control-server)
- [Tailscale auth keys](https://tailscale.com/kb/1085/auth-keys)
