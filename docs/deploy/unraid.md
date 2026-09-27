---
title: Unraid + Portainer
summary: Deploy Paperclip on Unraid via Portainer, pulling the published upstream image
---

Deploys the published `ghcr.io/paperclipai/paperclip` image (upstream's own
CI build) via a Portainer stack pasted from `docker/docker-compose.unraid.yml`
in this fork. This fork does not build the image itself — it only owns the
compose file, the deploy workflow, and this doc.

Note: the stack is **pasted into Portainer's Web editor**, not sourced from
Portainer's Git-repository build method. This repo contains symlinks under
`.claude/skills/` (upstream's own files), and Portainer's git-stack clone
refuses to clone any repository containing a symlink, anywhere in the tree,
for security reasons. The compose file still lives in git here as the
source of truth — just copy its contents into Portainer manually when it
changes.

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

Stacks → Add stack → **Web editor** → paste the contents of
`docker/docker-compose.unraid.yml`. Re-paste and update the stack whenever
that file changes in git.

## 3. Set environment variables

In the stack's **Environment variables** UI (not committed to git):

| Variable | Required | Notes |
|---|---|---|
| `BETTER_AUTH_SECRET` | yes | `openssl rand -hex 32` |
| `PAPERCLIP_TOOL_ACTION_SIGNING_SECRET` | yes | `openssl rand -hex 32` |
| `PAPERCLIP_PUBLIC_URL` | no | defaults to `https://paperclip.rosenlund-fam.com` |
| `PAPERCLIP_ALLOWED_HOSTNAMES` | no | extra LAN/Tailscale aliases, comma-separated |
| `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` | no | only for API-key (metered) billing — see §7 for using an existing subscription instead |
| `PAPERCLIP_IMAGE_TAG` | no | defaults to `latest` (upstream's most recent tagged release) |

Do not set `DATABASE_URL` — this stack runs paperclip's embedded Postgres
(single container, no separate `db` service), which manages its own
database automatically. `DATABASE_URL` is only for the full-stack compose
variant with a separate `postgres` container; pointing it at a host that
doesn't exist in this stack (e.g. `postgres-paperclip`) will break startup.

`PAPERCLIP_DEPLOYMENT_MODE` and `PAPERCLIP_DEPLOYMENT_EXPOSURE` are already
hardcoded in the compose file itself — no need to set them again here.

Portainer strips a bare `$` from env var values — if a secret ever
contains one (e.g. an Argon2 hash), escape it as `$$` or it will be
silently corrupted.

## 4. Reverse proxy

Nginx Proxy Manager → Add Proxy Host:

- Domain: `paperclip.rosenlund-fam.com`
- Forward to: `<unraid-lan-ip>:3131`
- SSL: same certificate/setup used for other internal hosts

`PAPERCLIP_PUBLIC_URL` should be the browser-facing HTTPS domain
(`https://paperclip.rosenlund-fam.com`), not the LAN `ip:3131` address —
that address is only what NPM forwards *to*, not what users or the
auth/callback flow should see.

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

The webhook re-pulls the image (`pull_policy: always` in the compose file)
and recreates the container — that covers picking up a new upstream
`:latest` release, including via manual dispatch with no compose change.
It does **not** re-fetch the compose file itself from git (the stack is
pasted, not git-sourced — see the note at the top of this doc): if you
edit `docker/docker-compose.unraid.yml`, re-paste it into Portainer's
Web editor yourself before or instead of relying on the workflow.

## 7. Use your Claude Pro / ChatGPT Plus subscriptions instead of API keys

Paperclip's `claude_local` and `codex_local` adapters both support logging
in with an existing subscription instead of an API key — no extra
per-token billing. The image sets `HOME=/paperclip`, so credentials from
either CLI land under the `/paperclip` appdata mount and survive container
recreation (redeploys, image updates) without re-logging in.

1. In Portainer, open the `paperclip` container's **Console** (`>_`), shell
   as `node`.
2. Run `claude login` and follow the OAuth flow (works with Claude Pro or
   Max — Pro's lower usage limits apply, so if you run several agents
   concurrently against it you may hit throttling sooner than with a
   per-agent API key; fine for a personal single-user instance).
3. Run `codex login` and follow the ChatGPT OAuth flow (Plus, Pro, Team, or
   Enterprise all work).
4. Leave `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` unset in the stack's
   environment variables — their presence would make the adapters use
   metered API billing instead of the subscription login.
5. In the Paperclip UI, configure your agents' `claude_local` / `codex_local`
   adapters normally; no additional adapter-level env is needed once the
   host CLI login exists.

Both subscription logins are shared across every agent using that adapter
on this instance (they all read the same host-owned credential file) — fine
for personal use, but if you ever run many agents concurrently, a per-agent
API key avoids them contending over one subscription's rate limit.
