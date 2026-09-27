---
title: Unraid + Portainer
summary: Deploy Paperclip on Unraid via a Portainer Git stack, pulling the published upstream image
---

Deploys the published `ghcr.io/paperclipai/paperclip` image (upstream's own
CI build) via a Portainer stack sourced from `docker/docker-compose.unraid.yml`
in this fork. This fork does not build the image itself — it only owns the
compose file, the deploy workflow, and this doc.

## Prerequisites

- Portainer CE running on the Unraid box.
- A GitHub Actions self-hosted runner registered **to this fork**
  (Settings → Actions → Runners → New self-hosted runner). If you already
  run a self-hosted runner for another repo (e.g. a `homelab` stack), note
  that GitHub does not share repo-scoped runners across unrelated repos on
  a personal account — you need a separate runner registration for this
  repo, even if it's the same physical machine running both.
- Nginx Proxy Manager + Cloudflare Tunnel already set up for other
  LAN-only services (same pattern used here).

## 1. Create the appdata directory

```sh
mkdir -p /mnt/user/appdata/paperclip
```

Add `paperclip` to Unraid's **Appdata Backup / Restore** plugin's
container list, and confirm it's in the "stop containers during backup"
set. The embedded Postgres data under this path is only guaranteed
consistent after a clean container shutdown — a live copy while it's
running risks a torn snapshot, the same reason a raw copy of a live
Postgres data directory is unsafe.

## 2. Create the Portainer stack

Stacks → Add stack → **Git repository**:

- Repository URL: this fork's HTTPS clone URL
- Repository reference: `refs/heads/master`
- Compose path: `docker/docker-compose.unraid.yml`
- GitOps updates: leave **off** — redeploys are triggered explicitly by
  the GitHub Actions workflow below, not polled/auto-pulled by Portainer.

## 3. Set environment variables

In the stack's **Environment variables** UI (not committed to git):

| Variable | Required | Notes |
|---|---|---|
| `BETTER_AUTH_SECRET` | yes | `openssl rand -hex 32` |
| `PAPERCLIP_TOOL_ACTION_SIGNING_SECRET` | yes | `openssl rand -hex 32` |
| `PAPERCLIP_PUBLIC_URL` | no | defaults to `https://paperclip.rosenlund-fam.com` |
| `PAPERCLIP_ALLOWED_HOSTNAMES` | no | extra LAN/Tailscale aliases, comma-separated |
| `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` | no | enables `claude_local` / `codex_local` adapters inside the container |
| `PAPERCLIP_IMAGE_TAG` | no | defaults to `latest` (upstream's most recent tagged release) |

Portainer strips a bare `$` from env var values — if a secret ever
contains one (e.g. an Argon2 hash), escape it as `$$` or it will be
silently corrupted.

## 4. Reverse proxy

Nginx Proxy Manager → Add Proxy Host:

- Domain: `paperclip.rosenlund-fam.com`
- Forward to: `<unraid-lan-ip>:3100`
- SSL: same certificate/setup used for other internal hosts

Add a Cloudflare Tunnel public hostname route for
`paperclip.rosenlund-fam.com` pointing at the NPM proxy host, matching
the existing pattern (Cloudflare → NPM → container). No host port is
exposed directly to the internet.

## 5. First admin

With `PAPERCLIP_DEPLOYMENT_MODE=authenticated` and
`PAPERCLIP_DEPLOYMENT_EXPOSURE=private`, open the URL, sign in or create
an account, then choose **Claim this instance** on the setup screen.

## 6. Redeploy on push

`.github/workflows/deploy-unraid.yml` pokes the stack's Portainer webhook
from the self-hosted runner whenever `docker/docker-compose.unraid.yml`
changes on `master` (or on manual dispatch). Set it up once:

1. In the Portainer stack, enable **Webhook** and copy the generated URL.
2. Add it as the repo secret `PORTAINER_STACK_WEBHOOK_URL`
   (Settings → Secrets and variables → Actions).

Pushing a change to the compose file — or running the workflow manually
to pick up a new upstream `:latest` release without a compose change —
triggers Portainer to pull and recreate the container.
