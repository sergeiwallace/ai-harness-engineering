---
title: "Agent profiles, sub-agents and delegation"
permalink: /sections/08-delegation/
nav_order: 18
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Agent profiles, sub-agents and delegation

As of 2026-11-09. Overview page. Slides: [talk stop 6]({{ '/talk/' | relative_url }}#stop-6-4-minutes), about 4 minutes: sub-agents and agent profiles, delegation with worktrees, checking delegated work.

Delegation means a lead agent hands part of a task to another agent and keeps control of the result. Figure 1 follows one round from launch to merge. The parts work together like this:

1. The lead writes a brief and asks to launch an agent with a [profile]({{ '/sections/08-delegation/agent-profiles.html' | relative_url }}).
   - The profile fixes the agent's tools and model.
   - Launch checks can refuse when the machine is busy; the lead then does the item itself.
2. A writing agent works in its own [worktree]({{ '/sections/08-delegation/worktree-isolation.html' | relative_url }}).
   - Its commits land on its own branch, apart from every other writer.
3. The agent reports, and the lead [checks the artifact]({{ '/sections/08-delegation/checking-delegated-work.html' | relative_url }}).
   - Commits, diff and test output, read by the lead itself.
4. A reviewer from another model family reads the branch, and the lead merges it.

![A sequence across six participants. The lead asks the launch checks to start engineer A, which starts in its own worktree. A second launch is refused because the machine is busy, so the lead does that item itself. Engineer A reports a commit and a test line; the lead checks the log, diff and tests itself. When a slot frees, engineer B starts. A reviewer from another model family returns two findings, engineer A fixes them, and the lead merges branch A into main.]({{ '/assets/images/08-delegation/delegation-fanout.png' | relative_url }}){: style="width:100%"}

*Figure 1. One delegation round, top to bottom. Built with archify.*

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Sub-agents

### What a sub-agent is

<!-- IMAGE NEEDED: a main agent's context window beside a sub-agent's separate context window, with only a short summary passing back to the main one; search: subagent context window diagram, claude code subagents, multi-agent orchestrator worker diagram -->

A sub-agent is a second agent that the main agent starts to do part of a task.

- It runs in its own context window.
- It does its work with its own tools, set by its profile.
- It returns a summary as its final answer.
- The main agent sees only that summary, never the files the sub-agent read, so its own window stays free for coordination.

### What it sees, and what it does not

| The sub-agent starts with | It does not get |
|---|---|
| Its brief | The lead's conversation |
| Its profile's instructions and tools | Rules the lead learned this session |
| The project's context files | Files the lead read, unless named in the brief |

The brief is the only channel from parent to child, which is why its shape is fixed by a template: see [Sub-agent briefs]({{ '/sections/07-templates-and-stubs/briefs.html' | relative_url }}).

### When delegation tends to fit

These are common choices, each with the case it fits. A lead weighs them per task.

| Often delegated | Often kept by the lead |
|---|---|
| A self-contained search or research question | Work that needs the conversation to be done well |
| A review by a role that should not share the author's reasoning | A task smaller than the brief that would describe it |
| Parallel work on separate parts of a change | Anything you would have to re-check line by line anyway |
| A long mechanical edit across many files | A design decision |

> Note: Claude Code's `/subtask` starts a sub-agent that inherits the conversation, for the case where context matters. `/fork` copies the session into a separate background session. `/tasks` lists background work, including finished sub-agents.

#### Worked example: a research question

1. The main agent needs to know which services call a deprecated API.
2. It starts a sub-agent with the brief "list every caller of `old_client.fetch`, with file and line, as a table".
3. The sub-agent searches and reads thirty files in its own context.
4. It returns a twelve-row table.
   - The main agent's context gains twelve rows; the thirty files stay with the sub-agent.

## Shapes of delegation

Delegation can take three shapes. Most delegation today is a hub, with a tree where one part is large.

| Shape | How results flow | Status in Claude Code |
|---|---|---|
| Hub | Every sub-agent reports to one lead | Standard sub-agents |
| Tree | Some sub-agents start helpers and report up one level | Standard sub-agents, within the depth limit |
| Team | Agents message each other directly | Agent teams: experimental, off by default |

Session-to-session coordination is covered in [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }}).

## Agent profiles

### A role in a file

