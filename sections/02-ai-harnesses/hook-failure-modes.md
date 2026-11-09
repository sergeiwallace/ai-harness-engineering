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

The guard is written, tested and merged. The installer that copies hooks into place was never run on the machine, or ran from a branch that did not have it. Every check of the repository says the guard exists.

### The check

Measure the target, not the source. `/hooks` in a live session lists what the harness actually loaded. My harness learned this more than once: a guard pinned by a test can still be live in zero places, so the release check names the one artifact whose state proves the change reached its target, and reads it.

## Matcher misses

### The mechanism

A guard on `Bash` refuses deleting a file with `rm`. The agent has an MCP server with a file-delete tool, and a `Write` tool that can empty a file. The rule is enforced on one route of three.

### The check

For each guarded action, list every tool that can perform it, and widen the matcher or add a guard per route.

## False positives and route-arounds

### The mechanism

A guard refuses too much. The agent, being helpful, finds another way: writes the command to a script file and runs the script, or uses a different tool. The guard now teaches evasion.

### The check

Log every refusal with the command that caused it. Read the log weekly. A refusal of a command that was safe is a bug in the guard; fix it the same day, or the agent's workaround becomes a habit.

## Bypass by spelling

| Intended block | Spellings that slip past a string match |
|---|---|
| `rm -rf dir` | `rm -r -f dir`, `rm --recursive --force dir`, `find dir -delete` |
| `pkill node` | `killall node`, `kill $(pgrep node)` |
| `env` | `printenv`, `export -p`, `declare -x`, `set` |

Each row is a test case. My harness's secret guard keeps a list of every spelling that prints the environment, and a shell's tracing mode and a throwaway subshell are on that list too.

## Loops

A Stop hook that blocks whenever tests fail traps an agent that cannot fix the tests. Check the flag the harness passes that says the agent was already sent back once, and on the second stop, let it through with a note to the person.

## Cost and noise

### Time

A hook on `PreToolUse` with an empty matcher runs before every tool call in every session. At a hundred milliseconds it is felt; at a second it is a tax on everything. Start a fast interpreter, read only what you need, and exit early when the event is not yours.

### Tokens

A hook that adds context on every prompt or tool call spends tokens every time. A reminder that fires on every call becomes text the model learns to skim. Fire on the condition, not on the event.

## Sources

- Claude Code hooks, exit codes and decision control: <https://code.claude.com/docs/en/hooks.md>, verified 2026-11-08.
- The failure cases are from the author's own harness, described conceptually.

{% include page_nav.html %}
