# review-pr

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that takes a pull request from
open to merged on its own. It reads every bot review comment and CI result, decides what to fix,
pushes the fixes, replies to and resolves each thread, waits for the re-review, and repeats until
the PR is green. Then it merges.

It works with any review bot that comments as a GitHub `Bot` user (Qodo, Copilot, Devin, …), and it
also works with CI alone.

## How it works

```
/review-pr 123
```

**Setup (once per PR)**

- **Ledger.** The skill posts a single PR comment marked `<!-- review-pr-ledger -->`. The comment
  holds the round count plus one row per bot finding, with its class and disposition. All loop
  state lives there, so any later session can pick the loop up where it stopped, and the round
  count never resets.
- **Shape gate.** If a diff spans two or more concerns (schema, runtime, infra, integrations,
  installer, docs with recipes, prompts) or has more than ~400 non-test lines, the skill stops
  before bot review and asks for a stack of smaller PRs.
- **Self-review.** Before any bot sees the PR, one subagent reviews it and the skill fixes the
  critical findings in the opening head.

**Loop (until GREEN)**

1. **Settle.** Wait for CI and every expected bot to finish. A bot that stays silent ~10 minutes
   after CI settles is treated as timed out.
2. **Assess.** Fetch all comments and threads (paginated) and subtract the ledger by substance,
   not by comment ID. What remains is the new-actionable set. Re-raises of findings already
   decided get a pointer reply and a resolved thread, never more work.
3. **Work.** Critically evaluate each finding with `superpowers:receiving-code-review`, then accept,
   push back, or defer it. Bot-suggested patches are treated as context, not as specs. Red CI is
   debugged with `superpowers:systematic-debugging`.
4. **Close the round.** Commit once, push, reply to every comment (with the fix SHA or the
   pushback reasoning), resolve the settled threads, re-fetch to confirm the replies landed, and
   update the ledger.

**GREEN** means CI is settled with no unaddressed failures, every bot comment has a ledger
disposition, every bot thread is resolved, and no accepted fix is still pending.

**Merge.** PRs into `main` are squash-merged and other bases get a merge commit. A PR that is
behind its base is updated and re-looped. If the only blocker is a required human approval, the
skill stops and tells you. It never uses `--admin`.

## Churn control

Endless fix-and-reflag cycles are diagnosed instead of being allowed to run on.

- **Checkpoint@4.** At round 4, each ledger row is classified as `original`, `fix-induced`,
  `scope`, `re-raise`, or `noise`. If fix-induced plus scope rows meet or exceed the original
  rows, or if the loop added a migration, dependency, or integration, the extra work is split
  into its own PR.
- **Cap@8.** Round 8 always produces a written diagnosis, never just a request for more rounds.
  The outcome is one of three: split the PR, re-approach the problem in a new issue, or ask you
  whether to continue (only when the last two rounds show convergence).
- **Bails.** The skill stops and reports when the same CI check fails 3 times, when two passes in
  a row change nothing while the PR is still not green, after a few hours of wall-clock time, or
  when an accepted fix is destructive, ambiguous, or security-sensitive.

## Requirements

- Claude Code
- [`gh`](https://cli.github.com/), authenticated with `repo` scope
- The [superpowers](https://github.com/obra/superpowers) plugin, which provides the
  `requesting-code-review`, `receiving-code-review`, and `systematic-debugging` skills

## Installation

```bash
git clone https://github.com/jonazri/review-pr.git ~/.claude/skills/review-pr
```

To update, run `git pull` in that directory.

## Usage

```
/review-pr          # PR for the current branch
/review-pr 123      # a specific PR
```

The skill also triggers on phrases like "address PR comments", "watch CI", "resume PR review", and
"iterate PR until green".

> **Heads up:** the skill runs without asking between rounds. It commits, pushes, replies to and
> resolves review threads, and merges when the PR is green. Run it only on PRs where you want that.

## License

MIT
