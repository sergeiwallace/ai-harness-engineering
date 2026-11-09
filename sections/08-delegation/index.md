---
title: "Agent profiles, sub-agents and delegation"
nav_order: 18
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Agent profiles, sub-agents and delegation

As of 2026-11-09.

Stop 6 on the talk path, about 4 minutes. [Delegation in detail]({{ '/sections/08-delegation/detail.html' | relative_url }}) covers profiles, worktrees, nesting, model choice and checking results.

## Sub-agents

A sub-agent is a second agent that the main agent starts to do part of a task. It runs in its own context window, does its work, and returns a summary. The main agent's context stays small, because it sees only the summary, never the hundred files the sub-agent read.

Use a sub-agent when a part of the task is self-contained: search a codebase, review a diff, research a question, write the tests while the main agent writes the code. Several sub-agents can run at once.

### Example: a research question

The main agent needs to know which services call a deprecated API. It starts a sub-agent with the brief "list every caller of `old_client.fetch`, with file and line, as a table". The sub-agent searches and reads thirty files, then returns a twelve-row table. The main agent continues with twelve rows in its context, and none of the thirty files.

## Agent profiles

An agent profile is a file that fixes a role's settings: which tools it has, which model it runs on, which permissions it gets, and its standing instructions. In Claude Code profiles live in `.claude/agents/` as Markdown with frontmatter; in Codex they are TOML files in `.codex/agents/`. A read-only reviewer profile, for example, has no edit tools at all, so it cannot change the code it reviews.

### Example: a reviewer profile

```markdown
---
name: reviewer
description: Reviews a diff for bugs and missing tests. Read-only.
tools: Read, Grep, Glob
model: opus
---
Review the diff you are given. Report each finding with file, line, severity
and a one-line fix. Do not suggest style changes the linter already enforces.
```

## Delegation with worktrees

![A lead agent sends a brief to each of three sub-agents. Each sub-agent commits in its own worktree and returns a summary that the lead agent reads.]({{ '/assets/diagrams/delegation.svg' | relative_url }})

When several agents edit the same repository at once, they need separate checkouts or they overwrite each other. A git worktree is a second checkout of the same repository on its own branch. Give each writing agent its own worktree; Claude Code's `isolation: worktree` setting does this for a sub-agent. Bound how many agents run at once and how deep they nest.

## Pick the model per task

Use the strongest model for decisions, design and synthesis, and cheaper ones for bulk reading and mechanical edits. A sub-agent that greps a codebase does not need the model that plans the architecture.

## Check delegated work

Agents report work they did not do: a complete fix report with passing tests while no file changed. A summary can drop the caveat that changes the conclusion. Check the files, the commits and the test output yourself, and read the key passages of a summary's source.

### Example: catching a false report

A sub-agent returns "Fixed the race; all 214 tests pass." The lead agent runs `git -C <worktree> log --oneline -1` and sees no new commit, then `git diff --stat` and sees no change. The report described a plan; no work was done. The lead agent sends it back with the evidence.

## Commands

- `/tasks` lists the session's background work, including sub-agents that have finished.
- `/subtask` starts a sub-agent that inherits the conversation; `/fork` copies the session into a separate background session.

## Pages for this topic

- [Delegation in detail]({{ '/sections/08-delegation/detail.html' | relative_url }}): sub-agents and their context, profiles field by field, briefs, worktree isolation, nesting and concurrency limits, model choice, agent teams, checking results, my harness's profiles and guards, and recording who did what.

{% include page_nav.html %}
