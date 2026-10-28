---
name: shipping-pull-requests
description: Use when the developer says, in Italian or English, "fai la PR", "fai la MR", "apri la PR", "controlla la PR", "mergia la PR", "do the PR", "do the MR", "open the PR", "review the PR", "check the PR", "merge the PR", "ship it", or asks to open, review or merge the current feature branch into the target branch on a GitHub repository (gh CLI). Covers both the first run (open and review) and the re-run after the developer committed the fixes (verify and merge). The project's root CLAUDE.md must carry a "Shipping pull requests" section with at least the review checklist (the rule files the reviewer judges the diff against); without it the review has no rule book and the run falls back to defaults.
---

# Shipping pull requests

## Overview

The developer finishes and commits; this skill replaces asking a human for a review. Each run
reads the branch state and does the next step: open the PR if missing, review it, write every
finding as a review thread on the PR **first**, then fix the code in the working tree and hand
back. The developer reads the threads and the fixes, commits, pushes and runs the skill again. A
run that finds nothing to fix waits for the PR checks and merges.

The skill never commits: the developer's commit is the review of the fixes. It may `git push`
the developer's commits and `gh pr create` / `gh pr merge` (root `CLAUDE.md` exception).

Built on: `superpowers:requesting-code-review` and `superpowers:receiving-code-review` (required
sub-skills, step 3); the procedure is adapted from a GitLab merge-request workflow built on
`glab`, mapped here onto GitHub pull requests, review threads and checks.

## Project profile (read first)

