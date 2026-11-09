---
title: "Logging, observability and evaluation"
permalink: /sections/16-logging-and-evaluation/
nav_order: 26
has_children: true
has_toc: false
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Logging, observability and evaluation

Stop 11 on the talk path, about 2 minutes.

A harness is a system you keep changing, so you need two instruments:

| Instrument | Tells you | Without it |
|---|---|---|
| Logs | what the agents and the harness actually did: commands, files, time and money | you optimise the wrong step |
| Evaluations | whether a change made things better: a fixed set of tasks, run before and after, with one thing changed | you cannot tell whether a change helped |

![Sequence diagram. The person proposes a new hook. The eval runner runs a baseline loop, giving each task to the agent with the old harness, grading the result and writing pass, cost and time to the ledger. It then runs the same loop with the new harness. The ledger returns both runs, and the runner reports keep or revert to the person.]({{ '/assets/images/16-logging-and-evaluation/eval-run-sequence.svg' | relative_url }}){: style="width:100%"}

*Testing one harness change. The model, the tasks and the number of runs stay fixed; only the change differs.*

## At a glance

| Idea | In one line | Go deeper |
|---|---|---|
| Activity logs | Record commands, files, time and cost for every agent | [Logging and observability]({{ '/sections/16-logging-and-evaluation/logging-and-observability.html' | relative_url }}) |
| Provenance | Each commit records which model and agent made it | [Commit provenance]({{ '/sections/16-logging-and-evaluation/logging-and-observability.html#commit-provenance' | relative_url }}) |
| Spend | Estimate cost continuously; check the budget before a paid job | [Spend tracking and budget checks]({{ '/sections/16-logging-and-evaluation/spend-and-budgets.html' | relative_url }}) |
| Harness measurement | Find where the time goes before optimising | [Measuring the harness]({{ '/sections/16-logging-and-evaluation/logging-and-observability.html#measuring-the-harness-itself' | relative_url }}) |
| Evaluation | A fixed task set, before and after one change | [Evaluating a harness change]({{ '/sections/16-logging-and-evaluation/evaluating-changes.html' | relative_url }}) |
| pass@k, pass^k | Success in any of k tries, or in all of them | [pass@k and pass^k]({{ '/sections/16-logging-and-evaluation/pass-at-k.html' | relative_url }}) |

### Pages in this section

1. This overview: the whole topic in one page.
2. [Logging and observability]({{ '/sections/16-logging-and-evaluation/logging-and-observability.html' | relative_url }}): what to record, JSONL ledgers, commit provenance, Claude Code telemetry, and measuring the harness.
   - [Spend tracking and budget checks]({{ '/sections/16-logging-and-evaluation/spend-and-budgets.html' | relative_url }}): estimating cost and stopping a job before it runs.
3. [Evaluating a harness change]({{ '/sections/16-logging-and-evaluation/evaluating-changes.html' | relative_url }}): task sets, graders, fair runs and reading the result.
   - [pass@k and pass^k]({{ '/sections/16-logging-and-evaluation/pass-at-k.html' | relative_url }}): the two measures of repeated attempts, with the arithmetic.

## Log what agents do

An agent session does a lot you never see: dozens of commands, file edits, retries, model calls. When something goes wrong, or when a run is slow or expensive, the log is the only record. Log at least:

| Field | Example | Answers |
|---|---|---|
| What ran | `npm test -- settings` | what did it actually do? |
| What changed | `src/settings.tsx`, 42 lines | what should I review? |
| How long | 3 min 10 s | where did the time go? |
| What it cost | tokens in and out, estimated dollars | was it worth it? |
| Who | model, effort level, agent role | which configuration produced this? |

Append-only JSON Lines files, one JSON object per line, are a good default:

- easy to write from any script;
- easy to query with `jq` or Python;
- no migration when a field is added.

```json
{"ts":"2026-11-09T14:02:11Z","agent":"reviewer","model":"opus","event":"tool","tool":"Bash","command":"npm test -- settings","seconds":41,"exit":0}
```

### Build the logging into the system

Asking agents to log their own work is an instruction, applied inconsistently. Two mechanisms record every call without the agent's help:

| Mechanism | What it records | Setup |
|---|---|---|
| A PostToolUse hook | each tool call as it completes, appended to your own JSONL ledger | a hook entry in `.claude/settings.json` |
| OpenTelemetry export | session counts, token use, cost, tool results, API requests | `CLAUDE_CODE_ENABLE_TELEMETRY=1` and an exporter, set in your shell or user settings |

