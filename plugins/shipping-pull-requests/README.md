# shipping-pull-requests

Say "fai la PR", "do the PR", "review the PR" or "ship it". The skill opens the pull request of
the current branch on GitHub, reviews it with a fresh subagent, writes every finding as a review
thread on the PR, fixes the code in the working tree and hands back; a clean re-run with green
checks merges. It never commits.

## Trigger phrases

Italian and English both work; the skill also picks up any request to open, review or merge the
current branch's pull request.

- Italian: "fai la PR", "fai la MR", "apri la PR", "controlla la PR", "mergia la PR".
- English: "do the PR", "do the MR", "open the PR", "review the PR", "check the PR",
  "merge the PR", "ship it".

Or call it explicitly: `/shipping-pull-requests:shipping-pull-requests`.

## Requirements

| Requirement | What the skill uses it for | Checked by the skill |
|---|---|---|
| [GitHub CLI](https://cli.github.com) `gh`, logged in to `github.com` (default scopes `repo`, `read:org`) | Everything on GitHub: `gh pr create/list/view/checks/merge`, review threads through the REST API (`gh api .../pulls/<n>/comments`), resolving and re-opening threads through GraphQL, failing-check logs (`gh run view --log-failed`) | Yes, first step of every run: missing or logged out → it prints the setup steps and stops |
| Write access to the repository, with `git` able to push to `origin` | Pushing the developer's unpushed commits, opening and merging the PR, deleting the merged branch; the merge also has to satisfy the branch protection (no `--admin` bypass) | Indirectly: a rejected push or a refused merge is a hard stop |
| [superpowers](https://github.com/obra/superpowers) plugin | `superpowers:requesting-code-review` builds the prompt of the reviewer subagent, `superpowers:receiving-code-review` makes the skill verify each finding on the code before acting on it | Yes, first step of every run: either sub-skill missing → it prints the install commands below and stops |
| A `## Shipping pull requests` section in the project's root `CLAUDE.md` | The project facts: target branch, merge method, verification commands, check paths and, above all, the **review checklist** (the rule files the review applies) | Read on every run; missing entries fall back to defaults and the report says which ones did |
| The toolchains of the project's verification commands (compilers, Docker, …) | Re-running the touched module's CI commands locally after a fix | No: a failing command counts as a red verification, so the fix is reverted |

Install the first and the third once per developer:

```bash
# GitHub CLI (macOS; other platforms: https://github.com/cli/cli#installation)
brew install gh
gh auth login --hostname github.com --git-protocol ssh --web
gh auth status --hostname github.com

# superpowers
claude plugin marketplace add obra/superpowers-marketplace
claude plugin install superpowers@superpowers-marketplace
```

## Install

```bash
claude plugin marketplace add uschti/claude-skills
claude plugin install shipping-pull-requests@uschti
```

More options (per-project install, updates) in the [repository README](../../README.md#install).
A new session lists the skill as `/shipping-pull-requests:shipping-pull-requests`.

## What it builds on

This skill is an orchestration of existing pieces, not a review engine of its own:

- **superpowers** — `superpowers:requesting-code-review` shapes the prompt of the fresh reviewer
  subagent, `superpowers:receiving-code-review` governs how its findings are verified on the code
  before anything is changed. Both are required sub-skills.
- **A GitLab merge-request workflow** — the procedure (thread first, fix in the working tree, the
  developer commits, re-run to merge pinned to the reviewed commit) is adapted from a skill built
  on `glab` for GitLab merge requests; this version maps it onto GitHub pull requests, review
  threads and checks through `gh`.

## What a project must provide

The skill reads a `## Shipping pull requests` section in the project's root `CLAUDE.md` with these
entries: target branch, merge method, per-module verification commands, check paths, **review
checklist**, ignored dirt, what a merge deploys. The review checklist is the one a project cannot
skip: it lists the files whose rules the reviewer judges the diff against and the PR threads cite;
without it every finding is only a suggestion and nothing gets fixed. The other entries fall back
to the skill's defaults, and the run's report says which ones did. Example:

```markdown
## Shipping pull requests

- **target**: `main`.
- **merge method**: `merge`.
- **modules**: `firmware/` — `esphome compile firmware/app.yaml`; `tests/` — `cmake -S tests -B
  tests/build && cmake --build tests/build && ctest --test-dir tests/build`.
- **check paths**: `firmware/**`, `tests/**`, `.github/workflows/**`.
- **review checklist**: `CLAUDE.md` + `docs/SPEC.md`.
- **ignored dirt**: none.
- **deploys**: nothing.
```

## License

This project is licensed under the **MIT License** (see [LICENSE](../../LICENSE)).

## Development & Support

For bugs or feature requests, please open an issue on the
[GitHub repository](https://github.com/uschti/claude-skills/issues).

Contributions are welcome!

---

Made with ❤️ by Andrea Pellegrini ([@uschti](https://github.com/uschti))
