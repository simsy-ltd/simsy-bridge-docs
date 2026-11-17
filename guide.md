# Running the SIMSY Tailscale bridge yourself

The bridge puts the devices on your SIMSY SIMs onto your Tailscale network. Each device answers on your tailnet at its SIM address, and nothing is installed on it. You run the bridge as one container on a Linux host; a setup command writes its configuration for you.

Set-up takes about 30 minutes. You need admin access to your Tailscale network, a SIMSY account, and a Linux host you can open firewall ports on.

<!-- Video: "How the bridge works", 90 seconds, animated from the diagram below. -->

## How it works

```
your devices ── SIMSY SIMs ── SIMSY core ══ WireGuard tunnel ══ bridge (your host) ══ your tailnet ── your team
```

Three parts work together:

| Part | What it is | What it does |
|---|---|---|
| **SIMSY** | Your SIMs, on SIMSY's own mobile core | Carries your devices' traffic privately to the bridge, through an encrypted WireGuard tunnel it dials to your host |
| **The bridge** | One container on your Linux host | Passes traffic between that tunnel and your tailnet. It never changes addresses, and it records who reaches which device |
| **Your tailnet** | Your Tailscale network | Joined by the bridge as one machine. Your Tailscale access rules decide who can reach which devices |

What the bridge sets up for you, and checks every minute:

