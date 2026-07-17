# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

See [README.md](README.md) for what each action does and [DESIGN.md](DESIGN.md) for conventions and
how release/renovate/testing fit together.

## Repo layout

- `actions/<name>/action.yml` — one composite action per directory. Each is self-contained: inputs,
  outputs, and a `runs: using: composite` steps list, generally implemented as inline `shell: bash`.
  Each directory also has its own `README.md` documenting that action's inputs/outputs/behavior in
  detail — the root [README.md](README.md) only summarizes.
- `ci/test-data/<name>/` — fixtures for that action's test workflow (e.g. before/after Flux manifest
  trees, a sample `mise.toml`).
- `.github/workflows/test-<name>.yaml` — one integration test workflow per action that has runtime
  behavior worth exercising (`flux-diff`, `setup-repository-tools`). These invoke the action via
  `uses: ./actions/<name>/` against the fixtures in `ci/test-data/`, run on PRs that touch the
  action or its fixtures, and also on a weekly schedule to catch upstream drift (tool releases,
  external API changes) between PRs.
- `.github/workflows/lint.yaml`, `release.yaml`, `renovate.yaml` — repo-wide CI, all thin wrappers
  around reusable workflows from `ppat/github-workflows@<pinned-sha>`. This repo owns none of the
  linting/release/renovate logic itself — only the `with:`/`secrets:` wiring.

## Commands

This repo has no root-level toolchain manifest or build step — `pre-commit` (with yamllint,
markdownlint-cli2, shellcheck, commitlint as hooks) is the only local tooling. Install it, then:

```bash
pre-commit install
```

Run all lint hooks against the whole repo (yamllint, markdownlint, shellcheck; commitlint hook only
lints commit messages, not files):

```bash
pre-commit run --all-files
```

Run a single hook:

```bash
pre-commit run yamllint --all-files
pre-commit run shellcheck --all-files
pre-commit run markdownlint-cli2 --all-files
```

Lint a commit message locally:

```bash
echo "fix(dev-tools): bump astral-sh/uv" | npx commitlint
```

There is no build step. To exercise an action's actual runtime behavior, don't try to run its
`shell: bash` steps locally — push a branch/PR that touches the action or its `ci/test-data/**`
fixtures, which triggers the matching `test-<name>.yaml` workflow (`test-flux-diff.yaml`,
`test-setup-repository-tools.yaml`) in GitHub Actions. These also run weekly and via
`workflow_dispatch`.

## Working in this repo

- Each action in `actions/<name>/action.yml` is fully self-contained — inputs, outputs, and
  composite steps in one file. When changing one, check whether a matching
  `.github/workflows/test-<name>.yaml` + `ci/test-data/<name>/` fixture exists and update/extend it
  rather than adding a separate ad hoc test path.
- When changing an action's inputs, outputs, or behavior, update its `actions/<name>/README.md` in
  the same change — that's the source of truth for that action's interface, not the root README.
- Pin any new third-party Action reference to a commit SHA with the version as a trailing comment
  (`uses: owner/repo@<sha> # vX.Y.Z`), and add a `# renovate: datasource=... depName=...` comment
  above any tool version embedded directly in shell/YAML (see DESIGN.md) so Renovate can track it.
- Commit messages must pass commitlint (`commitlint.config.js`): conventional-commit format, scope
  must be one of `dev-tools`, `github-actions`, `renovate`, `release`, or empty, header ≤120 chars.
  Commit messages, versioning, and `CHANGELOG.md` are otherwise fully automated by release-please —
  don't hand-edit `CHANGELOG.md` or version numbers.
- `lint.yaml`, `release.yaml`, and `renovate.yaml` call reusable workflows from
  `ppat/github-workflows@<pinned-sha>` — there is very little logic to change in this repo's own
  workflow files beyond the `with:`/`secrets:`/path-filter wiring and the pinned ref itself.
