# AGENT.md — 021/ci

Coding rules: 021is/code-rules v0.6.0 (PRINCIPLES; workflows only, no language file), then
§ Conventions below.

> **If you touched this project, update this file before you commit.**
> The next agent loads it on boot to continue from where you left off.
> One or two lines per entry — never essays.

## Stack

- **Language:** YAML (GitHub Actions). No runtime code lives here.
- **Versioning:** semver via git tags (`v1.0.0`, `v1.1.0`, …). Consumers pin `@v1`.
- **Its own CI:** `self-check.yml` runs `check-repo.yml` from the same commit (`repo / check`, required by main's ruleset): every YAML/JSON parses, `bash -n`, gitleaks over the tree and history. The other reusable workflows run only when a consumer calls them.

## Boot

```bash
git clone git@github.com:021is/ci.git
```

That's it. Edit a workflow, commit, tag a release. Consumers pick up the change on the next workflow invocation.

## Layout

- `.github/workflows/` — every reusable workflow lives here.
  - `notify-failure.yml` — Alertmanager push (called by all deploy workflows on `failure()`).
  - `ssh-rsync-deploy.yml` — rsync + post-deploy bash.
  - `ssh-systemd-deploy.yml` — atomic-symlink Bun/Node deploys behind systemd.
  - `cf-worker-deploy.yml` — wrangler deploy for CF Workers.
  - `pr-check-node.yml` — typecheck + lint + build for PRs on Node/TS repos.
  - `pr-check-gradle.yml`, `ghcr-compose-deploy.yml`, `release-please.yml` — Kotlin/Gradle PR gate, GHCR image + compose roll, and release-please (as a GitHub App: `RELEASE_APP_CLIENT_ID` / `RELEASE_APP_PRIVATE_KEY`, org secrets); copied from DanceClub-Platform/ci (private, not callable from here), runner label `[self-hosted, "021"]` (private repos only). Re-sync by diff when that repo changes.
  - `check-go.yml` — the Go module gate (two jobs, `check` and `test`; tests `-race -shuffle=on`), reads 021is/code-rules with the release app's token; `ghcr-compose-deploy.yml`'s `go_module` / `image_var` build and switch a Go twin. Both ported from DanceClub-Platform/ci (2026-11-17, places' Go twin first). `danceclub_platform: true` mints each job a read token for DanceClub's platform module (TF144, 021is/designer). Its `image` job still writes the Kotlin env names (`SPRING_PROFILES_ACTIVE`, `SECRET_DB_URL`): a Go service needs DanceClub-Platform/ci's Go names there first.
  - `main-gated.yml` — on push to main, every new commit came through a merged PR whose head was green.
- `README.md` — humans + how to call from a consumer.
- `CHANGELOG.md` — Keep-a-Changelog with `## [Unreleased]` block; `git tag v*` rolls via `git cliff`.

## Conventions

- **Inputs:** explicit `type:` + `description:` + sensible defaults. Required inputs go first.
- **Secrets:** declared `required: true` so GitHub blocks calls without them. Use `secrets: inherit` in consumers — don't list each one.
- **Concurrency:** `group: ${{ inputs.service }}-deploy` so two pushes to the same service don't race. `cancel-in-progress: true` for idempotent deploys; `false` for atomic-symlink ones (don't interrupt mid-swap).
- **Naming:** kebab-case files. `<deploy-style>-deploy.yml` for deploys, `pr-check-<runtime>.yml` for PR gates, `notify-<channel>.yml` for notifications.
- **Failure path:** every deploy workflow ends with a `notify` job that calls `notify-failure.yml@main` on `failure()`. Don't write inline curl-to-AM scripts in consumers — they drift.
- **Nothing stays on the runner:** SSH keys and known_hosts live in `$RUNNER_TEMP/ssh` (used with `ssh -F`), removed in an `always()` step; never `~/.ssh`. The runners are persistent.
- **A roll checks the platform:** the pulled image's os/arch must equal the host's before `up -d`.

## Active threads

- v1 rollout (2026-05-10): migrating consumers `parked`, `021-vault`, `delvix`, `021-mon` (when it has CI) to call these reusables.
- Future: `pr-check-gradle.yml` for Kotlin/Gradle repos (xauth, DC backend, 021-platform-kotlin).
- Future: `docker-compose-deploy.yml` if a pattern emerges that ssh-rsync-deploy can't cover.

## Recent context

- 2026-05-10 — Initial scaffold. Five reusable workflows: notify-failure, ssh-rsync-deploy, ssh-systemd-deploy, cf-worker-deploy, pr-check-node. Standard secret names locked: `ALERTMANAGER_URL` / `_USER` / `_PASSWORD`, `SSH_HOST` / `_KEY`, `CLOUDFLARE_API_TOKEN` / `_ACCOUNT_ID`.
- Decision: nest `notify-failure.yml` via `uses:` rather than inline curl. Trade slight indirection for one-place fixes.
- Decision: `@main` accepted during initial rollout; switch consumers to `@v1` after first stable tag lands.

## Gotchas

- **Reusable workflow `uses:` paths must be repo-qualified** (`021is/ci/.github/workflows/foo.yml@main`), even when referenced from within the same repo. Relative paths (`./.github/workflows/foo.yml`) work too but break when called from a different repo.
- **`secrets: inherit` passes ALL repo secrets** to the called workflow. Fine when caller and callee trust each other (same org), but means a reusable workflow with `${{ secrets.X }}` reads X from the *caller's* repo, not its own.
- **Private repos calling reusable workflows from another private repo in the same org** requires `Settings → Actions → General → Access` set on the *called* repo. Default for new private repos is "Not accessible" — must be flipped to "Accessible from repositories in the '021is' organization".
- **Nested reusable workflow depth limit is 4.** Don't build deep call chains.
- **`workflow_call` cannot use matrix-strategy outputs.** If you need fan-out, do it in the caller.

## Where to read more

- `README.md` — usage for humans + consumer table.
- `axon/axon.md` § Discipline #17 — every-repo CI standard this enforces.
- `monitoring/rules/` — alert rules consuming the metrics CI alerts produce.

## Mirror

`CLAUDE.md` is a single-line `@AGENTS.md` so Claude Code auto-loads this on dir entry.
