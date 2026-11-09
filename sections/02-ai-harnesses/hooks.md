---
title: "Harness hooks"
parent: "AI harnesses"
nav_order: 6
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Harness hooks

As of 2026-11-09. Part of [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}). The full section on this topic is [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}).

A hook is a command the harness runs at a fixed point in a session. It sees what the agent is about to do or has just done, and it can let the action through, refuse it with a reason, or add text to the conversation. A rule in a hook holds whether or not the model remembers it. This page covers how a hook is configured, what it receives, what it can return, and how to write one. Three sub-topic pages go further: [Hook lifecycle events]({{ '/sections/02-ai-harnesses/hook-lifecycle-events.html' | relative_url }}), [Guard hooks]({{ '/sections/02-ai-harnesses/guard-hooks.html' | relative_url }}) and [Hook failure modes]({{ '/sections/02-ai-harnesses/hook-failure-modes.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Where hooks sit

![Where hooks fire in a session: SessionStart, then UserPromptSubmit, then a loop of PreToolUse, the tool, and PostToolUse, then Stop, which can send the agent back to work; PreCompact, SubagentStop and SessionEnd fire alongside.]({{ '/assets/images/02-ai-harnesses/hook-lifecycle.svg' | relative_url }})

*Hook events across one session. Orange events can change what happens next.*

### Hooks and the other controls

| Control | Runs when | Can see | Can do |
|---|---|---|---|
| Permission rule | before a tool call | tool name and a pattern on its input | allow, ask, deny |
| Hook | at any lifecycle event | the full event as JSON | allow, deny, add context, send the agent back |
| Git hook | at commit or push | the staged change | refuse the commit or push |
| CI check | after push | the whole repository | fail the build |

A permission rule matches a pattern. A hook runs code, so it can parse a shell command, read a file the agent is about to write, check the git state or call another tool before it decides. Use a [permission rule]({{ '/sections/02-ai-harnesses/permissions-and-settings.html' | relative_url }}) when a pattern is enough and a hook when it is not.

## Configuring a hook

### The settings entry

Hooks are declared in `settings.json` at user, project or local scope, under the event they listen to. A matcher picks which tools trigger the hook; it is a regular expression on the tool name.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR\"/tools/guards/no_pattern_kill.py",
            "timeout": 10
          }
        ]
      },
      {
        "matcher": "Edit|Write",
        "hooks": [
          { "type": "command", "command": "python3 \"$CLAUDE_PROJECT_DIR\"/tools/guards/no_generated_edits.py" }
        ]
      }
    ]
  }
}
```

### Matchers

| Matcher | Fires on |
|---|---|
| `Bash` | every shell command |
| `Edit\|Write` | file edits and new files |
| `mcp__.*` | every tool from any MCP server |
| `mcp__github__.*` | every tool from one MCP server |
| empty or omitted | every tool, for tool events; the event itself for others |

### Seeing what is installed

The `/hooks` command lists every hook the session has loaded, by event and source.

```text
> /hooks
PreToolUse
  Bash        python3 tools/guards/no_pattern_kill.py       (project)
  Edit|Write  python3 tools/guards/no_generated_edits.py    (project)
PostToolUse
  Edit|Write  uv run ruff check --quiet                     (user)
```

> Run it after every install. A hook that is in the repository and missing from this list is not running. See [Hook failure modes]({{ '/sections/02-ai-harnesses/hook-failure-modes.html' | relative_url }}#never-installed).
{: .important }

## What a hook receives

The harness writes the event to the hook's standard input as one JSON object.

```json
{
  "session_id": "8c1e…",
  "transcript_path": "<path to the session transcript>",
  "cwd": "<working directory>",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {
    "command": "pkill -f pytest",
    "description": "Stop the running tests"
  }
}
```

The fields after `hook_event_name` depend on the event. `PostToolUse` adds the tool's response; `UserPromptSubmit` carries the prompt; `Stop` carries a flag saying whether a stop hook has already sent the agent back once.

## What a hook returns

### Exit codes

The simplest hook answers with its exit code.

| Exit code | Meaning | What the model sees |
|---|---|---|
| 0 | allow, no comment | nothing, or the hook's output where the event adds it to context |
| 2 | block | the hook's standard error, as the reason |
| any other | the hook itself failed | the action continues; the person sees an error |

> Exit code 1 does not block. A guard that crashes with an ordinary error lets the action through. See [Hook failure modes]({{ '/sections/02-ai-harnesses/hook-failure-modes.html' | relative_url }}#fail-open-by-accident).
{: .warning }

### JSON output

For a richer answer, the hook prints JSON and exits 0.

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Pattern kills can stop other people's processes. Find the PID with `ps -eo pid,args` and kill that one PID."
  }
}
```

