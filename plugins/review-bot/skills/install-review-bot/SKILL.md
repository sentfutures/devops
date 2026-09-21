---
name: install-review-bot
description: Install the sentfutures Claude PR review bot and @claude mention handler on the current repository. Use when someone asks to "install the review bot", "add Claude review to this repo", "set up the sentfutures reviewer", or to onboard a repo to the shared review workflows. Works on sentfutures repos (nothing to set up) and on personal or other-org repos (needs the per-repo app and secret first). Creates the escalation label, adds the two caller workflows from sentfutures/devops, installs the pr-review-watch skill, and opens a setup PR — never pushes to the default branch and never merges.
---

# Installing the review bot on this repo

Everything lands via a PR. Never commit to the default branch directly, and
never merge the setup PR — a human reviews and merges it. The full runbook
and design rationale live in the sentfutures/devops README; this skill is the
executable version of its install steps.

## 1. Preconditions

- `gh auth status` must be logged in with `repo` scope on this repository.
- **Confirm a `CLAUDE_CODE_OAUTH_TOKEN` reaches this repo.** Either source
  works; check the org one first, then fall back to a repo-level secret:

  ```
  gh api repos/<owner>/<repo>/actions/organization-secrets --jq '.secrets[].name'
  gh secret list --repo <owner>/<repo>
  ```

  A repo outside any org returns **422** from the first command — that is not
  an error to report, just a signal to rely on the second. STOP only if
  *neither* lists `CLAUDE_CODE_OAUTH_TOKEN`. Which advice to give depends on
  where the repo lives: inside sentfutures, it isn't covered by the org secret
  (see "The org setup" in the devops README); outside, the user needs the
  per-repo fallback below.
- **Decide which install path this repo is on.** The workflows are
  org-agnostic — `sentfutures/devops` is public, so the `uses:` reference
  resolves from any owner. What is *not* automatic outside the org is the pair
  of prerequisites the org-wide setup provides.
  - **Inside the sentfutures org** — the Claude GitHub App covers all
    repositories (org-wide install, 2026-08-20) and the org secret reaches all
    of them. Nothing to do; proceed.
  - **Outside it** (a personal repo, or another org) — proceed, but first
    confirm both prerequisites from "Per-repo fallback" in the devops README:
    the Claude GitHub App installed on this specific repo, and a repo-level
    `CLAUDE_CODE_OAUTH_TOKEN` holding the *shared bot account's* token, not a
    personal one. If either is missing, say which, point at that README
    section, and stop — you can neither install an app nor supply a secret
    value on the user's behalf. Also tell them this install depends on
    `sentfutures/devops` staying public: making it private breaks the
    cross-owner `uses:` and forces a transfer into the org.
- Check `.github/workflows/` for existing files named `claude-pr-review.yml`
  or `claude-mention.yml` — if present, this repo may already be onboarded;
  report instead of overwriting. When they are present, check one thing before
  you report: does `claude-pr-review.yml`'s `permissions:` block include
  `actions: read`? Repos onboarded before 2026-09-21 predate it, and without it
  the review cannot read its `required_check` — it reports "Resource not
  accessible by integration" in its summary and reviews blind to whether the
  suite passed. Tell the user; the fix is that one line, copied from
  `callers/claude-pr-review.caller.yml`.

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
  - **Solo-maintainer repos**: if the only admin is the person who will author
    the PRs, say so plainly at install time. GitHub rejects a review request
    naming a PR's own author (422), so the shared workflow drops them and
    escalation degrades to the `needs-human-review` label alone — nobody is
    notified, and a `NOT_REVIEWED` or neutral verdict can pass unseen. Still
    fill `escalate_to` with their login (it starts working the moment anyone
    else opens a PR), and record the caveat in a comment in the caller so it
    is discoverable later.
- Branch protection: `gh api repos/<owner>/<repo>/branches/<default>/protection`.
  A **403** with "Upgrade to GitHub Pro or make this repository public" means
  protection is unavailable on this plan, so no check can be required and the
  bot is **advisory only** — its approval gates nothing and its red check
  blocks nothing. Say this at install time rather than letting the user assume
  the bot is enforcing something. Note it in the pr-review-watch skill's
  "Merge gate" bullet in step 3.
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

If you set a `required_check`, add one more thing to check on that test PR:
the review's summary should **refer to the check's result**, and must not say
it could not read `statusCheckRollup`. Inline comments, a verdict and a green
check all appear even when the review cannot see CI, which is how that went
unnoticed for 114 PRs on the origin repo — so it is worth naming as its own
expectation rather than trusting the green.
