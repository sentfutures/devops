# sentfutures/devops

Shared GitHub automation for the sentfutures org. Today that is two reusable
workflows — the **Claude PR review bot** (a standing auto-reviewer for every
pull request) and the **Claude mention handler** (`@claude` on demand) — plus
the `pr-review-watch` skill their PR authors use to respond to reviews.
`docs/` is the future home for other shared dev-ops resources (CI templates,
test-writing guidelines).

The review bot was extracted from
[`animal-welfare-data-pipeline`](https://github.com/sentfutures/animal-welfare-data-pipeline),
where it reviewed 114 PRs over six weeks. Its verification and escalation
logic each encode a real incident from that period — the inline comments in
`claude-pr-review.yml` say which. Treat those comments as load-bearing.

## What a repo gets

On every pull request, claude[bot]:

1. reads the diff of **every changed file** — each saved as its own diff
   file, with your `generated_paths` left out — and leaves **inline
   comments** on specific lines;
2. files exactly one **verdict** — approve, or request changes — with a
   summary that must agree with itself (a "minor nit" summary files as an
   approval, never as a limbo comment);
3. is **checked up on**: a verification step confirms a verdict for the
   current commit actually landed, and fails the `review / claude-review`
   check if the bot looked but wouldn't commit to a verdict. A coverage
   check counts which diffs it actually read. The bot's verdict is held
   back (at most twice) until it has read every diff, and a follow-up pass
   reads any it still skipped. Anything left unread after that is named in
   a PR comment with a ready-to-paste `@claude` prompt. Coverage is
   report-only — it never fails the check, and a verdict is never
   withheld or downgraded for unread files;
4. **escalates to humans** when it can't supply a verdict: the
   `needs-human-review` label plus a review request to the maintainers named
   in your caller.

The `@claude` handler is the companion: comment `@claude <request>` on any
issue or PR and Claude responds in a thread. It is also the fallback for the
two cases the review bot deliberately skips — fork PRs (no secrets on fork
triggers) and PRs whose automated review never ran (`@claude please review
this PR`).

## Installing on a repo

> **Org setup status: DONE for sentfutures (2026-08-20).** The Claude GitHub
> App covers all repositories and the `CLAUDE_CODE_OAUTH_TOKEN` org secret is
> available to all repositories — installing on a new org repo needs **no
> admin steps at all**. (What that setup was and why its scoping is safe:
> see "The org setup" near the end of this README.)

### The fast path — one skill

One-time, on your own machine:

```
/plugin marketplace add sentfutures/devops
/plugin install review-bot@sentfutures
```

Then, inside any repo you want the bot on, run **`/install-review-bot`** and
answer its questions (mainly: which maintainers to page when the bot can't
decide). It discovers the repo's CI check name, creates the
`needs-human-review` label, adds the two caller workflows, installs the
`pr-review-watch` skill, and opens a setup PR for a human to review and
merge. The same plugin ships **`/disable-review-bot`** — the emergency stop
(see "Disabling the bot" below).

This works outside sentfutures too — on a personal repo or another org. This
repo is public, so the `uses:` reference resolves from any owner; what does
not come for free is the app-and-secret pair the org-wide setup provides, so
do ["Per-repo fallback"](#per-repo-fallback-if-an-org-ever-cant-do-the-org-wide-setup)
first and the skill will take it from there. Two things to expect off-org:

- The install depends on **this repo staying public**. Making it private
  breaks the cross-owner `uses:` for every outside consumer at once.
- **Branch protection may be unavailable.** On a private repo on a free plan
  GitHub returns `403: Upgrade to GitHub Pro or make this repository public`,
  so `review / claude-review` cannot be required and the bot is advisory: its
  approval gates nothing, its red check blocks nothing.

**Solo-maintainer repos**: escalation degrades to label-only. GitHub rejects a
review request naming a PR's own author (422), so when the sole maintainer is
also the sole PR author the workflow drops them and nobody is paged — the
`needs-human-review` label is the only signal that a PR went unreviewed. Fill
`escalate_to` with their login anyway; it starts working the moment a second
person opens a PR.

### The manual path

1. **Label** — create `needs-human-review` in the repo (Issues → Labels). A
   missing label only degrades to a warning in the run log, so do it up
   front where it's visible.
2. **Workflows** — repo → Actions → New workflow → under **"By sentfutures"**
   choose **"Claude PR review bot — reviews every PR"** → Configure → replace
   the two `FILL_ME_IN` values (`required_check`: your CI check's name as it
   appears on a PR, or delete the line if there is none; `escalate_to`:
   maintainer GitHub logins) → commit. Repeat for **"Claude @mention handler
   — on-demand"** (nothing to fill in). Equivalent: copy both files from this
   repo's `callers/` into your `.github/workflows/`.
3. **Skill** — copy `skills/pr-review-watch/` from this repo into your repo's
   `.claude/skills/` and complete its "In this repo" fill-in bullets from
   your repo's own CLAUDE.md and CI. That skill is how a Claude Code session
   responds to the bot's reviews: verify every claim against the code before
   complying, never post on the PR, escalate disagreements.
4. **Branch protection — decide, don't inherit.** See
   [the decision every repo owes](#branch-protection-the-decision-every-repo-owes).
5. **Verify** — open a trivial test PR (one-line README change) and watch the
   run: the review posts inline comments and a verdict, "Verify a review
   verdict was posted" goes green, and the escalation step is skipped. If you
   set a `required_check`, also read the review's summary: it must review the
   code on its merits and must not caveat about tests or CI it could not see
   (no "could not read `statusCheckRollup`", no "Resource not accessible by
   integration") — that wording is a prompt regression in this repo, not a
   setup problem on yours; see the troubleshooting table. Then comment
   `@claude say hello` on the PR to confirm the mention handler.

### Customizing what the bot looks for

Per-repo behavior lives in your caller's `with:` block — edit it any time in
the GitHub web editor. Most customization belongs in `extra_instructions`:

```yaml
    with:
      required_check: "preview"
      escalate_to: "Deco354"
      extra_instructions: |
        This is a static-site repo. Also flag:
        - Broken internal links in changed HTML
        - Images added without alt text
```

All inputs: `required_check`, `generated_paths` + `generated_paths_note`
(leave committed generated data out of the review — whitespace-separated git
globs such as `outputs/**` or `uv.lock`), `escalate_to`,
`escalation_label` (default `needs-human-review`), `extra_instructions`,
`model`. Each is documented at the top of `claude-pr-review.yml`. What is
*not* an input — the verdict rules, the tool allowlist, the verification
logic — is deliberately unreachable from a caller.

## Disabling the bot (when something goes wrong)

- **One repo misbehaving** → run **`/disable-review-bot`** in that repo, or
  manually: `gh workflow disable "Claude PR review bot"` (re-enable later
  with `gh workflow enable`). **If `review / claude-review` is a required
  status check there, also remove it from branch protection** — otherwise
  every PR waits forever on a check that will never run.
- **Every repo broken at once, right after a merge to this repo** → the
  problem is the shared release, not the consumers: a devops admin re-points
  the `v1` tag at the previous commit (the exact command is in
  `release.yml`'s header) and all repos are back on the old bot on their
  next PR event.

## Branch protection: the decision every repo owes

> **claude[bot]'s approval counts as "1 approving review".** On the pipeline
> repo, whose protection requires the `smoke` check plus one approval, the
> measured consequence was that **113 of 124 merged PRs carried no human
> approval** (per that repo's process retrospective). Nothing about
> installing this bot forces that trade — but leaving branch protection
> unexamined makes it silently.

**Default: do not make `review / claude-review` a required status check.**
Two facts decide this (2026-09-29):

- A required check that never reports blocks every merge, and the bot has
  failure modes that report nothing: a startup failure (a permissions
  mismatch, a stale caller on a PR branch, a syntax error) creates no check,
  no label and no escalation — the PR just says "Expected — waiting for
  status" until someone finds the cause. On a repo whose team barely knows
  the bot exists, that freezes development over a bot bug. Not acceptable in
  any circumstance.
- The bot still bites without the required check. Under a "require a pull
  request before merging" rule, a **changes requested** review from
  claude[bot] blocks the merge until a human dismisses it or the bot approves
  the next push — observed on the pipeline repo, whose protection never
  required the review check (each of its five dismissed change requests
  needed that dismissal to merge). A red check for a no-verdict review stays
  visible on the PR, and the label plus review request page the maintainers.
  What the team does with that is the team's call.

**Tests are enforced by branch protection, not by the bot.** The review never
reads CI (its token cannot, and the prompt tells it not to try), and the
post-approval cross-check of the verdict against `required_check` that lived
in the verify step from 2026-09-21 to 2026-09-30 was removed: it gated
nothing under the default above, it could only see a suite that finished
before the review did (on website roughly one push in three finished after
it, two of them 10-25 minutes late in a runner queue), and it spent runner
minutes waiting on every push where it could not. If a red suite must block
merges, *require the CI check* in the branch rules — that blocks the merge
itself, costs nothing, and does not depend on the bot.

A repo owner who wants the check to gate merges can add it to the branch
rules later; nothing in the install assumes it. Then choose how approvals
count, deliberately, per repo:

- **Human approval required, bot advisory.** Add a `CODEOWNERS` file naming
  human owners and enable *Require review from Code Owners*. The bot's
  request-changes still blocks; its approval cannot merge anything — a
  human's can.
- **Bot approval satisfies the gate.** Require 1 approval with no code-owner
  rule. Velocity for repos where the team explicitly accepts that a PR can
  merge with no human having read it. If you choose this, say so in the
  repo's README.

## Changing the bot

Three tiers, in order of how often they should happen:

1. **Change how it reviews *your* repo** — edit your caller's `with:` block
   (usually `extra_instructions`). No PR to this repo needed.
2. **Change the bot for everyone** — PR to this repo editing
   `claude-pr-review.yml`. The selftest calls the shared workflow **by local
   path**, so your PR is reviewed by the *changed* bot itself — you watch it
   work before it can merge, and the selftest is a required check here so a
   change that breaks the bot cannot land — with one blind spot: a change
   that fails at *startup* (a permission the job declares that the selftest
   caller does not grant, an unknown input, a syntax error) reports no check
   at all, and the PR shows "Expected — waiting for status". Treat that as a
   failing selftest; an admin merging past it releases a bot no caller can
   start (2026-09-21, see the changelog). **Merging is releasing**:
   `release.yml` moves the `v1` tag to the merge commit and every consumer
   picks it up on their next PR event. Rollback is re-pointing one tag
   (command in `release.yml`'s header). This tier is fine to hand to Claude
   Code: *"in sentfutures/devops, add <X> to the review bot's prompt and open
   a PR"*.
3. **Breaking changes** (rename/remove an input): do not ride `v1`. Tag `v2`,
   update `callers/` and the `sentfutures/.github` templates to `@v2`, and
   add a changelog entry below.
4. **Never** edit the verdict-verification step's jq or the tool allowlist
   without reading their inline comments first — each line exists because of
   a production incident, and the selftest may not catch the failure mode you
   reintroduce.

## Troubleshooting

| Symptom | Cause → fix |
|---|---|
| `workflow was not found` / `failed to resolve` at the caller's `uses:` line | This repo's sharing setting is off (devops → Settings → Actions → General → Access → *Accessible from repositories in the organization*), or the consuming repo's allowed-actions policy doesn't include `sentfutures/devops/*`. |
| Review step fails on its first turn; full-output stream shows an auth error | The repo isn't in the `CLAUDE_CODE_OAUTH_TOKEN` org secret's selected list (install step 2) — the secret evaluates to empty. |
| `gh api` 403s in the verify or escalation steps | The caller's `permissions:` block was trimmed — restore it exactly as in `callers/claude-pr-review.caller.yml`. |
| Bot never runs on a PR | Fork PRs are skipped by design — comment `@claude please review this PR`. Also check the PR event types in your caller match the template's. |
| No `review / claude-review` check appears on the PR at all (or a required one sits at "Expected — waiting for status"), and the Actions tab shows the run as `startup_failure` — "This run likely failed because of a workflow file issue" | The workflow failed before any job started, so nothing could report; open the run for the one-line reason. Seen so far: (a) the caller's `permissions:` block granted less than the called job declared — fixed 2026-09-29 by making the job inherit the caller's grant, so a caller on `@v1` can no longer hit this; (b) a PR branch carrying an old copy of the caller. GitHub reads the workflow from the PR's *merge ref*, which it does not refresh after an automatic base change, so rebase the branch onto the default branch — that push refreshes it. |
| A PR comment from github-actions says "The automated review of `<sha>` did not read N of M changed files" | The first pass skipped those diffs and the follow-up pass did not finish them (the job summary's "Review coverage" and the "Review the diffs the first pass did not read" log say why). The verdict stands but does not cover those files: post the comment's `@claude` prompt, or review them yourself. If it keeps happening, open an issue here with the run link. |
| `review / claude-review` is red with "no binary verdict" | Working as designed: the bot looked and wouldn't decide; the PR now carries `needs-human-review` and a human review request. A human reviews, then dismisses or supersedes. |
| Review summary says it could not read `statusCheckRollup` / CI ("Resource not accessible by integration"), or caveats its verdict on not having seen the tests | The review is told it cannot read CI and must not try or caveat (the prompt block in `claude-pr-review.yml`). If that wording is back, the prompt on `v1` has regressed (it happened 2026-09-17 to 21): roll `v1` back per `release.yml`'s header and open an issue here. Nothing to fix in your caller — `required_check` is prompt context only. |

## The org setup (done for sentfutures 2026-08-20 — kept for reference)

What the one-time setup was, should another org adopt this or the secret need
recreating:

Ask an org owner, in one sitting: install the **Claude GitHub App** for
**All repositories** (Org Settings → GitHub Apps), and create the
**`CLAUDE_CODE_OAUTH_TOKEN`** org secret — the shared bot account's token,
never a personal one — with **All repositories** access (Org Settings →
Secrets and variables → Actions). Every step below is then doable by anyone
with write access, or by pasting this section to Claude Code with: *"install
the sentfutures review bot on this repo, following the sentfutures/devops
README"*.

Why all-repositories is acceptable here: the token authenticates to Anthropic
only — GitHub write access (posting as claude[bot]) comes from the GitHub
App's own per-run token, never this secret — and the bot account is a
dedicated subscription with no metered billing, so a leaked token buys an
attacker rate-limited Claude usage until rotation, not money and not repo
access. Two habits are the compensating controls: **rotate the token** if
reviews start failing inexplicably (someone else draining the usage windows
is what that looks like) or if a Claude workflow nobody recognizes appears in
a repo's CI; and **note the token's expiry** (subscription OAuth tokens last
about a year — `claude setup-token` prints the date) somewhere it will be
seen before the bot goes quiet org-wide.

### Per-repo fallback, if an org ever can't do the org-wide setup

A **repo admin** can do both prerequisites for their own repo with no org
admin involved — this is exactly how animal-welfare-data-pipeline was set up
(its secret is repo-level, created 2026-07-01):

1. **Claude GitHub App** — run `/install-github-app` from Claude Code inside
   the repo; it walks through the app install and creates the repo secret.
   When it asks for credentials, use the shared bot account's token, not your
   own. (Manual equivalent: install the app for the repo, then
   `gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo sentfutures/<repo>`.) If
   GitHub refuses the app install, that one step needs an org owner.
2. Trade-off to know: the token value is then stored per repo, so rotating
   the shared token means updating every repo that did this.

## Changelog

- **v1, 2026-10-09** — the review reads every non-generated diff, and is
  checked for it (#11). On Deco354/factory-farm-em#29 (25 files, ~150 KB of
  diff) all seven approvals between 2026-10-06 and 10-08 said they had read
  only part of the PR, and none had read `analysis/summary.py`, which
  computes its headline numbers; `@claude` reviews of the unread files then
  found a bug the approvals had missed. Across the five consuming repos, 15
  of 134 approvals since 2026-09-01 admitted unread code. Cause: `gh pr diff`
  was the review's only view of the diff; above Claude Code's tool-output
  limit it arrives as a 2 KB preview plus a saved file, it takes no path
  argument, and nothing told the review to open the file. Sonnet 5 (the
  action's default until late September) opened it in 2 of 2 sampled runs,
  Sonnet 5.5 in 1 of 9. Now the workflow writes one diff per changed file
  (`git diff` of the merge commit; checkout is `fetch-depth: 2`) and lists
  them in the prompt; `generated_paths` files get no diff and are listed by
  name and size only, so `generated_paths` is now whitespace-separated git
  globs (both live values, `outputs/**` and `uv.lock`, already are). The
  list alone was not enough: in a replay of #29 (factory-farm-em#40) the
  first pass read 12, 12 and 7 of 24 diffs under three prompt wordings,
  skipping every test and doc. So a Claude Code hook, passed through the
  action's `settings` input, refuses the review's `gh pr review` while any
  diff is unread and names what is left; it refuses at most twice. A step
  counts the lines each diff's Read calls returned — not the lines they
  asked for: Read stops at 25,000 tokens without an error, and on
  2026-10-09 a 2,000-line diff came back as lines 1–935. If any diff is
  unread, a second action run resumes the review's session to read the
  rest and file one more verdict for the whole PR, which the verify step
  checks as before. Anything still unread gets a PR comment with a
  ready-to-paste `@claude` prompt. Coverage is report-only by decision:
  tying the verdict to it would block too many PRs. The review body is
  told to describe the change, not how it was reviewed: the gated
  approvals on #40 opened with "I read every diff in this PR to its last
  line". Cost: a full read is what September's reviews cost (18–24 turns
  and $0.49–0.99 on 2,000–6,000-line PRs, against ~10 turns and ~$0.13 for
  the partial reads). No input renamed; callers need no change.
- **v1, 2026-09-30** — the post-approval CI cross-check is removed, and
  `required_check` is now prompt context only. The verify step had polled the
  named job for a fixed 90s after an approval and failed `review /
  claude-review` if it had finished red. Removed because: it gated nothing
  (no consumer requires this check, and the section above says not to); it
  could only see a suite that finished before the review did (website: one
  push in three finished after it, two of them 10-25 minutes late in a
  runner queue), so every fixed wait is tuned to one repo's timings and
  loses to a queue anyway; on a suite slower than the review it spent 90s of
  runner time per approved push to learn nothing; and it never fired.
  oTullio's #7 found the gap on enfiyeci/farm-welfare-eval (a 7-12 min suite)
  and proposed waiting to completion behind a new `type: number` input; not
  taken — runner cost proportional to the suite, a typed input that fails
  the workflow at startup if a caller quotes the value, and the wait length
  being verification logic that is deliberately not caller-configurable.
  Enforcement of tests is branch protection's: require the CI check. The
  prompt still tells the review plainly that it cannot read CI and must not
  caveat — that block is what ended the 2026-09-17..21 "Resource not
  accessible by integration" caveats and is unchanged in substance. Callers
  no longer need `actions: read` (the template and selftest drop it; an
  existing caller that keeps it is harmless), and the action step no longer
  requests `additional_permissions: actions: read`. The two 2026-09-21
  entries below are superseded.
- **v1, 2026-09-29** — the reusable job no longer declares its own
  `permissions:`; it inherits the caller's. The 2026-09-21 release had made
  `actions: read` a startup requirement, and a caller without it did not
  degrade — it failed before any job ran, with no check, no label and no
  escalation. That silenced this repo's own selftest (its caller had never
  been given the line; #4 and #5 were merged past a check that never
  reported), left the `sentfutures/.github` template shipping a caller that
  could not start, and jammed PRs on Deco354/factory-farm-em whose branches
  carried the older caller (#17). Now `actions: read` is needed only for the
  CI cross-check of an approval (`required_check`): a caller without it
  reviews normally and the run log warns that the approval stands unchecked.
  The selftest caller and the org template carry the line. The
  branch-protection recommendation changed to **advisory by default** — do
  not require `review / claude-review`; a bot that cannot report must never
  be able to freeze a team.
- **v1, 2026-09-21** — the review can now read its `required_check`. It had
  always been told to, but was never granted `actions: read`, so
  `statusCheckRollup` returned "Resource not accessible by integration" and the
  FAILURE branch of that prompt block was unreachable: a PR with a red check was
  eligible for a clean approval. **Action required for repos installed before
  this**: add `actions: read` to your caller's `permissions:` block. Permissions
  can only be reduced down a reusable-workflow chain, never elevated, so the
  shared workflow cannot supply it for you — a caller without the line will fail
  to start (true until 2026-09-29; see above). New installs get it from the
  template.
- **v1, 2026-08-20** — org setup completed (app + secret, all repositories);
  `review-bot` plugin added (`/install-review-bot`, `/disable-review-bot`);
  runbook restructured around it.
- **v1** — initial release: review bot + mention handler extracted from
  animal-welfare-data-pipeline (its `.github/workflows/claude-code-review.yml`
  and `claude.yml` remain in place there until that repo migrates to a
  caller; until then, fixes belong in both places).

---

*Costs to watch as adoption grows: every consuming repo's reviews draw on one
shared Claude subscription (a failed window shows as a first-turn auth-style
error and a red verify step), and each review run spends the consuming repo's
own Actions minutes — a few minutes per push, which matters on private repos
near the free-tier cap.*
