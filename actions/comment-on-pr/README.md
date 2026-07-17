# comment-on-pr

Create a new comment on a pull request, or update an existing one, keyed by a caller-supplied
`message_id`.

## How it works

The action embeds a hidden `<!-- pr-comment-id: <message_id> -->` marker in the comment body, then
searches existing PR comments for that marker on each run. If found, it `PATCH`es that comment in
place; otherwise it creates a new one. This lets a workflow re-post updated content (e.g. a diff, a
status) to the same comment across multiple runs instead of accumulating duplicates. Uses `gh api`
and `jq` directly — no GitHub App/token exchange beyond the `token` input.

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `message` | yes | | Comment body |
| `message_id` | yes | | Unique identifier used to find and update an existing comment |
| `repository` | no | `${{ github.repository }}` | `owner/repo` |
| `pr_number` | no | `${{ github.event.pull_request.number }}` | Pull request number |
| `token` | yes | | GitHub token with write access to issues/PRs |

## Outputs

| Output | Description |
| --- | --- |
| `comment_id` | ID of the created/updated comment |
| `comment_url` | URL of the created/updated comment |
| `action_taken` | `created` or `updated` |

## Example

```yaml
- uses: ppat/homelab-ops-actions/actions/comment-on-pr@<ref>
  with:
    message: |
      `````diff
      ${{ steps.some-step.outputs.diff }}
      `````
    message_id: "${{ github.event.pull_request.number }}/some-diff"
    token: ${{ secrets.GITHUB_TOKEN }}
```

See usage in [`.github/workflows/test-flux-diff.yaml`](../../.github/workflows/test-flux-diff.yaml).
