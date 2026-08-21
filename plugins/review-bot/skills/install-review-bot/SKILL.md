---
name: install-review-bot
description: Install the sentfutures Claude PR review bot and @claude mention handler on the current repository. Use when someone asks to "install the review bot", "add Claude review to this repo", "set up the sentfutures reviewer", or to onboard a repo to the org's shared review workflows. Creates the escalation label, adds the two caller workflows from sentfutures/devops, installs the pr-review-watch skill, and opens a setup PR — never pushes to the default branch and never merges.
---

# Installing the review bot on this repo

Everything lands via a PR. Never commit to the default branch directly, and
never merge the setup PR — a human reviews and merges it. The full runbook
and design rationale live in the sentfutures/devops README; this skill is the
executable version of its install steps.

## 1. Preconditions

- `gh auth status` must be logged in with `repo` scope on this repository.
- Confirm the org secret reaches this repo:
  `gh api repos/<owner>/<repo>/actions/organization-secrets --jq '.secrets[].name'`
  must list `CLAUDE_CODE_OAUTH_TOKEN`. If it doesn't, STOP and tell the user
  this repo isn't covered by the org secret (see "The org setup" in the
  devops README) — nothing else in this skill will work until it is.
- The Claude GitHub App covers all sentfutures repositories (org-wide install,
  2026-08-20). If this repo is outside the sentfutures org, stop and say the
  bot is org-internal.
- Check `.github/workflows/` for existing files named `claude-pr-review.yml`
  or `claude-mention.yml` — if present, this repo may already be onboarded;
  report instead of overwriting.

## 2. Discover this repo's facts

- Default branch: `gh repo view --json defaultBranchRef`.
- CI check name: look at a recent PR's checks (`gh pr list --state merged
  --limit 3`, then `gh pr checks <n>`) and identify the check that builds or
  tests the repo (e.g. `smoke`, `preview`). If the repo has no CI, the caller
  simply omits `required_check` — that is a supported mode, not a blocker.
- Who to page: **ask the user** which GitHub logins `escalate_to` should name
  (suggest the repo's admins from
  `gh api repos/<owner>/<repo>/collaborators --jq '.[] | select(.permissions.admin) | .login'`).
  Do not guess silently.
- Read the repo's CLAUDE.md (if any) for its test command and conventions —
  needed to fill the skill in step 3.

## 3. Make the changes, on a branch

1. Create the label:
   `gh api -X POST repos/<owner>/<repo>/labels -f name=needs-human-review -f color=FF9F1C -f description="Automated review reached no verdict; a maintainer must decide."`
   (409 = already exists = fine.)
2. Fetch the two caller templates from devops and place them at
   `.github/workflows/claude-pr-review.yml` and
   `.github/workflows/claude-mention.yml`:
   `gh api repos/sentfutures/devops/contents/callers/claude-pr-review.caller.yml --jq '.content' | base64 -d`
   (same for `claude-mention.caller.yml`). Fill `required_check` and
   `escalate_to` with the discovered values; delete the `required_check` line
   entirely if the repo has no CI check. Leave everything else — especially
   the `permissions:` block and the `@v1` pin — exactly as shipped.
3. Fetch `skills/pr-review-watch/SKILL.md` from devops the same way, place it
   at `.claude/skills/pr-review-watch/SKILL.md`, and complete its
   "In this repo (fill in at install…)" section from what step 2 found: the
   test command, the check names (the bot's check appears as
   `review / claude-review`), and the merge gate. Remove the fill-in marker
   from the heading.
4. Open the PR. The body MUST open with a callout naming Claude as the author
   (e.g. a `> [!NOTE]` line), and MUST include a "How to test" section. Model
   it on sentfutures/website#202. Include these two facts: the setup PR
   itself gets no usable review (the action self-skips on PRs adding its own
   workflow — the designed `NOT_REVIEWED` path), and branch protection is a
   separate decision the PR does not make (link the devops README's
   "Branch protection: the decision every repo owes").

## 4. Report to the user

Hand back: the PR link; that a human merges it; that after merging they
should open a trivial test PR and expect inline comments + one verdict + a
green `review / claude-review` check; and that testing the mention handler
means **they** comment `@claude say hello` — comments are always posted by
the human, never by you from their account.
