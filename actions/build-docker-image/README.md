# build-docker-image

Build a Docker image from the caller's checkout with `docker/build-push-action`, optionally publish it to Docker
Hub, a private registry reached over Tailscale, and/or GHCR, and optionally sign every published repository by
digest with keyless [cosign](https://github.com/sigstore/cosign).

It is a composite action, so it runs as one step of the caller's job. Steps before it (producing files the build
copies in) and after it (inspecting, loading or testing the image) share the same workspace and Docker daemon. The
reusable workflow `ppat/github-workflows/.github/workflows/build-docker-image.yaml` is a thin wrapper around this
action for callers that want a whole job.

## How it works

1. **Validate targets.** A target is requested when its repository input is non-empty and well-formed (no
   leading or trailing `/`, no empty path segment). A requested Docker Hub or private-registry target without its
   credentials fails, except on a fork pull request, where it is disabled with a warning. Partial credential sets
   fail. A run with no target is a green build-only run.
2. **Resolve the version from the checkout.** HEAD on a branch gives `branch-<name>` tags, HEAD at a tag gives
   the tag plus semver tags and `latest`, and every build gets `git-<short sha>`. Publishing from a HEAD that is
   neither (for example a pull request's merge commit) fails, because the tags would carry no version. A build-only
   run accepts any HEAD.
3. **Registry build cache.** Only with `private_registry_build_cache` and a private-registry target, imported from
   and exported to `:branch-<name>` and `:cache-latest`. A commit message containing `[ci: bust-cache]` skips the
   import.
4. **Build**, pushing when any target is enabled, with SBOM and provenance attestations when pushing.
5. **Sign** (when `sign` is `true`), running `cosign sign --yes <repository>@<digest>` for each published
   repository.

State passes between the action's steps as step outputs, never through `$GITHUB_ENV`, so nothing leaks into the
caller's later steps.

## Requirements on the calling job

| Need | When |
| --- | --- |
| A checkout of the repository in the workspace, with `.git` | Always. The action does not check out, and reads the version from HEAD |
| `permissions: contents: read` | Private repositories, for the checkout |
| `permissions: packages: write` | Publishing to GHCR. The push uses the job's own `github.token` |
| `permissions: id-token: write` | `sign: true`. The action fails early with a clear message when it is missing |
| `timeout-minutes` and `runs-on` | Set on the calling job. A composite action cannot set either |
| Secrets passed as inputs | Composite actions cannot read the `secrets` context |

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `image_context_path` | yes | | Build context, relative to the workspace. Use `.` for the repository root |
| `dockerfile` | no | `""` | Dockerfile path relative to the workspace. Empty means `<image_context_path>/Dockerfile` |
| `build_args` | no | `""` | Newline-separated `NAME=value` build arguments |
| `target` | no | `""` | Build stage to build. Empty builds the last stage |
| `load` | no | `false` | Load the image into the runner's Docker daemon. Single platform only |
| `platforms` | no | `linux/amd64` | Comma-separated target platforms |
| `label_title` | no | `${{ github.event.repository.name }}` | OCI title label and annotation |
| `label_description` | no | `${{ github.event.repository.description }}` | OCI description label and annotation, and the Docker Hub short description (truncated to 100 characters) |
| `ghcr_repository` | no | `""` | GHCR repository as `owner/name`, lowercased before use |
| `dockerhub_repository` | no | `""` | Docker Hub repository |
| `dockerhub_readme_path` | no | `""` | README pushed as the Docker Hub description. Empty means `<image_context_path>/README.md` |
| `private_registry_repository` | no | `""` | Repository path in the private registry |
| `private_registry_build_cache` | no | `""` | Repository path in the private registry for the build cache |
| `sign` | no | `false` | Sign each published repository by digest with keyless cosign |
| `cosign_version` | no | pinned version (Renovate-tracked) | cosign release installed when signing |
| `build_secrets` | no | `""` | Build secrets, passed to `docker/build-push-action`'s `secrets` |
| `dockerhub_username`, `dockerhub_token` | no | `""` | Docker Hub credentials, both or neither |
| `private_registry`, `private_registry_username`, `private_registry_token` | no | `""` | Private registry host and credentials, all or none |
| `tailscale_oauth_client_id`, `tailscale_oauth_secret` | no | `""` | Tailscale OAuth pair, both or neither. Connects the runner to the tailnet for the private registry |

## Outputs

| Output | Description |
| --- | --- |
| `digest` | Image digest reported by `docker/build-push-action`, the pushed digest when publishing. Empty on a build-only run without `load` |
| `image_tag` | Primary version tag from `docker/metadata-action` |
| `image_refs` | Newline-separated pushed image references. Empty on a build-only run |
| `image_id` | Image ID from `docker/build-push-action`, the handle for an image built with `load` |

## Keyless signing

The signing certificate is issued against the job's GitHub OIDC token and records the repository and the workflow
file that ran, in the public [Rekor](https://docs.sigstore.dev/logging/overview/) transparency log. That holds for
private repositories too. Verify with:

```bash
cosign verify ghcr.io/<owner>/<name>@<digest> \
  --certificate-identity-regexp '^https://github\.com/<owner>/<repo>/\.github/workflows/<workflow file>@' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

## Example

A repository-root build context with the Dockerfile in a subdirectory, pushed to GHCR and signed:

```yaml
jobs:
  image:
    runs-on: ubuntu-24.04
    timeout-minutes: 30
    permissions:
      contents: read
      packages: write
      id-token: write
    steps:
    - uses: actions/checkout@<sha> # vX.Y.Z
      with:
        persist-credentials: false
    - id: build
      uses: ppat/homelab-ops-actions/actions/build-docker-image@<ref>
      with:
        image_context_path: .
        dockerfile: app/Dockerfile
        build_args: |
          VERSION=${{ github.ref_name }}
        target: runtime
        ghcr_repository: owner/app
        sign: "true"
```

See [`.github/workflows/test-build-docker-image.yaml`](../../.github/workflows/test-build-docker-image.yaml) and
[`ci/test-data/build-docker-image/`](../../ci/test-data/build-docker-image/) for a build-only run with `load`, the
refusal to publish from a detached HEAD, and a GHCR push verified with `cosign verify`.
