# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Initial scaffold of reusable workflows:
  - `notify-failure.yml` — Alertmanager push (called by all deploy workflows on `failure()`).
  - `ssh-rsync-deploy.yml` — rsync + post-deploy bash on remote host.
  - `ssh-systemd-deploy.yml` — atomic-symlink releases-style Bun/Node deploys behind systemd, with health-check + release pruning.
  - `cf-worker-deploy.yml` — `wrangler deploy` for Cloudflare Workers.
  - `pr-check-node.yml` — typecheck + lint + build for Node/TS repos on PRs.
- README documenting the consumer-call pattern and required secrets.
- AGENTS.md / CLAUDE.md per discipline #17.

### Locked

- Standard secret names: `ALERTMANAGER_URL` / `_USER` / `_PASSWORD`, `SSH_HOST` / `_KEY`, `CLOUDFLARE_API_TOKEN` / `_ACCOUNT_ID`.
- All deploy workflows MUST end with a `notify` job calling `notify-failure.yml` on `failure()`. No inline alert curl in consumers.
- Consumers pin `@v1` after first tagged release; `@main` accepted only during initial rollout.

[Unreleased]: https://github.com/021is/ci/compare/HEAD...HEAD
