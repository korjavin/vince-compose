# Vince Analytics

Docker Compose setup for [Vince Analytics](https://github.com/vinceanalytics/vince) — a privacy-friendly, self-hosted web analytics platform (a Plausible-compatible single-binary Go app) — designed for GitOps deployment with Portainer behind Traefik.

## Features

- **Self-hosted analytics** — Plausible-compatible UI, no cookies, GDPR/CCPA friendly
- **Single binary, single container** — no Postgres, no Redis, embedded Pebble DB
- **GHCR vendoring** — upstream image mirrored weekly to GHCR for pinnable digests
- **Fully parameterized** — all configuration via environment variables
- **GitOps ready** — no secrets in the repository
- **Portainer compatible** — auto-redeploy on push via webhook

> **No forward-auth.** Vince ships its own login, and the `/api/event` ingestion endpoint must be reachable unauthenticated by tracker scripts on the sites you instrument. Adding `forward-auth@docker` here would break analytics collection.

## Prerequisites

- Docker runtime
- Portainer (for GitOps deployment)
- Traefik reverse proxy with external `traefik_default` network and a configured `certresolver`
- DNS record pointing your chosen hostname (e.g. `vince.example.com`) at the server

## Quick Start with Portainer

### 1. Deploy in Portainer

1. **Stacks** → **Add Stack**
2. **Name**: `vince`
3. **Build method**: **Repository**
4. **Repository URL**: `https://github.com/korjavin/vince-compose`
5. **Repository reference**: `deploy` ← **not master**
6. **Compose path**: `docker-compose.yml`

### 2. Configure environment variables

In the **Environment variables** section, set at minimum:

```
VINCE_HOST=vince.yourdomain.com
TRAEFIK_CERTRESOLVER=myresolver
VINCE_ADMIN_NAME=admin
VINCE_ADMIN_PASSWORD=change-me-to-a-strong-password
```

Deploy the stack. Once you can log in at `https://vince.yourdomain.com/login`, **clear `VINCE_ADMIN_NAME` and `VINCE_ADMIN_PASSWORD`** from the stack config and redeploy — you don't want long-lived bootstrap credentials sitting in Portainer.

### 3. Enable auto-deploy webhook

In Portainer: **Stack** → **Webhooks** → enable and copy the URL.
Add it as a GitHub secret named `PORTAINER_REDEPLOY_HOOK`.

### 4. Trigger first deploy

Push to `master` (or run **Deploy Vince Stack** manually in GitHub Actions).

## Environment Variables

| Variable | Default | Required | Description |
|---|---|---|---|
| `VINCE_IMAGE` | `ghcr.io/korjavin/vince-vendor:latest` | No | Docker image to deploy |
| `VINCE_CONTAINER_NAME` | `vince` | No | Container name |
| `VINCE_HOST` | — | **Yes** | Hostname for Traefik routing (e.g. `vince.example.com`) |
| `TRAEFIK_NETWORK_NAME` | `traefik_default` | No | External Traefik network name |
| `TRAEFIK_CERTRESOLVER` | `myresolver` | No | Traefik TLS cert resolver name |
| `TRAEFIK_MIDDLEWARES` | _(empty)_ | No | Extra Traefik middlewares for the router. Do NOT add forward-auth. |
| `DATA_VOLUME_NAME` | `vince_data` | No | Named volume used for `/var/lib/vince-data` |
| `VINCE_LISTEN` | `:8080` | No | Listen address inside the container |
| `VINCE_ADMIN_NAME` | _(empty)_ | First boot | Admin username to create on startup |
| `VINCE_ADMIN_PASSWORD` | _(empty)_ | First boot | Admin password to create on startup |
| `VINCE_DOMAINS` | _(empty)_ | No | Comma-separated sites to pre-create on startup |
| `VINCE_DEMO_URL` | `vinceanalytics.com` | No | Demo site shown on the public landing page |
| `VINCE_PROFILE` | `false` | No | Expose `/debug/pprof/*` (leave off in prod) |

## GitHub Secrets

| Secret | Description |
|---|---|
| `PORTAINER_REDEPLOY_HOOK` | Portainer stack webhook URL (triggers redeploy on push) |

`GITHUB_TOKEN` is automatic — no setup needed for GHCR vendoring.

## How Updates Work

### Compose changes
```
git push origin master
  → GitHub Actions: force-push deploy branch
  → Portainer: pulls updated docker-compose.yml and redeploys
```

### Image updates (vendor-images.yml)
```
Weekly cron (Monday 04:00 UTC)
  → Pulls ghcr.io/vinceanalytics/vince:latest
  → Pushes to ghcr.io/<owner>/vince-vendor:latest
  → Records digest in vendor-digests.log on deploy branch
  → Triggers Portainer redeploy
```

To pin a specific digest, set `VINCE_IMAGE` in Portainer to:
```
ghcr.io/korjavin/vince-vendor:latest@sha256:<digest>
```
(Digests are recorded in `vendor-digests.log` on the `deploy` branch.)

## Adding a site

1. Log in at `https://${VINCE_HOST}/login`.
2. **Sites** → **+ Add a website**.
3. Copy the snippet shown under **Snippet** and paste it into the `<head>` of the site you want to track.
4. The tracker script POSTs to `https://${VINCE_HOST}/api/event` — this endpoint is intentionally public.

## Links

- [Vince Analytics on GitHub](https://github.com/vinceanalytics/vince)
- [Upstream image (ghcr.io/vinceanalytics/vince)](https://github.com/vinceanalytics/vince/pkgs/container/vince)
- [Project site](https://vinceanalytics.com)
