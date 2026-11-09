---
title: "Sessions and state"
parent: "Agentic AI fundamentals"
nav_order: 4
---

{% include page_nav.html %}

# Sessions and state

As of 2026-11-09.

A model keeps nothing between calls, so everything an agent "remembers" is something the client sends again or something written to a file. A session is the client's record of one conversation. The [Agentic AI fundamentals overview]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }}#sessions-and-state) is the short version, and [Context window and memory]({{ '/sections/11-context-window/' | relative_url }}) covers memory systems in more depth. This page covers:

- what a session holds, and what it does not;
- resuming, continuing, branching and forking a session in Claude Code;
- handoff notes, and how to make sure they get written;
- where each kind of state belongs, from a plan file to a check in CI.

> The model keeps nothing between sessions. Work carries over through files the next session reads: context files, plan files, memory files and git history.
{: style="border-left: 4px solid #0f766e; background: #f0fdfa; padding: 0.75rem 1rem;"}

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What a session is

### The transcript

Claude Code records each session as a transcript on disk: every message, tool call and tool result, in order.

| The transcript lets you | How |
|---|---|
| scroll back through the session | the terminal history, or the transcript file |
| resume it later | `claude --continue` or `claude --resume` |
| audit what the agent did | read the tool calls and their results |

After compaction, the live context holds a summary while the full record stays on disk. The agent works from the summary; only you, or a tool that reads the file, can see the rest.

### What a session holds and what it does not

| Held in the session | Not held anywhere unless written down |
|---|---|
| the conversation and every tool result | decisions you made in your head |
| the current working directory and environment | why a design was chosen, unless it was said |
| the task list the agent keeps for itself | what a later session should do next |
| the model and settings in use | anything a compaction summary dropped |

### Sessions and the model

```text
turn 1:  client sends [system, files, task]                     -> model replies
turn 2:  client sends [system, files, task, reply 1, result 1]  -> model replies
turn 3:  client sends [system, files, task, ... , result 2]     -> model replies
```

The model sees a longer list each time; it has no memory of turn 1 other than turn 1 being in the list. Close the session and the list is gone from the model's point of view.

## Resuming, continuing and forking

### The commands

| Goal | Claude Code | What you get |
|---|---|---|
| Pick up the most recent session in this directory | `claude -c` (`--continue`) | the same conversation carries on; sessions started with `claude -p` are skipped |
| Pick a particular earlier session | `claude -r` (`--resume`) with an ID or name, `claude -r` alone for a picker, or `/resume` inside a session | the chosen conversation |
| Name a session so it is easy to find | `claude -n "orders-migration"`, or `/rename` mid-session | the name appears in `/resume` and the terminal title |
| Try another direction without losing this one | `/branch` | a branch of the conversation at this point; you switch into it, and the original stays available through `/resume` |
| Resume, but as a new session | `claude -r <id> --fork-session` | a copy with a new session ID; the original is untouched |
| Hand a copy to a background agent | `/fork` | the conversation copied into a new background session while you keep working here |
| Start clean | `/clear`, optionally with a name for the old conversation | an empty conversation; project memory and context files still load |

Two things come back on resume without being in the transcript's text:

- an active `/goal` is restored, with its turn count and timer reset;
- the system prompt recorded on the conversation's first request is reused until the next compaction.

### When to resume and when to start fresh

| Situation | Resume | Start fresh |
|---|---|---|
| Same task, short transcript | yes: errors seen and files read are still useful | |
| Different task | | yes, so the old task's history does not crowd the new one |
| Near the window limit, or compacted several times | | yes, with a handoff note |
| The session went down a wrong path | | yes; or `/rewind` to before the wrong turn |

> A resumed session still contains its earlier mistakes. If the last session went down a wrong path, start a fresh session with a handoff note.
{: style="border-left: 4px solid #b91c1c; background: #fef2f2; padding: 0.75rem 1rem;"}

## Handoff notes

### Why write one

A handoff note is a short file the next session reads first. It carries work across sessions more reliably than the transcript:

- it survives compaction, closing the terminal and switching machines;
- it holds only what the next session needs, so it costs little context;
- a person can read and correct it too.

![Sequence diagram: in the afternoon you ask session 1 to migrate the orders schema; context files load; it commits tables A and B; you ask it to wrap up and it writes the plan file with done, next and watch out. The next morning you ask session 2 to continue; it loads the context files and the plan file, so it knows the next table and the nullable column; it runs make migrate for orders and updates the plan file]({{ '/assets/images/01-agentic-ai-fundamentals/session-handoff.svg' | relative_url }}){: style="width: 100%" }
*Figure: two sessions on one task. The only link between them is what session 1 wrote to disk.*

### What goes in it

```markdown
# Migration: orders schema v3

## Done
- Tables `customers` and `addresses` migrated; tests green.

## Next
- Migrate `orders`. Run `make migrate TABLE=orders` then `make test-db`.

## Watch out
- `orders.customer_id` is nullable in production data (about 2% of rows).
  The migration must keep the null values.

## Decisions
- Kept the old column for one release so rollback stays possible.
```

| Heading | What it holds | Why the next session needs it |
|---|---|---|
| Done | finished steps, with how they were checked | so it does not redo or re-verify them |
| Next | the next step and the exact command | so it can start in the first minute |
| Watch out | traps found along the way | so it does not rediscover them by failing |
| Decisions | choices made and their reasons | so it does not reopen them |

Plan mode writes a plan to disk too, and Claude Code re-injects that plan after compaction.

