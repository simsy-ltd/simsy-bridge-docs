# Running the SIMSY Tailscale bridge yourself

The bridge puts your SIMSY SIMs on your Tailscale network. You run it as a container on any Linux host with a public IPv4 address; it sets up SIMSY and Tailscale itself. Nothing is installed on your devices, and the container needs no special privileges.

```
your devices ── SIMSY SIMs ── SIMSY core ══ encrypted tunnel ══ bridge (your host) ══ your tailnet
```

## What you need

- A Linux host with Docker and a public IPv4 address. A small cloud instance is enough (1 vCPU, 1 GB).
- Inbound UDP 51820 from SIMSY (it dials the bridge; for Europe, from 46.247.38.21) and UDP 41641 from anywhere (Tailscale direct connections). No other ports.
- Admin access to your tailnet's policy file and OAuth clients.
- A SIMSY API token for your account, and the VSlice to bridge. Its SIM address range must not overlap Tailscale's 100.64.0.0/10; SIMSY support can renumber it if it does.

## 1. Tailscale (10 minutes)

Add to your tailnet policy file, replacing `fleet-a` with your VSlice's name:

```json
"tagOwners": {
    "tag:simsy-provisioner": ["autogroup:admin"],
    "tag:simsy":             ["autogroup:admin", "tag:simsy-provisioner"],
    "tag:simsy-fleet-a":     ["autogroup:admin", "tag:simsy-provisioner"],
},
"grants": [
    // Who may reach the SIM devices, and how.
    {"src": ["group:ops"], "dst": ["10.20.0.0/24"], "ip": ["tcp:22", "tcp:443"]},
    // Only if the devices should open connections to services on your tailnet.
    {"src": ["10.20.0.0/24"], "dst": ["tag:collector"], "ip": ["tcp:8883"]},
],
```

Then **Settings, OAuth clients, Generate**: scopes **Devices: Core**, **Devices: Routes** and **Auth Keys**, all write, tagged **only** `tag:simsy-provisioner`. Keep the client ID and secret for step 3. The owner tag matters: an OAuth client's keys must carry exactly its own tags, or tags those tags own.

## 2. The host

```bash
mkdir -p ~/simsy-bridge && cd ~/simsy-bridge
```

