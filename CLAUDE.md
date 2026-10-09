# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Shared GitHub Actions for the sentfutures org: two **reusable workflows** under
`.github/workflows/` (the Claude PR review bot and the `@claude` mention
handler), the **caller templates** consuming repos copy (`callers/`), the
**`pr-review-watch` skill** PR authors use to respond to the bot, and a
**Claude Code plugin marketplace** (`.claude-plugin/`, `plugins/review-bot/`)
whose `/install-review-bot` and `/disable-review-bot` skills operate all of it.

The YAML under `.github/workflows/` *is* the product. Most PRs change it. The
README is the full runbook and design rationale; read its "Changing the bot"
and "Branch protection" sections before touching the review workflow.

## There is no build, lint, or unit test — the selftest is the test

Nothing runs locally. Changes are tested and released by the PR flow itself:

- **Test:** `selftest-claude-pr-review.yml` is this repo's own caller and
  invokes the shared workflow **by local path**, so a PR editing
  `claude-pr-review.yml` is reviewed by the PR branch's *own* version of the
  bot. `review / claude-review` is a required check on `main`. Watch it with
  `gh pr checks <N>` and read the run with `gh run view <id> --log`.
- **Expected quirk:** `anthropics/claude-code-action` self-skips on a PR that
  edits a workflow file (it only runs the copy on the default branch). Such a
  PR lands in the `NOT_REVIEWED` path: green check, `needs-human-review`
  label, human reviewer requested. That path is itself under test.
- **Blind spot — treat as a failing test:** a change that fails at *startup*
  (a `permissions:` the selftest caller does not grant, an unknown input, a
  YAML syntax error) creates **no check at all**; the PR shows
  `review / claude-review — Expected, waiting for status`. Admin enforcement
  is off on `main`, so an admin *can* merge past it. Never do.
- **Release:** merging to `main` **is** the release. `release.yml` moves the
  `v1` tag to the merge commit and every consumer picks it up on its next PR
  event. Rollback is re-pointing the tag:
  `git push origin +<old-sha>:refs/tags/v1`.
- **Breaking changes** (renamed/removed input) must not ride `v1`: tag `v2`,
  update `callers/` and the `sentfutures/.github` templates to `@v2`, add a
  README changelog entry.

To exercise the plugin skills locally:
`/plugin marketplace add sentfutures/devops` then
`/plugin install review-bot@sentfutures`.

`plugins/review-bot/.claude-plugin/plugin.json` deliberately has no
`version`, so installs track this repo's commits. Claude Code updates a
plugin only when its computed version changes. The `"1.0.0"` pinned on
2026-08-20 kept every install on the August skills through four changes to
`install-review-bot` (09-10 to 09-30). Don't add a `version` back unless every
plugin change bumps it.

## How the review bot works (claude-pr-review.yml)

One job, `claude-review`: a job-level gate, then steps that hand state to
each other via `$GITHUB_OUTPUT`:

1. **Gate + concurrency** (`if:` / `concurrency:`): skips fork PRs (no
   secrets on `pull_request` from forks — the mention handler is the fallback)
   and title/body-only `edited` events; cancels superseded runs so a burst of
   pushes yields one verdict. Cancellation is why later steps guard on
   `!cancelled()`, not `always()` alone.
2. **Record review start time** — backdated 60s; the verify step only counts
   a verdict submitted after this, so a stale review at the same SHA is
   ignored.
