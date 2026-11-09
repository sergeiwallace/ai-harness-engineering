---
title: "Harness hooks"
parent: "What is a harness?"
nav_order: 6
has_children: true
has_toc: false
redirect_from:
  - /sections/02-ai-harnesses/hooks.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Harness hooks

The full section on this topic is [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}).

A hook is a command the harness runs at a fixed point in a session. It sees what the agent is about to do or has just done, and it can let the action through, refuse it with a reason, or add text to the conversation. A rule in a hook holds whether or not the model remembers it. This page covers how a hook is configured, what it receives, what it can return, and how to write one. Three sub-topic pages go further: [Hook lifecycle events]({{ '/sections/02-what-is-a-harness/hook-lifecycle-events.html' | relative_url }}), [Guard hooks]({{ '/sections/02-what-is-a-harness/guard-hooks.html' | relative_url }}) and [Hook failure modes]({{ '/sections/02-what-is-a-harness/hook-failure-modes.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Where hooks sit

![Where hooks fire in a session: SessionStart, then UserPromptSubmit, then a loop of PreToolUse, the tool, and PostToolUse, then Stop, which can send the agent back to work; PreCompact, SubagentStop and SessionEnd fire alongside.]({{ '/assets/images/02-ai-harnesses/hook-lifecycle.svg' | relative_url }})

*The main hook events across one session. Orange events can change what happens next. Claude Code has many more events than shown here; [Hook lifecycle events]({{ '/sections/02-what-is-a-harness/hook-lifecycle-events.html' | relative_url }}) lists them.*

### Hooks and the other controls

| Control | Runs when | Can see | Can do |
|---|---|---|---|
| Permission rule | before a tool call | tool name and a pattern on its input | allow, ask, deny |
| Hook | at any lifecycle event | the full event as JSON | allow, deny, add context, send the agent back |
| Git hook | at commit or push | the staged change | refuse the commit or push |
| CI check | after push | the whole repository | fail the build |

A permission rule matches a pattern. A hook runs code, so it can:

- parse a shell command into its parts;
- read the file the agent is about to write;
- check the git state or the time of day;
- call another tool or service before it decides.

Use a [permission rule]({{ '/sections/02-what-is-a-harness/permissions-and-settings.html' | relative_url }}) when a pattern is enough and a hook when it is not. The two combine: a hook can always refuse, and a hook's allow never overrides a deny or ask rule.

## Configuring a hook

### The settings entry

Hooks are declared under the event they listen to, in any of these places:

- `settings.json` at user, project, local or managed scope;
- a plugin's `hooks/hooks.json`;
- the front matter of a skill or a sub-agent, scoped to that skill or agent.

A matcher picks which tools trigger the hook.

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

| Matcher | Read as | Fires on |
|---|---|---|
| `Bash` | exact name | every shell command |
| `Edit\|Write` | a list of exact names | file edits and new files |
| `mcp__github__.*` | a regular expression | every tool from one MCP server |
| `mcp__.*` | a regular expression | every MCP tool |
| `mcp__github` | exact name | nothing: no tool has that name |
| `*`, `""` or omitted | match all | every tool, or the event itself for non-tool events |

A matcher made only of letters, digits, `_`, `-`, spaces, `,` and `\|` is an exact name or a list; any other character makes it an unanchored JavaScript regular expression. Use `^...$` to match a whole name.

### Hook types and timeouts

| `type` | Runs | Default timeout |
|---|---|---|
| `command` | a shell command, with the event on standard input | 600 seconds |
| `http` | a POST to a URL | 600 seconds |
| `mcp_tool` | a tool on a connected MCP server | 600 seconds |
| `prompt` | a single model call that judges the event | 30 seconds |
| `agent` | a sub-agent that can use tools before it judges | 60 seconds |

Some events have shorter defaults, such as 30 seconds for `UserPromptSubmit`. All hooks that match one event run in parallel, so one hook cannot rely on another having run first.

### Seeing what is installed

The `/hooks` command shows the hook configuration the session has loaded, by event and source. Its output looks roughly like this (illustrative):

```text
> /hooks
PreToolUse
  Bash        python3 tools/guards/no_pattern_kill.py       (project)
  Edit|Write  python3 tools/guards/no_generated_edits.py    (project)
PostToolUse
  Edit|Write  uv run ruff check --quiet                     (user)
```

> Run it after every install. A hook that is in the repository and missing from this list is not running. See [Hook failure modes]({{ '/sections/02-what-is-a-harness/hook-failure-modes.html' | relative_url }}#never-installed).
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

Every event carries common fields: `session_id`, `transcript_path`, `cwd`, `permission_mode` and `hook_event_name`. The rest depend on the event.

| Event | Adds |
|---|---|
| `PreToolUse` | `tool_name`, `tool_input` |
| `PostToolUse` | the tool's input and its response |
| `UserPromptSubmit` | the prompt |
| `Stop`, `SubagentStop` | `stop_hook_active`, true when a stop hook has already sent the agent back |
| `SubagentStop` | `agent_type`, `agent_transcript_path`, `last_assistant_message` |
| `SessionStart` | the source: `startup`, `resume`, `clear` or `compact` |

## What a hook returns

### Exit codes

The simplest hook answers with its exit code.

| Exit code | Meaning | What happens |
|---|---|---|
| 0 | success | JSON on standard output is applied; plain text is added as context on `SessionStart` and `UserPromptSubmit`; standard error is never shown to the model |
| 2 | block, on events that can block | the action is refused or the agent is sent back, with standard error as the reason |
| any other, including 1 | a non-blocking error | the action continues, and a hook error notice is shown |

> Exit code 1 does not block. A guard that crashes with an ordinary error lets the action through. See [Hook failure modes]({{ '/sections/02-what-is-a-harness/hook-failure-modes.html' | relative_url }}#fail-open-by-accident).
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

| Field | Event | Effect |
|---|---|---|
| `permissionDecision: "deny"` | `PreToolUse` | refuses; `permissionDecisionReason` goes to the model |
| `permissionDecision: "allow"` | `PreToolUse` | skips the prompt, unless a deny or ask rule matches |
| `permissionDecision: "ask"` | `PreToolUse` | shows the person a prompt |
| `permissionDecision: "defer"` | `PreToolUse` | leaves the decision to the normal permission flow |
| `updatedInput` | `PreToolUse` | replaces the tool's arguments before it runs |
| `additionalContext` | several | adds text the model sees, without blocking |
| `decision: "block"` with `reason` | `PostToolUse` | shows the reason beside the result; the tool has already run |
| `updatedToolOutput` | `PostToolUse` | replaces the output the model sees, for example to redact it |
| `decision: "block"` with `reason` | `Stop`, `SubagentStop` | sends the agent back to work |

> `updatedInput` can turn a risky call into a safe one, such as adding `--dry-run`, without a refusal. Use it with care: the model believes it ran what it asked for, so say what you changed in `additionalContext`.
{: .note }

## Writing a hook

### A complete guard

```python
#!/usr/bin/env python3
"""Refuse kill commands that select processes by name pattern."""
import json
import re
import shlex
import sys

import os

event = json.load(sys.stdin)
command = event.get("tool_input", {}).get("command", "")

PATTERN_KILLS = {"pkill", "killall"}
PREFIXES = {"sudo", "env", "nohup", "time"}

def words(cmd):
    try:
        return shlex.split(cmd)
    except ValueError:
        return cmd.split()

def program(parts):
    while parts and (parts[0] in PREFIXES or "=" in parts[0]):
        parts = parts[1:]
    return os.path.basename(parts[0]) if parts else ""

for segment in re.split(r"[;&|\n]+", command):
    if program(words(segment)) in PATTERN_KILLS:
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
2. It splits compound commands on `;`, `&`, `|` and newlines, so `cd x && pkill y` is still caught.
3. It looks past `sudo`, `env` and variable assignments, and strips the path, so `sudo /usr/bin/pkill` is caught too.
4. Its refusal says why and what to do instead, in one sentence each.
5. It exits 0 for everything it does not recognise, so it never blocks unrelated work.
6. It runs in milliseconds; hooks on `Bash` run before every shell command.

It does not catch `kill $(pgrep node)`, which kills by pattern through a substitution. [Hook failure modes]({{ '/sections/02-what-is-a-harness/hook-failure-modes.html' | relative_url }}#bypass-by-spelling) lists spellings like this; each one belongs in the hook's tests.

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

Many of my harness's hooks came from the same pattern, which is the [systems approach]({{ '/sections/02-what-is-a-harness/' | relative_url }}#a-systems-approach-to-harness-engineering) in practice: an agent broke a written rule, and the fix went into the system as a hook, with the written rule kept as a second layer. Today a rule that can be checked from the tool input goes to a hook first. The refusal text repeats the rule at the moment it applies. A few of the kinds, described conceptually:

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

## Hooks in Codex

Codex declares hooks in a `[hooks]` table in `config.toml`.

- Its documented events include `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PermissionRequest`, `PostToolUse`, `PreCompact`, `PostCompact`, `SubagentStart`, `SubagentStop`, `Stop` and `SessionEnd`, plus model-request and interrupt events of its own.
- A separate `notify` key runs a command with a JSON payload for notifications.
- The event names match Claude Code's closely. This page has not verified that the input and output formats match, so check the Codex configuration reference before porting a hook.

## Sub-topics

- [Hook lifecycle events]({{ '/sections/02-what-is-a-harness/hook-lifecycle-events.html' | relative_url }}): each event, what it receives and what it can change.
- [Guard hooks]({{ '/sections/02-what-is-a-harness/guard-hooks.html' | relative_url }}): hooks that refuse, and how to write a refusal an agent learns from.
- [Hook failure modes]({{ '/sections/02-what-is-a-harness/hook-failure-modes.html' | relative_url }}): failing open, false positives, bypasses and cost.

## Sources

- [Claude Code hooks reference](https://code.claude.com/docs/en/hooks.md), verified 2026-11-09.
- [Hooks and permission rules](https://code.claude.com/docs/en/permissions.md), verified 2026-11-09.
- [Codex hooks configuration](https://learn.chatgpt.com/docs/config-file/config-reference), verified 2026-11-09.
- The hook kinds and refusal texts are from the author's own harness, described conceptually.

{% include page_nav.html %}
