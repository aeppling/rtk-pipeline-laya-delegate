# rtk pipeline · laya-delegate

Numbers on this card are self-reported by the author's own rtk. rtk verifies nothing.

## Install

```
rtk pipeline add github:aeppling/rtk-pipeline-laya-delegate
```

rtk shows this pipeline's rules and asks before anything installs, and pins the install to a commit. A pipeline can rewrite the commands your agent runs, filter and replace what it reads, and send content to any upstream it declares — review the rules and install only what you trust. RTK does not review or endorse shared pipelines; the author is responsible for this repo, and you are responsible for what you install.

## What it does

- **before read** — delegate -> bulk-read when file_lines_gt=350, score.need_lte=0.65, whole_file=true
- **before read** — delegate -> bulk-read when class.kind_in=["write", "data"], recent_calls_matches='^(Bash|Read|Grep|Glob) [^|]*( \| (Bash|Read|Grep|Glob) [^|]*)?$'
- **before shell** — delegate -> investigate when class.kind_in=["write", "data"], command_matches='^(cd [^;&]+(;|&&) *)?(ls|cat|head|tail|grep|rg|find|tree|pwd|echo|which|wc|file|stat|du|for|while|awk|nl|xxd|git|sed -n)\b', recent_calls_matches='^(Bash|Read|Grep|Glob) [^|]*( \| (Bash|Read|Grep|Glob) [^|]*)?$'
- **before shell** — rewrite -> rtk
- **after read** — compress [lockfile] when basename_in=["package-lock.json", "Cargo.lock", "yarn.lock", "pnpm-lock.yaml"]
- **after shell** — compress [term] when is_rtk_command=false
- **after search** — compress [matches] when format="matches"
- **after web** — compress [web]
- **proxy tool result** — compress [ansi, progress, dedup]

_No recorded activity yet — run some sessions with this pipeline active, then re-run `rtk pipeline share`._

## user pipeline.toml

