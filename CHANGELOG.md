# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.1.1] — 2026-10-28

### Fixed

- `ssh-rsync-deploy.yml` installs rsync when the runner has none. The self-hosted runner image
  ships without it, so 021-mon's first deploy there stopped at `rsync: command not found` (before
  copying anything).

## [1.1.0] — 2026-10-28

### Added

- `runs_on` on `ssh-rsync-deploy.yml` and `notify-failure.yml`: the runner as JSON, default
  `"ubuntu-latest"`, so every existing caller is unchanged. `'["self-hosted","021"]'` runs the job
  on 021is's own slot on the CI box (danceclub-infra `ci/README.md` § 021is): no billed minutes,
  and it runs while the org's Actions spending is blocked, as it was on 2026-10-28. The deploy
  passes it on to its failure notice.
- `spine-quality-gate.yml` (2026-05-25): the PR check for Spine-shaped repos, every static gate in
  parallel.
- `spine-metrics.yml` (2026-05-25): post-merge metrics from a Spine repo to 021-mon.
- `deploy-helvix` composite action (2026-06-01): build, push, bump `image.tag` in `021is/021-infra`.

### Changed

- `actions/checkout`, `setup-node`, `cache`, `upload-artifact` and `download-artifact` at `@v5`
  (2026-06-01), before GitHub forced Node 24 on 2026-06-16.

### Fixed

- `deploy-helvix` takes the registry as host and path apart, for `ghcr.io/021is` (2026-06-01).

## [1.0.1] — 2026-05-10

### Fixed

- `ssh-systemd-deploy.yml`'s prune deleted every release, the active one included: `KEEP` was
  expanded in the remote shell, where it was unset. It is now interpolated on the runner, and the
  just-deployed release is always excluded.

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

[Unreleased]: https://github.com/021is/ci/compare/v1.1.1...HEAD
[1.1.1]: https://github.com/021is/ci/compare/v1.1.0...v1.1.1
[1.1.0]: https://github.com/021is/ci/compare/v1.0.1...v1.1.0
[1.0.1]: https://github.com/021is/ci/commits/v1.0.1
[1.0.0]: https://github.com/021is/ci/commits/9edbf6f
