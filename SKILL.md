---
name: review-pr
description: Use when reviewing a pull request with automated review comments from Qodo or Copilot, addressing PR feedback, watching CI, or merging a PR. Runs an autonomous loop — handle bot/CI feedback, commit fixes, wait for re-review, repeat until green, then merge. Triggers on "review PR", "PR review", "address PR comments", "handle PR feedback", "watch CI", "merge PR", "resume PR review", "iterate PR until green".
---

# Review PR

Autonomous PR-review loop: iterate until GREEN, then merge. Each round ingests every new review
comment and CI result, critically evaluates it, applies the fixes you agree with, replies to and
resolves every comment, commits, and waits for the re-review that push triggers. Fully autonomous —
no per-round prompts; auto-merges at GREEN. Convergence is tracked in a durable **ledger**; the
loop bails (stops + notifies) on the safety conditions below. Churn is diagnosed, not waived: a
classified checkpoint at round 4 and a split-or-re-approach verdict at round 8.

**Announce at start:** "I'm using the review-pr skill to run an autonomous review-until-green loop on PR #N."

## GREEN — the termination condition

All of:
- CI settled with no unaddressed failures.
- No new-actionable bot comments — every comment ledgered as fixed / pushed-back / deferred. Two
  consecutive clean bot passes on the same head settle this; don't wait for more.
- Every bot review thread **resolved** — replied-but-unresolved is open work.
- No accepted-but-unfixed items.

Merge-state (conflicts, behind-base, required approvals) is handled at **Merge**, not here.

## Round accounting

A **round** is one push that changes the PR's code (the opening head counts) followed by at least
one new-actionable bot finding. The count is cumulative for the PR's life. These never advance
it — each is logged as a pointer row:

- a bot timeout (silent through the Step 1 wait);
- a "no findings" / approval-only review;
- a re-raise of a finding already in the ledger: same id, same file+line+text, or same substance
  under a fresh id and wording.

A **clean pass** is a bot's pass on a head that adds no new-actionable finding.

## Phase 0 — Setup

1. **Resolve the PR** from the argument or current branch:
   `gh pr view --json number,baseRefName,headRefName,url,state`. Capture `owner/repo`, PR number,
   `baseRefName`.
2. **Load or create the ledger** — one top-level PR comment starting with
   `<!-- review-pr-ledger -->`, holding the round count, the `self-review` flag, the
   `checkpoint@4:` / `cap@8:` verdict lines, and one row per bot comment:
   `round | comment | class | disposition` (`fixed@<sha>` / `pushed-back` / `deferred` /
   `pointer`; classes under **Checkpoint@4**). Fetch top-level comments and search for the
   marker: found → resume it (the round count stands — **re-invocation never resets it**; fill any
   missing `class`); absent → post it empty. All loop state a future session needs lives in this
   comment, nowhere else.
   ```
   <!-- review-pr-ledger -->
   round: 4 · self-review: done@<sha>
   checkpoint@4: CONTINUE — original 5, fix-induced 2, scope 1, re-raise 0, noise 3
   | round | comment | class | disposition |
   | 3 | Devin inline 4135853006 (override defeats the recipe added in round 2) | fix-induced | fixed@<sha> |
   ```
3. **Shape gate** (round 0 only). Concerns: schema/migration, core runtime,
   infrastructure/deploy, external integration, setup/installer, docs-with-recipes, agent
   skills/prompts. If the diff touches any two, or exceeds ~400 non-test lines
   (`git diff --numstat origin/<base>...HEAD`), stop — request no bot review — and cut it into a
   stacked series: one concern per PR, each based on the previous PR's branch, merged bottom-up,
   each tracked by its own issue. Run this skill on each PR of the series, bottom-up. Docs with
   shell recipes: `bash -n` plus a fixture run of every recipe block before round 1, or an
   explicit "not executed" marker on the block.
