---
title: "Guard hooks"
parent: "Harness hooks"
grand_parent: "What is a harness?"
nav_order: 2
redirect_from:
  - /sections/02-ai-harnesses/guard-hooks.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Guard hooks

A guard is a `PreToolUse` hook that refuses an action as the agent tries it. This page covers when to write a guard, the ways a guard can answer, how to word its refusal, patterns that keep it accurate, and a worked example from my harness.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## When a rule becomes a guard


![A vertical flow chart. One agent action enters at the top and moves down through four layers. Layer 1, instructions in context files, skills and prompts, is read by the model and guides it. Layers 2 to 4 are enforced by a program: workflows and tools refuse invalid input, hooks, scripts and linters refuse or flag the action as it happens, and validation and tests fail the change and send it back. At each layer a side branch shows what that layer caught, and what it missed passes down: a rule the model dropped, an action outside the workflow, a mistake not visible yet. A change that passes all four layers lands.]({{ '/assets/images/diagrams/02-ai-harnesses/enforcement-flow.svg' | relative_url }})
*Figure: one agent action moving down the four layers. A guard is layer 3, between the workflows above it and the tests below. Made with archify.*

![The Swiss cheese model with the four layers of a harness labelled. An agent action enters at the left and passes four slices in order: 1, instructions in context files, skills and prompts, which guide because the model reads them; 2, workflows and tools, with the steps built in; 3, hooks, scripts and linters, which check at the moment of action; 4, validation and tests before a change lands. Layers 2 to 4 are enforced by a program. Each slice has holes; where the holes in every slice line up, a bad change lands.]({{ '/assets/images/diagrams/02-what-is-a-harness/swiss-cheese-harness-layers.svg' | relative_url }})
*Figure: the Swiss cheese model applied to a harness. Each layer catches some mistakes, and a bad change lands only where the holes in every layer line up. Adapted from [Swiss cheese model of accident causation](https://commons.wikimedia.org/wiki/File:Swiss_cheese_model_of_accident_causation.png) by Davidmack, after James Reason, [CC BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0/); labels added, shared under the same licence.*

*The Swiss cheese model of layered defence (James Reason). Each layer catches some mistakes; what gets through one layer is caught by the next. Source: [Swiss cheese model of accident causation on Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Swiss_cheese_model_of_accident_causation.png), by Davidmack, [CC BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0/). Resized for this page.*

A guard is layer 3: it acts at the moment of action, after the instructions and workflows of layers 1 and 2, and before the tests of layer 4. The [full ladder]({{ '/sections/02-what-is-a-harness/guides-sensors-and-enforcement.html' | relative_url }}#the-layers) shows what each layer catches and misses.

Under the [systems approach]({{ '/sections/02-what-is-a-harness/' | relative_url }}#a-systems-approach-to-harness-engineering), a rule that matters is enforced in code from the start where it can be, without waiting for agents to break it. A guard is the right place when both of these are true.

1. Breaking it is costly or hard to undo: lost work, a leaked secret, someone else's process stopped.
2. The bad action can be recognised from the tool input alone, reliably enough that false refusals are rare.

If the second is false, the rule belongs in a later layer of the [enforcement ladder]({{ '/sections/02-what-is-a-harness/guides-sensors-and-enforcement.html' | relative_url }}#the-enforcement-ladder): a test or a review step that sees the whole result.

## Refuse, ask or rewrite

A guard has more than one way to answer.

| Answer | How | When |
|---|---|---|
| refuse | exit 2 with the reason on standard error, or JSON `permissionDecision: "deny"` with `permissionDecisionReason` | the action is wrong in every case |
| ask the person | JSON `permissionDecision: "ask"` | the action is risky only in some cases the hook cannot tell apart |
| rewrite | JSON `updatedInput` with safer arguments, such as an added `--dry-run` | a safe form of the same action exists |
| let it through | exit 0 with no output | everything else |

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Refused: pkill can stop processes that are not yours. Find the PID with `ps -eo pid,etime,args`, then `kill <pid>`."
  }
}
```

A guard's allow never overrides a permission deny or ask rule, so a guard can only make the harness stricter.

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

1. Split the command on `;`, `&&`, `||`, `|`, `&` and newlines.
2. In each segment, skip `sudo`, `env` and variable assignments to find the program.
3. Strip its path, so `/usr/bin/pkill` reads as `pkill`.
4. Judge the program and its flags.
5. Refuse what you cannot parse, such as `eval` or a command substitution, if the guard protects something costly.

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

A `PreToolUse` hook on Bash, shown in full on the [Harness hooks]({{ '/sections/02-what-is-a-harness/hooks.html' | relative_url }}#a-complete-guard) page.

1. It splits the command into segments.
2. It finds each segment's program, past `sudo`, `env` and a full path.
3. It refuses any segment whose program is `pkill` or `killall`.
4. Its refusal tells the agent to find the process ID and kill that one process.
5. It exits 0 for everything else.

Its tests cover both arms: `pkill -f pytest`, `cd x && pkill y`, `sudo /usr/bin/pkill node` and a pattern kill after a newline are refused; `kill 41822`, `echo 'pkill is banned'` and `git status; npm test` are allowed.

### The result

| | Before | After |
|---|---|---|
| Rule location | one line in a long context file | a hook that runs on every shell command |
| What the agent sees | nothing at the moment of the mistake | a refusal with the safe command |
| Outcome | other sessions' runs stopped | the agent kills its own process by id |

## Guards and permissions together

| Need | Better tool | Why |
|---|---|---|
| never read `.env` with the `Read` tool | permission `deny` rule `Read(./.env)` | a pattern on a path is enough |
| never read `.env` by any route, `cat` included | the sandbox's read limits | it acts on the file, whatever the command |
| never kill by pattern, in any spelling | guard hook | needs parsing |
| ask before any push | permission `ask` rule `Bash(git push *)` | a prefix is enough |
| never push to `main` from a sub-agent's worktree | guard hook | needs the git state and the session's role |

## Pitfalls

| Pitfall | Result | See |
|---|---|---|
| the guard crashes with exit 1 | the action runs | [fail open by accident]({{ '/sections/02-what-is-a-harness/hook-failure-modes.html' | relative_url }}#fail-open-by-accident) |
| the guard is merged but not installed | nothing runs it | [never installed]({{ '/sections/02-what-is-a-harness/hook-failure-modes.html' | relative_url }}#never-installed) |
| the same action through another tool | the guard never sees it | [matcher misses]({{ '/sections/02-what-is-a-harness/hook-failure-modes.html' | relative_url }}#matcher-misses) |
| a vague refusal | the agent tries the next way around | [the refusal message](#the-refusal-message) |

## Sources

- [Claude Code hooks, `PreToolUse` decision control](https://code.claude.com/docs/en/hooks.md), verified 2026-11-09.
- [Hooks and permission rules](https://code.claude.com/docs/en/permissions.md), verified 2026-11-09.
- [Swiss cheese model of accident causation](https://commons.wikimedia.org/wiki/File:Swiss_cheese_model_of_accident_causation.png), Wikimedia Commons, by Davidmack, [CC BY-SA 3.0](https://creativecommons.org/licenses/by-sa/3.0/).
- The incident, guards and refusal texts are from the author's own harness, described conceptually.

{% include page_nav.html %}
