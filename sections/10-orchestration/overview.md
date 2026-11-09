---
title: "Orchestration and workflows"
nav_order: 20
has_children: true
has_toc: false
permalink: /sections/10-orchestration/
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Orchestration and workflows

Orchestration is how multi-step work gets done without you driving every step:

- In prompt-driven orchestration a lead agent holds the plan and directs other agents through briefs.
- In programmatic orchestration a script holds the plan and starts agents for each step.
- Goals and loops keep either kind going until a condition is met.
- Every loop needs an exit condition and a limit.

![A sequence diagram. In a parallel fragment the lead briefs two readers, who return a table and notes. The lead rechecks a count, then briefs a writer, which reports a commit, and the lead verifies the diff.]({{ '/assets/images/10-orchestration/lead-splits-and-joins.svg' | relative_url }}){: style="width:100%"}

*Figure: prompt-driven orchestration: the lead writes the next brief only after reading the results. Made with archify.*

Stop 7 on the talk path, about 2 minutes.

Pages in this section, in reading order:

- This overview: the two kinds, who holds the plan, goals and loops, hill-climbing, and the commands.
- [Prompt-driven orchestration]({{ '/sections/10-orchestration/prompt-driven-orchestration.html' | relative_url }}): the lead agent's job, splitting and joining work, briefs, and when to move to a script.
  - [Chaining skills into a pipeline]({{ '/sections/10-orchestration/skill-pipelines.html' | relative_url }}): research, decide, audit, build and audit as stages that hand over files.
- [Programmatic workflows]({{ '/sections/10-orchestration/programmatic-workflows.html' | relative_url }}): Claude Code dynamic workflows, graph frameworks, and designing for resume.
  - [Orchestration frameworks]({{ '/sections/10-orchestration/orchestration-frameworks.html' | relative_url }}): LangGraph, OpenAI Agents SDK, CrewAI, Google ADK, Microsoft Agent Framework, and how to choose.
- [Goals and loops]({{ '/sections/10-orchestration/goals-and-loops.html' | relative_url }}): `/goal`, Stop hooks, `/loop` and scheduled tasks, conditions and limits.
  - [Hill-climbing and quality ratchets]({{ '/sections/10-orchestration/hill-climbing.html' | relative_url }}): run, measure, change one thing, keep it only if better.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Two kinds of orchestration

### Prompt-driven

A coordinator session or lead agent writes briefs, launches sub-agents and joins their results.

- The plan lives in the lead agent's context and in the prompts and [agent profiles]({{ '/sections/08-delegation/' | relative_url }}) it uses.
- It fits work whose steps depend on judgment and change as the work goes.
- The figure at the top of this page shows one round: two parallel reads, a check, then a write.

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

A script holds the plan in code and starts agents for each step.

- It fits work with known steps, many parallel agents, or a need to resume after a failure.
- Claude Code's dynamic workflows are JavaScript scripts that Claude writes and a runtime executes in the background; intermediate results stay in script variables, so Claude's context holds only the final answer.
- Graph frameworks such as LangGraph describe a workflow as steps with shared state, retries and human approval points.

![A sequence diagram. A script starts agent batches that write checkpoints 1 to 13; batch 14 fails; the script lists the done checkpoints, restarts at 14, and joins all 20 into the report.]({{ '/assets/images/10-orchestration/workflow-resume.svg' | relative_url }}){: style="width:100%"}

*Figure: programmatic orchestration: the script holds the plan and resumes after a failure. Made with archify.*

#### Example: review two hundred files

You want every file in a large repository checked against a new logging rule.

1. You ask Claude Code to "use a workflow" for it, and approve the plan it shows.
2. The script lists the files, then runs one agent per file through `pipeline()`, up to 16 at a time by default.
3. An agent hits a rate limit at file 140 and the run stops.
4. You ask Claude to relaunch it. The runtime returns the saved results of the agents that finished, and runs the rest.
5. The script joins the verdicts into one report.

No model has to keep two hundred file names in its context. More in [programmatic workflows]({{ '/sections/10-orchestration/programmatic-workflows.html' | relative_url }}).

### Who holds the plan

