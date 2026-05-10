# 021 CI

Central library of reusable GitHub Actions workflows for every repo in `021is/`.

One source of truth for: deploy patterns, alert wiring, PR gates. Fix a bug here, every consumer inherits the fix on the next workflow run.

## What's here

| Workflow | Purpose |
|---|---|
| [`notify-failure.yml`](.github/workflows/notify-failure.yml) | Push a single alert to Alertmanager (Telegram + email fan-out via configured receivers). Called from other deploy workflows on `failure()`. |
| [`ssh-rsync-deploy.yml`](.github/workflows/ssh-rsync-deploy.yml) | rsync source → run post-deploy script on remote. For docker-compose stacks, Caddy configs, Vaultwarden-style deploys. |
| [`ssh-systemd-deploy.yml`](.github/workflows/ssh-systemd-deploy.yml) | Atomic releases-style deploy for Bun/Node apps behind systemd. Builds in `releases/<sha>`, swaps `current` symlink, restarts service, health-checks. |
| [`cf-worker-deploy.yml`](.github/workflows/cf-worker-deploy.yml) | `wrangler deploy` for Cloudflare Workers (parked, future edge services). |
| [`pr-check-node.yml`](.github/workflows/pr-check-node.yml) | typecheck + lint + build dry-run for Node/TS repos on PRs. |

## How to use from a consumer repo

Every consumer's `.github/workflows/deploy.yml` becomes ~10 lines:

```yaml
name: Deploy
on:
  push: { branches: [main] }
  workflow_dispatch: {}

jobs:
  deploy:
    uses: 021is/ci/.github/workflows/ssh-rsync-deploy.yml@v1
    with:
      service: vault
      remote_path: /opt/vaultwarden
      rsync_paths: "compose caddy scripts"
      post_deploy: |
        cd /opt/vaultwarden
        docker compose pull && docker compose up -d
    secrets: inherit
```

`secrets: inherit` passes all repo-level secrets through. The reusable workflow declares which ones it requires; GitHub enforces presence.

## Required secrets (per consumer repo)

Every consumer that uses a deploy workflow MUST have these set:

| Secret | Used by | What |
|---|---|---|
| `ALERTMANAGER_URL` | every deploy workflow | `https://grafana.021.is/alertmanager/api/v2/alerts` |
| `ALERTMANAGER_USER` | every deploy workflow | `ci-alerts` (basic-auth user) |
| `ALERTMANAGER_PASSWORD` | every deploy workflow | basic-auth password (in vault → `Grafana stack` → `ALERTMANAGER_PUSH_PASSWORD`) |
| `SSH_HOST` | ssh-*-deploy | target host IP or hostname |
| `SSH_KEY` | ssh-*-deploy | private SSH key (pubkey installed on remote `authorized_keys`) |
| `CLOUDFLARE_API_TOKEN` | cf-worker-deploy | CF API token with Workers Scripts:Edit |
| `CLOUDFLARE_ACCOUNT_ID` | cf-worker-deploy | `1918ff4c5c5ff163bfe7b0262ceb5cdd` |

Bootstrap a new repo: `gh secret set <NAME> --repo 021is/<repo> --body "$(vault get '...' --field ...)"`.

## Versioning

Tag `v1.0.0` etc. Consumers should pin a major: `@v1`. Breaking changes get a new major; existing consumers keep working.

Currently: `@main` accepted during initial rollout. Switch to `@v1` after first stable tag.

## Adding a new reusable workflow

1. Drop a new `.github/workflows/<name>.yml` with `on: workflow_call:` and explicit `inputs:` + `secrets:`.
2. If it deploys, end with a `notify` job that calls `notify-failure.yml@main` on `failure()`.
3. Document the inputs + required secrets in the table above.
4. Tag a release (`gh release create v1.X.0`).

## Why not... ?

- **GitHub workflow templates** (org-level): they get *copied* into new repos at creation, then diverge. We want live updates across the org.
- **Composite actions**: less expressive than reusable workflows (no concurrency:, no nested job graph). Use composite when the unit is a *step* (e.g. "install deps"), use reusable when the unit is a *job graph* (e.g. "full deploy + notify").
- **One monorepo for all platform code**: that's a different conversation. This repo is just the CI primitives.

## See also

- `axon/axon.md` § Discipline #17 — every-repo CI requirements.
- `monitoring/rules/*.yml` — alert rules that fire on the metrics CI alerts produce.
- `vault → Grafana stack` — alertmanager push credentials.
