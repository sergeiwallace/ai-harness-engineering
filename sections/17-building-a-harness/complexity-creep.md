---
title: "Complexity creep and pruning"
parent: "Building your own harness"
nav_order: 3
last_updated: "2026-11-09"
talk_stop: "12"
stop_minutes: "5"
---

{% include page_nav.html %}

# Complexity creep and pruning

Every fix in the [improvement loop]({{ '/sections/17-building-a-harness/reflexivity-loop.html' | relative_url }}) adds a mechanism: a line in a context file, a hook, a script, a gate. Each one is reasonable on its own. Together they can grow into a patchwork that nobody can hold in their head, where a change in one place breaks another, and a growing share of the agent's context goes to rules. This page is about noticing that and pruning on the same loop.

## How complexity creeps in

### The usual paths

| Path | What it looks like | Why it happens |
|---|---|---|
| Ad hoc rules | A new, detailed context-file rule after each mistake | Writing a line is faster than building a check |
| Overlap | Two rules, or a rule and a hook, cover the same failure | Each was added for a different incident, months apart |
| Out-of-date rules | A rule compensates for a weakness the current model no longer has | Nobody re-tested after the model changed |
| Over-broad guards | A hook fires on correct work and the agent learns to work around it | The first version matched too much |
| Tangled layers | Reusable parts and personal settings live in the same file | It was quicker to add a line where you already were |

### What it costs

