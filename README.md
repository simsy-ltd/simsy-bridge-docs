# SIMSY Tailscale bridge

The bridge puts the SIMs on your SIMSY VSlice onto your Tailscale network. Your devices keep their SIMSY addresses and answer on your tailnet; nothing is installed on them. You run the bridge as a single container on a Linux host; it sets up SIMSY and Tailscale itself and keeps them correct.

```
your devices ── SIMSY SIMs ── SIMSY core ══ encrypted tunnel ══ bridge (your host) ══ your tailnet
```

## Documents

Also on the SIMSY website: https://s-imsy.com/developers/tools/tailscale

| File | What |
|---|---|
| [guide.md](guide.md) | Setting up and running the bridge |
| [webhooks.md](webhooks.md) | Bridge events and access records sent to your systems, and how to verify their signatures |
| [docker-compose.yml](docker-compose.yml) | The compose file from the guide |
| [config.example.yaml](config.example.yaml) | An example `config.yaml` |
| [env.example](env.example) | The variables your `.env` file needs |

## The image

`public.ecr.aws/simsy/simsy-bridge` (Amazon ECR Public; no login needed to pull), for `linux/amd64` and `linux/arm64`. Every release is signed, with an SBOM and build provenance; the guide shows how to verify a signature before you run it.

## Support

support@s-imsy.com

This repository holds published documentation only. It is maintained by SIMSY and updated from SIMSY's own sources, so changes made here are not kept: please send corrections to support instead.
