# flux-diff

Run [`flux-local diff`](https://github.com/allenporter/flux-local) between a "before" and "after"
version of a Flux-managed Kubernetes manifest tree (e.g. two checkouts, or a PR base vs. head) and
expose the resulting unified diff as an output.

## How it works

Runs the `ghcr.io/allenporter/flux-local` Docker image's `diff <resource_type>` command against
`path_before`/`path_after`, writing the result to `diff.patch`, then reads that file into the
`diff` output. `strip_attrs`/`skip_params` default to stripping noisy, non-semantic fields (Helm
chart checksums, versions) so diffs reflect real spec changes only.

## Choosing the pinned image

The pinned `flux-local` image bundles its own `helm` binary, and that binary is what renders every
chart the diff reports on. Its major must track the one Flux's `helm-controller` links against:
when the two drift apart, the diff still renders and still looks healthy, but it stops describing
what Flux would deploy. The failure is silent and asymmetric — a chart that renders identically
under both majors hides the drift, so the modules that expose it are found by rendering the whole
estate under both images and comparing, not by reading a job's exit code.

Two rendering differences observed across that boundary, as a sense of what changes: a
quoted-numeric value in a chart's values (`"08"`) losing its quoting and arriving as an integer,
and templates gated on API availability rendering under one major and not the other.

Because the diff renders the before and the after tree with the *same* image, a systematic
rendering difference cancels and does not surface as review noise — the cost of being on the wrong
major is fidelity, not churn.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `resource_type` | yes | | Flux resource type to diff, e.g. `kustomization`, `helmrelease` |
| `path_before` | yes | | Path to the "before" manifest tree |
| `path_after` | yes | | Path to the "after" manifest tree |
| `sources` | yes | | `--sources` value passed to `flux-local diff`, e.g. `name=path` |
| `log_level` | no | `INFO` | `--log-level` for `flux-local` |
| `skip_params` | no | `--skip-crds --skip-secrets` | Extra `--skip-*` flags |
| `strip_attrs` | no | `helm.sh/chart,checksum/config,app.kubernetes.io/version,chart` | Fields to strip before diffing |
| `other_params` | no | `""` | Any additional `flux-local diff` flags (e.g. `--api-versions`) |

## Outputs

| Output | Description |
| --- | --- |
| `diff` | Unified diff produced by `flux-local diff` (empty string if no changes) |

## Example

See [`.github/workflows/test-flux-diff.yaml`](../../.github/workflows/test-flux-diff.yaml) and the
fixtures under [`ci/test-data/flux-diff/`](../../ci/test-data/flux-diff/) for a full working example,
including piping the `diff` output into
[`comment-on-pr`](../comment-on-pr/README.md).

```yaml
- uses: ppat/homelab-ops-actions/actions/flux-diff@<ref>
  id: flux-diff
  with:
    resource_type: kustomization
    path_before: before
    path_after: after
    sources: "my-source=./"
```
