---
title: "Nesting and concurrency"
parent: "Agent profiles, sub-agents and delegation"
nav_order: 3
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Nesting and concurrency

A lead agent can start several sub-agents at once, and a sub-agent can start sub-agents of its own. Both multiply work, and both need limits: nesting limits how deep the tree of agents goes, and concurrency limits how many agents run at the same time on one machine. This page covers the shapes delegation takes, the depth limit and what each level costs, three kinds of concurrency limit, and what a lead does when a launch is refused.

![A sequence across five participants. The lead asks the launch checks to start a researcher at depth 1. The researcher asks for a search helper, which is allowed for its role, and the helper starts at depth 2, returns twelve matches, and the researcher answers the lead with sources. An engineer asks for a helper and is refused because its role may not start agents. The lead asks for a fifth engineer and is refused because four are running, so the lead works the item itself until an engineer reports and a slot frees.]({{ '/assets/images/08-delegation/fanout-bounds.png' | relative_url }}){: style="width:100%"}

*Figure 1. Role, depth and count limits applied at launch. Built with archify.*

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Shapes of delegation

### Hub

One lead, several sub-agents, no deeper levels. Every result comes back to the lead, which checks it and decides what happens next.

| Fits when | Costs |
|---|---|
| The parts are independent and the lead can check each one | The lead's context holds every summary |

### Tree

Some sub-agents start helpers of their own: a researcher starts search helpers; a coordinator starts engineers. Results flow up one level at a time.

| Fits when | Costs |
|---|---|
| A part is itself large enough to split | Each level summarises the one below and loses detail; harder to see who did what |

### Team

Agents message each other directly and share a task list, with a lead that coordinates. In Claude Code this is agent teams, which are experimental and off by default.

```json
{
  "env": { "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1" }
}
```

| Part | Role |
|---|---|
| Team lead | The main session; spawns teammates and coordinates work |
| Teammates | Separate Claude Code instances, each with its own context window |
| Task list | Shared work items with states pending, in progress and completed; tasks can depend on others |
| Mailbox | Messages between agents, delivered automatically |

| Fits when | Costs and limits |
|---|---|
| Parallel research or review, where teammates challenge each other's findings | Token use scales with the number of teammates |
| New modules where each teammate owns separate files | One team per session; teammates cannot start teams of their own |
| Debugging with competing hypotheses | Experimental: in-process teammates are not restored on resume, and task status can lag |

- The documentation suggests starting with three to five teammates.
- Three hooks enforce quality gates on a team. With exit code 2, `TeammateIdle` sends feedback and keeps the teammate working, and `TaskCreated` or `TaskCompleted` sends feedback and stops the task from being created or marked complete.
- With agent teams enabled, a sub-agent that Claude names launches as a teammate, so teams can form when nobody asked for one. Setting the variable to `0` turns that off.

<!-- IMAGE NEEDED: sub-agents reporting back to one main agent beside an agent team whose teammates share a task list and message each other; search: subagents vs agent teams diagram, claude code agent teams (candidate: the comparison diagram on the Claude Code agent teams docs page; check its reuse terms first) -->

> Note: these shapes are options, and a harness can use more than one. Most delegation today is hub-shaped, with a tree where one part is large.

## Nesting

### The depth limit

| Setting | Effect |
|---|---|
| Default | A sub-agent may start sub-agents up to three levels below the main conversation |
| `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH=2` | Caps nesting at two levels |
| `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH=1` | Turns nesting off |

### What each level costs

A level adds a summary between the work and the person. Following one fact up a three-level tree:

1. A search helper at depth 2 finds a function and notes it is only called in tests.
2. The researcher at depth 1 summarises twelve helpers' results.
   - The "only in tests" note becomes one clause among many.
3. The lead at depth 0 reads the researcher's summary.
   - The clause may be gone, and the lead plans as if the function were live.

The check at each level is the same as for any delegation: read the artifact the summary describes. See [checking delegated work]({{ '/sections/08-delegation/checking-delegated-work.html' | relative_url }}).

### Which roles may nest

