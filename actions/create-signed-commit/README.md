# create-signed-commit

Commit currently staged changes to a branch via the GitHub GraphQL `createCommitOnBranch` mutation,
so the resulting commit is signed by GitHub rather than pushed with `git commit`/`git push`.

## How it works

1. Reads `git diff --cached --name-status -z` in `working_directory` and converts additions,
   modifications, and renames into base64-encoded `fileChanges.additions`, and deletions/rename
   sources into `fileChanges.deletions`.
2. Resolves the branch's current `expectedHeadOid` via a GraphQL query so the mutation fails fast on
   a stale base instead of silently overwriting concurrent commits.
3. Calls `createCommitOnBranch` with the file changes, `commit_message`, and `expectedHeadOid`.
4. **Exits 1 on a successful commit.** The commit just created is expected to trigger the calling
   workflow's own re-run (e.g. via `push`), which will then find no staged changes and exit 0. This
   is intentional, not a bug — don't "fix" it by changing the exit code.

If there are no staged changes, the action exits 0 without attempting a commit.

## Inputs

| Input | Required | Description |
| --- | --- | --- |
| `commit_message` | yes | Commit headline |
| `branch` | yes | Branch name to commit onto |
| `repository` | yes | `owner/repo` |
| `token` | yes | GitHub token with permission to write commits to `branch` |
| `working_directory` | yes | Directory containing the git checkout with staged changes |

## Outputs

None.

## Example

```yaml
- run: |
    # ...make and `git add` changes...
  working-directory: current
- uses: ppat/homelab-ops-actions/actions/create-signed-commit@<ref>
  with:
    commit_message: "chore: update generated file"
    branch: ${{ github.head_ref || github.ref_name }}
    repository: ${{ github.repository }}
    token: ${{ secrets.GITHUB_TOKEN }}
    working_directory: current
```
