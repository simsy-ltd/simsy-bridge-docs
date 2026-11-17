# Changelog

Releases of the SIMSY Tailscale bridge image, `public.ecr.aws/simsy/simsy-bridge`. Security fixes are marked **Security**. Verify an image before running it; see the guide.

## 0.4.0 (17 November 2026)

- New `setup` command writes the bridge's configuration for you: it lists your VSlices, reads the SIM range, prints the Tailscale policy lines for your VSlice, checks your OAuth client, and writes `config.yaml`, `docker-compose.yml` and the secret files. See the guide.
- The guide is rewritten around `setup`, with an explanation of how the bridge works, the terms it uses and where to find each value. Writing the files by hand is still covered.
- `redundancy_zone` is no longer in the example configuration: `europe-primary`, the default, is the only zone.
- **Security:** dependency updates, including a fix for a crash on malformed compressed data in a library Tailscale uses (GO-2026-6865), and Tailscale 1.104.5. Built with Go 1.27.4, which fixes a crash in Go's HTTP server (GO-2026-6859).

## 0.3.0 (6 November 2026)

The first public release.

- **Security:** SIM devices can no longer query the bridge's own Tailscale DNS resolver.
- **Security:** one device can no longer fill the bridge's connection table and slow the other devices.
- **Security:** access records and webhook delivery are bounded, so a scan of the SIM range cannot exhaust disk or hide accesses.
- **Security:** webhooks are sent over HTTPS only, to public addresses only, without following redirects, and need a secret of at least 32 characters.
- **Security:** ICMP errors are accepted only from the address a device was talking to.
- The bridge keeps working with read-only Tailscale device scopes.
- The guide keeps secrets in files that Docker Compose mounts, not in `.env`, so they no longer appear in `docker inspect` or `docker compose config`. `.env` still works.
- `/status` and `/metrics` count access sessions and events the bridge could not record.
- **Security:** the guide now asks for an OAuth client that can only read machines (Auth Keys write; Devices: Core and Routes read), with the route approved by `autoApprovers` in your policy. Existing write clients keep working; replace yours when convenient.
- Restarts no longer report a missing route or handshake for their first minute.
