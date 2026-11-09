---
title: "Prompt-driven orchestration"
parent: "Orchestration and workflows"
nav_order: 1
has_children: true
has_toc: false
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Prompt-driven orchestration

In prompt-driven orchestration a lead agent, or a coordinator session, holds the plan and directs other agents through briefs and agent profiles.

This page covers:

- The lead's job, and the Claude Code tools it uses.
- How it splits work and writes briefs.
- How it joins and checks results.
- When it should hand off to a script, and how it fails.

The [section overview]({{ '/sections/10-orchestration/' | relative_url }}) compares it with programmatic orchestration. The sub-topic page [chaining skills into a pipeline]({{ '/sections/10-orchestration/skill-pipelines.html' | relative_url }}) covers one fixed shape of it.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The lead agent's job

![A sequence diagram. Inside a parallel fragment the lead briefs reader 1 to search the code and reader 2 to read the spec; reader 1 returns a table and reader 2 returns notes. The lead rechecks reader 1's count, then briefs the writer to build the exporter in its own worktree. The writer reports a commit and the lead verifies the diff.]({{ '/assets/images/10-orchestration/lead-splits-and-joins.svg' | relative_url }}){: style="width:100%"}

*Figure: a lead agent splits work, joins the results, and only then writes the next brief. Made with archify.*

The lead works in a loop:

1. Plan the next steps.
2. Write a brief for each.
3. Start the agents.
4. Read and check what came back.

The lead does not do the leaf work itself. That keeps its context for the plan and the results, the same reason a [coordinator session]({{ '/sections/09-session-coordination/coordinator-sessions.html' | relative_url }}) hands its work to other sessions.

### Planning in rounds

A prompt-driven plan can change as results come in.

1. The lead plans as far as it can see.
2. It runs those steps.
3. It reads the results and plans again.
   - A step can appear that did not exist before, such as finding replacements for two queries the first reports flagged.

A script can only do this where its author wrote the decision point in.

### Lead agent or coordinator session

| | Lead agent | Coordinator session |
|---|---|---|
| Lives for | One task | Hours or days |
| Directs | Its own sub-agents | Other sessions, and sub-agents |
| Plan held in | Its context | Files and a task tracker, since it compacts often |
| Typical work | A research question, a feature | A stream of requests from a person |

## The tools in Claude Code

| Tool or feature | Role in prompt-driven orchestration |
|---|---|
| The `Agent` tool | Starts a sub-agent with a brief and, optionally, an agent profile; `isolation: worktree` gives a writer its own checkout ([delegation]({{ '/sections/08-delegation/' | relative_url }})) |
| Background sub-agents | Run while the lead keeps working; `/tasks` lists them, and the lead hears when each finishes |
| Named sub-agents | Sub-agents that Claude named when it spawned them can message each other with `SendMessage` |
| Agent teams (experimental) | Teammates with a shared task list and direct messages, under one lead; see [SendMessage and ListAgents]({{ '/sections/09-session-coordination/sendmessage-and-listagents.html' | relative_url }}#agent-teams) |
| Skills | A fixed procedure the lead follows, such as a review checklist or a pipeline stage ([skills]({{ '/sections/05-skills-and-plugins/' | relative_url }})) |

In all of these the plan stays with Claude, turn by turn. That is what separates them from a [dynamic workflow]({{ '/sections/10-orchestration/programmatic-workflows.html' | relative_url }}), where the script holds it.

## Splitting the work

### Independent parts run in parallel

Parts that do not need each other's results start at once.

- Three searches over different parts of a codebase.
- A read of the docs for a new dependency.
- An inventory of the tests that touch the area.

### Dependent parts run in order

- A part that needs another's output waits for it.
  - Writing the migration needs the list of queries first.
  - The review needs the code and the test to exist.

### Writers get their own worktrees

Readers can share a checkout. Agents that edit files each need their own [worktree]({{ '/sections/08-delegation/' | relative_url }}), or they overwrite each other.

#### Example: splitting a feature

A lead agent building an export feature splits it like this:

| Part | Kind | Depends on | Worktree |
|---|---|---|---|
| Find the existing export code | Read | Nothing | Shared |
| Read the file-format spec | Read | Nothing | Shared |
| Write the exporter | Write | Both reads | Own |
| Write the end-to-end test | Write | Format spec only | Own |
| Review the result | Read | Exporter and test | Shared |

The parts start in this order:

1. The two reads start together.
2. The test writer starts as soon as the spec read finishes, without waiting for the exporter.
3. The review waits for both writers.

## Briefs

The brief is the only thing a sub-agent knows about the task, so a rule that is not in the brief does not reach it. Of the fields in [templates and stubs]({{ '/sections/07-templates-and-stubs/' | relative_url }}), two matter most for orchestration, because the lead has to join several results:

- The output format.
- The evidence wanted back.

```markdown
## Objective
List every caller of `export_rows` and how it passes the format argument.

## Output
A Markdown table: file, line, format value, notes. No prose before or after.

## Evidence
The grep command you ran and its match count.

## Boundaries
Read only. Do not edit files.
```

When every sub-agent returns the same shape, joining is mechanical.

## Joining and checking results

| The sub-agent reports | The lead checks | How |
|---|---|---|
| "Committed abc123 in my worktree" | The commit exists and touches the files named | `git -C <worktree> show --stat abc123` |
| "All tests pass" | The test run happened after the last edit | Run the named tests itself |
| "Found 12 callers" | The search covered the whole question | Read the command it ran, and rerun it if the scope looks narrow |
| "No changes needed" | The sub-agent read the target | Ask for the file and line it based that on |

### Conflicts between results

Two sub-agents can disagree: one reports nine callers, another eleven.

1. Compare how each got its number.
   - Different search patterns, different directories, one counting tests.
2. Rerun the search that explains the difference.
3. Use the result whose method covers the question asked.

Agreement can hide the same gap: two reports of "nine" from different populations do not confirm each other.

## When to hand off to a script

Signs that a prompt-driven chain should become a [programmatic workflow]({{ '/sections/10-orchestration/programmatic-workflows.html' | relative_url }}):

1. The same steps run in the same order every time.
2. The task needs more agents than one conversation can coordinate.
   - The Claude Code docs give a 500-file migration and a codebase-wide bug sweep as examples.
3. A failure halfway means starting over, and that is expensive.
4. The lead's context fills with bookkeeping: lists of agents, their states, their results.
5. You want the orchestration itself saved and rerun, such as a review run on every branch.

## Failure modes

### The lead does the work

- What happens: it starts reading files "to save a step", fills its context and loses the plan.
- What helps: a lead profile without edit tools, so the narrow role is enforced by the tool list.

```markdown
---
name: lead
description: Plans, delegates and checks. Starts sub-agents for all reading and writing.
tools: Agent, Read, Bash
---
Split the request into parts. Brief a sub-agent for each part and check its
evidence before you combine results.
```

The `tools` list has no `Edit` or `Write`, so the lead cannot make the change itself; `Bash` stays for running checks.

### Brief drift

- What happens: each layer of delegation paraphrases the request, and by the third layer the sub-agent solves a slightly different problem.
- What helps: pass the original request verbatim alongside each brief, as a fixed block in the brief template ([templates and stubs]({{ '/sections/07-templates-and-stubs/' | relative_url }})).

### The engine grows

- What happens: each failure gets a new instruction, a new step or a new special case in the lead's brief, and the brief becomes long enough that the lead follows parts of it.
- What helps:
  - Before adding a step, ask whether an existing one can be removed or merged.
  - Move a repeated check out of the brief and into a tool or hook that runs every time.
  - Keep the brief template short, and put detail in files the sub-agent reads when it needs them.

## Pitfalls and lessons

| Pitfall | What helps |
|---|---|
| The lead does leaf work | No edit tools in its profile |
| Paraphrase drift across layers | The original request verbatim in every brief |
| Results that cannot be joined | A fixed output shape in every brief |
| Summaries taken on trust | The evidence check table above |
| Two writers in one checkout | A worktree per writer |
| A growing brief | Cut steps; move checks into tools |

## Examples from my own harness

- Coordinator agents run each stage of a pipeline ([chaining skills into a pipeline]({{ '/sections/10-orchestration/skill-pipelines.html' | relative_url }})).
- Every brief they write carries the person's original request verbatim, so drift is visible at each layer.
- Each brief names the evidence to return, and the coordinator checks it before reporting.

To start: one lead agent, two parallel read-only sub-agents, and a brief that fixes the output table.

## Sources

- Sub-agents, background sub-agents and `isolation`: [Create custom subagents (Claude Code docs)](https://code.claude.com/docs/en/sub-agents.md).
- Named sub-agents messaging and agent teams: [Orchestrate teams of Claude Code sessions (Claude Code docs)](https://code.claude.com/docs/en/agent-teams.md), verified 2026-11-09.
- When to use a workflow: [Orchestrate subagents at scale with dynamic workflows (Claude Code docs)](https://code.claude.com/docs/en/workflows.md), verified 2026-11-09.
- Opus 5.5 harness guidance: [Prompting Claude Opus 5.5 (Claude Platform docs)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5), verified 2026-11-08.

{% include page_nav.html %}
