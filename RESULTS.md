# laya-delegate-v12 — benchmark results

SkillsBench via Harbor, Claude Opus 5.5 (subscription billing, 1h prompt cache),
6 tasks × 4 trials per arm. OFF = stock Claude Code; v12 = Claude Code + RTK with
this pipeline. Both arms ran back to back on 2026-10-18, so they share cluster
conditions. Per-trial rows: `results.tsv`.

| task | OFF $/trial | v12 $/trial | change | reward OFF → v12 | sub-agent calls (v12) |
|---|---|---|---|---|---|
| parallel-tfidf-search | 1.260 | 1.007 | −20% | 1.00 → 1.00 | 4 |
| python-scala-translation | 0.949 | 0.885 | −7% | 1.00 → 1.00 | 6 |
| fix-visual-stability | 0.534 | 0.451 | −16% | 0.25 → 0.50 | 3 |
| fix-build-google-auto | 0.315 | 0.329 | +4% | 1.00 → 1.00 | 2 |
| fix-erlang-ssh-cve | 0.188 | 0.199 | +6% | 1.00 → 1.00 | 0 |
| debug-trl-grpo | 0.121 | 0.240 | +98% | 0.60 → 0.60 | 0 |
| **total** | **3.367** | **3.111** | **−7.6%** | **0.81 → 0.85** | 15 |

- Reward is equal or better on every task.
- Savings come from long write/port tasks, where Laya's label lets RTK route the
  opening investigation to a Haiku sub-agent.
- Short bug/CVE tasks are left alone by the routing and pay Laya's overhead.
  debug-trl-grpo's +98% is one trial ($0.44) that distrusted a condensed grep,
  bypassed RTK and read a 1,360-line file in chunks; the other three are near OFF.
- 4 trials per arm: per-task differences under ~20% are within run-to-run noise;
  the total is the number to quote.
