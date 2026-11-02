# Changelog

Releases of the SIMSY Tailscale bridge image, `public.ecr.aws/simsy/simsy-bridge`. Security fixes are marked **Security**. Verify an image before running it; see the guide.

## Unreleased

- **Security:** SIM devices can no longer query the bridge's own Tailscale DNS resolver.
- **Security:** one device can no longer fill the bridge's connection table and slow the other devices.
- **Security:** access records and webhook delivery are bounded, so a scan of the SIM range cannot exhaust disk or hide accesses.
- **Security:** webhooks are sent over HTTPS only, to public addresses only, without following redirects, and need a secret of at least 32 characters.
- **Security:** ICMP errors are accepted only from the address a device was talking to.
- The bridge keeps working with read-only Tailscale device scopes.
- **Security:** the guide now asks for an OAuth client that can only read machines (Auth Keys write; Devices: Core and Routes read), with the route approved by `autoApprovers` in your policy. Existing write clients keep working; replace yours when convenient.
- Restarts no longer report a missing route or handshake for their first minute.