The procedure is the same in every project; the facts differ. The project's root `CLAUDE.md`
carries them in a section titled `## Shipping pull requests`. The **review checklist is the one
entry a project cannot do without**: it names the files whose rules the reviewer applies and the
threads cite (a module's `CLAUDE.md`, a spec, a component catalog); a project without it gets a
review with no rule book, so every finding becomes a suggestion and nothing is fixed. The entries:

- **target**: the branch PRs go to (`develop` where there is one, otherwise `main`);
- **merge method**: `merge`, `squash` or `rebase` (the `gh pr merge` flag);
- **modules**: for each top-level module, the local verification that mirrors its CI job (the
  commands, their prerequisites such as a toolchain or Docker, and what CI adds that no local
  run reproduces, e.g. a coverage floor);
- **check paths**: the paths whose change triggers checks on the PR (the `paths` filters of the
  GitHub Actions workflows); a diff outside them, or a repository without workflows, has no
  checks and the gate is skipped;
- **review checklist**: the file(s) the reviewer uses as the rule book per module;
- **ignored dirt**: tracked files that may be modified locally without belonging to a feature;
- **deploys**: what merging into the target deploys, if anything, for the final report.

A project without the section gets the defaults: target `develop` if `origin/develop` exists else
the repository's default branch (`gh repo view --json defaultBranchRef --jq .defaultBranchRef.name`),
merge method `merge`, verification = the commands the module's `CLAUDE.md` names under its
build/verify instructions, every change expected to have checks if `.github/workflows/` exists,
no ignored dirt. Say in the report which facts came from the defaults.

## Prerequisites (once per developer)

1. Install the GitHub CLI: `brew install gh` (macOS; other platforms: https://github.com/cli/cli#installation).
2. Log in: `gh auth login --hostname github.com --git-protocol ssh --web` (browser flow; the
   token is stored in the OS keyring). The default scopes (`repo`, `read:org`) are enough.
3. Check: `gh auth status --hostname github.com` prints "Logged in to github.com account <you>",
   and `gh pr list` (from the repo) lists the open pull requests.
4. Install the superpowers plugin (the review sub-skills): `claude plugin marketplace add
   obra/superpowers-marketplace && claude plugin install superpowers@superpowers-marketplace`.

## Hard stops (report, never work around)

- Tracked files modified or staged under a module or any path the PR touches
  (`git status --porcelain --untracked-files=no -- <modules> $(git diff --name-only
  origin/<target>...HEAD)` not empty): the developer commits their work first, the skill's own
  fixes from the previous run included; the skill never commits for them. A dirty file in the
  profile's "ignored dirt" is not the feature's and is ignored.
- `gh` missing (`command -v gh` empty) or not logged in (`gh auth status --hostname github.com`
  fails): print the Prerequisites section above, all four steps verbatim, and stop. Nothing
  else runs before the developer has done them.
- `superpowers:requesting-code-review` or `superpowers:receiving-code-review` missing from the
  skills available in this session: print Prerequisites step 4 verbatim, then "then run
  `/reload-plugins` (or start a new session) and ask again; details:
  https://github.com/uschti/claude-skills/blob/main/plugins/shipping-pull-requests/README.md#requirements",
  and stop. Never review inline instead: the review needs both sub-skills.
- Current branch is the target or `main`.
- `git fetch` still fails after 3 retries.
- `git merge-tree --write-tree origin/<target> HEAD` shows conflicts.
- Push rejected (never `--force`: branches may be shared).
- A human left an unresolved review thread on the PR, or a human review is in state
  `CHANGES_REQUESTED`. Threads come from GraphQL, paged (`--paginate` is mandatory):
  ```bash
  gh api graphql --paginate -F owner='{owner}' -F repo='{repo}' -F n=<number> -f query='
    query($owner:String!,$repo:String!,$n:Int!,$endCursor:String){
      repository(owner:$owner,name:$repo){pullRequest(number:$n){
        reviewThreads(first:100,after:$endCursor){
          pageInfo{hasNextPage endCursor}
          nodes{id isResolved path line
            comments(first:1){nodes{databaseId body author{login}}}}}}}}'
  ```
  A human thread is an unresolved one whose first comment does not start with
  `[shipping-pull-requests]` (the marker every thread of this skill carries; the author is no
  criterion, the skill posts with the developer's own token). Reviews:
  `gh pr view <number> --json latestReviews --jq '.latestReviews[] | select(.state=="CHANGES_REQUESTED")'`.
- Failing or cancelled checks. Never skip a check, never `--admin`, never lower a coverage gate.

## Steps

1. **Preflight.** Read the project profile. `command -v gh && gh auth status --hostname
   github.com` first (a failure is the Prerequisites hard stop), then the superpowers
   sub-skills in the session's skill list (missing is the superpowers hard stop);
   `git status --short --branch`;
   branch not the target/`main`; `git fetch origin <target>` (retry 3x; add `<branch>` only if
   `git ls-remote --heads origin <branch>` finds it); then, against the fresh `origin/<target>`,
   the dirty-tree hard stop (no change in the paths it names). Unpushed commits
   (`git log origin/<branch>..HEAD`, or no remote branch) → `git push -u origin <branch>`.
   Then the conflict preview (`git merge-tree`, see Hard stops).
2. **PR.** `gh pr list --head <branch> --state open --json number,url`. None → `gh pr create
   --base <target> --head <branch> --title "<conventional title>" --body "<What / Why / How to
   test>"`. Keep the number, the URL and the head SHA (`gh pr view <number> --json headRefOid
   --jq .headRefOid`, for line comments).
3. **Review with fresh eyes.** Dispatch a subagent with the PR URL, `git diff
   origin/<target>...HEAD` and the touched module's review checklist from the profile.
   REQUIRED SUB-SKILLS: superpowers:requesting-code-review for the prompt,
   superpowers:receiving-code-review for the answer (verify each finding on the code before
   acting on it). On a re-run the whole diff is reviewed again: the earlier fixes are part of
   it. Then, before any new thread is posted, resolve the threads of previous runs (from the
   paginated read of the Hard stops, first comment with the marker):
   `gh api graphql -f query='mutation($id:ID!){resolveReviewThread(input:{threadId:$id}){thread{isResolved}}}' -f id=<thread id>`,
   the suggestions too: the developer answered them by committing or not, the review re-read
   the whole diff, and anything still wrong comes back below.
4. **Findings → comment, then fix, then hand back.** For each finding, in this order:
   1. A review thread on the PR line, in English (like commits and files), body = the marker
      `[shipping-pull-requests]`, what is wrong, the rule (`<file>` "…") and the fix:
      `gh api 'repos/{owner}/{repo}/pulls/<number>/comments' -f body='…' -f commit_id=<head sha>
      -f path=<repo path> -F line=<n> -f side=RIGHT` (`side=LEFT` with the old line number for a
      removed line). A `422` means the line is not in the diff: post the same body as a
      file-level thread (`-f subject_type=file`, no `line`/`side`) with `path:line` in the text;
      a file not in the diff at all gets `gh pr comment <number> --body '…'` (not a thread: say
      so in the report). A finding already threaded on the PR (a suggestion, or a violation that
      came back after a reverted or uncommitted fix) gets a reply in that thread (`gh api
      'repos/{owner}/{repo}/pulls/<number>/comments/<first comment databaseId>/replies' -f
      body='…'`), not a new one; for a violation the thread is re-opened too
      (`unresolveReviewThread`, same shape as the resolve mutation above), so the developer
      sees it open on GitHub.
   2. The finding breaks a written rule (a file of the review checklist) → fix it in the
      working tree, however big: when the fix reaches a file outside the feature, the thread
      says so, so the developer knows why the diff grew. The finding is a reviewer's suggestion
      with no rule behind it ("nicer if…") → the thread only, marked `suggestion`; the
      developer decides.
   Then, only when a fix was made (on an all-suggestions run CI is the check), run the touched
   module's verification from the profile, the same commands as CI, on the fixed tree (a
   formatter first where the module has one). What CI adds and no local run reproduces means
   green here can still be red at the gate. Red after one fix attempt → revert that fix, say so
   in its thread: an unfixed rule violation is handed back, never merged.
   **A rule violation was found (fixed or reverted) → stop here, no commit.** Report: PR URL,
   one line per thread (file, what, fixed, reverted or suggestion), `git status --short` with
   the new files to `git add` named (the dirty-tree stop does not see untracked files), and
   "review the threads and the diff, commit, push, run the skill again". Findings that are all
   suggestions → they are posted and the run goes on to step 5: a declined suggestion must
   not block the merge.
5. **Nothing to hand back → checks gate.** The suggestions this run posted are listed in the
   report as "not applied". Check for human threads and reviews (Hard stops). Checks exist only
   when the repository has workflows triggered on the PR and the diff touches the profile's
   check paths: when `git diff --name-only origin/<target>...HEAD` touches none of them, or
   `gh pr checks <number>` answers "no checks reported", there is nothing to wait for (if branch
   protection requires status checks, GitHub then refuses the merge: say so and stop). Otherwise
   first make sure the checks belong to the reviewed commit (`gh pr view <number> --json
   headRefOid --jq .headRefOid` equals `git rev-parse HEAD`), then poll every 30 s with the
   Monitor tool (foreground `sleep` is blocked) `gh pr checks <number> --json
   name,bucket,link` until no `bucket` is `pending`; no check appeared after 5 minutes → stop and
   report (workflow not triggered, runner queue). Any `fail` or `cancel` → `gh run view <run id>
   --log-failed` for each failing run, stop, report job names and links.
6. **Merge, pinned to the reviewed commit.** `gh pr merge <number> --<merge method>
   --delete-branch --match-head-commit "$(git rev-parse HEAD)"`. Never `--auto` (it arms
   auto-merge and returns before the merge: the gate is explicit) and never `--admin` (it
   bypasses branch protection). When the branch protection requires conversation resolution,
   an open suggestion thread of this run blocks the merge: right before merging, reply in each
   still-open skill thread that it is resolved to unblock the merge and is left to the
   developer, then resolve it; a human's thread is never resolved this way (Hard stops).
   Confirm `gh pr view <number> --json state --jq .state` is `MERGED`.
7. **Clean up and report.** `gh pr merge --delete-branch` already deleted the remote branch and,
   when it could, the local one. `git checkout <target> && git pull --ff-only && git fetch
   --prune`, then `git branch -d <branch>` if it still exists. Report PR URL, merge commit SHA
   (`gh pr view <number> --json mergeCommit --jq .mergeCommit.oid`), checks links, and what the
   merge deploys according to the profile (often nothing: say so).

## Common mistakes

| Mistake | Fix |
|---|---|
| Committing the fixes "to save the developer a step" | The developer's commit is their review of the fixes; hand back with the diff in the tree |
| Fixing first, commenting later (or not at all) | The thread is the record the developer reads on GitHub; it goes in before the edit |
| Reading checks of an older commit | Compare the PR's `headRefOid` with `git rev-parse HEAD` before trusting `gh pr checks` |
| `gh pr merge --auto` or `--admin` | `--auto` walks away before the merge, `--admin` bypasses protection; merge explicitly, pinned with `--match-head-commit` |
| Posting findings with `gh pr comment` | Issue comments are not resolvable threads; use the review comments API, `gh pr comment` only for a file outside the diff |
| Reviewing your own diff inline | A fresh subagent; the author misses what it just wrote |
| Verifying one module when several changed | Each module has its own CI job: run every touched module's verification locally |
| Merging over a human's open thread or a "changes requested" review | Only the skill's own threads get resolved by the skill |
