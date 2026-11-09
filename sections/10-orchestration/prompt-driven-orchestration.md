---
title: "Prompt-driven orchestration"
parent: "Orchestration and workflows"
nav_order: 1
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Prompt-driven orchestration

As of 2026-11-09.

In prompt-driven orchestration a lead agent, or a coordinator session, holds the plan and directs other agents through briefs and agent profiles. This page covers the lead's job, how it splits work, how it joins results, when it should hand off to a script, and how it fails. The [section overview]({{ '/sections/10-orchestration/' | relative_url }}) compares it with programmatic orchestration. A sub-topic page covers [chaining skills into a pipeline]({{ '/sections/10-orchestration/skill-pipelines.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The lead agent's job

![A sequence diagram. Inside a parallel fragment the lead briefs reader 1 to search the code and reader 2 to read the spec; reader 1 returns a table and reader 2 returns notes. The lead rechecks reader 1's count, then briefs the writer to build the exporter in its own worktree. The writer reports a commit and the lead verifies the diff.]({{ '/assets/images/10-orchestration/lead-splits-and-joins.svg' | relative_url }}){: style="width:100%"}

*Figure: a lead agent splits work, joins the results, and only then writes the next brief. Made with archify.*

The lead does four things, in a loop: plan the next steps, write a brief for each, start the agents, then read and check what came back. It does not do the leaf work itself. That keeps its context for the plan and the results, the same reason a [coordinator session]({{ '/sections/09-session-coordination/coordinator-sessions.html' | relative_url }}) routes instead of working.

### Planning in steps, not all at once

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

The two reads start together. The test writer starts as soon as the spec read finishes, without waiting for the exporter. The review waits for both writers.

## Briefs

The brief is the only thing a sub-agent knows about the task. A rule that is not in the brief does not reach the sub-agent. [Templates and stubs]({{ '/sections/07-templates-and-stubs/' | relative_url }}) covers the fields; for orchestration the ones that matter most are the output format and the evidence wanted back, because the lead has to join several results.

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

<details markdown="1">
<summary>Why the lead must check, not only merge</summary>

The lead treats each sub-agent's summary as unverified until checked. Agents report work they did not do, and a summary can drop the caveat that changes the conclusion. The lead checks the evidence the brief asked for: the commit exists, the test output shows the test passing, the match count makes sense. A lead that only merges summaries passes every sub-agent's mistake on as its own conclusion.

</details>

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
2. The number of agents grows past what the lead can track.
3. A failure halfway means starting over, and that is expensive.
4. The lead's context fills with bookkeeping rather than judgment.

## Failure modes

### The lead does the work

- What happens: it starts reading files "to save a step", fills its context and loses the plan.
- What helps: a lead profile without edit tools, so the narrow role is enforced by the tool list.

### Brief drift

- What happens: each layer of delegation paraphrases the request, and by the third layer the sub-agent solves a slightly different problem.
- What helps: pass the original request verbatim alongside each brief, as a fixed block in the brief template ([templates and stubs]({{ '/sections/07-templates-and-stubs/' | relative_url }})).

### The engine grows

> Adding a fix for each failure grows an orchestration engine until the fixes cause failures. Cutting steps can work better than adding them.
{: .warning }

## Examples from my own harness

- Coordinator agents run each stage of a pipeline ([chaining skills into a pipeline]({{ '/sections/10-orchestration/skill-pipelines.html' | relative_url }})).
- Every brief they write carries the person's original request verbatim, so drift is visible at each layer.
- Each brief names the evidence to return, and the coordinator checks it before reporting.

To start: one lead agent, two parallel read-only sub-agents, and a brief that fixes the output table.

## Sources

- Sub-agents in Claude Code: <https://code.claude.com/docs/en/sub-agents.md>.
- Opus 5.5 harness guidance: <https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-claude-opus-5-5>, verified 2026-11-08.

{% include page_nav.html %}
