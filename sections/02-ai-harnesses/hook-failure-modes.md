---
title: "Hook failure modes"
parent: "Harness hooks"
grand_parent: "AI harnesses"
nav_order: 3
---

{% include page_nav.html %}

# Hook failure modes

As of 2026-11-09. A sub-topic of [Harness hooks]({{ '/sections/02-ai-harnesses/hooks.html' | relative_url }}), in [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}).

A hook is code, and code fails. A failed guard leaves everyone believing the rule is enforced when it is not. This page lists the ways hooks fail and the check that catches each one.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The failure modes at a glance

| Failure | What it looks like | Check |
|---|---|---|
| Fails open by accident | the guard crashes and the action runs | test that a malformed event still refuses |
| Never installed | the guard exists in the repository and nowhere else | `/hooks` on a live session |
| Matcher misses | the guard is on `Bash` and the action came through an MCP tool | list every tool that can do the action |
| False positives | good commands refused, agents learn to route around | log refusals and read them |
| Bypass | the same action, spelled differently | parse, and test the alternate spellings |
| Loops | a Stop hook that never lets go | honour the already-sent-back flag |
| Cost | a slow hook on every tool call | time it; keep it in milliseconds |
| Noise | context hooks that add text every turn | measure what they add |

![Where hooks fire in a session: SessionStart, UserPromptSubmit, the PreToolUse, tool and PostToolUse loop, Stop, PreCompact, SubagentStop and SessionEnd.]({{ '/assets/images/02-ai-harnesses/hook-lifecycle.svg' | relative_url }})

*Each event is a place a hook can fail. Guards on PreToolUse fail open on a crash; Stop hooks can loop.*

## Fail open by accident

### The mechanism

Only exit code 2, or an explicit deny in JSON, blocks. A Python guard that raises an exception exits with 1, which the harness treats as the hook failing, and the action continues.

```text
$ echo 'not json' | python3 guard.py; echo "rc=$?"
json.decoder.JSONDecodeError: Expecting value: line 1 column 1 (char 0)
rc=1          <- the tool call runs
```

### The fix

Wrap the guard so any unexpected error refuses, for guards where refusing is the safe default.

```python
try:
    main()
except SystemExit:
    raise
except Exception as exc:
    print(f"Guard error, refusing to be safe: {exc}", file=sys.stderr)
    sys.exit(2)
```

> Decide per guard whether failing closed is right. A guard on every shell command that fails closed on a bug stops all work until it is fixed. That is often the correct trade for a secrets guard and the wrong one for a style hint.
{: .important }

## Never installed

### The mechanism

1. The guard is written, tested and merged.
2. The installer that copies hooks into place was never run on the machine, or ran from a branch without it.
3. Every check of the repository says the guard exists. No session runs it.

### The check

Measure the live target.

```text
> /hooks
```

- Run it in a live session after every install, on every machine.
- My harness learned this more than once: a guard pinned by a test can still be live in zero places.
- Its release check now names the one artifact whose state proves the change reached its target, and reads it.

## Matcher misses

### The mechanism

A guard on `Bash` refuses deleting files with `rm`. The same action has other routes.

| Route | Guarded |
|---|---|
| `Bash` with `rm` | yes |
| an MCP server's file-delete tool | no |
| `Write` with empty content | no |

### The check

1. For each guarded action, list every tool that can perform it, MCP tools included.
2. Widen the matcher (`Bash|Write|mcp__files__.*`) or add a guard per route.
3. Add a test per route, so a new tool that can do the action is noticed.

## False positives and route-arounds

### The mechanism

A guard refuses too much, and the agent, being helpful, finds another way.

- It writes the command into a script file and runs the script.
- It uses a different tool that does the same thing.
- The guard now teaches evasion, and the workaround becomes a habit.

### The check

1. Log every refusal with the command that caused it.
2. Read the log weekly.
3. A refusal of a safe command is a bug in the guard; fix it the same day.
4. Add the safe command to the guard's tests as a must-allow case.

## Bypass by spelling

| Intended block | Spellings that slip past a string match |
|---|---|
| `rm -rf dir` | `rm -r -f dir`, `rm --recursive --force dir`, `find dir -delete` |
| `pkill node` | `killall node`, `kill $(pgrep node)` |
| `env` | `printenv`, `export -p`, `declare -x`, `set` |

Each row is a test case. My harness's secret guard keeps a list of every spelling that prints the environment, and a shell's tracing mode and a throwaway subshell are on that list too.

## Loops

A Stop hook that blocks whenever tests fail traps an agent that cannot fix the tests.

1. The hook blocks the stop with the failing test names.
2. The agent tries, fails, and tries to stop again.
3. Without a limit, this repeats until the session runs out of room.

> The Stop event carries a flag saying a stop hook already sent the agent back once. Check it, and on the second stop let the agent through with a note to the person.
{: .warning }

## Cost and noise

### Time

A hook on `PreToolUse` with an empty matcher runs before every tool call in every session.

| Hook run time | Effect |
|---|---|
| under 50 ms | not noticed |
| around 100 ms | felt on busy sessions |
| a second or more | a tax on every action |

Start a fast interpreter, read only what you need, and exit early when the event is not yours.

### Tokens

A hook that adds context spends tokens every time it fires.

- A reminder that fires on every tool call becomes text the model learns to skim.
- Fire on the condition: add the background-task reminder only while a background task is running.
- Measure what hooks add with `/context` before and after enabling them.

## Sources

- Claude Code hooks, exit codes and decision control: <https://code.claude.com/docs/en/hooks.md>, verified 2026-11-08.
- The failure cases are from the author's own harness, described conceptually.

{% include page_nav.html %}
