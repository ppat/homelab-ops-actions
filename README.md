# Homelab-Ops Actions

A collection of reusable [GitHub composite Actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action)
supporting CI/CD workflows across the `homelab-ops` and `ppat` GitHub organizations/repos.

## Actions

| Action | Description |
| --- | --- |
| [`actions/comment-on-pr`](actions/comment-on-pr/README.md) | Create or update a PR comment, keyed by a hidden `message_id` marker so repeat runs edit in place instead of piling up new comments. |
| [`actions/create-signed-commit`](actions/create-signed-commit/README.md) | Commit staged changes to a branch via the GitHub GraphQL API (`createCommitOnBranch`) so the commit is signed by GitHub, instead of `git commit` + `git push`. |
| [`actions/flux-diff`](actions/flux-diff/README.md) | Run [`flux-local diff`](https://github.com/allenporter/flux-local) between a before/after version of a Flux-managed Kubernetes manifest tree and expose the resulting patch. |
| [`actions/setup-repository-tools`](actions/setup-repository-tools/README.md) | Checkout a repository and install its toolchain via [`mise`](https://mise.jdx.dev/), with caching. Used as the common setup step by reusable workflows in [`ppat/github-workflows`](https://github.com/ppat/github-workflows). |

Each action's README documents its inputs, outputs, and a usage example in full.

See [DESIGN.md](DESIGN.md) for how these actions are structured, tested, and released, and
[CLAUDE.md](CLAUDE.md) for repo-specific guidance when working with Claude Code.