An agent profile fixes a role's settings: which tools it has, which model it runs on, its permissions, and its standing instructions. In Claude Code, profiles are Markdown files with frontmatter in `.claude/agents/`; in Codex they are TOML files in `.codex/agents/`.

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

### Why roles with limits

A profile with no edit tools cannot edit, whatever its brief says. A reviewer cannot "fix" what it reviews, and a researcher cannot change a file by accident. An example role set:

| Role | Tools | Typical model | Starts sub-agents? |
|---|---|---|---|
| Researcher | Read, search, web | Mid or cheapest | One level of search helpers |
| Engineer | Read, edit, shell | Mid | No |
| Reviewer | Read, search | Strongest or another family | No |
| Architect | Read, write docs | Strongest | Researchers only |

The fields one by one, how the runtime picks a profile, and how the tool list is enforced are on the [agent profiles]({{ '/sections/08-delegation/agent-profiles.html' | relative_url }}) page. Choosing a model per role has its own page: [model choice]({{ '/sections/08-delegation/model-choice.html' | relative_url }}).

## Delegation with worktrees

### The collision problem

Two agents editing one checkout overwrite each other's files, and their changes mix in one diff that nobody can review. A git worktree is a second checkout of the same repository, on its own branch, in its own directory. Give each writing agent its own.

```bash
git fetch origin
git worktree add ../repo-agent-a -b agent-a/fix-parser origin/main
git worktree add ../repo-agent-b -b agent-b/add-tests   origin/main
```

<!-- IMAGE NEEDED: one git repository with two or three worktrees, each a separate directory on its own branch, sharing one object store; search: git worktree diagram, git worktrees multiple branches -->

Claude Code's `isolation: worktree` profile setting does this for a sub-agent: a temporary worktree branched from the default branch, removed automatically if the agent changed nothing.

### Merge back one branch at a time

1. Each agent commits on its own branch, in its own worktree.
2. The lead, or a person, reviews one branch.
   - The diff shows only that agent's change.
3. The branch is merged, or sent back with findings.
4. The next branch is reviewed against the new main.
   - A conflict shows up here, one branch at a time, where it can be resolved on purpose.

### Remove only on proof

A worktree holds work until it is merged. Remove one only after checking that its content reached the main branch and that no process still runs in it. Removing it early deletes unmerged work.

> Warning: isolation can fail. A sub-agent can commit onto its parent's branch, or write into a second repository's main checkout because only the first had a worktree. Check where the commits landed.

The full lifecycle, from creation to proved removal, is on [worktree isolation]({{ '/sections/08-delegation/worktree-isolation.html' | relative_url }}).

## Bounding the fan-out

Delegation multiplies work, so a harness bounds it in two directions.

| Bound | What it limits | Typical setting |
|---|---|---|
| Nesting | How deep the tree of agents goes | Three levels by default in Claude Code |
| Concurrency | How many agents run at once on one machine | A count per role, a role limit, or a load check before launch |

Shapes of delegation, the depth limit, three kinds of concurrency limit and what a refused launch means are on [nesting and concurrency]({{ '/sections/08-delegation/nesting-and-concurrency.html' | relative_url }}).

### Nesting

- Sub-agents can start sub-agents of their own.
- Claude Code allows three levels below the main conversation by default.
- `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` changes the limit; `1` turns nesting off.
- Each level summarises the one below and loses detail, and deep trees make it hard to see who did what.

### Concurrency

Many agents at once slow every session on the machine. What they share:

- CPU: every test run and build slows.
- Memory: processes are killed or swap.
- Test databases and ports: tests collide.
- Model rate limits: calls queue or fail.

Two bounds cover most cases: how many agents run at once, and which roles may start others.

#### Worked example: a bounded lead

One way to set the bounds, as in figure 1:

1. The lead may start up to four engineers.
   - Engineers may not start sub-agents.
   - Researchers may start one level of search helpers.
2. Before each launch, the harness reads the machine's load.
   - Free capacity: the launch goes ahead.
   - Busy: the launch is refused, and the lead works the next item itself.

## Pick the model per task

One common split, the one my harness uses. Other splits work too; the trade-offs are on [model choice]({{ '/sections/08-delegation/model-choice.html' | relative_url }}).

