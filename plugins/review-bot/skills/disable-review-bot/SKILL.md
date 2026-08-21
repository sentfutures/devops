---
name: disable-review-bot
description: Disable (or re-enable) the Claude PR review bot on the current repository — the emergency stop when the reviewer misbehaves, spams verdicts, burns Actions minutes, or blocks merges. Use when someone says "turn off the review bot", "disable claude review", "the reviewer is broken", "stop the bot reviewing PRs", or asks to re-enable it after a pause.
---

# Disabling the review bot

## Triage first — pick the right lever

- **Misbehaving in this repo only** → disable here (next section).
- **Broken in every repo, right after a merge to sentfutures/devops** → the
  problem is the shared release, not the repos. The fix is a devops repo
  admin re-pointing the `v1` tag at the previous commit (the exact command is
  in `release.yml`'s header comment in that repo). Say so and offer to do it
  if the user has devops admin rights; per-repo disabling is the wrong tool.
- **Just noisy on one PR** (e.g. repeated verdicts after a burst of pushes) →
  no disable needed; the newest verdict is the current one and the rest are
  superseded. Explain rather than act.

## Disable in this repo

1. `gh workflow list --repo <owner>/<repo>` and find **"Claude PR review
   bot"**. Leave **"Claude mention handler"** running unless the user asks —
   it only acts when explicitly @-mentioned, and it is the documented
   fallback while the reviewer is off.
2. `gh workflow disable "Claude PR review bot" --repo <owner>/<repo>` —
   immediate for new events; a run already in progress finishes (cancel it
   with `gh run cancel <id>` if the user wants silence now).
3. **Check branch protection before finishing** — this is the step people
   miss: `gh api repos/<owner>/<repo>/branches/<default>/protection`. If
   `review / claude-review` is listed as a REQUIRED status check, every PR
   will now wait forever on a check that never runs. Removing a required
   check changes the repo's merge gate — confirm with the user, then update
   the protection to drop it (leave everything else in the protection
   payload untouched).
4. Report: what was disabled, whether the required check was removed, and
   that open PRs keep whatever verdicts/labels they already have — don't
   strip `needs-human-review` labels; they still mean what they said.

## Re-enable

`gh workflow enable "Claude PR review bot" --repo <owner>/<repo>`, restore
the required check if step 3 removed it, and reviews resume on the next PR
event (an existing PR can be nudged with a push, or close-and-reopen).

## What not to do

- Don't delete the caller workflow file to disable the bot — that loses the
  repo's configuration and hides the state; a disabled workflow is visible
  and reversible in the Actions tab.
- Don't delete or move the shared `v1` tag as a stop — every consuming
  repo's runs would fail red, which is noise, not a disable.
- Never merge, close, or approve PRs as part of this, and never post PR
  comments from the user's account.
