---
title: "Guard hooks"
parent: "Harness hooks"
grand_parent: "AI harnesses"
nav_order: 2
---

{% include page_nav.html %}

# Guard hooks

As of 2026-11-09. A sub-topic of [Harness hooks]({{ '/sections/02-ai-harnesses/hooks.html' | relative_url }}), in [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}).

A guard is a `PreToolUse` hook that refuses an action. It acts on the agent's action as it happens, and a guard that refuses too much blocks good work. This page covers when to write a guard, how to write its refusal, and how to keep it from blocking good work.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## When a rule becomes a guard

![The enforcement ladder, from context rule to path rule to skill to hook or guard to test or CI gate.]({{ '/assets/images/02-ai-harnesses/enforcement-ladder.svg' | relative_url }})

Under the [systems approach]({{ '/sections/02-ai-harnesses/' | relative_url }}#a-systems-approach-to-harness-engineering), a rule that matters is enforced in code from the start where it can be, without waiting for agents to break it. A guard is the right place when both of these are true.

1. Breaking it is costly or hard to undo: lost work, a leaked secret, someone else's process stopped.
2. The bad action can be recognised from the tool input alone, reliably enough that false refusals are rare.

If the second is false, the rule belongs further up the [enforcement ladder]({{ '/sections/02-ai-harnesses/guides-sensors-and-enforcement.html' | relative_url }}#the-enforcement-ladder), in a test or a review step that sees the whole result.

## The refusal message

### Three parts

The refusal is the only thing the model learns from a guard. A good one has three parts.

| Part | Example |
|---|---|
| What was refused | "Refused: this command prints environment variables." |
| Why | "Their values include credentials, and the transcript is stored." |
| The safe way | "To check a variable is set, run `[[ -n \"${NAME:-}\" ]] && echo set`." |

### Before and after

```text
BEFORE  Blocked by policy.
AFTER   Refused: `git -C <shared checkout>` would change another session's tree.
        Run the same command from your own worktree, without -C.
```

The first message tells the agent it hit a wall and nothing more, so it tries the next way around. The second tells it the exact command that will work, and the next call succeeds.

## Patterns that work

### Parsing the command

A guard that searches the raw command string misses spellings and blocks innocent text.

| Command | String search for `rm -rf` | Parsed check |
|---|---|---|
| `rm -r -f build/` | missed | caught: `rm` with recursive and force flags |
| `echo "never rm -rf"` | wrongly refused | allowed: the program is `echo` |
| `cd x && rm -rf y` | caught | caught: each segment checked |

Split the command on `;`, `&&`, `||` and `|`, find the program in each segment, and judge its flags.

### Allowlist the safe form

Where a dangerous command has a safe variant, allow the variant by name.

- `rm -r` inside the session's own scratch directory: allowed.
- `rm -r` anywhere else: refused, with the scratch path in the message.
- `kill <pid>` for one process: allowed.
- `pkill` and `killall`: refused, with `ps -eo pid,etime,args` as the way to find the PID.

### Refusing what it cannot judge

When the command is too complex to analyse (nested subshells, `eval`, a long pipeline that changes directory), refuse with a message asking for simpler commands. The isolation guard in my harness does exactly this:

```text
... this command is too complex to verify that it stays inside the worktree.
Refusing to run it. Split it into plain, separate commands ...
```

The cost is an extra turn; the gain is that no clever command can slip past.

### Scope the guard

Some guards apply only to some sessions.

| Guard | Active for |
|---|---|
| worktree isolation | sub-agents given their own worktree |
| destructive-command guard | every session |
| secret guard | every session |
| publish guard | sessions working on public files |

My harness's isolation guard is active only for sub-agents, so a person's own session can still work across the repository.

## A worked example

### The incident

On a shared machine, an agent ran a pattern kill to stop its own test run.

- The pattern matched other sessions' test runs too, and they died mid-run.
- The context file already said "never kill by pattern".
- Nothing in the system enforced the rule at the moment of the command.

### The guard

A `PreToolUse` hook on Bash, shown in full on the [Harness hooks]({{ '/sections/02-ai-harnesses/hooks.html' | relative_url }}#a-complete-guard) page.

1. It splits the command into segments.
2. It refuses any segment whose program is `pkill` or `killall`.
3. Its refusal tells the agent to find the process ID and kill that one process.
4. It exits 0 for everything else.

### The result

| | Before | After |
|---|---|---|
| Rule location | one line in a long context file | a hook that runs on every shell command |
| What the agent sees | nothing at the moment of the mistake | a refusal with the safe command |
| Outcome | other sessions' runs stopped | the agent kills its own process by id |

<details markdown="1">
<summary>Why the context-file line was not enough</summary>

The agent was deep in a long task, the line was one of many, and the action looked routine. A written rule has to be recalled at the right moment; a guard runs at that moment every time. The line stays in the context file, now with a note that a hook enforces it, so the agent knows why the refusal will come.

</details>

## Guards and permissions together

| Need | Better tool |
|---|---|
| Never read `.env` | permission `deny` rule |
| Never kill by pattern, in any spelling | guard hook (needs parsing) |
| Ask before any push | permission `ask` rule |
| Never push to `main` from a sub-agent's worktree | guard hook (needs git state) |

## Sources

- Claude Code hooks, `PreToolUse` decision control: <https://code.claude.com/docs/en/hooks.md>, verified 2026-11-08.
- The incident, guards and refusal texts are from the author's own harness, described conceptually.

{% include page_nav.html %}