| Field | Effect |
|---|---|
| `permissionDecision: "allow"` | runs the tool without a permission prompt |
| `permissionDecision: "ask"` | shows the person a prompt |
| `permissionDecision: "deny"` | refuses, and the reason goes to the model |
| `additionalContext` | adds text to the conversation without blocking |
| `decision: "block"` on Stop | sends the agent back to work with the reason |

## Writing a hook

### A complete guard

```python
#!/usr/bin/env python3
"""Refuse kill commands that select processes by name pattern."""
import json
import re
import shlex
import sys

event = json.load(sys.stdin)
command = event.get("tool_input", {}).get("command", "")

PATTERN_KILLS = {"pkill", "killall"}

def words(cmd):
    try:
        return shlex.split(cmd)
    except ValueError:
        return cmd.split()

for segment in re.split(r"[;&|]+", command):
    parts = words(segment)
    if parts and parts[0] in PATTERN_KILLS:
        print(
            "Refused: a pattern kill can stop processes that are not yours. "
            "Find the process with `ps -eo pid,etime,args`, then `kill <pid>`.",
            file=sys.stderr,
        )
        sys.exit(2)

sys.exit(0)
```

### What makes it a good hook

1. It reads only what it needs from the event.
2. It splits compound commands, so `cd x && pkill y` is still caught.
3. Its refusal says why and what to do instead, in one sentence each.
4. It exits 0 for everything it does not recognise, so it never blocks unrelated work.
5. It runs in milliseconds; hooks on `Bash` run before every shell command.

<details markdown="1">
<summary>Testing a hook before you install it</summary>

Feed it events by hand and check the exit code.

```text
$ echo '{"tool_input":{"command":"pkill -f pytest"}}' | python3 no_pattern_kill.py; echo "rc=$?"
Refused: a pattern kill can stop processes that are not yours. ...
rc=2
$ echo '{"tool_input":{"command":"kill 41822"}}' | python3 no_pattern_kill.py; echo "rc=$?"
rc=0
```

Test both arms. A guard tested only on the case it should block can be a guard that blocks everything. My harness keeps a test file per hook with both a positive and a negative case for every rule the hook enforces.

</details>

## Hooks in my harness

Many of my harness's hooks came from the same pattern: an agent broke a written rule, and the fix went into the system as a hook, with the written rule kept as a second layer. Today a rule that can be checked from the tool input goes to a hook first. The refusal text repeats the rule at the moment it applies. A few of the kinds, described conceptually:

| Kind | Event | What it does |
|---|---|---|
| Destructive-command guard | PreToolUse on Bash | refuses pattern kills, recursive deletes outside scratch, and system changes |
| Secret guard | PreToolUse on Bash | refuses commands that would print the environment or a profile file |
| Worktree isolation guard | PreToolUse on Bash and Edit | keeps a sub-agent's git commands and edits inside its own worktree |
| Evidence guard | PreToolUse on Edit and Write | refuses a document that cites an untracked file as proof |
| Background reminder | PreToolUse | adds a note when a background task is still running, so the agent keeps working |
| Compaction backstop | PreCompact | makes sure session state is written down before the summary |

The isolation guard's refusal, as an agent sees it:

```text
This agent is isolated in the worktree <its worktree>, but this command is too
complex to verify that it stays inside the worktree. Refusing to run it. Split it
into plain, separate commands and run them from <its worktree>.
```

The refusal names the safe form of the same action, so the agent's next try succeeds.

## Sub-topics

- [Hook lifecycle events]({{ '/sections/02-ai-harnesses/hook-lifecycle-events.html' | relative_url }}): each event, what it receives and what it can change.
- [Guard hooks]({{ '/sections/02-ai-harnesses/guard-hooks.html' | relative_url }}): hooks that refuse, and how to write a refusal an agent learns from.
- [Hook failure modes]({{ '/sections/02-ai-harnesses/hook-failure-modes.html' | relative_url }}): failing open, false positives, bypasses and cost.

## Sources

- Claude Code hooks reference: <https://code.claude.com/docs/en/hooks.md>, verified 2026-11-08.
- The hook kinds and refusal texts are from the author's own harness, described conceptually.

{% include page_nav.html %}
