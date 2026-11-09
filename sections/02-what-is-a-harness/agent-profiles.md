---
title: "Agent profiles"
parent: "What is a harness?"
nav_order: 7
redirect_from:
  - /sections/02-ai-harnesses/agent-profiles.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Agent profiles

- Part of [What is a harness?]({{ '/sections/02-what-is-a-harness/' | relative_url }}).
- The full section on this topic is [Agent profiles, sub-agents and delegation]({{ '/sections/08-delegation/' | relative_url }}).

An agent profile fixes a role: what it is for, which model runs it, and which tools it may use.

- The main session starts a sub-agent from a profile, hands it a brief, and gets back a summary.
- Profiles are how a harness turns "ask another agent" into a repeatable division of labour.

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

Only `name` and `description` are required in Claude Code.

| Field | Does |
|---|---|
| `name` | the id the main session uses to pick the profile |
| `description` | when to delegate to it; like a skill's, it is a trigger |
| `tools` | the tools the sub-agent may use; omit to inherit all |
| `disallowedTools` | tools removed from the inherited or listed set |
| `model` | `sonnet`, `opus`, `haiku`, `fable`, a full model id, or `inherit` for the main session's model |
| `permissionMode` | the sub-agent's own permission mode, such as `plan` for a read-only role |
| `skills` | skills whose full text is loaded into the sub-agent at start |
| `hooks` | hooks scoped to this sub-agent |
| `isolation` | `worktree` runs it in a temporary git worktree |
| `memory` | `user`, `project` or `local`: its own memory across sessions |
| `maxTurns`, `effort`, `mcpServers`, `omitClaudeMd` | turn limit, effort, extra MCP servers, skipping the `CLAUDE.md` files |
| body | the sub-agent's standing instructions |

### Where profiles live

| Tool | Location |
|---|---|
| Claude Code | in order of precedence: managed settings, the `--agents` flag for one session, `.claude/agents/` in the project, `~/.claude/agents/`, a plugin's `agents/` folder |
| Codex | custom roles declared in the `agents` table of `config.toml`, each pointing at its own TOML config layer with `config_file` |

When two Claude Code profiles share a name, the one from the higher-precedence location wins. Both folders are scanned recursively.

### Built-in agents

| Agent | Tools | Used for |
|---|---|---|
| `Explore` | read-only; skips `CLAUDE.md` and git status for speed | searching and reading a codebase |
| `Plan` | read-only | gathering context in plan mode |
| `general-purpose` | every tool available to sub-agents | multi-step tasks that read and act |

## How a sub-agent runs

![A sequence diagram of one change delegated to sub-agents. The main session briefs a researcher, which returns three rules with sources. It briefs an engineer, which writes tests then code in its own worktree and reports the head commit, checks and gaps. It sends the diff and the original ask to a read-only reviewer from another model family, which returns two findings. The main session fixes one, answers one, and the change is ready to merge.]({{ '/assets/images/02-ai-harnesses/delegation.png' | relative_url }})

*Each sub-agent gets a profile, a brief and its own context; only summaries return. Our own diagram, made with archify.*

### Its own context

A Claude Code sub-agent starts with a fresh context window.

| It starts with | It does not see |
|---|---|
| its profile's body as its system prompt, plus environment details | the main session's conversation |
| every level of `CLAUDE.md` the main session loads (not for `Explore` and `Plan`) | skills the main session has already used |
| a git status snapshot | files the main session has already read |
| skills named in its `skills` field | earlier sub-agents' work, unless the brief includes it |
| the brief the main session writes | the person's earlier corrections |

A fork is the exception: it inherits the parent conversation.

Its exploration stays out of the main session's window, and only its summary comes back.

### The brief carries its context

Because the sub-agent sees nothing else, the brief must carry everything.

1. The goal, in the person's own words where possible.
2. What has already been tried or ruled out.
3. The files and documents to read.
4. The scope: which paths it owns, what it must not touch.
5. The limits: what it may not do, whatever it decides.
6. The report shape: which fields come back.

A short brief produces a confident answer to a different question. See [Templates and briefs]({{ '/sections/02-what-is-a-harness/' | relative_url }}#templates-and-briefs).

### Isolation

A sub-agent that edits files should work in its own git worktree.

- `isolation: worktree` in the profile gives it a temporary worktree, branched from the default branch, cleaned up automatically if it changes nothing.
- Its changes cannot collide with the main session's or another sub-agent's.
- Its branch can be reviewed and merged, or thrown away, as one unit.
- My harness enforces this with a [guard hook]({{ '/sections/02-what-is-a-harness/guard-hooks.html' | relative_url }}) that keeps an isolated agent's git commands and edits inside its own worktree, and refuses commands too complex to check.

### Nesting

- By default a sub-agent can start sub-agents of its own, up to three layers below the main conversation.
- A sub-agent at the limit does its work itself and returns one summary.
- `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` changes the limit; `1` turns nesting off.
- Every layer costs a fresh context and tokens, so a brief should say whether delegating further is allowed.

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

## Pitfalls

| Pitfall | Result | Fix |
|---|---|---|
| a brief that assumes shared context | a confident answer to a different question | write the brief as if to a new colleague |
| a reviewer with edit tools | it fixes what it should judge | drop `Edit` and `Write` from `tools` |
| several editing sub-agents in one folder | their changes collide | `isolation: worktree` |
| delegating a small task | more tokens and waiting than doing it | keep small tasks in the main session |
| trusting a sub-agent's summary without checking | its mistakes arrive in the same confident tone as its findings | verify the claims that matter before acting on them |

## Profiles in my harness

- Profiles live in one source folder and install everywhere, so a role means the same thing on every machine.
- The ground truth for each profile's model is its own front matter; a summary of it can go stale.
- Every sub-agent launch gets its own child issue in the task tracker, filed at launch and closed when its work is collected.
- A crashed agent leaves an open issue behind, so its unfinished work is visible.

## Sources

- [Claude Code sub-agents, fields, starting context and nesting](https://code.claude.com/docs/en/sub-agents.md), verified 2026-11-09.
- [Codex `agents` configuration](https://learn.chatgpt.com/docs/config-file/config-reference), verified 2026-11-09.
- The roles and launch tracking are from the author's own harness, described conceptually.

{% include page_nav.html %}
