# Bridge events and webhooks

The bridge sends events to the HTTPS address you give it, so your own systems know when it changes state and who accessed which device. Access events are the evidence trail for NIS2, the EU Cyber Resilience Act and UK PSTI: they record access, they do not make you compliant by themselves.

## Events

| Event | When |
|---|---|
| `bridge.provisioning` | The bridge is being set up, or set up again after being re-enabled. |
| `bridge.online` | Everything is working: SIMSY tunnel up, Tailscale node running, route approved. |
| `bridge.degraded` | Something needs attention. `reason` says what (below); `detail` says more. |
| `bridge.disabled` | The bridge was switched off. Nothing is forwarded; SIMSY stays configured. |
| `bridge.deleting`, `bridge.deleted` | The bridge is being removed, and has been. |
| `policy.downlink_changed` | Uplink-only was switched on or off. `data.downlink_allowed` is the new setting. |
| `access.opened` | A device on your tailnet started talking to a SIM device. |
| `access.closed` | That session ended (5 minutes without traffic), with its duration and bytes. |

Degraded reasons: `handshake_stale` (no recent WireGuard handshake with SIMSY), `route_unapproved`, `tags_not_assignable` (your tailnet policy does not let the bridge use its tags), `ts_offline`, `simsy_api_error`, `tailscale_api_error`, `simsy_key_unknown`, `config_error`, `data_plane_error`.

## Payload

```json
{
  "id": "evt_6b1d0f3c9a2e4f5d8c7b6a51",
  "type": "access.closed",
  "tenant": "fleet-a",
  "time": "2026-10-13T12:12:05Z",
  "data": {
    "who": "sam@example.com",
    "device": "sam-laptop",
    "tailnet_address": "100.101.102.103",
    "sim_name": "depot-router-3",
    "sim_address": "10.20.0.70",
    "started": "2026-10-13T12:00:01Z",
    "ended": "2026-10-13T12:12:05Z",
    "duration_seconds": "724",
    "bytes_to_sim": "18342",
    "bytes_from_sim": "1203391"
  }
}
```

`who` and `device` are what your tailnet knows about the address that opened the session. Sessions are counted per tailnet address and SIM; replies to connections the SIM device opened itself are not access.

## Delivery

- `POST` with `Content-Type: application/json`. Answer with any 2xx status within 15 seconds.
- Headers: `X-SIMSY-Event` (the type), `X-SIMSY-Delivery` (the event ID, the same on every retry: use it to ignore duplicates), `X-SIMSY-Signature`.
- Events for one bridge arrive in order. If your endpoint fails, the bridge retries after 10 s, 30 s, 1, 2, 5 and 10 minutes, then every 15 minutes, for 24 hours. Events are kept on disk, so a bridge restart does not lose them. Later events wait for earlier ones.

## Verifying the signature

`X-SIMSY-Signature` looks like `t=1792000000,v1=5f2b…`. `v1` is the hex HMAC-SHA256, keyed with your webhook secret, of the timestamp, a full stop, and the raw request body. Check it before trusting the event, and reject timestamps more than five minutes from now so a captured request cannot be replayed.

Python:

```python
import hashlib, hmac, time

def verify(secret: bytes, header: str, body: bytes, tolerance=300) -> bool:
    parts = dict(p.split("=", 1) for p in header.split(","))
    t, sig = parts.get("t", ""), parts.get("v1", "")
    if not t.isdigit() or abs(time.time() - int(t)) > tolerance:
        return False
    expected = hmac.new(secret, f"{t}.".encode() + body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, sig)
```

Node.js:

```js
const crypto = require("crypto");

function verify(secret, header, rawBody, toleranceSeconds = 300) {
  const parts = Object.fromEntries(header.split(",").map(p => p.split("=", 2)));
  const t = Number(parts.t);
  if (!Number.isInteger(t) || Math.abs(Date.now() / 1000 - t) > toleranceSeconds) return false;
  const expected = crypto.createHmac("sha256", secret).update(`${parts.t}.`).update(rawBody).digest("hex");
  return parts.v1 && crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(parts.v1));
}
```

Verify against the raw body exactly as received, before any JSON parsing or reformatting.
