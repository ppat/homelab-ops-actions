# Design

## Purpose

This repo hosts standalone, reusable GitHub composite Actions consumed by workflows in
[`ppat/github-workflows`](https://github.com/ppat/github-workflows) and by other `homelab-ops`/`ppat`
repositories directly. Actions live here (rather than inline in workflow YAML) so they can be
versioned, pinned by SHA, and tested independently of any one consuming repo.

See [CLAUDE.md](CLAUDE.md) for repo layout.

## Conventions used by the actions themselves

- Actions favor plain `shell: bash` composite steps with `set -euo pipefail`, explicit input
  validation with clear `ERROR:` messages, and `jq`/`gh` for GitHub API and JSON work, over
  JavaScript/TypeScript actions. There is no build step for any action in this repo.
- External tool versions pinned as action `default:` values (e.g. `mise_version` in
  `setup-repository-tools`) or workflow `env:` (e.g. `UV_VERSION` in
  `test-setup-repository-tools.yaml`) carry a `# renovate: datasource=... depName=...` comment
  driving Renovate's custom regex manager (`.github/renovate.json`) — this is what lets Renovate
  bump versions embedded in shell/YAML rather than a lockfile.
- Third-party Actions (`actions/checkout`, `actions/cache`, `jdx/mise-action`, `docker://...`) are
  pinned to a commit/digest with the version as a trailing comment, per
  `helpers:pinGitHubActionDigests`/`docker:pinDigests` in the Renovate config.
- `create-signed-commit` intentionally exits non-zero after a successful commit — the commit it
  just created is expected to trigger the workflow's own re-run, which then finds no staged changes
  and exits 0. Don't treat that as a bug.

## Release & versioning

Releases are automated by `release-please` (via `ppat/github-workflows`'s `release-please.yaml`,
triggered from `release.yaml` on push to `main`). Conventional commit messages (enforced by
commitlint, see below) drive version bumps and `CHANGELOG.md` generation. Consumers pin to a
released tag (or commit SHA) of this repo when referencing `homelab-ops-actions/actions/<name>`.

## Dependency updates

Renovate (`.github/renovate.json`) runs on a schedule and via `renovate.yaml`, opening PRs to bump
pinned Action SHAs and tool versions marked with `# renovate:` comments. `packageRules` excludes
`ci/test-data/**` from updates since those fixtures are intentionally frozen for reproducible diffs.
