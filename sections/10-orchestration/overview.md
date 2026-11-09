---
title: "Orchestration and workflows"
nav_order: 20
has_children: true
has_toc: false
permalink: /sections/10-orchestration/
---

{% include page_nav.html %}

# Orchestration and workflows

As of 2026-11-09.

Orchestration is how multi-step work gets done without you driving every step. There are two kinds. In prompt-driven orchestration a lead agent holds the plan and directs other agents through briefs. In programmatic orchestration a script or graph holds the plan and starts agents for each step. On top of either, goals and loops keep work going until a condition is met. Every loop needs an exit condition and a limit.

![A sequence diagram of one goal. You set the goal "suite shows 0 failures". Inside a loop fragment the agent runs the test suite after a fix, gets 2 failures printed, ends its turn, and the evaluator, which only reads, answers "not met: continue". The agent fixes and runs again, gets 0 failures printed, ends its turn, and the evaluator answers "condition met". The agent reports done to you with the output.]({{ '/assets/images/10-orchestration/goal-run.svg' | relative_url }}){: style="width:100%"}

*Figure: a `/goal` run. The evaluator can judge only what the agent printed. Made with archify.*

Stop 7 on the talk path, about 2 minutes. The deeper pages:

| Page | What it covers |
|---|---|
| [Prompt-driven orchestration]({{ '/sections/10-orchestration/prompt-driven-orchestration.html' | relative_url }}) | Lead agents, coordinator agents, briefs, joining results |
| [Chaining skills into a pipeline]({{ '/sections/10-orchestration/skill-pipelines.html' | relative_url }}) | Research, decide, audit, build, audit as a chain of skills |
| [Programmatic workflows]({{ '/sections/10-orchestration/programmatic-workflows.html' | relative_url }}) | Claude Code dynamic workflows, LangGraph and other frameworks |
| [Orchestration frameworks]({{ '/sections/10-orchestration/orchestration-frameworks.html' | relative_url }}) | LangGraph, OpenAI Agents SDK, CrewAI, Google ADK, Microsoft Agent Framework, and how to choose |
| [Goals and loops]({{ '/sections/10-orchestration/goals-and-loops.html' | relative_url }}) | `/goal`, `/loop`, exit conditions, limits |
| [Hill-climbing and quality ratchets]({{ '/sections/10-orchestration/hill-climbing.html' | relative_url }}) | Run, measure, change one thing, keep it only if better |

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Two kinds of orchestration

### Prompt-driven

A coordinator session or lead agent writes briefs, launches sub-agents and joins their results. The plan lives in the lead agent's context and in the prompts and [agent profiles]({{ '/sections/08-delegation/' | relative_url }}) it uses. It fits work whose steps depend on judgment and change as the work goes: the lead reads what came back and decides the next step.

![A sequence diagram. In a parallel fragment the lead briefs two readers, who return a table and notes. The lead rechecks a count, then briefs a writer, which reports a commit, and the lead verifies the diff.]({{ '/assets/images/10-orchestration/lead-splits-and-joins.svg' | relative_url }}){: style="width:100%"}

*Figure: prompt-driven: the lead writes the next brief only after reading the results. Made with archify.*

#### Example: a migration plan

A lead agent is asked to plan a database migration.

1. It starts three sub-agents at once.
   - One lists every query in the service.
   - One reads the new database's transaction docs.
   - One finds the tests that touch the data layer.
2. It reads the three reports and checks their evidence.
3. It notices that two queries use a feature the new database lacks.
4. It starts a fourth sub-agent to find replacements for those two queries.

The fourth step did not exist until the lead read the first three results. More in [prompt-driven orchestration]({{ '/sections/10-orchestration/prompt-driven-orchestration.html' | relative_url }}).

### Programmatic

Programmatic orchestration holds the plan in code. It fits work with known steps, many parallel agents, or a need to resume after a failure. Claude Code's dynamic workflows are JavaScript scripts that run many sub-agents; the plan does not take up Claude's context. LangGraph describes a workflow as a graph of steps with shared state, retries and human approval points.

![A sequence diagram. A script starts agent batches that write checkpoints 1 to 13; batch 14 fails; the script lists the done checkpoints, restarts at 14, and joins all 20 into the report.]({{ '/assets/images/10-orchestration/workflow-resume.svg' | relative_url }}){: style="width:100%"}

*Figure: programmatic: the script holds the plan and resumes from checkpoints. Made with archify.*

#### Example: review two hundred files

You want every file in a large repository checked against a new logging rule.

1. A workflow script lists the two hundred files.
2. It starts one agent per batch of ten, up to the concurrency limit.
   - Each agent writes its verdicts to a file for its batch.
3. If the run stops at batch 14, a rerun skips the finished batch files and resumes at 14.
4. The script joins every batch file into one report.

No model has to keep two hundred file names in its context. More in [programmatic workflows]({{ '/sections/10-orchestration/programmatic-workflows.html' | relative_url }}).

### Choosing between them