Open UDP 51820 (from SIMSY's address) and UDP 41641 in the host's firewall or cloud security group.

## 3. Configuration

`config.yaml`:

```yaml
mode: tenant
state_dir: /var/lib/simsy-bridge
listen:
  public_address: 203.0.113.10      # this host's public IPv4 address
simsy:
  token: ${SIMSY_API_TOKEN}
  redundancy_zone: europe-primary
tenants:
  - name: fleet-a
    vslice: fleet-a                  # your VSlice's moniker
    sim_range: 10.20.0.0/24          # its SIM address range
    downlink_allowed: true           # false: uplink only
    tailscale:
      oauth_client_id: ${TS_OAUTH_CLIENT_ID}
      oauth_client_secret: ${TS_OAUTH_CLIENT_SECRET}
    webhook:                         # optional: see Bridge events and webhooks
      url: https://example.com/hooks/simsy
      secret: ${WEBHOOK_SECRET}
api:
  listen: 0.0.0.0:9100
```

`.env` (readable only by you: `chmod 600 .env`):

```
SIMSY_API_TOKEN=...
TS_OAUTH_CLIENT_ID=...
TS_OAUTH_CLIENT_SECRET=...
WEBHOOK_SECRET=...
```

`docker-compose.yml`:

```yaml
# SIMSY Tailscale bridge, run yourself: https://s-imsy.com/developers/tools/tailscale/bridge-guide
services:
  simsy-bridge:
    image: ghcr.io/simsy-ltd/simsy-bridge:0.1   # pin a version; see the guide to verify it
    container_name: simsy-bridge
    restart: unless-stopped
    # No NET_ADMIN and no /dev/net/tun: WireGuard and Tailscale both run in
    # userspace inside the container. Only two UDP ports are published.
    ports:
      - "51820:51820/udp"            # SIMSY dials this (fixed by SIMSY)
      - "41641:41641/udp"            # Tailscale direct connections
      - "127.0.0.1:9100:9100/tcp"    # API, reachable from this host only
    volumes:
      - ./config.yaml:/etc/simsy-bridge/config.yaml:ro
      - state:/var/lib/simsy-bridge  # keys, Tailscale identity, event queue
    env_file: .env
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs: [/tmp]                    # scratch space; the root filesystem is read-only
    cap_drop: [ALL]

volumes:
  state:
```

## 4. Start

```bash
docker compose up -d
docker compose logs -f
```

Within a minute or two the log shows `provisioning`, then `online`. The bridge has:

- created a WireGuard routing target `ts-fleet-a` on your VSlice and added one rule to the VSlice's routing policy, sending traffic for your tailnet to it. Your other rules, including your devices' internet route, are left exactly as they were;
- joined your tailnet as `simsy-fleet-a`, tagged, advertising your SIM range, with the route approved.

From a machine on your tailnet with subnet routes enabled (**Use Tailscale subnets**, or `tailscale set --accept-routes`), your devices answer at their SIM addresses.

Status, with the API token the bridge generated on first start:

```bash
TOKEN=$(docker compose exec simsy-bridge cat /var/lib/simsy-bridge/api-token)
curl -s -H "Authorization: Bearer $TOKEN" localhost:9100/status
```

Port 9100 is published to localhost only. `/metrics` serves Prometheus metrics with the same token.

## Everyday use

**Uplink only** (devices can reach the tailnet; nothing on the tailnet can open a connection to them):

```bash
curl -s -X PATCH -H "Authorization: Bearer $TOKEN" -d '{"allowed": false}' localhost:9100/tenants/fleet-a/downlink
```

Takes effect at once and survives restarts. `DELETE` on the same address returns to the config file's setting. Editing `config.yaml` and sending `docker compose kill -s HUP simsy-bridge` also works.

**Events**: state changes and every access to a device can be sent to your systems, signed. See *Bridge events and webhooks* (https://s-imsy.com/developers/tools/tailscale/webhooks).

**Upgrade**: change the image tag in `docker-compose.yml`, then `docker compose up -d`. The bridge keeps its keys and identity; SIM traffic pauses for a few seconds. To check an image was built by SIMSY:

```bash
cosign verify ghcr.io/simsy-ltd/simsy-bridge:<version> \
  --certificate-identity-regexp '^https://github.com/simsy-ltd/simsy-bridge/\.github/workflows/release\.yml@refs/tags/v' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

**Remove**: add `delete: true` to the tenant in `config.yaml`, run `docker compose kill -s HUP simsy-bridge`, wait for `deleted` in the log, then `docker compose down`. The bridge removes its Tailscale machine, its rule and its routing target. Remove the policy lines yourself.

## When something is wrong

`/status` and the `bridge.degraded` webhook give a reason:

| Reason | What to do |
|---|---|
| `tags_not_assignable` | The `tagOwners` lines in step 1 are missing, or the OAuth client is not tagged `tag:simsy-provisioner`. |
| `route_unapproved` | Usually clears by itself within a minute. If not, check the OAuth client has Devices: Routes (write). |
| `handshake_stale` | SIMSY cannot reach the bridge: check UDP 51820 is open from SIMSY's address and `public_address` is right. |
| `simsy_api_error` | Check the SIMSY token and the VSlice name. The detail says which call failed. |
| `ts_offline` | The host cannot reach Tailscale: check outbound internet access. |
| `config_error` | The detail names the setting. |

Devices answer by address but a device cannot reach your tailnet: add a grant with your SIM range as the source (step 1, second grant).

Support: support@s-imsy.com
