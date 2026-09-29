# Open Remediation PR

Composite GitHub Action that commits the current working-tree changes as
`github-actions[bot]` onto a new branch, pushes it, and opens a PR against a
given base branch. Built as the second half of an automated remediation flow
(after [`howdycom/claude-remediation-prepare`](https://github.com/howdycom/claude-remediation-prepare)
validates the trigger and gathers context), but usable anywhere a job needs
to turn worktree changes into a reviewable PR.

Licensed under the [MIT License](LICENSE).

## Usage

```yaml
steps:
  # ... produce changes in the working tree (apply a patch, run a fixer) ...
  - uses: howdycom/open-remediation-pr@v1
    id: pr
    with:
      branch: claude/remediate-pr-123-${{ github.run_id }}-${{ github.run_attempt }}
      base: feature-branch
      title: "fix: automated remediation for #123"
      body_file: remediation-body.md
      github_token: ${{ secrets.REMEDIATION_PAT }}
```

## Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `branch` | yes | — | Branch name to create and push. If it already exists, it is switched to instead. |
| `base` | yes | — | Base branch for the PR (typically the source PR's head branch). |
| `title` | yes | — | PR title. |
| `body` | no | `''` | PR body. Ignored when `body_file` is set. |
| `body_file` | no | `''` | Path to a file containing the PR body. Takes precedence over `body`. |
| `commit_message` | no | `fix: automated remediation` | Commit message for the bot commit. |
| `github_token` | yes | — | Token used to push the branch and open the PR. Use a PAT here if you want the resulting PR to trigger other workflows — the default `GITHUB_TOKEN` does not trigger them. |
| `draft` | no | `'true'` | Open the PR as a draft. |

## Outputs

| Output | Description |
|---|---|
| `url` | URL of the opened PR. |

## Notes

- The caller must confirm there are changes to commit first — an empty
  `git commit` fails the step.
- The commit runs `git add -A` with no exclusions. Keep scratch/context
  files outside the git worktree (e.g. under `RUNNER_TEMP`, which is
  `claude-remediation-prepare`'s default `context_dir`), or add them to
  `.git/info/exclude` before calling this action.
- Pin to a tag (`@v1`), never to `main`.

## Security notes for remediation consumers

This action only handles the git/PR mechanics — the agent invocation itself
(e.g. `anthropics/claude-code-action`) is wired up in each consuming repo's
own workflow, including its `--allowedTools` Bash allowlist. Scope that
allowlist to **non-executing commands only** (formatters and linters: `black`,
`isort`, `prettier`, `eslint --fix`, `flake8`, `mypy`, plus read-only
`git diff`/`git status`) — never test or build execution (`pytest`,
`npm run test`, `npm run build`, `npm ci`, `uv run`, `make test`, etc.).

Why: the agent's API key is present in the environment inherited by whatever
the Bash tool executes. A test/build command that reads its own environment
(intentionally or via a compromised dependency/test file) can exfiltrate the
key. Formatters/linters never execute the target code, so they don't have
this exposure — verification that a fix actually works should happen via the
normal CI that runs on the resulting draft PR, not inside the remediation job
itself.

## Versioning

Changes are tagged with semver (`v1`, `v1.1`, …). The major tag (`v1`) moves
to the latest compatible release; breaking changes bump the major version.
Don't reference `main` from a consumer workflow.

## Contributing

Changes go through a PR, not direct pushes to `main`. This action pushes
branches and opens PRs in consuming repos with a caller-supplied token, so
review matters here more than usual.