Claude Code's own comparison of its multi-agent features turns on one question: who decides what runs next.

| | Sub-agents | Skills | Agent teams | Dynamic workflows |
|---|---|---|---|---|
| What it is | A worker Claude spawns | Instructions Claude follows | A lead supervising peer sessions | A script the runtime executes |
| Who decides what runs next | Claude, turn by turn | Claude, following the prompt | The lead, turn by turn | The script |
| Where intermediate results live | Claude's context | Claude's context | A shared task list | Script variables |
| Scale | A few tasks per turn | Same as sub-agents | A handful of long-running peers | Dozens to hundreds of agents per run |
| After an interruption | The turn restarts | The turn restarts | Teammates keep running | Resumable in the same session |

The first three are prompt-driven; the last is programmatic.

### Choosing between them

| Question | Prompt-driven | Programmatic |
|---|---|---|
| Are the steps known in advance? | No, they emerge | Yes |
| How many agents? | A handful | Tens to hundreds |
| Does a failure need a clean resume? | The lead or a person recovers | The runtime or the script resumes |
| Where is the plan? | In the lead's context | In code |
| Changing the plan mid-run | The lead decides | Edit and relaunch the script |

## Goals and loops

Both kinds of orchestration need something that keeps the work going after the first answer, and something that stops it. Claude Code has three ways to keep one session running between prompts:

| Mechanism | Next turn starts when | Stops when |
|---|---|---|
| `/goal <condition>` | The previous turn finishes | A small model judges the condition met or impossible, a turn fails on an error you must fix, or you run `/goal clear` |
| `/loop [interval] [prompt]` | An interval elapses, or one Claude picks | You stop it, Claude ends a self-paced loop, or 7 days pass |
| A Stop hook | The previous turn finishes | Your own script or prompt decides |

### `/goal`

`/goal` sets a completion condition. After each turn, a small fast model reads the conversation, without calling tools, and returns "not yet met", "met" or "impossible".

![A sequence diagram of one goal. You set the goal "suite shows 0 failures". Inside a loop fragment the agent runs the test suite after a fix, gets 2 failures printed, ends its turn, and the evaluator, which only reads, answers "not met: continue". The agent fixes and runs again, gets 0 failures printed, ends its turn, and the evaluator answers "condition met". The agent reports done to you with the output.]({{ '/assets/images/10-orchestration/goal-run.svg' | relative_url }}){: style="width:100%"}

*Figure: a `/goal` run. The evaluator judges only what the agent printed. Made with archify.*

#### Example: a condition that can be checked

```text
/goal all tests in tests/export pass, shown by pytest output printed after
the last code change, and no other test file is modified; or stop after 15 turns
```

- It names the evidence the evaluator will read: printed `pytest` output.
- It states a constraint: no other test file changes.
- It bounds the run with a turn clause, which the docs recommend for limiting how long a goal runs.

Anthropic's guidance for Opus 5.5 recommends the same evaluator pattern for custom harnesses, and stopping after two or three automatic continuations. More in [goals and loops]({{ '/sections/10-orchestration/goals-and-loops.html' | relative_url }}).

### A Stop hook

A Stop hook runs after every turn like `/goal`, but it lives in settings and can run a script.

| | `/goal` | Stop hook running a script |
|---|---|---|
| Who judges | A small model reading the conversation | Your script |
| Sees | Only printed output | The repository, the tests, anything a script can run |
| Scope | One session | Every session in the settings scope |

Exit code 2 from the hook keeps Claude working, with the hook's message as the reason. The [goals and loops page]({{ '/sections/10-orchestration/goals-and-loops.html' | relative_url }}#a-stop-hook-as-an-enforced-goal) has a full example.

### `/loop`

`/loop` repeats a prompt on a schedule within the open session.

| You type | What happens |
|---|---|
| `/loop 10m check the nightly job and tell me only if it failed` | Runs every 10 minutes (units `s`, `m`, `h`, `d`; cron granularity is one minute) |
| `/loop check whether CI passed and address review comments` | Claude picks the next delay each time, between one minute and one hour |
| `/loop` | Runs a built-in maintenance prompt, or your `.claude/loop.md` |

