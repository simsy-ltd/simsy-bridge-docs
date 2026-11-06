# Running the SIMSY Tailscale bridge yourself

The bridge puts your SIMSY SIMs on your Tailscale network. You run it as a container on any Linux host with a public IPv4 address; it sets up SIMSY and Tailscale itself. Nothing is installed on your devices, and the container needs no special privileges.

```
your devices ── SIMSY SIMs ── SIMSY core ══ encrypted tunnel ══ bridge (your host) ══ your tailnet
```

## What you need

- A Linux host with Docker and a public IPv4 address. A small cloud instance is enough (1 vCPU, 1 GB).
- Inbound UDP 51820 from SIMSY (it dials the bridge; for Europe, from 46.247.38.21) and UDP 41641 from anywhere (Tailscale direct connections). No other ports. Use your cloud provider's security group for this: ports published by Docker bypass host firewalls such as `ufw`.
- Admin access to your tailnet's policy file and OAuth clients.
- A SIMSY API token for your account, and the VSlice to bridge. Treat the token like a password: it can change your VSlice's routing. Its SIM address range must not overlap Tailscale's 100.64.0.0/10; SIMSY support can renumber it if it does.

## 1. Tailscale (10 minutes)

Add to your tailnet policy file, replacing `fleet-a` with your VSlice's name:

```json
"tagOwners": {
    "tag:simsy-provisioner": ["autogroup:admin"],
    "tag:simsy":             ["autogroup:admin", "tag:simsy-provisioner"],
    "tag:simsy-fleet-a":     ["autogroup:admin", "tag:simsy-provisioner"],
},
// The bridge's route is approved by the policy, not by the bridge.
"autoApprovers": {
    "routes": {"10.20.0.0/24": ["tag:simsy-fleet-a"]},
},
"grants": [
    // Who may reach the SIM devices, and how.
    {"src": ["group:ops"], "dst": ["10.20.0.0/24"], "ip": ["tcp:22", "tcp:443"]},
    // Only if the devices should open connections to services on your tailnet.
    {"src": ["10.20.0.0/24"], "dst": ["tag:collector"], "ip": ["tcp:8883"]},
],
```

Grant only what the devices need: a grant to the SIM range reaches every device on it. Add a policy test for each grant, so a later edit cannot quietly widen it:

```json
"tests": [
    {"src": "ops-person@example.com", "accept": ["10.20.0.10:22"], "deny": ["10.20.0.10:80"]},
],
```

Then **Settings, OAuth clients, Generate**: scopes **Auth Keys** (write), **Devices: Core** (read) and **Devices: Routes** (read), tagged **only** `tag:simsy-provisioner`. The bridge needs nothing more: it only reads your machines, and the policy approves its route. Keep the client ID and secret for step 3. The owner tag matters: an OAuth client's keys must carry exactly its own tags, or tags those tags own.

Protect this secret like an admin password: anyone holding it can add machines with the bridge's tags to your tailnet, and the policy would approve their route to your SIMs too. Keep it only on the bridge's host, and rotate it if the host is ever compromised. The bridge should be the only machine with `tag:simsy-fleet-a`: a second one in the admin console, or a Tailscale **Node created** notification you did not expect, means the secret is being used elsewhere.

## 2. The host

```bash
mkdir -p ~/simsy-bridge && cd ~/simsy-bridge
```

