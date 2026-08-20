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

1. reads the diff and leaves **inline comments** on specific lines;
2. files exactly one **verdict** — approve, or request changes — with a
   summary that must agree with itself (a "minor nit" summary files as an
   approval, never as a limbo comment);
3. is **checked up on**: a verification step confirms a verdict for the
   current commit actually landed, and fails the `review / claude-review`
   check if the bot looked but wouldn't commit to a verdict;
4. **escalates to humans** when it can't supply a verdict: the
   `needs-human-review` label plus a review request to the maintainers named
   in your caller.

The `@claude` handler is the companion: comment `@claude <request>` on any
issue or PR and Claude responds in a thread. It is also the fallback for the
two cases the review bot deliberately skips — fork PRs (no secrets on fork
triggers) and PRs whose automated review never ran (`@claude please review
this PR`).

## Installing on a repo

### The one-time org setup (do this once, then no repo ever needs an admin)

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

### If the org-wide setup hasn't happened

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

### Per repo

3. **Label** — create the `needs-human-review` label in the repo (Issues →
   Labels). A missing label only degrades to a warning in the run log, so do
   this up front where it's visible.
4. **Branch protection — decide, don't inherit.** See
   [the decision every repo owes](#branch-protection-the-decision-every-repo-owes) below.
5. **Add the workflows** — either path:
   - *Point-and-click*: repo → Actions → New workflow → under **"By
     sentfutures"** choose **"Claude PR review bot — reviews every PR"** →
     Configure → replace the two `FILL_ME_IN` values (`required_check`: your
     CI check's name as it appears on a PR, or delete the line if you have
     none; `escalate_to`: maintainer GitHub logins) → commit. Repeat for
     **"Claude @mention handler — on-demand"** (nothing to fill in).
   - *Copy the files*: `callers/claude-pr-review.caller.yml` and
     `callers/claude-mention.caller.yml` from this repo into your repo's
     `.github/workflows/`, filling the same two values.

   Then copy `skills/pr-review-watch/` from this repo into your repo's
   `.claude/skills/` and complete its "In this repo" fill-in bullets from
   your repo's own CLAUDE.md and CI. That skill is how a Claude Code session
   responds to the bot's reviews: verify every claim against the code before
   complying, never post on the PR, escalate disagreements.
6. **Verify** — open a trivial test PR (one-line README change) and watch the
   run: the review posts inline comments and a verdict, "Verify a review
   verdict was posted" goes green, and the escalation step is skipped. Then
   comment `@claude say hello` on the PR to confirm the mention handler.

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
(skim committed generated data instead of analyzing it), `escalate_to`,
`escalation_label` (default `needs-human-review`), `extra_instructions`,
`model`. Each is documented at the top of `claude-pr-review.yml`. What is
*not* an input — the verdict rules, the tool allowlist, the verification
logic — is deliberately unreachable from a caller.

## Branch protection: the decision every repo owes

> **claude[bot]'s approval counts as "1 approving review".** On the pipeline
> repo, whose protection requires the `smoke` check plus one approval, the
> measured consequence was that **113 of 124 merged PRs carried no human
> approval** (per that repo's process retrospective). Nothing about
> installing this bot forces that trade — but leaving branch protection
> unexamined makes it silently.

Choose one, deliberately, per repo:

- **Recommended default — human approval required, bot advisory.** Add a
  `CODEOWNERS` file naming human owners, enable *Require review from Code
  Owners*, and add `review / claude-review` as a required status check. The
  bot still blocks bad PRs (its check fails on request-changes, on a
  no-verdict review, and on a missing verdict) but its approval cannot merge
  anything — a human's can.
- **Opt-in — bot approval satisfies the gate.** Require 1 approval with no
  code-owner rule. Velocity for repos where the team explicitly accepts that
  a PR can merge with no human having read it. If you choose this, say so in
  the repo's README.

## Changing the bot

Three tiers, in order of how often they should happen:

1. **Change how it reviews *your* repo** — edit your caller's `with:` block
   (usually `extra_instructions`). No PR to this repo needed.
2. **Change the bot for everyone** — PR to this repo editing
   `claude-pr-review.yml`. The selftest calls the shared workflow **by local
   path**, so your PR is reviewed by the *changed* bot itself — you watch it
   work before it can merge, and the selftest is a required check here so a
   change that breaks the bot cannot land. **Merging is releasing**:
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
| `review / claude-review` is red with "no binary verdict" | Working as designed: the bot looked and wouldn't decide; the PR now carries `needs-human-review` and a human review request. A human reviews, then dismisses or supersedes. |
| Reviews mention a check that never finishes | Your `required_check` value doesn't match the check's real name — copy it exactly from a PR's checks list, or delete the line if the repo has no CI. |

## Changelog

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
