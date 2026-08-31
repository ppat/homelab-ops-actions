# release-sweep

Evaluates open release-please pull requests and merges the safe, non-breaking subset with a
GitHub App token that is already authorized by the repository's branch rules. Every ineligible
pull request is reported as a skip; the action fails only if an attempted merge fails for a reason
other than a head change.

The caller owns the policy: the token and its branch-rules bypass, the schedule, the component
selection expression, and any permanently manual components. The action owns the repeatable
release-please interpretation and merge safeguards.

## Preconditions

- Check out the repository's base branch with full history and an `origin` remote before invoking
  the action. The staleness guard compares the release PR's parent with `origin/<base_branch>`.
- Pass a GitHub App installation token with `contents: write` and `pull-requests: write`. The App
  must be authorized by the target repository's rules; this action does not bypass policy itself.
- The selected repository must use `release-please-config.json` and
  `.release-please-manifest.json` in `working_directory`.

## Eligibility

A release pull request is merged only when all of these are true:

1. Its title matches `release_title_pattern`, which supplies component and semantic-version
   capture groups.
2. Its component matches `component_selector`, is present in the release-please configuration,
   and is not manually held.
3. It is a non-breaking patch/minor bump under the repository's release-please settings.
4. It has no hold label, no merge conflicts, all check runs are green/skipped/neutral, and no
   newer base-branch commit touches its release-please package path.
5. Its head SHA still matches at the moment of the REST squash merge.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `token` | yes | | GitHub App installation token used for API calls and the squash merge. |
| `repository` | no | current repository | Target in `owner/repo` form. |
| `working_directory` | no | `.` | Checkout directory holding release-please state. |
| `base_branch` | no | `main` | Base branch available as `origin/<base_branch>`. |
| `dry_run` | no | `true` | Only the literal `false` permits a merge in GitHub Actions. |
| `release_pr_label` | no | `pr-type:release` | Label identifying release pull requests. |
| `hold_label` | no | `automerge:off` | Label that holds an otherwise eligible pull request. |
| `release_title_pattern` | no | release-please default | Bash ERE whose capture groups are component, major, minor, patch. |
| `component_selector` | no | `.*` | Bash ERE selecting release-please component names. |
| `manual_components` | no | empty | Space-separated components that always require human review. |

`manual_components` applies to named release-please components. A single root package has an
empty component name; use the hold label to keep its release pull request manual.

`component_selector` replaces a hard-coded module class. For example, an applications sweep can
use `^apps-`, infrastructure can use `^infra-`, and a single-package repository can keep `.*`.

## Example

```yaml
- name: Sweep application releases
  uses: ppat/homelab-ops-actions/actions/release-sweep@<ref>
  with:
    token: ${{ steps.app-token.outputs.token }}
    dry_run: 'false'
    component_selector: '^apps-'
    manual_components: 'apps-sensitive'
```