```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "python3 \"$CLAUDE_PROJECT_DIR\"/tools/log_tool_call.py" }] }
    ]
  }
}
```

The hook receives the completed call as JSON on standard input and appends one line to the ledger. [Logging and observability]({{ '/sections/16-logging-and-evaluation/logging-and-observability.html' | relative_url }}) shows the ledger format and the telemetry setup.

![Sequence diagram. In a loop over every tool call, the agent's test run and commit each trigger a logging hook that appends a line to a JSONL ledger with the duration, exit code, model and agent. Later the person queries the ledger for seconds by stage, learns the full test run takes 20 minutes, and asks the agent to shard the full suite.]({{ '/assets/images/16-logging-and-evaluation/logging-sequence.svg' | relative_url }}){: style="width:100%"}

*From one agent action to an answered question: each tool call appends a line, and a later query answers why merging is slow.*

### Example: a question the log answers

1. A person notices that last week's agent runs cost twice the usual.
2. A query over the ledger groups cost by agent role:

```bash
jq -s 'group_by(.agent) | map({agent: .[0].agent, usd: (map(.usd) | add)}) | sort_by(-.usd)' logs/model-calls.jsonl
```

3. The reviewer role tops the list. Its records show the largest model at maximum effort for every review, one-line changes included.
4. The fix goes into the launcher, which now picks the effort level from the size of the diff, so it holds without anyone remembering it.

## Measure the harness itself

The harness has its own bottlenecks, and they are often somewhere unexpected. In a busy setup, most of the time from first commit to merge can be spent waiting for the final full test run. Measure each stage before optimising any of them:

| Stage | Log at start and end | Typical fix when it is the slow one |
|---|---|---|
| Writing the change | agent session start, first commit | smaller tasks, better briefs |
| Focused tests | test command start and exit | run only the tests the change touches |
| Review | review requested, review returned | a cheaper reviewer for small diffs |
| Waiting for a slot | queued, started | more runners, or batching changes |
| Final full test run | start, exit | shard the suite; cache dependencies |
| Merge | merge started, merged | fewer conflicts through smaller changes |

![An illustrative timeline of one change: writing, focused tests, review, a wait for a slot and a long final full test run. The waits and the final run take most of the time. The proportions are an illustration, not a measurement.]({{ '/assets/images/16-logging-and-evaluation/merge-time.svg' | relative_url }}){: style="width:100%"}

*An illustration of where the time to merge can go. Your proportions will differ; measure them.*

## Watch the spend

![Sequence diagram. The agent gives the budget check a 40,000-call batch and a measured sample. The check asks the spend ledger what remains, learns $100 is left, and refuses the job at about $264. The agent proposes a small model and a sample to the person, who approves, and the estimate is recorded in the ledger.]({{ '/assets/images/16-logging-and-evaluation/budget-check-sequence.svg' | relative_url }}){: style="width:100%"}

*A budget check stops a batch before it runs, and the person decides on the cheaper proposal.*

Put a budget check before any job that makes paid model calls, so unplanned spend stops before the bill shows it:

1. The agent describes the job: how many calls, on which model.
2. The check estimates the cost.
   - Tokens per call come from a measured sample, multiplied by the price.
3. It compares the estimate with what is left in the budget, read from the spend ledger.
4. It refuses a job that would exceed it, and names cheaper options.
   - Raising the budget goes to the person, because spending is one of the [human gates]({{ '/sections/14-human-gates/' | relative_url }}).

### Example: a batch that did not run

| Step | Value |
|---|---|
| Job | classify 40,000 documents |
| Measured on a 50-document sample | 900 input and 40 output tokens per call |
| Estimate on the large model, with a 20 percent margin | about $264 (illustrative prices) |
| Remaining monthly budget for jobs | $100 |
| Check result | refused, with cheaper options named |
| Agent's proposal | small model on all documents (about $53), large model on a 500-document sample to compare accuracy |

[Spend tracking and budget checks]({{ '/sections/16-logging-and-evaluation/spend-and-budgets.html' | relative_url }}) works through the arithmetic.

## Evaluate each change

Anthropic's guide to agent evaluation puts it this way: "When we evaluate 'an agent,' we're evaluating the harness and the model working together." Your harness changes are changes to the agent, and they can make it worse as easily as better. The way to know is an evaluation:

1. Build a fixed set of tasks. Anthropic suggests starting with 20 to 50 simple tasks drawn from real failures.
2. Run the baseline: each task several times, with the current harness.
3. Change one thing: a rule, a hook, a skill or the model.
4. Run the same tasks the same number of times.
5. Compare pass rate, cost and time. Keep the change or revert it.

OpenAI recommends the same discipline as eval-driven development, with evals run on every change. [Evaluating a harness change]({{ '/sections/16-logging-and-evaluation/evaluating-changes.html' | relative_url }}) covers task sets, graders and how many runs you need.

### Example: does the new hook help?

A team adds a hook that refuses a commit when the focused tests have not run since the last edit. They run 30 tasks from past failures, three times each, before and after (the numbers are an illustration):

| | Before | After |
|---|---|---|
| Runs passed, of 90 | 58 | 71 |
| Median time per task | 6.1 min | 7.0 min |
| Regression set, 60 runs | 57 | 58 |

Pass rate rose, the regression set held, and the time cost is small, so they keep the hook.

### Capability and regression evals

Keep two task sets and report them separately:

| Set | Holds | Answers |
|---|---|---|
| Capability | tasks the agent fails today | what did the change unlock? |
| Regression | tasks the agent passes today | did the change lose ground? |

A change that solves two new capability tasks and breaks three regression tasks has made the agent worse at work it already did; a single combined score would hide that.

### pass@k and pass^k

Agents are not deterministic, so one run per task tells you little. Two measures from Anthropic's guide describe repeated runs:

| Measure | A task counts as solved when | For an agent that succeeds 70 percent of the time, k = 3 |
|---|---|---|
| pass@k | any of k attempts succeeds | about 0.97 |
| pass^k | all k attempts succeed | about 0.34 |

[pass@k and pass^k]({{ '/sections/16-logging-and-evaluation/pass-at-k.html' | relative_url }}) works through the arithmetic and when to use each.

![Bar chart for a per-attempt success rate of 0.7: pass@k rises to 1.00 by k = 5 while pass^k falls to 0.17.]({{ '/assets/images/16-logging-and-evaluation/pass-at-k.svg' | relative_url }}){: style="width:100%"}

*The same agent looks reliable under pass@k and unreliable under pass^k. Pick the one that matches how you use it.*

## Tools that help

| Tool | What it does |
|---|---|
| `/insights` in Claude Code | writes a report on your recent sessions, friction points included, for improving your setup |
| `claude plugin eval` | compares a plugin's results against a run without it |
| OpenTelemetry export | sends Claude Code metrics and events to your own monitoring |
| A JSONL ledger and `jq` | answers ad hoc questions about your own runs |

## Examples from my own harness

- Each commit records the model, effort level and agent that produced it, so any line of code can be traced to the configuration that wrote it.
- Each subsystem keeps its own append-only JSONL ledger, and spend is estimated from the local session transcripts at list prices, never mistaken for an invoice.
- The test gate's own timing is logged, which is how waiting for the final test run showed up as the largest share of time to merge.

To start: ten tasks taken from your own past failures, run before and after one harness change.

## Pitfalls and lessons

- Without a baseline, you cannot tell whether a harness change helped.
- Slow gates are a cost too. Measure what each gate costs, and give it a budget.
- One run per task measures luck. Run each task several times.
- A log that records only successes hides the failures you need. Log the exit code and the error.
- A log that records full command lines or environment variables can record secrets. Strip tokens before writing, and never log the environment.
- An evaluation with a different model in the two runs measures the model, not your change.

## Sources

- `/insights`: described from the Claude Code command list; not re-checked against the docs for this page.

- Anthropic on agent evals: [Anthropic, Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) (2026-01-09), verified 2026-11-08.
- OpenAI evaluation best practices: [OpenAI, Evaluation best practices](https://developers.openai.com/api/docs/guides/evaluation-best-practices), verified 2026-11-08.
- `claude plugin eval`: [Claude Code docs: Plugin evals](https://code.claude.com/docs/en/plugin-evals.md), verified 2026-11-08.
- Claude Code monitoring and OpenTelemetry: [Claude Code docs: Monitoring](https://code.claude.com/docs/en/monitoring-usage.md), verified 2026-11-09.
- The broken-benchmark figure: [OpenAI, Separating signal from noise in coding evaluations](https://openai.com/index/separating-signal-from-noise-coding-evaluations/); the page returned HTTP 403 to a fetch on 2026-11-09, so the figure is not yet verified.
- Commit provenance, ledgers, spend estimates and gate timing: from the author's own harness, described conceptually.

{% include page_nav.html %}
