# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Changed

- `ghcr-compose-deploy.yml`: `danceclub_platform: true` for a `go_module` on DanceClub's private
  platform module. The build mints the same read token as check-go.yml (secrets
  `RELEASE_APP_CLIENT_ID` and `RELEASE_APP_PRIVATE_KEY`, now declared) and removes it right after
  the binary is built.

- `deploy-helvix`: the 021-infra bump is a PR that merges itself once 021-infra's checks pass,
  and the step waits for the merge (15 minutes at most), since 021-infra's main takes only PRs.
  Its commit drops `[skip ci]`, which would skip those checks.

### Added

- `check-go.yml`: `danceclub_platform: true` for a 021 module on DanceClub's private platform
  module: `check`, `test` and `image` each mint a read token for DanceClub-Platform/danceclub-platform
  from the release app (installed in both orgs) through the new composite action
  `.github/actions/danceclub-platform-token`: the token in one git config file under RUNNER_TEMP
  (0600), included by GIT_CONFIG_*, removed by an always-run step that reads the action's
  `config-file` output. With it, `image` builds with BuildKit (`COPY --chmod`) and starts the
  service with the platform's configuration names (`APP_ENV`, `DB_URL`, `DB_USER`,
  `SECRET_DB_PASSWORD`, `NATS_URL`, `SECRET_NATS_SEED` made by nats-box, `INGRESS_CIDRS`), as
  DanceClub-Platform/ci does; without it, the Kotlin-era names. First callers: 021is/designer
  (`check`, `test`); 021is/support (`image`, TF189: unproven until 021is/support's first PR run).
- `check-repo.yml`: the PR check floor for repos with no build of their own (every YAML and JSON
  file parses, every shell script passes `bash -n`, gitleaks over the tree and history), as
  `<caller job> / check`. `self-check.yml` runs it on this repo's own PRs.

- `check-go.yml`: the gate for a Go module, in two jobs at once (`check`: build, vet,
  golangci-lint with 021is/code-rules' shared config at `code_rules_ref`, go-rules, govulncheck,
  gitleaks; `test`: the race tests), so a run stays under 3 minutes on the org's two slots. Copied
  from DanceClub-Platform/ci's one-job version, which a 021is repo cannot call.
- `ghcr-compose-deploy.yml`: `go_module` (the static Go binary built outside Docker, the commit in
  its vcs stamp) and `image_var` (the whole image reference written to `.env`, the one line that
  says which of a service's twins runs; `tag_var` keeps naming the other, the way back). Both
  optional: every existing caller is unchanged.
- `check-go.yml` `image_service` (with `image_schema`, `image_extensions`, `image_env`): a job
  `image` that builds the deploy image and runs it as the host does: non-root, `/actuator/health`
  200 against a real Postgres, `/actuator/info` naming the commit, SIGTERM exit 0 inside 60 s.
  `ghcr-compose-deploy.yml` `info_url`: the roll fails unless the running build names its commit.
  Both as DanceClub-Platform/ci's (TF96), one design.

- `release-please.yml`: versions a repo from its conventional commits and tags `v<version>` when
  its release PR is merged, acting as a GitHub App (`RELEASE_APP_CLIENT_ID`,
  `RELEASE_APP_PRIVATE_KEY`). Copied from DanceClub-Platform/ci, whose private workflow a 021is
  repo cannot call: places' release workflow failed on every push before a job started.

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
