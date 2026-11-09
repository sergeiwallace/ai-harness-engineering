---
title: "Spend tracking and budget checks"
parent: "Logging and observability"
grand_parent: "Logging, observability and evaluation"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Spend tracking and budget checks

Part of [Logging and observability]({{ '/sections/16-logging-and-evaluation/logging-and-observability.html' | relative_url }}), in [Logging, observability and evaluation]({{ '/sections/16-logging-and-evaluation/' | relative_url }}).

Model calls cost money per token, and agents make a lot of them. Most of the time the spend is small and steady. The surprises come from jobs: a batch over thousands of documents, an evaluation run with many repeats, a loop that does not stop. Spend tracking tells you where the money went. A budget check stops the surprise before it starts.

## Contents

- [Two kinds of spend](#two-kinds-of-spend)
- [Estimating spend](#estimating-spend)
- [The budget check](#the-budget-check)
  - [Making the check unavoidable](#making-the-check-unavoidable)
  - [Interactive spend](#interactive-spend)
- [Worked example](#worked-example)
- [Examples from my own harness](#examples-from-my-own-harness)
- [Pitfalls](#pitfalls)

## Two kinds of spend

| Kind | Example | Control |
|---|---|---|
| Interactive | a person and an agent working in a session | a report after the fact, by session and repository |
| Jobs | a batch classification, an eval run, a pipeline | a check before the job starts |

Interactive spend is hard to stop without stopping the work, so watch it. Job spend is easy to estimate in advance, so gate it.

## Estimating spend

### From your own records

Each model call reports its input and output tokens. Multiply by the model's list price and sum. An estimate built from local session transcripts at list price is useful for trends and comparisons; label it as an estimate, because it will differ from an invoice (discounts, caching, rounding).

```python
PRICE_PER_MTOK = {"large": {"in": 5.00, "out": 25.00}, "small": {"in": 1.00, "out": 5.00}}  # illustrative

def call_cost(model, tokens_in, tokens_out):
    p = PRICE_PER_MTOK[model]
    return tokens_in / 1e6 * p["in"] + tokens_out / 1e6 * p["out"]
```

The prices here are placeholders. Read the current ones from your provider's pricing page.

### From telemetry

Claude Code's OpenTelemetry export includes cost directly:

- `claude_code.cost.usage`: cost of the session in US dollars.
- `claude_code.token.usage`: tokens used.

See [Logging and observability]({{ '/sections/16-logging-and-evaluation/logging-and-observability.html#claude-code-telemetry' | relative_url }}) for the setup.

## The budget check

![Sequence diagram. The agent gives the budget check a 40,000-call batch and a measured sample. The check asks the spend ledger what remains, learns $100 is left, and refuses the job at about $264. The agent proposes a small model and a sample to the person, who approves, and the estimate is recorded in the ledger.]({{ '/assets/images/16-logging-and-evaluation/budget-check-sequence.svg' | relative_url }}){: style="width:100%"}

*A budget check stops a batch before it runs, and the person decides on the cheaper proposal.*

A budget check is a short program that runs before a paid job and refuses to start it if the estimate exceeds what is left.

### The steps

1. Count the calls the job will make (documents times prompts times repeats).
2. Measure tokens per call on a small sample, not from a guess.
3. Multiply by the price and add a margin for retries.
4. Compare with the remaining budget for the period.
5. Refuse with a clear message, or proceed and record the estimate in the ledger.

### A sketch

```python
import sys

def preflight(calls, tokens_in, tokens_out, model, remaining, margin=1.2):
    estimate = calls * call_cost(model, tokens_in, tokens_out) * margin
    if estimate > remaining:
        sys.exit(f"refused: estimated ${estimate:,.0f} exceeds remaining ${remaining:,.0f}. "
                 "Try a smaller model, a sample, or ask the person to raise the budget.")
    return estimate
```

The refusal message names the alternatives, so the agent that hits it knows what to propose. Spending is one of the actions a person decides ([Human gates, decisions and alignment]({{ '/sections/14-human-gates/' | relative_url }})), so raising the budget goes to them.

`remaining` comes from the spend ledger, not from memory:

```python
import json

def remaining(budget, ledger="logs/model-calls.jsonl", month="2026-11"):
    records = (json.loads(line) for line in open(ledger) if line.strip())
    spent = sum(r["usd"] for r in records if r["ts"].startswith(month))
    return budget - spent
```

> A check before each job catches large spends with one decision. A check on every call slows interactive work and still misses a loop of many small calls.

### Making the check unavoidable

A preflight the agent is asked to run is an instruction. Build it into the system:

1. The batch runner calls `preflight()` itself and will not start without a passing estimate.
2. A hook or deny rule refuses direct calls to the paid API's script outside the runner.
3. The estimate and the actual cost both go to the ledger, so a run that costs far more than its estimate shows up in the next query.

### Interactive spend

Interactive sessions are hard to gate without stopping the work, so watch them instead:

| Signal | Source | Action |
|---|---|---|
| Session cost far above its usual range | `claude_code.cost.usage` per session from telemetry, or the ledger | look for a loop: the same command or file read again and again |
| One role costs much more than others | the ledger grouped by agent | check its model and effort settings |
| Cost rises after a harness change | the ledger before and after | evaluate the change ([Evaluating a harness change]({{ '/sections/16-logging-and-evaluation/evaluating-changes.html' | relative_url }})) |

## Worked example

| Step | Value |
|---|---|
| Job | classify 40,000 support tickets into ten categories |
| Sample | 50 tickets: 900 input and 40 output tokens per call on the large model |
| Tokens | about 36 million input and 1.6 million output |
| Large model | about $180 + $40 = $220; $264 with a 20 percent margin (illustrative prices) |
| Small model | about $53 with margin |
| Remaining budget for jobs this month | $100 |

The check refuses the large-model run, and the agent proposes:

1. Run the small model on all 40,000 tickets (about $53).
2. Run the large model on a 1,000-ticket sample (about $7) to measure the accuracy difference.
3. Ask the person whether the difference justifies a budget increase.

The person approves steps 1 and 2.

## Examples from my own harness

- Spend is estimated from local session transcripts at list price, reported per session, repository and account, and always labelled an estimate rather than an invoice.
- Before any pipeline or training run, a preflight states the expected cost, and rented compute is always shut down at the end and verified by its state.

To start: before your next batch job, run it on 50 items and multiply. Then put that arithmetic into the runner, so every later job estimates itself before it starts.

## Pitfalls

- An estimate from a guessed token count can be off by several times. Measure on a sample of real inputs.
- A list-price estimate is not an invoice. Batch discounts and prompt caching can lower the real cost, and retries raise it; label the number as an estimate.
- A budget with no owner never gets raised or lowered on purpose. Name who decides.
- A job that retries on every error can spend its whole budget on one failing input. Cap retries per item.

## Sources

- Claude Code cost metric: <https://code.claude.com/docs/en/monitoring-usage.md>, verified 2026-11-09.
- The prices in the snippets are placeholders, not current list prices.
- Spend estimates and preflight checks: from the author's own harness, described conceptually.

{% include page_nav.html %}
