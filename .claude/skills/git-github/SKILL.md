---
name: git-github
description: >
  Use this skill for ANY git or GitHub operation on this project's repositories: creating
  branches, staging and committing, pushing, opening or updating pull requests, reviewing and
  merging PRs, and managing issues and releases. Trigger it whenever the user says "commit
  this", "push my changes", "open a PR", "create a branch", "merge the PR", "open an issue",
  "cut a release", "sync with main", or any version-control / GitHub request — even casually
  phrased. Authentication is HTTPS with a Personal Access Token read from the GITHUB_TOKEN
  environment variable; local operations use git, while PR/issue/release operations use the
  GitHub REST API via curl. This skill encodes the branching strategy, commit format, and
  safety rules so the agent never pushes to main directly and never leaks the token.
---

# Git & GitHub Workflow

Authentication: **HTTPS + Personal Access Token**, taken from the `GITHUB_TOKEN` environment
variable. Local history operations use `git`; pull-request, issue, and release operations use
the **GitHub REST API** via `curl` (there is no `gh` CLI in this setup).

## Hard safety rules (never violate)
- **Never push directly to `main`** (or `master`/`develop`). All changes go through a feature
  branch → pull request.
- **The agent may merge its OWN pull requests**, but only after: the PR is mergeable, required
  checks/CI are green, and there are no requested changes. Verify before merging.
- **Never force-push** to a shared branch (`main`, `develop`). Force-push is allowed only on the
  agent's own short-lived feature branch when explicitly needed (`--force-with-lease`, never
  bare `--force`).
- **Never print, log, echo, or commit the token.** Don't put it in a remote URL that gets saved
  to `.git/config`. Don't paste it into commit messages, PR bodies, or files. If a secret is
  found in a diff, stop and warn the user.
- **Build & test before pushing**: run `dotnet build && dotnet test` and only push if green
  (unless the user explicitly says to push a WIP branch).

## Step 0 — One-time auth setup (env-based, no token persisted)

Use a credential helper that reads the token from the environment at request time, so the
token is never written to disk or shell history:

```bash
git config --global credential.helper \
  '!f() { echo "username=x-access-token"; echo "password=${GITHUB_TOKEN}"; }; f'
git config --global user.name  "<configured name>"
git config --global user.email "<configured email>"
```
Confirm the token is present without printing it:
```bash
[ -n "$GITHUB_TOKEN" ] && echo "token present" || echo "GITHUB_TOKEN is missing — ask the user to set it"
```
For API calls, set the owner/repo once per session (derive from `git remote get-url origin`):
```bash
OWNER=Mr-Aristo
REPO=ECommerce_Microservices
API="https://api.github.com/repos/$OWNER/$REPO"
AUTH=(-H "Authorization: Bearer $GITHUB_TOKEN" -H "Accept: application/vnd.github+json" -H "X-GitHub-Api-Version: 2022-11-28")
```

## Step 1 — Branch (always off the latest base)

```bash
git checkout main && git pull --ff-only
git checkout -b <type>/<scope>-<short-desc>
```
**Branch naming** (mirror the service scopes of this solution): `feat/`, `fix/`, `refactor/`,
`chore/`, `test/`, `docs/` + a scope like the affected service:
`feat/basket-checkout-retry`, `fix/order-migration`, `refactor/catalog-pagination`.

## Step 2 — Commit (Conventional Commits)

Stage intentionally (avoid `git add .` blindly — never stage secrets, `bin/`, `obj/`,
`appsettings.*.json` with secrets):
```bash
git add <specific files>
git commit -m "feat(basket): retry checkout outbox dispatch on broker failure"
```
Format: `type(scope): summary`. Types: `feat`, `fix`, `refactor`, `chore`, `test`, `docs`,
`perf`. Scopes follow services/areas: `catalog`, `basket`, `order`, `discount`, `gateway`,
`messaging`, `buildingblocks`. Keep the summary imperative and under ~72 chars; add a body for
the "why" when non-trivial.

## Step 3 — Push the feature branch

```bash
dotnet build && dotnet test            # gate: only push if green
git push -u origin <branch>
```

## Step 4 — Open a Pull Request (REST API)

