---
title: "Agent profiles"
parent: "AI harnesses"
nav_order: 7
---

{% include page_nav.html %}

# Agent profiles

As of 2026-11-09. Part of [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}). The full section on this topic is [Agent profiles, sub-agents and delegation]({{ '/sections/08-delegation/' | relative_url }}).

An agent profile fixes a role: what it is for, which model runs it, and which tools it may use. The main session starts a sub-agent from a profile, hands it a brief, and gets back a summary. Profiles are how a harness turns "ask another agent" into a repeatable division of labour.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Anatomy of a profile

### The file

```markdown
---
name: researcher
description: Deep research specialist. Use for API docs, library comparisons and prior art. Returns a sourced summary.
tools: Read, Grep, Glob, WebFetch, WebSearch
model: opus
---

Answer the question you are given with sources. For each claim, give the URL and
the date you read it. Say plainly what you could not verify. Never edit files.
```

### The fields

| Field | Does |
|---|---|
| `name` | the id the main session uses to pick the profile |
| `description` | tells the main session when to use it; like a skill's, it is a trigger |
| `tools` | the only tools the sub-agent may use; omit to inherit all |
| `model` | which model runs it: a cheaper one for bulk work, the strongest for judgment |
| body | the sub-agent's standing instructions, added to every brief |

### Where profiles live

| Tool | Location |
|---|---|
| Claude Code | `.claude/agents/<name>.md` in the project, or `~/.claude/agents/` for all projects |
| Codex | `.codex/agents/` |

## How a sub-agent runs

![A sequence diagram of one change delegated to sub-agents. The main session briefs a researcher, which returns three rules with sources. It briefs an engineer, which writes tests then code in its own worktree and reports the head commit, checks and gaps. It sends the diff and the original ask to a read-only reviewer from another model family, which returns two findings. The main session fixes one, answers one, and the change is ready to merge.]({{ '/assets/images/02-ai-harnesses/delegation.png' | relative_url }})

*Each sub-agent gets a profile, a brief and its own context; only summaries return. Our own diagram, made with archify.*

### Its own context

A sub-agent starts fresh.

| It sees | It does not see |
|---|---|
| its profile's body | the main session's conversation |
| the project's context files | earlier sub-agents' work, unless the brief includes it |
| the brief | the person's earlier corrections |

Its exploration stays out of the main session's window, and only its summary comes back.

### The brief carries its context

Because the sub-agent sees nothing else, the brief must carry everything.

1. The goal, in the person's own words where possible.
2. What has already been tried or ruled out.
3. The files and documents to read.
4. The scope: which paths it owns, what it must not touch.
5. The limits: what it may not do, whatever it decides.
6. The report shape: which fields come back.

A short brief produces a confident answer to a different question. See [Templates and briefs]({{ '/sections/02-ai-harnesses/' | relative_url }}#templates-and-briefs).

### Isolation

A sub-agent that edits files should work in its own git worktree.

- Its changes cannot collide with the main session's or another sub-agent's.
- Its branch can be reviewed and merged, or thrown away, as one unit.
- My harness enforces this with a [guard hook]({{ '/sections/02-ai-harnesses/guard-hooks.html' | relative_url }}) that keeps an isolated agent's git commands and edits inside its own worktree, and refuses commands too complex to check.

## Roles

### A typical set

| Role | Tools | Model | Used for |
|---|---|---|---|
| researcher | read, web | strongest | questions with sources |
| engineer | read, edit, shell | balanced or strongest | writing code and tests |
| reviewer | read, shell; no edit | a different family from the author | finding problems in a change |
| auditor | read, write one report | strongest | adversarial checks against ground truth |
| analyst | read, shell | cheaper | bulk extraction and summaries |

### Why a reviewer cannot edit

| Reviewer with edit tools | Reviewer without |
|---|---|
| "fixes" what it was asked to judge | returns a list of findings |
| the person loses the record of what was wrong | each finding can be accepted or answered |
| the fix itself is unreviewed | the author makes the fix, and it can be reviewed |

Taking the tools away in the profile makes the role hold whatever the brief says.

### Why a different family

A model reviewing its own family's work tends to share its blind spots.

- My harness prefers a reviewer from a different model family for any change.
- It records which family reviewed, in the change's description.
- It does not block a merge when no other family is available; a same-family review is weaker and still useful.

## Worked example

### The task

Add a new export format to a reporting tool.

- The format has a public specification with a few rules that matter.
- The tool has tests and a documentation page per format.
- The change should land as one reviewed branch.

### The division

1. The main session reads the request and the code, and writes a plan.
2. A researcher sub-agent checks the format's specification and returns the three rules that matter, with sources.
3. An engineer sub-agent, in its own worktree, writes the tests and the code.
4. A reviewer sub-agent from another family reads the diff against the plan and the original request, and returns two findings.
5. The main session fixes one finding, answers the other, and lands the change.

The main session's context holds the plan, three summaries and the decisions. The pages of specification and the exploration of the code stayed in the sub-agents' windows.

## Profiles in my harness

- Profiles live in one source folder and install everywhere, so a role means the same thing on every machine.
- The ground truth for each profile's model is its own front matter, never a summary of it.
- Every sub-agent launch gets its own child issue in the task tracker, filed at launch and closed when its work is collected.
- A crashed agent leaves an open issue behind, so its unfinished work is visible.

## Sources

- Claude Code sub-agents: <https://code.claude.com/docs/en/sub-agents.md>.
- The roles and launch tracking are from the author's own harness, described conceptually.

{% include page_nav.html %}