3. **Write per-file diffs** — `git diff` of the `pull_request` merge commit
   against its first parent (checkout is `fetch-depth: 2`), one file per
   changed path under `.claude-review/diffs/`; `generated_paths` (git globs)
   get no diff. The review must not depend on `gh pr diff`: above Claude
   Code's tool-output limit its output arrives as a 2 KB preview, and it
   takes no path argument (#11). Also writes `coverage.jq`, the one
   definition of "read": the lines successful Read calls *returned* (the
   result's `startLine`/`numLines`) cover all of a diff's lines. Not the
   call's `limit`: Read stops early at its 25,000-token cap, without an
   error.
4. **Compose review prompt** — bash assembles the prompt from fixed quoted
   heredocs, the list of diff files, and the `workflow_call` inputs
   (`extra_instructions`, `required_check`, `generated_paths*`). The
   reviewing rules and the verdict rules are fixed text; only the
   repo-specific sections are inputs.
5. **Hold the verdict until every diff is read** — writes a Claude Code
   PreToolUse hook, passed to the action as its `settings` input. While
   any diff is unread, the hook refuses `gh pr review` and names the
   unread diffs and their unread line ranges (`unread.jq`, also used by
   the follow-up prompt). It refuses at most twice, and lets the verdict through
   whenever it cannot tell. Prompt wording alone got 7–12 of 24 diffs read
   on factory-farm-em#40.
6. **Run Claude Code Review** — `anthropics/claude-code-action@v1` with a
   narrow tool allowlist (Write, the inline-comment MCP tool, and
   `gh pr review|diff|view`), `Task` disallowed, and `--effort high`
   (Claude Code defaults Sonnet 5.5 to `medium`, which read about half the
   diffs before trying to approve). Claude Code itself also lets read-only
   Bash commands (`grep`, `sed -n`, `tail`) through, whatever the
   allowlist says, and has no Grep or Glob tool. The prompt therefore asks
   for `grep` to search and Read for everything else: coverage counts only
   Read. The review **cannot read CI** and the prompt tells it not to try.
7. **Measure which diffs the review read** — from the execution file.
8. **Review the diffs the first pass did not read** — only when some are
   unread: a second action run resumes the first session
   (`--resume <session_id>`), reads the rest, and files one more verdict for
   the whole PR. `continue-on-error`, so it can never sink the first verdict.
9. **Report review coverage** — job summary over both passes; anything still
   unread gets a `::warning::` and a PR comment with a ready-to-paste
   `@claude` prompt (posted with `GITHUB_TOKEN`, so it triggers nothing).
10. **Verify a review verdict was posted** — the action reports success even
   with no verdict, so this step reads the reviews API itself: last *bodied*
   claude review for this head SHA since the start time — the follow-up's,
   when one ran. `APPROVED` / `CHANGES_REQUESTED` pass; `COMMENTED` (no
   binary verdict) fails the check; `DISMISSED` is a notice; no verdict fails
   and dumps the agent's last turn.
11. **Request human review** — on `COMMENTED` or `NOT_REVIEWED`: apply
    `escalation_label` and request `escalate_to` reviewers, minus the PR
    author (GitHub 422s) and anyone already pending.

`claude-mention.yml` is the near-stock action with `contents: write` so
`@claude` can push commits when asked; it is deliberately broader than the
review bot.

## Invariants when editing the review workflow

Each of these exists because of a named production incident; the inline
comments say which. The selftest may not catch the failure mode you
reintroduce.

- **The job declares no `permissions:` block.** It inherits the caller's. A
  permission declared on the called job is a startup requirement on every
  caller, and a startup failure produces no check, no label, no escalation.
- **Inputs enter `run:` scripts only via `env:`.** Never `${{ inputs.x }}`
  inside a script body. Prompt text uses quoted heredocs (`<<'BLOCK'`) and
  placeholder substitution (`__PR__`, `__CHECK__`).
- **Do not weaken the verify step's jq** (`(.body|length) > 0`, the
  `jq -s 'add'` page merge, the `commit_id` and `submitted_at` filters) or
  the tool allowlist without reading their comments first.
- **A new input** needs a description, a safe empty default, and README
  documentation in the same PR. Verdict rules, the allowlist, and the
  verification logic are intentionally *not* inputs.
- **Coverage is report-only.** The measurement, the follow-up pass and the
  warning comment never fail the check or gate the verdict — decided
  2026-10-08 (#11): tying the verdict to coverage would block too many PRs.
  The verdict gate's hook decides only *when* the verdict goes out. It
  must keep its cap on refusals and keep letting the verdict through
  whenever it errors.
- **The follow-up step repeats the first pass's `claude_args`** (allowlist
  and `--effort`). Change both together.
- **`show_full_output: true` must already be on `main`** before a failure
  you need to diagnose; PRs editing the workflow self-skip.

## Files that move together

- `callers/*.caller.yml` are **mirrored** as org workflow templates in
  `sentfutures/.github/workflow-templates/` — update both.
- `skills/pr-review-watch/SKILL.md` is the **canonical** copy; consuming
  repos hold copies under `.claude/skills/`. Improve it here, and keep the
  "In this repo (fill in at install…)" section as a fill-in template.
- `/install-review-bot` fetches `callers/claude-pr-review.caller.yml`,
  `callers/claude-mention.caller.yml`, and `skills/pr-review-watch/SKILL.md`
  **by path** via `gh api repos/sentfutures/devops/contents/...`. Renaming or
  moving them breaks every install until the skill is updated.
- The README **Changelog** records every behavioral change to `v1` with its
  date and the incident behind it. Add an entry with the change.
- `animal-welfare-data-pipeline` (the origin repo) calls both shared
  workflows at `@v1` since 2026-09-21 (its #169); it has no copies left to
  keep in sync.
- This repo must stay **public**: outside-org consumers resolve
  `uses: sentfutures/devops/...` cross-owner.

## Conventions

- PR descriptions written by Claude open with a `> [!NOTE]` callout naming
  Claude as the author and include a "How to test" section.
- Never post comments, replies, or reviews on a PR from the user's account,
  and never merge; draft text for the human to post. Review responses are
  posted by humans, org-wide.
- Prose in this repo is dense and dated: when documenting a decision, say
  what was observed, when, and on which repo/PR, in the style of the existing
  README and inline comments.