```toml
# laya-delegate — RTK filtering + Laya-routed delegation to a Haiku sub-agent.
#
# PREREQUISITE: Laya running locally. This pipeline's routing decisions come
# from Laya (@receptron/laya, a small local classifier; Node 20+, ~1.7 GB model
# download on first start), served at http://127.0.0.1:41733/v1/systemone
# (rtk's classifier wire format), with RTK_LAYA_TOKEN exported. Without it,
# every Laya-gated rule stays silent and only RTK's output filtering applies.
#
# What it does
# - Laya (a small local classifier) labels the task: bug / setup / write / data.
# - On write and data tasks, RTK refuses the agent's first looking command or
#   file read (call 2 or 3) and asks it to hand the investigation to the Explore
#   sub-agent on Haiku; the agent then works from the sub-agent's report.
# - Bug and setup tasks are left alone (the agent needs the exact details there).
# - Whole-file reads over 350 lines that need little exact text are answered by
#   a Haiku summary.
# - Everything else is standard RTK output filtering (rtk:posthook).
# Nothing here keys on a task, a language or a file name.
#
# Results — SkillsBench, Claude Opus 5.5, 6 tasks x 4 trials per arm, OFF and
# v12 run back to back on the same day (2026-10-18). Per-trial data in
# results.tsv, summary in RESULTS.md.
#
#   task                       OFF $/trial   v12 $/trial   reward OFF -> v12
#   parallel-tfidf-search          1.260         1.007         1.00 -> 1.00
#   python-scala-translation       0.949         0.885         1.00 -> 1.00
#   fix-visual-stability           0.534         0.451         0.25 -> 0.50
#   fix-build-google-auto          0.315         0.329         1.00 -> 1.00
#   fix-erlang-ssh-cve             0.188         0.199         1.00 -> 1.00
#   debug-trl-grpo                 0.121         0.240         0.60 -> 0.60
#   total                          3.367         3.111 (-7.6%) 0.81 -> 0.85
#
# Savings come from long write/port tasks; short bug/CVE tasks pay a small
# overhead (+4-6%; trl's +98% is one trial that bypassed RTK's output filter).
# 4 trials per arm: per-task differences under ~20% are within run-to-run noise.
#
# Also
# - upstream.haiku calls the Anthropic API with your key. Subscription (OAuth)
#   logins cannot call it directly; the sub-agent hand-off still works, since
#   it runs inside Claude Code.
schema_version = 1

[pipeline]
name    = "laya-delegate"
extends = ["rtk:posthook", "rtk:delegate-reads"]

[upstream.judge]
provider     = "laya"
model        = "laya"
base_url     = "http://127.0.0.1:41733/v1/systemone"
api_key_env  = "RTK_LAYA_TOKEN"
timeout_secs = 60

[upstream.haiku]
provider     = "anthropic"
model        = "claude-haiku-5-5"
timeout_secs = 90

[classifier.need]
upstream     = "judge"
question     = "score"
instructions = "How much of this file's exact text does the agent need for its next step?"
rubric       = ["none: a short summary is enough", "a few exact lines", "most of the file verbatim"]
input_chars  = 2000

[classifier.kind]
upstream     = "judge"
question     = "choice"
instructions = "What kind of work is this coding task?"
input_chars  = 2000
# Laya's choice "confidence" sits well below its top probability (write: p 0.84,
# confidence 0.56); the engine default floor (0.8) dropped every verdict, so the
# rule never fired. The calibration used the label as is.
min_confidence = 0.0

[classifier.kind.labels]
bug   = "find and fix a bug by reading source code closely"
setup = "understand or repair a build, configuration, dependency or environment"
write = "write new code, a script or a port"
data  = "analyse data or files and report results"

# Named by the delegate-investigation route. In suggest mode the call is
# refused with the rule's message (pointing to the Explore sub-agent); the
# task only runs if the agent invokes it itself.
[task.investigate]
kind         = "digest"
upstream     = "haiku"
temperature  = 0
context      = true

[task.bulk-read]
kind         = "summarize"
upstream     = "haiku"
temperature  = 0
context      = true
instructions = "You read a file for a coding agent that will NOT see it, only your reply. First answer what the agent needs for its current step, in a few terse lines, with exact identifiers. Then map the file: one line on what it is; every class, function and method with its exact signature, in file order. Then quote verbatim, grouped under the name of the function that contains them, the lines whose exact text the agent would most need for its step (non-obvious arithmetic, constants, conditions, error handling, config values). Never give line numbers. Never paraphrase code you quote. No advice, no headings. End with exactly: (Exact text: grep -n a quoted line, then Read that file with offset and limit, at most 350 lines.)"

# The answer below replaces the refusal for the same reads.
[override.shunt-large-read]
enabled = false

[[hook.before_tool.read]]
name   = "answer-big-reads"
when   = { file_lines_gt = 350, whole_file = true, "score.need_lte" = 0.65 }
action = "route"
task   = "bulk-read"
mode   = "answer"


[[directive]]
name = "explore-in-subagent"
text = """
When RTK asks you to hand an investigation to the Explore sub-agent, call it on the haiku model (Agent tool: subagent_type "Explore", model "haiku") with one precise question built from the task, ask it for file paths, line numbers and the exact lines you will need, and work from its report instead of repeating its searches.
"""

# The session's second or third tool call (one or two earlier calls, none a
# sub-agent call), and only a looking command (list, print, search, history).
# recent_calls is absent before the first call, so a condition on it cannot
# match there. A refused call is recorded, so the rule can refuse at call 2 and
# again at call 3: at most two refusals per session.
[[hook.before_tool.shell]]
name    = "delegate-investigation"
when    = { command_matches = "^(cd [^;&]+(;|&&) *)?(ls|cat|head|tail|grep|rg|find|tree|pwd|echo|which|wc|file|stat|du|for|while|awk|nl|xxd|git|sed -n)\\b", recent_calls_matches = "^(Bash|Read|Grep|Glob) [^|]*( \\| (Bash|Read|Grep|Glob) [^|]*)?$", "class.kind_in" = ["write", "data"] }
action  = "route"
task    = "investigate"
mode    = "suggest"
message = "Do not run this search or open these files yourself: hand the investigation to the Explore sub-agent (Agent tool: subagent_type \"Explore\", model \"haiku\") with one precise question, and ask it for file paths, line numbers and the exact lines you will need. Then work from its report."

# The same decision when the agent opens a file instead of running a command.
[[hook.before_tool.read]]
name    = "delegate-investigation-read"
when    = { recent_calls_matches = "^(Bash|Read|Grep|Glob) [^|]*( \\| (Bash|Read|Grep|Glob) [^|]*)?$", "class.kind_in" = ["write", "data"] }
action  = "route"
task    = "bulk-read"
mode    = "suggest"
message = "Do not run this search or open these files yourself: hand the investigation to the Explore sub-agent (Agent tool: subagent_type \"Explore\", model \"haiku\") with one precise question, and ask it for file paths, line numbers and the exact lines you will need. Then work from its report."
```

Install this pipeline: `rtk pipeline add <url-of-this-card>`