4. **Self-review** (once, before round 1). With no `self-review` flag at round 0, launch ONE
   subagent running `/superpowers:requesting-code-review` on the head (never in the main thread)
   before bot review is requested — while the PR is a draft (bots skip drafts) or, on an
   already-open PR, before the first Assess. Fix only its critical/high-confidence findings, in the
   opening head; record `self-review: done@<sha>`. It never runs mid-loop: a ledger resumed past
   round 0 without the flag records `self-review: skipped (mid-loop)`. A resumed session must not
   re-run it.

## The Loop

Repeat until GREEN or a bail fires.

### Step 1 — Wait for settle (CI + every bot)

A push re-triggers CI **and** the bot reviewers; wait for both before assessing. (Fresh
invocation with nothing pushed since the last assessment? Skip to Step 2.)

1. Record the push time and the **expected reviewer set**: every bot that has reviewed this PR
   or is a requested reviewer.
2. CI: `gh pr checks <pr> --watch --interval 30` blocks until every check completes (pass or fail).
3. Bots: poll every 30–60s for reviews/comments created after the push. Done when every expected
   bot has posted, or ~10 min past CI settle for any that haven't (a bot with nothing to add may
   stay silent).

The wait ends when every expected reviewer has responded or timed out — **never when the first
ones have**. Foreground `sleep` is blocked: run the whole wait as one background shell
(`run_in_background: true`); for very long waits yield via `ScheduleWakeup` (~270s) and re-enter
on wake. If any bot comment says *"analysing this pull request"* or *"come back in a few
minutes"*, that re-review isn't done — keep waiting; never parse partials.

### Step 2 — Assess

```bash
gh pr view <pr> --json statusCheckRollup,mergeable,mergeStateStatus,reviewDecision,state,baseRefName
```

- **CI:** record failing checks. (`statusCheckRollup` doesn't mark which are required; required
  checks are enforced by `mergeStateStatus` at Merge.)
- **Comments:** fetch ALL comments and threads (see Fetching PR Comments). Filter to bot authors,
  dedup (see Reading Qodo Comments), subtract the ledger **by substance, not comment ID** → the
  **new-actionable** set.
- **Re-raises** (see Round accounting): bookkeeping only — reply once pointing at the prior
  disposition, resolve the thread, log a `re-raise` pointer row. A re-raise never makes a round
  actionable and never triggers rework.
- **Timeouts and no-findings reviews:** a `noise` pointer row each.
- **Count:** new-actionable set non-empty after a code-changing push → increment the round and
  add a classified row per finding; otherwise the count stands. Round 4 → **Checkpoint@4** before
  Work. Round 8 → **Cap@8** instead of Work.

### Step 3 — Work

