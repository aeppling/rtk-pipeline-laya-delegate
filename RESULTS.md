# laya-delegate — benchmark results

## Best gains

- **−30% cost on parallel-tfidf-search** ($1.345 → $0.946 per trial). It delegated in every trial and passed every trial.
- **−19.5% on the whole suite on 2026-10-19** ($3.803 → $3.063 for the 6 tasks).
- **−17% on fix-visual-stability and −13% on python-scala-translation.**
- **−13.9% across all 48 trials, with reward unchanged (0.81 → 0.81).**
- **28 hand-offs** to the Haiku sub-agent, in half of all trials, decided by Laya and enforced by RTK, with no task-specific rules.

**Setup:**
- Benchmark: SkillsBench via Harbor, Claude Opus 5.5 (subscription billing, 1h prompt cache).
- Size: 6 tasks × 4 trials per arm, run on two days, so 48 trials per arm.
- Arms: OFF is stock Claude Code; laya-delegate is Claude Code + RTK with this pipeline.
- Both arms of each day ran in the same batch.
- Per-trial rows are in `results.tsv`.

## Cost and reward

Mean cost per trial (each day's mean, averaged over the two days):

| task | OFF | laya-delegate | change | reward OFF → laya-delegate |
|---|---|---|---|---|
| parallel-tfidf-search | $1.345 | $0.946 | −30% | 1.00 → 1.00 |
| python-scala-translation | $1.002 | $0.867 | −13% | 1.00 → 1.00 |
| fix-visual-stability | $0.538 | $0.448 | −17% | 0.25 → 0.25 |
| fix-build-google-auto | $0.319 | $0.303 | −5% | 1.00 → 1.00 |
| fix-erlang-ssh-cve | $0.198 | $0.191 | −4% | 1.00 → 1.00 |
| debug-trl-grpo | $0.185 | $0.333 | +80% | 0.60 → 0.60 |
| **total** | **$3.587** | **$3.088** | **−13.9%** | **0.81 → 0.81** |

Per day:

| day | OFF | laya-delegate | change |
|---|---|---|---|
| 2026-10-18 | $3.367 | $3.111 | −7.6% |
| 2026-10-19 | $3.803 | $3.063 | −19.5% |

## What the pipeline triggered (48 trials)

| action | count |
|---|---|
| RTK refusals pointing to the sub-agent | 29 |
| Haiku Explore sub-agent hand-offs | 28 (in 24 trials) |
| Whole-file reads answered by a Haiku summary | 0 (needs an API key; the runs used a subscription login) |
| Tokens removed by RTK output filtering | 77.5K of 554K (14%) |

The hand-offs happen on the write and port tasks:
- **tfidf and python-scala:** delegated in every trial.
- **google-auto:** delegated in about half the trials.

These are also the tasks with the clear savings.

## Reading the numbers

- **4 trials per arm per day:** a per-task difference under about 20% is within run-to-run noise. The two-day total is the number to quote.
- **The trl outlier is the agent's path, not the pipeline.** debug-trl-grpo is a bug task, so the pipeline routes nothing there, and the extra cost comes from one or two trials per day that read the source by hand instead of diffing it against the original package. Some of those trials also re-ran commands with `rtk bypass` after a condensed `sed`/`grep` result.
- **visual-stability passes rarely in every arm** (1 in 4 for OFF on both days), so its reward column is noise at this sample size.

## Why this rule set (and not the variant without the read refusal)

A same-day run on 2026-10-19 also tested a lighter variant. It refused only shell commands, not file reads, and fired only at call 2.

Over the same 6 tasks:
- The lighter variant refused 10 times, but the agent actually handed off only 3 times. Most refusals were sidestepped by opening the file directly.
- With the read refusal, refusals turned into hand-offs: 11 refusals, 13 sub-agent calls, in 11 trials.

Measured on the tasks where the two actually behave differently, `laya-delegate` was cheaper:

| task | lighter variant | laya-delegate |
|---|---|---|
| python-scala-translation | $0.960 | $0.848 |
| fix-build-google-auto | $0.293 | $0.276 |
