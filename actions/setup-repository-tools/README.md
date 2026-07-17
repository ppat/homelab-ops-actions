# setup-repository-tools

Checkout a repository (into `current/`) and install its toolchain via [`mise`](https://mise.jdx.dev/),
with caching for both the mise installs and an optional extra path. This is the common first step
used by reusable workflows in [`ppat/github-workflows`](https://github.com/ppat/github-workflows)
before running lint/test/build steps against a checked-out repo.

## How it works

1. Checks out `current_repository`@`current_git_ref` into `current/`.
2. Restores/saves a cache for `~/.local/share/mise`, keyed by year-month plus a hash of
   `current/mise.toml` (so a new month or a changed `mise.toml` gets a fresh cache).
3. Optionally restores/saves a second cache at `extra_cache_path` under `extra_cache_key`, for
   tool-specific caches (e.g. package manager download caches) that aren't part of mise's own store.
4. Runs `jdx/mise-action` to install tools, honoring `mise_ignore_cfg` (paths to exclude from mise's
   config resolution) and an optional inline `mise_toml` (merged/overridden config, useful for
   supplying tool versions from the calling workflow rather than committing them to the repo).

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `current_repository` | yes | | Repository to checkout, `owner/repo` |
| `current_git_ref` | yes | | Ref to checkout |
| `token` | yes | | Token for checkout and `jdx/mise-action` |
| `current_fetch_depth` | no | `1` | `actions/checkout` `fetch-depth` |
| `current_persist_credentials` | no | `false` | `actions/checkout` `persist-credentials` |
| `current_dir` | no | `""` | Subdirectory of the checkout to use as mise's working directory |
| `extra_cache_path` | no | `""` | Additional path to cache (skipped if empty) |
| `extra_cache_key` | no | `""` | Cache key prefix for `extra_cache_path` |
| `mise_ignore_cfg` | no | `""` | Path (relative to `current/`) to exclude via `MISE_IGNORED_CONFIG_PATHS` |
| `mise_log_level` | no | `info` | `jdx/mise-action` log level |
| `mise_toml` | no | `""` | Inline `mise.toml` contents passed to `jdx/mise-action` |
| `mise_version` | no | pinned version (Renovate-tracked) | `jdx/mise-action` mise version |

## Outputs

None.

## Example

See [`.github/workflows/test-setup-repository-tools.yaml`](../../.github/workflows/test-setup-repository-tools.yaml)
and [`ci/test-data/setup-repository-tools/mise.toml`](../../ci/test-data/setup-repository-tools/mise.toml)
for a full working example, including supplying an inline `mise_toml`.

```yaml
- uses: ppat/homelab-ops-actions/actions/setup-repository-tools@<ref>
  with:
    current_git_ref: ${{ github.head_ref || github.ref }}
    current_repository: ${{ github.repository }}
    token: ${{ secrets.GITHUB_TOKEN }}
```