Open UDP 51820 (from SIMSY's address) and UDP 41641 in your cloud security group. A host firewall such as `ufw` does not filter ports Docker publishes; if you have no security group, add the rules to Docker's `DOCKER-USER` chain instead. Do the same for IPv6 if the host has it.

## 3. Configuration

`config.yaml`:

```yaml
mode: tenant
state_dir: /var/lib/simsy-bridge
listen:
  public_address: 203.0.113.10      # this host's public IPv4 address
simsy:
  token: file:/run/secrets/simsy_api_token
  redundancy_zone: europe-primary
tenants:
  - name: fleet-a
    vslice: fleet-a                  # your VSlice's moniker
    sim_range: 10.20.0.0/24          # its SIM address range
    downlink_allowed: true           # false: uplink only
    tailscale:
      oauth_client_id: file:/run/secrets/ts_oauth_client_id
      oauth_client_secret: file:/run/secrets/ts_oauth_client_secret
    webhook:                         # optional: see Bridge events and webhooks
      url: https://example.com/hooks/simsy   # https only
      secret: file:/run/secrets/webhook_secret   # at least 32 characters
api:
  listen: 0.0.0.0:9100               # inside the container; see the ports in docker-compose.yml
```

The secrets, one file each, readable only by the container's user (65532). Type each value at the prompt so it stays out of your shell history:

```bash
mkdir -m 700 secrets
read -rsp "SIMSY API token: " v && printf %s "$v" > secrets/simsy_api_token
read -rsp "Tailscale OAuth client ID: " v && printf %s "$v" > secrets/ts_oauth_client_id
read -rsp "Tailscale OAuth client secret: " v && printf %s "$v" > secrets/ts_oauth_client_secret
openssl rand -hex 16 > secrets/webhook_secret   # only if you use a webhook
unset v; sudo chown 65532 secrets/*; sudo chmod 400 secrets/*
```

The `chown` is needed: Docker Compose mounts these files as they are, and ignores `uid` and `mode` settings for them. Without it the bridge stops with `permission denied` on `/run/secrets/...`. If you do not use a webhook, remove the `webhook:` lines from `config.yaml`; if you do, give your receiver the same value (`sudo cat secrets/webhook_secret`). To change a secret later, `sudo rm` its file, repeat its line and the `chown`, then `docker compose up -d --force-recreate`.

Never paste these files, or anything else from the `secrets` directory, into a support ticket.

`docker-compose.yml`:

```yaml
# SIMSY Tailscale bridge, run yourself: https://s-imsy.com/developers/tools/tailscale/bridge-guide
services:
  simsy-bridge:
    image: public.ecr.aws/simsy/simsy-bridge:0.3   # pin a version; see the guide to verify it
    container_name: simsy-bridge
    restart: unless-stopped
    # No NET_ADMIN and no /dev/net/tun: WireGuard and Tailscale both run in
    # userspace inside the container. Only two UDP ports are published.
    ports:
      - "51820:51820/udp"            # SIMSY dials this (fixed by SIMSY)
      - "41641:41641/udp"            # Tailscale direct connections
      - "127.0.0.1:9100:9100/tcp"    # API, this host only: keep the 127.0.0.1
                                     # (with host networking, set api.listen
                                     # to 127.0.0.1:9100 instead)
    volumes:
      - ./config.yaml:/etc/simsy-bridge/config.yaml:ro
      - state:/var/lib/simsy-bridge  # keys, Tailscale identity, event queue
    # Secrets as files, not environment variables: they stay out of
    # docker inspect and docker compose config. Mounted in /run/secrets.
    secrets:
      - simsy_api_token
      - ts_oauth_client_id
      - ts_oauth_client_secret
      # - webhook_secret             # if you use a webhook
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs: [/tmp]                    # scratch space; the root filesystem is read-only
    cap_drop: [ALL]
    logging:                         # access records appear in the log too
      options: {max-size: 10m, max-file: "5"}

volumes:
  state:

# Files on this host, readable only by the container's user (see the guide).
secrets:
  simsy_api_token: {file: ./secrets/simsy_api_token}
  ts_oauth_client_id: {file: ./secrets/ts_oauth_client_id}
  ts_oauth_client_secret: {file: ./secrets/ts_oauth_client_secret}
  # webhook_secret: {file: ./secrets/webhook_secret}
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
TOKEN=$(docker compose cp simsy-bridge:/var/lib/simsy-bridge/api-token - | tar -xO)
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
cosign verify public.ecr.aws/simsy/simsy-bridge:<version> \
  --certificate-identity-regexp '^https://github.com/simsy-ltd/simsy-bridge/\.github/workflows/release\.yml@refs/tags/v' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

A tag can be moved; a digest cannot. After verifying, pin the digest `cosign verify` printed: `image: public.ecr.aws/simsy/simsy-bridge@sha256:...`. Each release is listed, with its security fixes, in the bridge's changelog.

**Remove**: add `delete: true` to the tenant in `config.yaml`, run `docker compose kill -s HUP simsy-bridge`, wait for `deleted` in the log, then `docker compose down`. The bridge removes its rule and its routing target, and logs the name of its Tailscale machine: remove that machine in the admin console (the OAuth client can only read machines). Remove the policy lines and the OAuth client yourself.

## When something is wrong

`/status` and the `bridge.degraded` webhook give a reason:

| Reason | What to do |
|---|---|
| `tags_not_assignable` | The `tagOwners` lines in step 1 are missing, or the OAuth client is not tagged `tag:simsy-provisioner`. |
| `route_unapproved` | Usually clears by itself within a minute. If not, check `autoApprovers` in your policy names your SIM range and your VSlice's tag exactly (step 1). |
| `handshake_stale` | SIMSY cannot reach the bridge: check UDP 51820 is open from SIMSY's address and `public_address` is right. |
| `simsy_api_error` | Check the SIMSY token and the VSlice name. The detail says which call failed. |
| `ts_offline` | The host cannot reach Tailscale: check outbound internet access. |
| `config_error` | The detail names the setting. |

Access records missing: `/status` counts what the bridge could not record. `access_sessions_capped` goes up when one person opens more than 256 sessions at once to devices (a scan of the SIM range, for example), and `access_events_dropped` when events arrive faster than they can be written. Both are also in `/metrics`.

Devices answer by address but a device cannot reach your tailnet: add a grant with your SIM range as the source (step 1, second grant).

One device does not answer at all while others on the same VSlice do: if it has another internet connection besides its SIMSY SIM (a second SIM, Ethernet or Wi-Fi), it probably sends its replies out of that one. Add a route on the device sending 100.64.0.0/10 out of its SIMSY cellular interface, and keep its default route as it is. Check on the device with `ip route get 100.100.100.100`: the answer should name the SIMSY interface.

## Security

Report a security problem to support@s-imsy.com with "Security" in the subject. Releases and the security fixes in them are listed in the changelog.

The bridge runs as a non-root user with no added privileges, on a read-only filesystem, with secrets never written to its log. Its API listens on the host only. Keep the host patched, and limit who can log in to it: anyone with root on the host can read the bridge's credentials.

## Privacy

The bridge records each access to a device through it: the tailnet user and machine, the SIM's address and name, times and data volumes. It writes these records to its log and, if you configure a webhook, sends them to you. Nothing is sent to SIMSY. You decide how long to keep them: the compose file above limits the log to 50 MB, and undelivered webhook events are dropped after 24 hours. As with any Tailscale client, the bridge's Tailscale node is subject to Tailscale's own logging; see Tailscale's documentation.

Support: support@s-imsy.com