```bash
curl -sS -X POST "${AUTH[@]}" "$API/pulls" -d @- <<'JSON'
{
  "title": "feat(basket): retry checkout outbox dispatch on broker failure",
  "head": "feat/basket-checkout-retry",
  "base": "main",
  "body": "## What\n- ...\n\n## Why\n- ...\n\n## Testing\n- dotnet test green\n\nCloses #<issue>"
}
JSON
```
Write a clear PR body: what changed, why, how it was tested, and any contract/migration/route
change that reviewers must know about (per the project's conventions). Link issues with
`Closes #N`.

To **update** an existing PR's title/body:
```bash
curl -sS -X PATCH "${AUTH[@]}" "$API/pulls/$PR_NUMBER" -d '{"body":"updated description"}'
```
(To update the code, just push more commits to the same branch.)

## Step 5 — Review before merge

Inspect the PR and its checks:
```bash
curl -sS "${AUTH[@]}" "$API/pulls/$PR_NUMBER"                       # .mergeable, .mergeable_state
HEAD_SHA=$(curl -sS "${AUTH[@]}" "$API/pulls/$PR_NUMBER" | jq -r .head.sha)
curl -sS "${AUTH[@]}" "$API/commits/$HEAD_SHA/check-runs"           # CI check runs
curl -sS "${AUTH[@]}" "$API/commits/$HEAD_SHA/status"              # combined status
```
Read the diff and self-review against the project's checklists (handler vs endpoint logic,
event contracts, migrations, gateway routes). Add review comments if useful:
```bash
curl -sS -X POST "${AUTH[@]}" "$API/issues/$PR_NUMBER/comments" -d '{"body":"Note: ..."}'
```

## Step 6 — Merge (own PRs only, when green)

Pre-merge gate — ALL must hold: `mergeable == true`, `mergeable_state` is clean, checks
green, no unresolved requested-changes reviews. Then:
```bash
curl -sS -X PUT "${AUTH[@]}" "$API/pulls/$PR_NUMBER/merge" \
  -d '{"merge_method":"squash"}'
```
Prefer `squash` to keep `main` history clean (use `merge` only if the user prefers preserving
commits). After a successful merge, clean up:
```bash
git checkout main && git pull --ff-only
git branch -d <branch>
git push origin --delete <branch>
```
If the merge call returns `405`/not mergeable, do NOT retry blindly — report why (conflicts,
failing checks, branch protection) and ask the user.

## Issues

```bash
# create
curl -sS -X POST "${AUTH[@]}" "$API/issues" \
  -d '{"title":"Basket stuck in CheckoutPending on consumer crash","body":"Repro: ...","labels":["bug","basket"]}'
# list open
curl -sS "${AUTH[@]}" "$API/issues?state=open"
# comment
curl -sS -X POST "${AUTH[@]}" "$API/issues/$N/comments" -d '{"body":"..."}'
# close
curl -sS -X PATCH "${AUTH[@]}" "$API/issues/$N" -d '{"state":"closed"}'
```

## Releases

Tag from `main` after the relevant PRs are merged. Use semantic versioning (`vMAJOR.MINOR.PATCH`).
```bash
curl -sS -X POST "${AUTH[@]}" "$API/releases" \
  -d '{"tag_name":"v1.3.0","target_commitish":"main","name":"v1.3.0","body":"## Changes\n- ...","draft":false,"prerelease":false}'
```
For a draft to review first, set `"draft":true`. Summarize merged PRs in the body.

## Handling conflicts & syncing

```bash
git checkout <feature-branch>
git fetch origin
git rebase origin/main          # or merge if the user prefers; rebase keeps history linear
# resolve conflicts, then:
git add <resolved> && git rebase --continue
git push --force-with-lease     # ONLY on your own feature branch
```
Never resolve conflicts by discarding the other side blindly — read both versions.

## Troubleshooting
- `403`/`401` from API → token missing/expired or lacks scopes (needs `repo` scope). Ask the
  user to refresh `GITHUB_TOKEN`; never print it.
- Push asks for a password → credential helper not configured (redo Step 0).
- `jq: command not found` → parse with `grep`/`python3 -c` instead, or ask to install `jq`.

## Checklist
- [ ] Token read from `GITHUB_TOKEN`; never printed, logged, or committed.
- [ ] Work done on a feature branch; never pushed directly to `main`.
- [ ] Commits follow `type(scope): summary` with service scopes.
- [ ] `dotnet build && dotnet test` green before pushing.
- [ ] PR opened with a clear what/why/testing body; issues linked.
- [ ] Before merging own PR: mergeable + checks green verified.
- [ ] Squash-merged, then branch deleted locally and remotely.
