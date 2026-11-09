---
title: "Logging and observability"
parent: "Logging, observability and evaluation"
nav_order: 1
has_children: true
has_toc: false
redirect_from:
  - /sections/16-logging-and-evaluation/detail.html
---

{% include page_nav.html %}

# Logging and observability

As of 2026-11-09.

Part of [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }}). Sub-page: [Spend tracking and budget checks]({{ '/sections/16-logging-and-evaluation/spend-and-budgets.html' | relative_url }}).

Observability is being able to answer questions about a system from what it records, without adding new code each time you ask. For a harness the questions are practical: what did this agent do, why did that run take an hour, which configuration wrote this line, and where did the money go. This page covers what to record, how to store it, how to get Claude Code's own telemetry, and how to measure the harness itself.

## Contents

- [What to record](#what-to-record)
- [JSON Lines ledgers](#json-lines-ledgers)
- [Commit provenance](#commit-provenance)
- [Claude Code telemetry](#claude-code-telemetry)
- [Measuring the harness itself](#measuring-the-harness-itself)
- [Session reports](#session-reports)
- [Examples from my own harness](#examples-from-my-own-harness)
- [Pitfalls](#pitfalls)

## What to record

![Sequence diagram. In a loop over every tool call, the agent's test run and commit each trigger a logging hook that appends a line to a JSONL ledger with the duration, exit code, model and agent. Later the person queries the ledger for seconds by stage, learns the full test run takes 20 minutes, and asks the agent to shard the full suite.]({{ '/assets/images/16-logging-and-evaluation/logging-sequence.svg' | relative_url }}){: style="width:100%"}

*From one agent action to an answered question: each tool call appends a line, and a later query answers why merging is slow.*

![Agent sessions, commits and model calls write to append-only JSONL ledgers; the ledgers answer questions about time and cost, and feed a budget check before any paid job.]({{ '/assets/images/16-logging-and-evaluation/ledger-flow.svg' | relative_url }}){: style="width:100%"}

*Three sources, one store, two uses.*

### Record structured events

Record each thing that happened as an event with fixed fields:

| Free text | Structured event |
|---|---|
| "Ran the tests, they passed" | `{"event":"test","exit":0,"seconds":41}` |
| "Committed the fix" | `{"event":"commit","model":"opus","agent":"engineer"}` |
| "The gate was slow today" | `{"event":"gate","stage":"full-tests","seconds":1260}` |

The right-hand column can be counted, summed and grouped; the left cannot.

### The minimum event

| Field | Type | Why |
|---|---|---|
| `ts` | ISO 8601 UTC time | ordering and durations |
| `session` | string | groups events from one run |
| `agent` | string | which role or profile |
| `model`, `effort` | string | which configuration |
| `event` | string | `tool`, `commit`, `gate`, `model_call`, `error` |
| `seconds` | number | duration |
| `exit` | number | success or failure |
| detail fields | varies | the command, the file, the token counts |

### Record failures with the same care

A log of successes says nothing about what went wrong. For a failure, record:

- the exit code;
- the first lines of the error, with secrets and tokens removed;
- the stage it failed in and how long it ran first;
- whether a retry followed, and its result.

## JSON Lines ledgers

### Why JSONL

A JSON Lines file holds one JSON object per line.

| Property | Why it helps a harness |
|---|---|
| Writing is an append | concurrent writers rarely conflict; a crash loses at most one line |
| No database | any script can write it; `jq` or Python can read it |
| No schema migration | a new field appears on new lines; old lines lack it |
| Plain text | it diffs, greps and compresses well |

### Writing

```python
import json
import time
from pathlib import Path

LEDGER = Path("logs/gate.jsonl")

def record(**fields):
    fields.setdefault("ts", time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()))
    LEDGER.parent.mkdir(parents=True, exist_ok=True)
    with LEDGER.open("a") as f:
        f.write(json.dumps(fields, sort_keys=True) + "\n")

record(event="gate", stage="full-tests", seconds=1260, exit=0)
```

### Querying

```bash
# total seconds per gate stage over the last week's runs
jq -s 'group_by(.stage) | map({stage: .[0].stage, seconds: (map(.seconds) | add)})' logs/gate.jsonl
```

### One ledger per subsystem

Keep a ledger for each part of the harness rather than one file for everything:

```text
logs/
  gate.jsonl          test gate stages and durations
  launcher.jsonl      agent launches: role, model, effort, outcome
  merge.jsonl         merges: PR, wait time, result
  model-calls.jsonl   tokens and estimated cost per call
```

- Each file stays small and has a clear owner.
- Each subsystem can change its own fields without affecting the others.

## Commit provenance

Commit provenance records, in the commit itself, which model, effort level and agent produced the change. Git trailers are the natural place: key-value lines at the end of the commit message that tools can parse.

```text
fix(export): stream large files instead of loading them whole

The export endpoint read each file into memory, which failed above 2 GB.

Model: opus
Effort: high
Agent: engineer
Co-Authored-By: Claude <noreply@anthropic.com>
```

```bash
# which model wrote the commits on this branch?
git log --format='%(trailers:key=Model,valueonly)' main..HEAD | sort | uniq -c
```

#### What it is good for

When a bug turns up, provenance tells you which configuration wrote the code. Over time it shows which models and roles produce changes that survive review, and which produce changes that get reverted.

## Claude Code telemetry

Claude Code can export metrics and events over OpenTelemetry to a monitoring backend you run. From the documentation:

```bash
export CLAUDE_CODE_ENABLE_TELEMETRY=1
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

### What comes out

| Kind | Examples |
|---|---|
| Metrics | `claude_code.session.count`, `claude_code.token.usage`, `claude_code.cost.usage`, `claude_code.lines_of_code.count`, `claude_code.commit.count` |
| Events | `claude_code.user_prompt`, `claude_code.tool_result`, `claude_code.api_request`, `claude_code.api_error` |

To check the setup, start a session and look for `claude_code.session.count` in your backend.

> These exporter variables are ignored in a repository's `.claude/settings.json`. Set them in your shell, in managed settings or in your user settings, so a cloned repository cannot redirect your telemetry.

## Measuring the harness itself

### Time every stage

A change passes through stages: writing, focused tests, review, waiting, the final full test run, merge. Log the start and end of each. Then the question "why is merging slow" has an answer instead of a guess.

![An illustrative timeline: writing, focused tests, review, a wait for a slot and a long final full test run. The waits and the final run take most of the time.]({{ '/assets/images/16-logging-and-evaluation/merge-time.svg' | relative_url }}){: style="width:100%"}

*An illustration, not a measurement. In a busy setup the final test run and the waits around it can dominate.*

### Worked example: finding the slow stage

#### The question

Merges feel slow; the team suspects code review.

#### The query

```bash
jq -s 'group_by(.stage) | map({stage: .[0].stage, median: (map(.seconds) | sort | .[length/2|floor])})' logs/gate.jsonl
```

#### The answer

Review has a median of four minutes. The final full test run has a median of twenty, and the queue before it another fifteen. The fix is to run the full suite on fewer, larger batches and to shard it.

### Budget each gate

Every gate costs time on every run. For each one, track:

| Measure | From the ledger | Action when it is bad |
|---|---|---|
| Duration | median and slowest run | shard it, cache it, or run it less often |
| Over budget | runs above its time limit | investigate the slow runs |
| False refusals | refusals of correct work | narrow the rule or fix the check |
| Catches | real problems it stopped | a slow gate that rarely catches anything is a candidate for removal |

## Session reports

`/insights` in Claude Code writes a report on your recent sessions, friction points included. Use it in a loop:

1. Run `/insights` at the end of a week.
2. Pick the friction point that repeats most.
3. Fix it in the system: a workflow step, a tool, a hook or a test.
4. Check the next report to see whether it went away.

## Examples from my own harness

- Each commit carries trailers for model, effort and agent, and a commit gate refuses a commit whose provenance is missing.
- Each subsystem keeps an append-only JSONL ledger: the test gate, the launcher, the reaper of old worktrees, the merge step.
- The test gate logs each stage's duration, which is how waiting for the final test run showed up as the largest share of time to merge.

To start: add `Model:` and `Agent:` trailers to agent commits, and one JSONL ledger for your slowest step.

## Pitfalls

- A log that prints secrets is a leak with a timestamp. Never record environment variables, tokens or full command lines that may carry them.
- Free-text logs cannot be counted. Use fields.
- A ledger nobody queries is storage. Write the first query when you create the ledger.

## Sources

- Claude Code monitoring and OpenTelemetry: <https://code.claude.com/docs/en/monitoring-usage.md>, verified 2026-11-09.
- Commit trailers, ledgers and gate timing: from the author's own harness, described conceptually.

{% include page_nav.html %}