| Cost | How it shows up | How to measure it |
|---|---|---|
| Context | Every always-loaded line takes room the task could use, and each added instruction lowers the odds that the others are followed | Count the always-loaded lines and their tokens; see the follow-rate figures in [Evidence for enforcing over instructing]({{ '/sections/17-building-a-harness/self-improving-harnesses.html' | relative_url }}#evidence-for-enforcing-over-instructing) |
| Time | Every hook runs on every matching event, so a slow one delays every tool call of that kind | Log each hook's duration per run |
| Understanding | With many overlapping mechanisms, the reason for a refusal is hard to find, and fixes start to fight each other | Ask, for each refusal in a week of transcripts, whether its reason was clear from the message alone |
| False blocks | A guard that fires on correct work teaches the agent to work around it | Sample refusals and check whether the refused action was correct |

> Warning: put removal on a schedule, the same as fixes, or the harness only grows.

## Ad hoc rules after every mistake

The most common path into complexity creep is answering each mistake with another context-file rule. It feels like a fix, because the rule describes exactly what went wrong. Over time it fails in a predictable way.

1. The agent makes a mistake.
2. You add a detailed rule to `CLAUDE.md` that names the mistake.
3. The next sessions follow it, while the context is short and the task is simple.
4. Weeks later, a long session with several tasks in flight breaks the rule again.
   - The rule is one of dozens, and the model applies instructions less consistently as it juggles more and its context window fills.
5. You add a second, more emphatic rule, and the list keeps growing.

### The fix: put it in the system

| Instead of this context-file rule | Build this | Layer |
|---|---|---|
| "Never edit files under generated/" | A PreToolUse hook that refuses edits to `generated/` and names the schema file | 3 |
| "Always run the tests before you finish" | A Stop hook that checks the last passing test run is newer than the last edit | 3 |
| "Use the status note format: answer, blockers, next" | A template the status skill fills in, plus a check for the three headings | 2 and 4 |
| "Do not commit to main" | A permission rule or a branch protection rule | 3 |
| "Remember to update the changelog" | A merge gate that fails when code changed and the changelog did not | 4 |

### Deciding what to do with a correction

When a correction arrives, check it against what the harness already has before adding anything. The TRACE study uses five outcomes for this, which work as a checklist:

| Outcome | When it applies | What changes |
|---|---|---|
| Noop | An existing rule or check already covers it | Attach the new case as evidence, for example as a test |
| Update | An existing rule covers it but is too narrow or vague | Revise that rule or widen that check |
| Supersede | The correction conflicts with an older rule | Retire the older rule |
| Split | The correction bundles several points | Handle each point on its own |
| New | Nothing covers it | Add one mechanism, at the layer that can enforce it |

Noop, Update and Supersede add nothing new, and Split and New add only what the correction needs. Running every correction through these five keeps the rule set from growing by one rule per mistake.

### The context-file line

Keep a context-file line only as the second layer: one short line that tells the agent the guard exists, so it is not surprised when the guard refuses.

> Note: a rule that no program can check, such as a point of taste, does belong in a context file. Keep those few and short, and review them when the model changes.

## Pruning on the same loop

Pruning uses the same steps as the improvement loop, pointed at the harness itself.

![Sequence diagram of pruning two guards that cover the same failure. The agent reads a month of guard fires and blocks, finds that Hook B blocked correct work and that Rule A covers the same failure, proposes merging both into one merge gate, and after your approval runs the task set before and after; with the same pass rate and fewer false blocks, it removes Rule A and replaces Hook B.]({{ '/assets/images/17-building-a-harness/pruning-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. Pruning two overlapping guards with a before-and-after run. The numbers in the diagram are illustrative. Diagram made with archify.*

### 1. Inventory

List every mechanism and the failure it exists to prevent:

- Always-loaded context lines, one row per rule.
- Skills and templates.
- Hooks, with their event and matcher.
- Scripts and linters.
- Tests and merge gates.

```text
mechanism                          layer  prevents
CLAUDE.md: "NEVER hand-edit generated/, ALWAYS regenerate"   1   hand edits to generated files
PreToolUse hook: refuse-generated-edits                       3   hand edits to generated files   <- same failure
CLAUDE.md: "tests must pass after your last edit (Stop hook)"  1   pointer to the Stop hook
Stop hook: require-fresh-tests                                3   done with a failing test
pre-commit: no-todo-check                                     3   TODO comments left in code      <- check its false blocks
```

Read the inventory for three patterns:

- Two entries for the same failure.
  - Keep the stronger one. If a context-file line restates a guard in detail, shorten it to a pointer, as in the third row.
- An entry with no answer in the last column.
  - A removal candidate.
- A guard with a history of false blocks.
  - Measure it next.

### 2. Measure

For each row of the inventory, record:

1. How often it fired in the last month.
2. How many of those fires were on correct work.
3. Its run time and the context it adds.

The questions and a worked table are in [Measuring a guard's cost](#measuring-a-guards-cost) below.

### 3. Decide

| Finding | Decision |
|---|---|
| Fires often, blocks only bad work, cheap | Keep |
| Covers the same failure as another mechanism | Merge into the stronger one |
| Fires on correct work | Narrow its match, or move it later (from a hook to a merge gate) |
| Never fires, and the failure no longer happens | Retire it, then re-test |
| Context-file line that restates a guard in detail | Shorten it to a one-line pointer to the guard |
| Written rule that agents break | Move it down the [enforcement ladder]({{ '/sections/17-building-a-harness/' | relative_url }}#3-prefer-a-fix-that-a-program-enforces) |

### 4. Re-test

1. Run your [task set]({{ '/sections/18-trends/building-a-task-set.html' | relative_url }}) with the current harness, several times.
2. Make the one pruning change.
3. Run the same tasks again, the same number of times.
4. Keep the change if the pass rate holds and the false blocks fall.
   - A rule that "never fires" may be preventing the failure by its presence; the re-test is how you find out.

## Measuring a guard's cost

### The five questions

1. How often does it fire?
2. How often does it fire on work that was correct (a false block)?
3. How long does it take to run?
4. How much context does it add, counting both its always-loaded text and its refusal messages?
5. When it refuses, does the next attempt succeed?

### A worked example

An illustrative month of logs for three guards:

| Guard | Fires | False blocks | Time per run | Next attempt succeeds | Verdict |
|---|---|---|---|---|---|
| Pre-push secret scan | 40 | 0 | 1 s | Yes | Keep |
| "Run the tests" line in context | n/a | n/a | n/a | Agents still skip tests | Replace with a Stop hook |
| Pre-commit "no TODO" check | 25 | 9 | 2 s | Agents delete the comment and keep the problem | Retire; the review covers it |

The third row shows a guard with a net cost: it fires on correct work, and when it fires, agents delete the comment and leave the problem in place.

<details markdown="1">
<summary>How to get the numbers</summary>

1. Every hook appends one line per run to a log.
   - The time, the hook name, the decision and the duration.
2. An agent samples refusals from session transcripts.
   - For each one, it records whether the refused action was correct.
3. The agent counts fires, false blocks and run time per guard, and writes the table above.

```text
2026-11-03T14:02:11Z  pre-push-secret-scan  allow   0.9s
2026-11-03T14:20:45Z  no-todo-check         block   1.8s  file=src/parse.py
2026-11-03T14:21:02Z  no-todo-check         allow   1.7s
```

</details>

## Design habits that slow the creep

### Prefer one strong check over many weak ones

A single gate that checks the result often replaces several instructions that try to prevent the problem.

| Several weak mechanisms | One strong one |
|---|---|
| Three context-file lines about test coverage, naming and edge cases | A merge gate that runs the test suite and a coverage threshold |
| A rule to format code and a rule to sort imports | A PostToolUse hook that runs the formatter and the import sorter in one step |
| Rules for each section of a status note | A template plus a check for its headings |

### Keep reusable parts apart from personal settings

- Reusable parts get their own files and tests.
  - A skill, a hook script, a template, a linter rule.
- Personal preferences go in a separate layer.
  - In Claude Code, user settings in your home directory, apart from the project's `.claude/settings.json`.
- Each part can then be understood, tested and removed on its own.

### Retire with the model

1. Tag each layer 1 rule with the model behaviour it compensates for.
2. When the model changes, re-run the tasks those rules cover.
3. Remove the rules that no longer change the outcome.

The procedure is on [Re-testing after a model change]({{ '/sections/18-trends/retesting-after-a-model-change.html' | relative_url }}).

## Examples from my own harness

| What happened in my harness | What fixed it |
|---|---|
| Rules written for the same failure at different times | Merged into one mechanism |
| Context-file rules added after mistakes | Moved into hooks, workflows or tests, with a one-line pointer left behind |
| Guards whose match was too broad | Narrowed, or moved to a later gate |
| Context lines that compensated for an older model | Retired after a re-test |

To start: list every always-loaded line in your context files and, next to each, the failure it prevents. Any line without an answer is a candidate for removal.

## Sources

- The five outcomes for a correction: TRACE, [Getting Better at Working With You: Compiling User Corrections into Runtime Enforcement for Coding Agents](https://arxiv.org/abs/2606.13174), from my research survey; not yet verified against the primary source.
- From the author's own harness, described conceptually.

{% include page_nav.html %}