| Task | Tier | Why |
|---|---|---|
| Design decisions, plans, synthesis of many reports | Strongest | Mistakes here are expensive and spread |
| Implementing a clear brief | Mid | Good code at lower cost |
| Bulk reading, searching, classifying | Cheapest | Volume work with checkable output |
| Review of another model's work | A different family | Does not share the author's blind spots |

`/model` and `/effort` set the model and reasoning effort for a session; a profile's `model` field fixes it for a role. More, with a worked example across four tiers: [model choice]({{ '/sections/08-delegation/model-choice.html' | relative_url }}).

## Check delegated work

### Reports are claims

Two ways a report misleads:

| Failure | Looks like | Check |
|---|---|---|
| Work reported but not done | "Fixed; all tests pass", with an empty diff | `git log` and `git diff --stat` on the agent's branch |
| A dropped caveat | "The API supports batching", without "only on the paid tier" | Read the key passage of the cited source |

#### Worked example: catching a false report

```text
sub-agent> Fixed the race; all 214 tests pass.
lead$ git -C ../repo-agent-a log --oneline -1
9b02f1d chore: start branch            <- the branch's first commit, nothing new
lead$ git -C ../repo-agent-a diff --stat
                                       <- empty: no file changed
lead> Returned with the evidence: no commit and no diff on your branch.
```

The report described a plan; no work was done.

In the systems approach these checks are built in: a collection step in the delegation workflow runs them on every report, and a guard can refuse to merge a branch whose report quotes no test output. The full checking routine, with a decision path and the research case, is on [checking delegated work]({{ '/sections/08-delegation/checking-delegated-work.html' | relative_url }}).

## Examples from my own harness

- Profiles by role: analyst, architect, auditor, engineer, planner, researcher and reviewer. Read-only roles have no write tools, and a check fails the build if a profile's stated authority and its tools disagree.
- A launch guard refuses a writing sub-agent without its own worktree; another reads the machine's load before each launch.
- Codex reviews what Claude writes (see [Code review]({{ '/sections/13-code-review/' | relative_url }})).
- Each commit records who did it in trailers: family, role, model, effort, profile and worktree.
- A worktree is removed only after an audit proves its content reached the main branch.

To start: one read-only researcher profile and one brief template.

## Pitfalls

| Pitfall | What happens | Fix in the system |
|---|---|---|
| Trusting a report | A fix is reported with no change made | The lead's collection step checks commits, diff and test output every time |
| Trusting a summary | The caveat that changes the plan is gone | Read the cited passage before deciding on it |
| Isolation fails | Commits land on the parent's branch or another repository's main checkout | A launch guard that requires a worktree in every repository the agent writes to |
| Unbounded fan-out | Every session on the machine slows | Role limits in profiles and a load check before each launch |
| Limits only in the brief | An agent does what its profile allows | Put safety limits in the profile's `tools` |

## Pages in this section

- [Agent profiles]({{ '/sections/08-delegation/agent-profiles.html' | relative_url }}): fields, how a profile is picked and enforced, roles, and recording who did what.
  - [Model choice]({{ '/sections/08-delegation/model-choice.html' | relative_url }}): tiers, effort, and cross-family review.
- [Worktree isolation]({{ '/sections/08-delegation/worktree-isolation.html' | relative_url }}): why, the lifecycle, merging back, removal, and how isolation fails.
- [Nesting and concurrency]({{ '/sections/08-delegation/nesting-and-concurrency.html' | relative_url }}): hub, tree and team shapes, the depth limit, and count, role and load limits.
- [Checking delegated work]({{ '/sections/08-delegation/checking-delegated-work.html' | relative_url }}): false reports, dropped caveats, and the checking routine.

## Sources

- Sub-agent profiles, fields, `isolation`, nesting depth: <https://code.claude.com/docs/en/sub-agents.md>, verified 2026-11-09.
- `/tasks`, `/subtask`, `/fork`, `/model`, `/effort`: <https://code.claude.com/docs/en/commands.md>, verified 2026-11-09.
- Agent teams: <https://code.claude.com/docs/en/agent-teams.md>, verified 2026-11-08.
- Codex sub-agents: <https://learn.chatgpt.com/docs/agent-configuration/subagents.md>, verified 2026-11-08.
- The profiles, launch guards, trailers and worktree audit are from the author's own harness, described conceptually. The other examples are illustrations written for this page. Figure 1 was drawn with archify for this page.

{% include page_nav.html %}
