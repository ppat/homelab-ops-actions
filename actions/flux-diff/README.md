# flux-diff

Run [`flate diff`](https://github.com/home-operations/flate) between a "before" and "after"
version of a Flux-managed Kubernetes manifest tree (e.g. two checkouts, or a PR base vs. head) and
expose the resulting unified diff as an output.

## How it works

1. Runs [`setup-repository-tools`](../setup-repository-tools/README.md), which checks out
   `github.ref` into `current/` and installs `flate_version` through mise's `aqua:` backend. For a
   `pull_request` event `github.ref` is the PR's merge commit, so a branch behind its base does not
   diff the base's newer changes in reverse. The `aqua:` backend verifies the release against the
   cosign bundle flate publishes with it, plus the checksum. The `github:` backend would check only
   the checksum, because flate publishes no GitHub artifact attestation.
2. Checks out the PR's base branch (the default branch outside a PR) into `base/`.
3. Runs `flate diff <resource_type> --path current/<path_after> --path-orig base/<path_before>
   --output diff --concurrency 1` and writes the result to `diff.patch`. A tree missing on one side
   is created empty first, so a tree added or removed by the PR renders as a full addition or
   removal.
4. Reads `diff.patch` into the `diff` output.

The calling job needs no checkout or tool setup of its own.

flate skips Secrets and CustomResourceDefinitions and strips non-semantic metadata (Helm chart
labels, `checksum/*` annotations, app versions) by default, so diffs reflect real spec changes only.
A render failure on either side fails the step instead of producing a partial diff.

## Sources

flate resolves every `sourceRef` itself, so callers no longer stage sources on disk:

| Source in the tree | How flate resolves it |
| --- | --- |
| `GitRepository` whose `spec.url` matches the checkout's `origin` remote | the local checkout of that side |
| `GitRepository` referenced but not declared anywhere in the tree | the local checkout of that side |
| Any other `GitRepository`, `OCIRepository`, `HelmRepository` | fetched over the network at its pinned ref |

The first rule depends on the checkout carrying its real remote URL, which `actions/checkout`
provides. When `origin` is missing or points elsewhere, the tree's own `GitRepository` stops being
read from the checkout, and changes to the checkout's own files drop out of the diff without an
error. Modules fetched from other repositories still render, so the diff looks plausible. The same happens when flate runs from a `git worktree`, where `.git` is a file rather than a
directory, which matters only when reproducing a diff locally: use a plain clone.

## Gotchas

- Each path is one scan root. Two objects with the same kind, namespace and name in different
  subdirectories of it make flate refuse with `conflicting entry coordinates`, because it reads them
  as two clusters. Two same-named Flux `Kustomization` objects in one directory are not reported:
  the last file read wins and the other is silently absent from the diff. Keep fixtures that
  represent one object at several points in time out of the scan root with `krmignore`, whose
  patterns are relative to each path and apply to both sides.
- flate reconciles the whole graph whichever `resource_type` is asked for, so a `kustomization` diff
  templates HelmReleases too. Helm flags such as `--api-versions` belong in `other_params` for every
  resource type, or a chart gated on a missing API fails the `kustomization` diff. Because each run
  renders everything anyway, `all` reports both outputs for the cost of one: the Kustomization
  output (including the HelmRelease objects themselves) and the Helm-rendered output.
- Reconciles run one at a time (`--concurrency 1`). In parallel, a HelmRelease whose `valuesFrom`
  object is produced by a different Kustomization with no `dependsOn` between them is sometimes
  rendered before that object exists, on one side only. The diff then reports the missing values as
  a change nobody made, differently on each run.
- An edit to such a `valuesFrom` object is reported as a change to that object, but the consuming
  HelmRelease is not re-rendered, so its rendered effect is absent from the diff.
- Two identical trees render nothing, so a green run on a PR that changes no manifests says nothing
  about whether a real change would render.
- The diff uses 3 lines of context; flate has no flag to change it.

## Choosing the pinned version

`flate_version` defaults to a release pinned here and tracked by Renovate, so a bump is one PR in
this repository and reaches each consumer with its next ref bump of this action.

flate links Helm as a library, and that Helm renders every chart the diff reports on. Its major must
track the one Flux's `helm-controller` links against: when the two drift apart, the diff still
renders and still looks healthy, but it stops describing what Flux would deploy. The failure is
silent and asymmetric. A chart that renders identically under both majors hides the drift, so the
modules that expose it are found by rendering the whole estate under both versions and comparing,
not by reading a job's exit code. Check the `helm.sh/helm/vN` line in flate's `go.mod` at the new
tag before accepting a bump.

Two rendering differences observed across a Helm major boundary, as a sense of what changes: a
quoted-numeric value in a chart's values (`"08"`) losing its quoting and arriving as an integer, and
templates gated on API availability rendering under one major and not the other.

Because the diff renders the before and the after tree with the *same* binary, a systematic
rendering difference cancels and does not surface as review noise. The cost of being on the wrong
major is fidelity, not churn.

A flate bump also needs a diff replayed against a consumer whose tree declares `GitRepository`
objects pinned to tags of other repositories (a cluster definition), not only against these
fixtures. The fixtures exercise only the local-checkout rules above, and the network-fetch rule is
where flate's source discovery has regressed between versions: the regression showed up as
`BootstrapAliases`, a failing `diff kustomization`, and a `diff helmrelease` that exited 0 with no
output. This repository's own test passes on a bump, since its fixtures reach only the local-checkout
rules, and so does a consumer's ref-bump PR, since it compares two identical trees and renders
nothing.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `resource_type` | yes | | `all`, `helmrelease` (`hr`) or `kustomization` (`ks`) |
| `path_before` | yes | | Manifest tree in the base-branch checkout, relative to the repository root |
| `path_after` | yes | | Manifest tree in the PR checkout, relative to the repository root |
| `token` | yes | | Token for both checkouts and for mise's GitHub API calls |
| `krmignore` | no | `""` | gitignore-style file in the PR checkout, relative to the repository root, of paths to leave out of both trees |
| `log_level` | no | `info` | `debug`, `info`, `warn` or `error`, in any case |
| `other_params` | no | `""` | Additional `flate diff` flags, split on whitespace (e.g. `--api-versions monitoring.coreos.com/v1`) |
| `flate_version` | no | pinned release (Renovate-tracked) | flate release to install |

## Outputs

| Output | Description |
| --- | --- |
| `diff` | Unified diff produced by `flate diff` (empty string if no changes) |

## Example

See [`.github/workflows/test-flux-diff.yaml`](../../.github/workflows/test-flux-diff.yaml) and the
fixtures under [`ci/test-data/flux-diff/`](../../ci/test-data/flux-diff/) for a full working example,
including piping the `diff` output into
[`comment-on-pr`](../comment-on-pr/README.md).

```yaml
- uses: ppat/homelab-ops-actions/actions/flux-diff@<ref>
  id: flux-diff
  with:
    resource_type: all
    path_before: clusters/homelab
    path_after: clusters/homelab
    other_params: --api-versions monitoring.coreos.com/v1
    token: ${{ secrets.GITHUB_TOKEN }}
```