Recurring loops expire 7 days after creation. For work that should run without an open session, Claude Code also has cloud and Desktop scheduled tasks.

### Exit conditions and limits

| Mechanism | Its exit condition | Its limit |
|---|---|---|
| `/goal` | The condition you write | A turn or time clause in the condition; Claude Code also stops a loop that makes no tool calls for several turns |
| `/loop` | A stop rule in the prompt, or Claude ending a self-paced loop | The 7-day expiry |
| A dynamic workflow | The script's own logic | 1,000 agents per run |

## Hill-climbing

Hill-climbing is a loop with a number in it: run, measure, change one thing, and keep the change only if the measure improved. It works for anything with a score: test time, a benchmark, a prompt's pass rate on a fixed set of examples.

#### Example: speeding up a test suite

1. Baseline: the suite takes 14 minutes, all green.
2. Round one: the agent changes one fixture from per-test to per-module.
   - 11 minutes, all green: the change stays.
3. Round two: it parallelises one slow module.
   - 11 minutes and two flaky failures: the change is reverted.

The [hill-climbing page]({{ '/sections/10-orchestration/hill-climbing.html' | relative_url }}) covers local maxima and quality ratchets, which apply the same rule to a whole codebase.

## Commands

| Command | What it does |
|---|---|
| `/goal <condition>` | Sets a completion condition and starts working toward it |
| `/goal` | Shows the active goal: condition, time, turns evaluated, token spend, last reason |
| `/goal clear` | Removes the active goal (aliases: `stop`, `off`, `reset`, `none`, `cancel`) |
| `/loop [interval] [prompt]` | Repeats a prompt on a schedule, or self-paced |
| `/workflows` | Lists dynamic workflow runs; watch, pause (`p`), stop (`x`) or save (`s`) one |
| `/deep-research <question>` | Runs the bundled research workflow |
| `ultracode` in a prompt | Asks Claude to write a dynamic workflow for that task |

## Examples from my own harness

- Goal templates, such as "work every task in progress to the end", started and ended through skills.
  - The condition always names the tracker's printed list as its evidence.
  - Every run ends with the same report: what landed, what is blocked, what needs me.
- A prompt-driven pipeline that runs research, decision, audit, build and a second audit, with coordinator agents doing each step ([chaining skills into a pipeline]({{ '/sections/10-orchestration/skill-pipelines.html' | relative_url }})).
- A quality ratchet that lets code-quality numbers improve and refuses a change that makes them worse ([hill-climbing and quality ratchets]({{ '/sections/10-orchestration/hill-climbing.html' | relative_url }})).

To start: one `/goal` with a clear finish condition and a turn clause.

## Pitfalls and lessons

| Pitfall | What happens | What helps |
|---|---|---|
| A fix added for each failure | The orchestration engine grows until the fixes cause failures | Cut steps before adding them |
| A condition the evaluator cannot see | It accepts a confident summary | Name printed output, or use a Stop hook that runs the check |
| A goal with no bound | It runs as long as it keeps making progress | A turn or time clause in the condition |
| A loop that reports every check | The one report that matters is buried | Report only changes |
| A large workflow started blind | Many agents, many tokens | Run it on a small slice first; Claude Code warns above 25 agents or 1.5 million projected tokens |

## Sources

- `/goal` and Stop hooks: [Keep Claude working toward a goal (Claude Code docs)](https://code.claude.com/docs/en/goal.md), verified 2026-11-09.
- `/loop` and scheduled tasks: [Run prompts on a schedule (Claude Code docs)](https://code.claude.com/docs/en/scheduled-tasks.md), verified 2026-11-09.
- Dynamic workflows: [Orchestrate subagents at scale with dynamic workflows (Claude Code docs)](https://code.claude.com/docs/en/workflows.md), verified 2026-11-09.
- LangGraph: [LangGraph overview (LangChain docs)](https://docs.langchain.com/oss/python/langgraph/overview), verified 2026-11-08.
- Opus 5.5 harness guidance: [Prompting Claude Opus 5.5 (Claude Platform docs)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5), verified 2026-11-08.

{% include page_nav.html %}