- **In SIMSY:** a WireGuard *routing target* (where SIMSY sends your tailnet's traffic) and one *routing rule* in your vSlice's routing policy. Your other rules, including your devices' internet route, are left as they are.
- **In Tailscale:** a machine called `simsy-<your vSlice>`, tagged so you can recognise it and write rules for it, advertising your SIM range as a *subnet route*.

What stays yours: the bridge never edits your Tailscale policy file. Setup shows you the lines to add, and you decide.

### Words used in this guide

| Term | Meaning |
|---|---|
| vSlice | Your private network slice on SIMSY: a group of SIMs that share an address range and routing rules. You see your vSlices in the SIMSY portal |
| SIM range | The block of addresses your vSlice gives its SIMs, such as `10.20.0.0/24`. Your devices are reached at these addresses |
| SIMSY API token | A key the bridge uses to set up the SIMSY side. You create it in the SIMSY portal |
| Tailnet | Your Tailscale network: everyone and everything signed in to your Tailscale account |
| Policy file | Your tailnet's access rules, edited in the Tailscale admin console under **Access controls** |
| Tag | A label for machines that are not a person's, such as `tag:simsy`. Rules can name tags |
| Grant | A rule in the policy file: who (source) may reach what (destination), on which ports |
| Subnet route | A range of addresses a machine offers to reach on the tailnet's behalf. The bridge offers your SIM range |
| Auto approver | A policy entry that approves a subnet route without you clicking in the console |
| OAuth client | A credential that lets the bridge add its machine to your tailnet. You create it in the admin console |

## Before you start

| You need | Why | Where to get it |
|---|---|---|
| A Linux host with Docker | It runs the bridge | Any small cloud instance is enough (1 vCPU, 1 GB). Install Docker with Docker Compose |
| The host's public IPv4 address | SIMSY dials the bridge on it | Your cloud provider's console, or run `curl -4 https://checkip.amazonaws.com` on the host |
| Two inbound ports open on the host | SIMSY's tunnel arrives on UDP 51820; Tailscale connects directly on UDP 41641 | Your cloud provider's security group (step 1) |
| Admin access to your tailnet | To add policy lines and create an OAuth client | The Tailscale admin console, signed in as an Admin or Owner |
| A SIMSY API token | The bridge sets up the SIMSY side with it | The SIMSY portal, https://portal.s-imsy.com/api-tokens/create (step 2) |
| The vSlice to bridge | Its SIMs are the devices on your tailnet | Setup lists your vSlices; nothing to look up |

Your vSlice's SIM range must not overlap Tailscale's own addresses, `100.64.0.0/10`. Setup checks this. If it does overlap, you can update the range in your SIMSY portal account.

## 1. Prepare the host

On your Linux host:

```bash
mkdir -p ~/simsy-bridge && cd ~/simsy-bridge
docker compose version
```

The second command should print a version. If it does not, install Docker with the Compose plugin first.

Open two inbound ports in your cloud provider's security group:

| Port | From | Why |
|---|---|---|
| UDP 51820 | SIMSY's address for your region (for Europe, `46.247.38.21`; setup prints yours) | SIMSY dials the bridge here |
| UDP 41641 | Anywhere | Tailscale direct connections; without it Tailscale relays traffic, which is slower |

No other inbound ports are needed. Use the security group rather than a host firewall: ports Docker publishes bypass host firewalls such as `ufw`. If you have no security group, add the rules to Docker's `DOCKER-USER` chain instead, and do the same for IPv6 if the host has it.

<!-- Screenshot: an AWS security group with the two inbound rules. -->

## 2. Run setup

Setup asks for your SIMSY token, lets you pick your vSlice, shows the Tailscale lines to add, checks your OAuth client works, then writes `config.yaml`, `docker-compose.yml` and a `secrets` directory. Nothing is written until every check passes, and nothing you type as a secret is shown or logged.

Before you start it, create a SIMSY API token in the SIMSY portal: https://portal.s-imsy.com/api-tokens/create. Name it after this bridge, so you know what to revoke later. Treat it like a password: it can change your vSlice's routing.

<!-- Screenshot: creating an API token in the SIMSY portal. -->

Then, in `~/simsy-bridge`:

```bash
docker run --rm -it --user 0 -v "$PWD":/setup public.ecr.aws/simsy/simsy-bridge:0.4 setup
```

`--user 0` lets setup give the secret files to the bridge's user and leave `config.yaml` yours to edit. It touches nothing outside this directory.

<!-- Video: "Running setup", 3 minutes, the whole of this step. -->

Setup walks through four stages:

1. **SIMSY.** Paste your token; it is not shown as you type. Setup lists the vSlices the token can see and reads the chosen vSlice's SIM range.
2. **This host.** Enter the public IPv4 address. Setup prints the firewall rule with SIMSY's exact address for your region; open it now if you have not already.
3. **Tailscale.** Setup prints the lines to add to your policy file, for your vSlice and SIM range. Keep setup open while you:
    1. In the Tailscale admin console, open **Access controls** and merge the lines into your policy file: each entry goes into the section of the same name. The grant is an example, commented out: adapt it, or add your own grants now or later (step 4). Save.
    2. Open **Settings**, then **OAuth clients**, and generate a client with scopes **Auth Keys** (write), **Devices: Core** (read) and **Devices: Routes** (read), tagged **only** `tag:simsy-provisioner`.
    3. Paste its client ID and secret into setup. Setup checks the client can add a machine with the bridge's tags, and reads your machines to make sure no other bridge already carries this vSlice. If something is missing, it says what; fix it and press Enter to check again.
4. **Options.** Choose whether machines on your tailnet may open connections to your devices (say no for *uplink only*: devices can still reach your tailnet), and optionally give a webhook URL for events and access records.

<!-- Screenshot: Tailscale Access controls with the lines added. -->
<!-- Screenshot: generating the OAuth client with the scopes and tag. -->

Protect the OAuth client's secret like an admin password: anyone holding it can add machines with the bridge's tags to your tailnet, and your policy would approve their route to your SIMs. It stays only in `secrets/` on this host.

## 3. Start the bridge and check it

```bash
docker compose up -d
docker compose logs -f
```

Within a minute or two the log shows `provisioning`, then `online`. Then check each side:

- **Tailscale:** in the admin console's **Machines** page, `simsy-<your vSlice>` is listed with the tags `tag:simsy` and `tag:simsy-<your vSlice>`, and its subnet route shows as approved.
- **Your computer:** turn on subnet routes in your Tailscale client (**Use Tailscale subnets** in the menu, or `tailscale set --accept-routes` on Linux), then reach a device at its SIM address, for example `ping 10.20.0.10`, or open its web page.

<!-- Screenshot: the bridge in Machines, route approved. -->
<!-- Video: "Your first device", 1 minute. -->

If a check fails, see [When something is wrong](#when-something-is-wrong).

## 4. Decide who can reach what

Access to your devices is ordinary Tailscale grants on your SIM range. A grant with the range as its destination says who may reach the devices; a grant with the range as its source says what the devices may reach. Grant only what is needed: a grant to the range reaches every device on it.

| You want | Grant |
|---|---|
| Your ops team to reach device web pages and SSH | `{"src": ["group:ops"], "dst": ["10.20.0.0/24"], "ip": ["tcp:22", "tcp:443"]}` |
| One person to reach one device | `{"src": ["engineer@example.com"], "dst": ["10.20.0.10"], "ip": ["tcp:443"]}` |
| Devices to publish to your MQTT broker | `{"src": ["10.20.0.0/24"], "dst": ["tag:collector"], "ip": ["tcp:8883"]}` |

Add a policy test for each grant, so a later edit cannot quietly widen it:

```json
"tests": [
    {"src": "ops-person@example.com", "accept": ["10.20.0.10:22"], "deny": ["10.20.0.10:80"]},
],
```

## Everyday use

The bridge's API listens on the host only. Read its token first:

```bash
TOKEN=$(docker compose cp simsy-bridge:/var/lib/simsy-bridge/api-token - | tar -xO)
curl -s -H "Authorization: Bearer $TOKEN" localhost:9100/status
```

`/metrics` serves Prometheus metrics with the same token.

**Uplink only** (devices can reach the tailnet; nothing on the tailnet can open a connection to them):

```bash
curl -s -X PATCH -H "Authorization: Bearer $TOKEN" -d '{"allowed": false}' localhost:9100/tenants/<your vSlice>/downlink
```

It takes effect at once and survives restarts. `DELETE` on the same address returns to the config file's setting. Editing `downlink_allowed` in `config.yaml` and sending `docker compose kill -s HUP simsy-bridge` also works.

**Events:** state changes and every access to a device can be sent to your systems, signed. See *Bridge events and webhooks* (https://s-imsy.com/developers/tools/tailscale/webhooks).

**Upgrade:** change the image tag in `docker-compose.yml`, then `docker compose up -d`. The bridge keeps its keys and identity; SIM traffic pauses for a few seconds. To check an image was built by SIMSY:

```bash
cosign verify public.ecr.aws/simsy/simsy-bridge:<version> \
  --certificate-identity-regexp '^https://github.com/simsy-ltd/simsy-bridge/\.github/workflows/release\.yml@refs/tags/v' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

A tag can be moved; a digest cannot. After verifying, pin the digest `cosign verify` printed: `image: public.ecr.aws/simsy/simsy-bridge@sha256:...`. Each release is listed, with its security fixes, in the bridge's changelog.

**Change a secret:** run setup again with `-force` on the end, or replace the file in `secrets/` (`sudo rm` it, write the new value, `sudo chown 65532` it), then `docker compose up -d --force-recreate`.

**Remove:** add `delete: true` to the tenant in `config.yaml`, run `docker compose kill -s HUP simsy-bridge`, wait for `deleted` in the log, then `docker compose down`. The bridge removes its SIMSY rule and routing target, and logs the name of its Tailscale machine: remove that machine in the admin console. Remove the policy lines, the OAuth client and the SIMSY token yourself.

## When something is wrong

`/status` and the `bridge.degraded` webhook give a reason:

| Reason | What to do |
|---|---|
| `tags_not_assignable` | The `tagOwners` lines from setup are missing, or the OAuth client is not tagged `tag:simsy-provisioner`. |
| `route_unapproved` | Usually clears by itself within a minute. If not, check `autoApprovers` in your policy names your SIM range and `tag:simsy-<your vSlice>` exactly. |
| `handshake_stale` | SIMSY cannot reach the bridge: check UDP 51820 is open from SIMSY's address, and `public_address` in `config.yaml` is the host's public address. |
| `simsy_api_error` | Check the SIMSY token has not been revoked, and the vSlice still exists. The detail says which call failed. |
| `ts_offline` | The host cannot reach Tailscale: check outbound internet access. |
| `config_error` | The detail names the setting. |

The bridge stops at start with `permission denied` on `/run/secrets/...`: the secret files must belong to the container's user. Run `sudo chown 65532 secrets/*`, or run setup again with `--user 0`.

Devices answer by address but a device cannot reach your tailnet: add a grant with your SIM range as the source (step 4).

One device does not answer while others on the same vSlice do: if it has another internet connection besides its SIMSY SIM (a second SIM, Ethernet or Wi-Fi), it probably sends its replies out of that one. Add a route on the device sending `100.64.0.0/10` out of its SIMSY cellular interface, and keep its default route as it is. Check on the device with `ip route get 100.100.100.100`: the answer should name the SIMSY interface.

Access records missing: `/status` counts what the bridge could not record. `access_sessions_capped` goes up when one person opens more than 256 sessions at once to devices (a scan of the SIM range, for example), and `access_events_dropped` when events arrive faster than they can be written. Both are also in `/metrics`.

## Setting up by hand

Setup writes three things; you can write them yourself instead.

**`config.yaml`:**

```yaml
mode: tenant
state_dir: /var/lib/simsy-bridge
listen:
  public_address: 203.0.113.10      # this host's public IPv4 address
simsy:
  token: file:/run/secrets/simsy_api_token
tenants:
  - name: fleet-a                    # names the bridge's machine: simsy-fleet-a
    vslice: fleet-a                  # your vSlice, as the SIMSY portal names it
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

Other settings, all optional: `simsy.redundancy_zone` (default `europe-primary`, the only zone today), `tenants[].enabled: false` (stop forwarding but keep the set-up), `tenants[].delete: true` (remove the set-up; see Remove).

**The secret files,** one per value, readable only by the container's user (65532). Type each value at the prompt so it stays out of your shell history:

```bash
mkdir -m 700 secrets
read -rsp "SIMSY API token: " v && printf %s "$v" > secrets/simsy_api_token
read -rsp "Tailscale OAuth client ID: " v && printf %s "$v" > secrets/ts_oauth_client_id
read -rsp "Tailscale OAuth client secret: " v && printf %s "$v" > secrets/ts_oauth_client_secret
openssl rand -hex 16 > secrets/webhook_secret   # only if you use a webhook
unset v; sudo chown 65532 secrets/*; sudo chmod 400 secrets/*
```

The `chown` is needed: Docker Compose mounts these files as they are, and ignores `uid` and `mode` settings for them. Never paste these files, or anything else from the `secrets` directory, into a support ticket.

**`docker-compose.yml`:**

```yaml
# SIMSY Tailscale bridge, run yourself: https://s-imsy.com/developers/tools/tailscale/bridge-guide
services:
  simsy-bridge:
    image: public.ecr.aws/simsy/simsy-bridge:0.4   # pin a version; see the guide to verify it
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

For the Tailscale side by hand, add the policy lines setup would print, with your vSlice name in place of `fleet-a` and your SIM range in place of `10.20.0.0/24`:

```json
"tagOwners": {
    "tag:simsy-provisioner": ["autogroup:admin"],
    "tag:simsy":             ["autogroup:admin", "tag:simsy-provisioner"],
    "tag:simsy-fleet-a":     ["autogroup:admin", "tag:simsy-provisioner"],
},
"autoApprovers": {
    "routes": {"10.20.0.0/24": ["tag:simsy-fleet-a"]},
},
```

Then create the OAuth client as in step 2, stage 3.

## Security

Report a security problem to support@s-imsy.com with "Security" in the subject. Releases and the security fixes in them are listed in the changelog.

The bridge runs as a non-root user with no added privileges, on a read-only filesystem, with secrets never written to its log. Its API listens on the host only. Setup runs once, as root inside its own container, and writes only to the directory you mount. Keep the host patched, and limit who can log in to it: anyone with root on the host can read the bridge's credentials.

The bridge should be the only machine with `tag:simsy-<your vSlice>`: a second one in the admin console, or a Tailscale **Node created** notification you did not expect, means the OAuth client's secret is being used elsewhere. Revoke it and run setup again.

## Privacy

The bridge records each access to a device through it: the tailnet user and machine, the SIM's address and name, times and data volumes. It writes these records to its log and, if you configure a webhook, sends them to you. Nothing is sent to SIMSY. You decide how long to keep them: the compose file limits the log to 50 MB, and undelivered webhook events are dropped after 24 hours. As with any Tailscale client, the bridge's Tailscale node is subject to Tailscale's own logging; see Tailscale's documentation.

Support: support@s-imsy.com