### Worked example: resuming the next morning

An agent spends an afternoon on a migration and stops halfway, after writing the note above.

1. The next morning you start a fresh session and say "continue the migration".
2. The session loads the context files and reads the plan file first.
3. From the note it knows:
   - which table is next, `orders`;
   - the exact command, `make migrate TABLE=orders`;
   - the trap, about 2% of rows with a null `customer_id`.
4. It runs the migration, keeps the null values, and updates the plan file.

Without the note, step 3 becomes reading code and git history, and the null values are found when the migration fails.

### Making it automatic

The [systems approach]({{ '/' | relative_url }}#a-systems-approach) applies here as anywhere: the note is written reliably when a layer below the instruction asks for it.

| Layer | Mechanism | How reliable |
|---|---|---|
| Instruction | a line in `CLAUDE.md`: "before stopping, update the plan file with done, next, watch out and decisions" | most of the time |
| Workflow | a wrap-up skill that writes the note in the fixed format | whenever it is run |
| Hook | a `Stop` hook that blocks the stop when the plan file has not changed this session; a `PreCompact` hook that reminds the agent before compaction | every time it fires |

<details markdown="1">
<summary>A Stop hook that asks for the note</summary>

```bash
#!/bin/bash
INPUT=$(cat)
[ "$(echo "$INPUT" | jq -r '.stop_hook_active')" = "true" ] && exit 0
# block the stop if there is uncommitted work but PLAN.md has not changed
other_changes=$(git status --porcelain | grep -v ' PLAN.md$')
if [ -n "$other_changes" ] && git diff --quiet -- PLAN.md; then
  echo '{"decision": "block", "reason": "Update PLAN.md with done, next, watch out and decisions before stopping."}'
fi
exit 0
```

The `stop_hook_active` check lets the agent stop on the second attempt, so the hook cannot loop. [How an agent run ends]({{ '/sections/01-agentic-ai-fundamentals/how-a-run-ends.html' | relative_url }}#a-stop-hook) explains Stop hooks.
</details>

## Where each kind of state belongs

### The map

| Kind of state | Lifetime | Best home | Loaded how |
|---|---|---|---|
| A rule that must hold | as long as the project | a check: a hook, a lint rule, a test, a CI job | runs whatever the context holds |
| Project conventions and knowledge | months | the project `CLAUDE.md` or `AGENTS.md` | automatically at every start |
| Personal preferences | months | `~/.claude/CLAUDE.md`, or `CLAUDE.local.md` for one project | automatically |
| Corrections and preferences Claude picks up | weeks to months | auto memory, which Claude writes itself, per repository | the first 200 lines or 25 KB at every start; `/memory` to view and edit |
| Current task progress | days | a plan file or task tracker | read first by the next session |
| What changed and why | permanent | git commits and their messages | `git log` when needed |
| The detailed record of one run | as long as you keep it | the session transcript | resume, or read for an audit |

A lesson learned from a mistake starts in a plan file or auto memory. If it must hold for every session, it moves up to a check, and the context-file line stays as the explanation.

### Worked example: one fact, three places

A session discovers that the test database must be reset before the integration suite, or tests fail at random.

1. For the task in hand, it goes in the plan file under "watch out", so the next session on this task knows.
2. Because it is true for every task in the project, the fix goes into the system:
   - the `test-integration` target resets the database itself, so no session can run the suite against a dirty database;
   - a CI job runs the same target, so a change that breaks the reset fails before it lands.
3. A one-line note in the project's context file stays as a second layer: "the integration target resets the test database; do not run the suite by hand".

Writing only a new context-file rule ("run `make reset-test-db` before `make test-integration`") would rely on every session remembering it. Building the step into the target removes the need to remember. The introduction describes this [systems approach]({{ '/' | relative_url }}#a-systems-approach).

## Pitfalls

- Relying on the transcript as memory. Compaction and fresh sessions lose it; write decisions, next steps and traps to files.
- Resuming a confused session. Start fresh with a handoff note, or `/rewind` to before the wrong turn.
- Notes without commands. "Continue the migration" leaves the next session to work out how; "run `make migrate TABLE=orders`" does not.
- State in two places that disagree. If the plan file and the tracker say different things, the next session may follow either.
- Editing `CLAUDE.md` mid-session and expecting the running session to see it. The edit loads on the next `/clear`, `/compact` or restart.
- A gitignored `CLAUDE.local.md` exists only in the worktree where you created it; import a file from your home directory to share it across worktrees.

## Sources

- `--continue`, `--resume`, `--fork-session`, `--name`, and system prompts in resumed conversations: <https://code.claude.com/docs/en/cli-reference.md>, verified 2026-11-09.
- `/resume`, `/branch`, `/fork`, `/rename`, `/clear`, `/rewind`, `/memory`: <https://code.claude.com/docs/en/commands.md>, verified 2026-11-09.
- `CLAUDE.md` levels, `CLAUDE.local.md`, auto memory and its loading limit: <https://code.claude.com/docs/en/memory.md>, verified 2026-11-09.
- A goal restored on resume: <https://code.claude.com/docs/en/goal.md>, verified 2026-11-09.
- `CLAUDE.md` edits taking effect after `/clear`, `/compact` or restart: <https://code.claude.com/docs/en/prompt-caching.md>, verified 2026-11-09.
- The plan-mode plan re-injected after compaction: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-09.
- The handoff note, hook script, transcript sketch and examples are illustrations written for this page.

{% include page_nav.html %}
