# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.0.0] — 2026-05-10

### Added

- `notify-failure.yml` — Alertmanager push (called by all deploy workflows on `failure()`); fans out to Telegram + email via configured AM receivers.
- `ssh-rsync-deploy.yml` — rsync + post-deploy bash on remote host. Supports `service`, `remote_path`, `rsync_paths`, `rsync_excludes`, `post_deploy`, `remote_user`.
- `ssh-systemd-deploy.yml` — atomic-symlink Bun/Node deploys behind systemd. Inputs: `service`, `remote_user`, `use_sudo`, `release_root`, `env_symlink_src`, `install_cmd`, `pre_build_cmd`, `build_cmd`, `migrate_cmd`, `health_url`, `keep_releases`, `rsync_excludes`.
- `cf-worker-deploy.yml` — `wrangler deploy` for Cloudflare Workers (parked, future).
- `pr-check-node.yml` — typecheck + lint + build for Node/TS repos. Inputs: `runtime` (bun|node), `install_cmd`, `pre_build_cmd`, `typecheck_cmd`, `lint_cmd`, `build_cmd`, `build_env` (multi-line placeholder env), `skip_build`.

### Locked conventions

- Standard secret names across all consumer repos: `ALERTMANAGER_URL` / `_USER` / `_PASSWORD`, `SSH_HOST` / `_KEY`, `CLOUDFLARE_API_TOKEN` / `_ACCOUNT_ID`.
- Every deploy workflow MUST end with a `notify` job calling `notify-failure.yml` on `failure()`. No inline alert curl in consumers.
- Moving `v1` tag follows latest `v1.x.y` release; consumers pin `@v1` for major-version stability.
- Repo is `access_level: organization` so any `021is/*` repo can call these workflows.

### Migrated to v1.0.0 (initial rollout)

- `021is/parked` — deploy.yml 50 lines → 5 lines via `cf-worker-deploy.yml`.
- `021is/021-vault` — deploy.yml 70 lines → 22 lines via `ssh-rsync-deploy.yml`. Secret names normalized: `VAULT_HOST` → `SSH_HOST`, `VAULT_SSH_KEY` → `SSH_KEY`, `ALERTMANAGER_PUSH_*` → `ALERTMANAGER_*`.
- `021is/delvix` — deploy.yml 169 lines → 30 lines via `pr-check-node.yml` (quality gate) + `ssh-systemd-deploy.yml` (deploy). Secret rename: `SSH_PRIVATE_KEY` → `SSH_KEY`.

[Unreleased]: https://github.com/021is/ci/compare/v1.0.0...HEAD
[1.0.0]: https://github.com/021is/ci/releases/tag/v1.0.0