| Question | Prompt-driven | Programmatic |
|---|---|---|
| Are the steps known in advance? | No, they emerge | Yes |
| How many agents? | A handful | Tens to hundreds |
| Does a failure need a clean resume? | A person or the lead recovers | The script resumes |
| Where is the plan? | In the lead's context | In code |
| Cost of changing the plan mid-run | Low: the lead decides | Edit and rerun the script |

> Either kind can come first. One common path is to run a chain prompt-driven while its steps are still changing, and turn it into a script once it runs the same way every time.
{: .note }

## Goals and loops

Both kinds of orchestration need something that keeps the work going after the first answer, and something that stops it.

| Mechanism | Keeps going until | Stops on | Use for |
|---|---|---|---|
| `/goal <condition>` | An evaluator judges the condition met | The condition, or a continuation limit | Work with a describable end |
| `/loop <interval> <prompt>` | The session ends, or the prompt's own stop rule | A stop rule in the prompt | Checks that repeat on a clock |
| A workflow script | Its steps finish | Its exit condition and limit, in code | Known steps at scale |

### `/goal`

`/goal` adds a check at the end of each turn. A small fast model reads the conversation, without calling tools, and decides whether the condition is met. If not, the agent continues. Because the evaluator only reads, write the condition so it names evidence the agent will print.

#### Example: a condition that can be checked

```text
/goal The full test suite has been run after the last code change and its
output, printed in this conversation, shows 0 failures.
```

"The tests pass" is weaker: the evaluator cannot see a test run, only what was printed. Anthropic's guidance for Opus 5.5 recommends the same pattern for custom harnesses, and stopping after two or three automatic continuations. See [goals and loops]({{ '/sections/10-orchestration/goals-and-loops.html' | relative_url }}).

### `/loop`

`/loop` repeats a prompt on a time interval, for monitoring or recurring work: check a deploy every five minutes, look at a pull request's review comments every hour.

```text
/loop 10m check the nightly job's status page and tell me only if it failed
```

### Exit conditions and limits

> Every loop needs an exit condition and a limit. A condition alone can be wrong or unreachable; a limit alone stops good work early. With both, the loop ends when the work is done or when it has clearly stalled, and says which.
{: .warning }

## Hill-climbing

Hill-climbing is a loop with a number in it: run, measure, change one thing, and keep the change only if the measure improved. It works for anything with a score: test time, a benchmark, a prompt's pass rate on a set of examples.

#### Example: speeding up a test suite

1. Baseline: the suite takes 14 minutes, all green.
2. Round one: the agent changes one fixture from per-test to per-module.
   - 11 minutes, all green: the change stays.
3. Round two: it parallelises one slow module.
   - 11 minutes and two flaky failures: the change is reverted. The [hill-climbing page]({{ '/sections/10-orchestration/hill-climbing.html' | relative_url }}) covers quality ratchets, which apply the same idea to a whole codebase.

## Commands

| Command | What it does |
|---|---|
| `/goal <condition>` | Keeps the session working until an evaluator judges the condition met |
| `/loop <interval> <prompt>` | Repeats a prompt on a schedule |
| `/workflows` | Watches, pauses and saves dynamic workflows, the scripts Claude writes to run many sub-agents. A workflow starts when you ask for one or use the `ultracode` keyword |

## Examples from my own harness

- Goal templates, such as "work every task in progress to the end", started and ended through skills.
  - The condition always names the tracker's printed list as its evidence.
  - Every run ends with the same report: what landed, what is blocked, what needs me.
- A prompt-driven pipeline that runs research, decision, audit, build and a second audit, with coordinator agents doing each step ([chaining skills into a pipeline]({{ '/sections/10-orchestration/skill-pipelines.html' | relative_url }})).
- A quality ratchet that lets code-quality numbers improve and refuses a change that makes them worse ([hill-climbing and quality ratchets]({{ '/sections/10-orchestration/hill-climbing.html' | relative_url }})).

To start: one `/goal` that has a clear finish condition.

## Pitfalls and lessons

| Pitfall | What happens | What helps |
|---|---|---|
| A fix added for each failure | The orchestration engine grows until the fixes cause failures | Cut steps before adding them |
| A condition the evaluator cannot see | It accepts a confident summary | Put the proof in printed output |
| A loop with no limit | Cost with no ceiling | A continuation or round cap |
| A loop that reports every check | The one report that matters is buried | Report only changes |

## Sources

- Dynamic workflows: <https://code.claude.com/docs/en/workflows.md>, verified 2026-11-08.
- LangGraph: <https://docs.langchain.com/oss/python/langgraph/overview>, verified 2026-11-08.
- `/goal`: <https://code.claude.com/docs/en/goal.md>, verified 2026-11-08.
- Opus 5.5 harness guidance: <https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5>, verified 2026-11-08.
- `/loop`: <https://code.claude.com/docs/en/scheduled-tasks.md>, verified 2026-11-08.

{% include page_nav.html %}