- All GREEN conditions hold → go to **Merge**.
- **CI red** (not caused by a bot comment): diagnose and fix with
  `/superpowers:systematic-debugging`. Failing-job logs: `gh run view <run-id> --log-failed`
  (GitHub Actions; external CI → follow the check's details URL).
- **New-actionable comments:**
  1. Run `/superpowers:receiving-code-review` over the set — a gate, not a formality. A bot's
     suggested patch or "agent prompt" is context, not a spec; write fixes from your own
     understanding of the code.
  2. Decide per comment: **accept** / **push back** / **defer**. The reasoning becomes the reply
     text and the ledger disposition.
  3. Apply accepted fixes yourself.
- **Never add a migration, dependency, or new integration inside a review round** — that is a new
  PR by definition (never a silent scope expansion). Defer the finding to its own
  issue/PR.

### Step 4 — Close the round

1. **Commit & push** the round's fixes as one commit; its SHA goes in the replies. Nothing to
   commit (all pushed-back/deferred)? Skip commit and push; after the bookkeeping below return
   directly to Step 2 — nothing new was triggered, so don't wait.
2. **Reply to EVERY new comment** — confirm fix (what changed + SHA) or push back (the
   reasoning). Inline comments:
   ```bash
   gh api repos/{owner}/{repo}/pulls/{pr}/comments/{comment_id}/replies -f body="Fixed in <sha>: <what changed>"
   ```
3. **Resolve every settled thread:**
   ```bash
   gh api graphql -f query='mutation($t:ID!){resolveReviewThread(input:{threadId:$t}){thread{isResolved}}}' -F t=<thread-id>
   ```
4. **Verify it landed:** re-fetch comments and threads; every intended reply and resolution must
   exist. Composed is not posted.
5. **Update the ledger comment** with the round's rows (class + disposition) and pointer rows.

Pushed? → Step 1.

## Checkpoint@4 — mandatory

At round 4, before Work:

1. Classify every ledger row:

   | class | Row is |
   |-------|--------|
   | `original` | a defect in the opening diff |
   | `fix-induced` | in code added during the loop |
   | `scope` | a suggestion outside the issue's contract |
   | `re-raise` | a finding already in the ledger (pointer row) |
   | `noise` | a timeout, a no-findings review, or a bot summary of another row (pointer row) |

2. Verdict: **SPLIT** if `fix-induced + scope ≥ original`, or any round introduced a migration,
   new dependency, new option/payload model or new external integration; otherwise **CONTINUE**.
3. Write the counts and one line into the ledger:
   `checkpoint@4: <CONTINUE|SPLIT> — original <o>, fix-induced <f>, scope <s>, re-raise <r>, noise <n>[; trigger: <what>][; split → <issue/PR>]`.
4. On SPLIT: everything the loop added beyond fixing `original` rows — loop-added design and
   accepted `scope` changes — moves to its own issue/PR with its own contract; revert it
   here so this PR carries only the original concern, and move its open rows to
   `deferred → <issue/PR>`. The split-off PR starts at Phase 0. Continue the loop here; the round
   count stands. Nothing separable (the fix-induced rows come from fixing the original concern
   itself) → re-approach as in Cap@8.

## Cap@8 — diagnosis, not a waiver

At round 8, instead of Work. A bare request for more rounds is never valid.

1. **Diagnose:** the classification table (class counts per round), the concern list (Phase 0
   concerns the diff now touches, plus each design the loop added), and the proposed split (which
   concern or design → which new issue/PR; what stays).
2. Write it into the ledger with one line:
   `cap@8: <SPLIT|RE-APPROACH|CONVERGING> — original <o>, fix-induced <f>, scope <s>, re-raise <r>, noise <n>; rounds 7/8 original <a>/<b>, fix-induced <c>/<d>`.
3. **Converging** — rounds 7 and 8 each have ≤1 new `original` row and zero `fix-induced` rows:
   ask the user whether to continue, quoting the table and the rounds 7/8 counts. Approval →
   **Continuation** (below).
4. Otherwise the default is:
   - **Split** — the PR carries more than one concern or a loop-added design: execute the split
     as at Checkpoint@4; the reduced PR proceeds by **Continuation**.
   - **Re-approach** — the PR is already one concern: stop the loop, leave the PR open, file an
     issue for a new approach carrying the diagnosis, and report.

## Merge (reached only at GREEN)

Read `mergeStateStatus` / `reviewDecision` fresh:

| State | Action |
|-------|--------|
| `CLEAN`, or `UNSTABLE` (= only non-required checks failing) | **Merge.** Base `main` → `gh pr merge <pr> --squash`; other base → `--merge`. Report done. |
| `BLOCKED` + `reviewDecision: REVIEW_REQUIRED` | **Stop & notify:** "Everything is green; only your approving review is needed." Never `--admin`. |
| `BEHIND` | `gh pr update-branch <pr>`, then `git pull` (the update lands only on the remote; a stale local fails the next push) → Step 1. |
| `DIRTY` | True conflict — needs manual resolution. Stop & notify. |
| Anything else (`DRAFT`, `UNKNOWN`, …) | Re-read once after ~30s; if unchanged, stop & notify. |

## Convergence & Safety

The ledger prevents loops. Matched by substance, a decided issue stays decided: re-raises get a
pointer-reply and a resolved thread, nothing more. Only genuinely new issues advance the loop.

**Bail conditions** — on ANY, stop and notify the user with current state + what's outstanding.
The round cap is not a bail: it runs **Cap@8**.

| Condition | Why |
|-----------|-----|
| Same CI check fails **3×** despite fixes | Unfixable here — needs a human |
| **2 consecutive no-op passes** (no fix committed AND no new-actionable items) while not green | Stuck / oscillating |
| A few hours elapsed since the ledger comment's `created_at` | Runaway protection |
| An accepted fix is destructive / ambiguous / security-sensitive | Never auto-apply — ask |

**Continuation** — the user overrides a bail ("keep going"), approves a converging Cap@8 ask, or a
Cap@8 split leaves a reduced PR: do not resume the same loop. The counter keeps counting. Map
every finding from all rounds to where it landed; findings clustered in prose you added (spec, PR
body, README, comments) mean the reviewable surface is over-extended — cut it, don't patch it
(every claim you add is new material for bots to audit); then at most ONE consolidation round,
then report back.

## Fetching PR Comments

Always `--paginate`; never combine it with `--jq` (jq runs **per page** — counts and slices are
silently wrong). Fetch raw, filter locally.

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate    # inline code comments
gh api repos/{owner}/{repo}/issues/{pr}/comments --paginate   # top-level comments (ledger lives here)
gh api repos/{owner}/{repo}/pulls/{pr}/reviews --paginate     # review summaries
# review threads + resolution state; loop with $after while hasNextPage
gh api graphql -f query='query($o:String!,$r:String!,$n:Int!,$after:String){repository(owner:$o,name:$r){pullRequest(number:$n){reviewThreads(first:100,after:$after){pageInfo{hasNextPage endCursor} nodes{id isResolved comments(first:1){nodes{databaseId}}}}}}}' -F o=<owner> -F r=<repo> -F n=<pr>
```

## Reading Qodo Comments

**Bot authors:** any commenter with `user.type == "Bot"` (e.g. `qodo-merge[bot]`,
`copilot-pull-request-reviewer[bot]`), plus Qodo's user-typed logins `pr-agent-pro` /
`pr-agent-pro-staging`.

**Deduplication:** Qodo posts each issue twice — a PR-level summary and an inline comment. Merge
them *before* ledger subtraction (one issue = one ledger entry); the inline comment ID is the one
you reply to.

**Severity labels** ("Action required" / "Review recommended") are the bot's confidence, not
ground truth — `/receiving-code-review` makes the call.

## Rules & Pitfalls

| Rule | Why |
|------|-----|
| `/superpowers:receiving-code-review` before any fix, every round | Every comment gets critical evaluation before you touch code |
| Never invoke `/qodo-pr-resolver` | It bypasses critique |
| Every new comment replied, every settled thread resolved, verified by re-fetch | Bookkeeping IS convergence |
| Loop state lives in the ledger PR comment, never only in session memory | Any session can resume; the cap can't reset |
| Shape gate and self-review before round 1, never mid-loop | Their fixes belong in the opening head, not in extra rounds |
| No migration, dependency or new integration inside a review round | A new PR by definition |
| Checkpoint@4 and Cap@8 verdicts written into the ledger | Churn is auditable without reading the threads |
| Waits run as background shells, never foreground `sleep` | Foreground `sleep` is blocked |
| Never bypass branch protection | Approval-blocked → stop & notify; no `--admin` |

**Stop if you catch yourself thinking:**
- "The bot already reasoned about this" — it's a reviewer, not an oracle.
- "I'll just apply the bot's suggested patch" — context, not a spec.
- "The bot re-raised it, so I must re-fix it" — decided items stay decided: pointer-reply + resolve.
- "It's still red after many tries, I'll keep going" — honor the bails.
- "Round 8 — I'll ask for more rounds" — run Cap@8; only the converging case asks, and the ask carries the table.
- "The bot wants a new table / dependency / integration, I'll add it" — that is a new PR.
- "The verification obviously passes, I'll report it as run" — never assert a check you didn't run this round.
- "I'll add a defensive comment so the bot stops flagging it" — narration is new falsifiable surface; shrink it, don't explain it.