A limit by role is often clearer than a limit by depth alone. One way to set it:

| Role | May start sub-agents | Why this choice |
|---|---|---|
| Lead or coordinator | Yes, engineers and researchers | Splitting work is its job |
| Researcher | One level of search helpers | Searches split cleanly and are easy to check |
| Engineer | No | Its work is one change in one worktree |
| Reviewer | No | Its reading should stay its own |

The role limit is enforced the same way as any other tool limit: a role whose [profile]({{ '/sections/08-delegation/agent-profiles.html' | relative_url }}) lacks the tool that starts agents cannot start one.

## Concurrency

### What agents share

Agents in separate [worktrees]({{ '/sections/08-delegation/worktree-isolation.html' | relative_url }}) have separate files. They still share the machine.

| Shared resource | What happens when many agents run |
|---|---|
| CPU | Every test run and build slows, in every session |
| Memory | Processes are killed or swap, and runs fail for reasons unrelated to the change |
| Test databases and ports | Tests collide and fail intermittently |
| Model rate limits | Calls queue or fail |

### Three kinds of limit

#### A count limit

- At most N agents of a role at once, for example four engineers.
- Easy to state and to check.
- Blind to what each agent is doing: four idle agents count the same as four running test suites.

#### A role limit

- Which roles may start other agents, as in the role table above.
- Bounds the width of the tree at each level.
- Enforced by the profile: a role without the launch tool cannot launch.

#### A load check

1. Before each launch, read the machine's current contention with one shared check.
2. Free capacity: launch.
3. Busy: refuse, and the launching session does the item itself.
   - The check sees other people's sessions too, which a count limit cannot.

### What a refused launch means

A refused launch is work the lead now does itself. In figure 1:

1. The lead asks for a fifth engineer.
2. The launch checks refuse: four engineers are running.
3. The lead works the item inline.
   - When an engineer reports and its slot frees, the next item can go out again.

Waiting idle for a slot is the other option. It wastes the lead's time when there is work it could do.

<details markdown="1">
<summary>Measuring load: what to read and what to avoid</summary>

A load average on its own mixes waiting on disk with waiting on CPU, and a slot count ignores what each slot is doing. A purpose-built check that reads the machine's actual contention, and is called the same way before every launch, gives every session the same answer.

</details>

## Worked example: a migration with bounds

A lead plans a database migration with these bounds: up to four engineers, researchers may start one level of helpers, engineers may not start agents, and a load check before each launch.

1. The lead starts a researcher to list every query in the service.
   - The researcher starts three search helpers, one per package.
   - Each helper returns a table; the researcher merges them and cites each row.
2. The lead starts three engineers, one per package, each in its own worktree.
3. The load check refuses a fourth launch while the three run their tests.
   - The lead writes the migration's rollback notes itself in the meantime.
4. Engineers report one at a time; the lead checks each branch and merges it.

## Examples from my own harness

- Each agent profile states whether that role may start sub-agents.
  - A build check fails if a profile's stated authority and its tools disagree.
- Before each launch, the harness reads the machine's contention with one shared check.
  - Busy: the launching session works its next item itself.
- A coordinator's whitelist bounds which roles it may start, and a guard refuses anything outside it.

## Related pages

- [Agent profiles]({{ '/sections/08-delegation/agent-profiles.html' | relative_url }}), where role limits are set.
- [Coordinating several sessions]({{ '/sections/09-session-coordination/' | relative_url }}), when the agents are separate sessions.
- [Orchestration and workflows]({{ '/sections/10-orchestration/' | relative_url }}), for longer multi-stage runs.

## Sources

- Nesting depth and `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`: [Claude Code docs: sub-agents](https://code.claude.com/docs/en/sub-agents.md), verified 2026-11-09.
- Agent teams, enabling them, task list, hooks and limitations: [Claude Code docs: Orchestrate teams of Claude Code sessions](https://code.claude.com/docs/en/agent-teams.md), verified 2026-11-09.
- The role limits and the contention check are from the author's own harness, described conceptually. The examples are illustrations written for this page.

{% include page_nav.html %}
