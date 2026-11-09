---
title: "Hooks"
parent: "Hooks, scripts and linters"
nav_order: 2
has_children: true
has_toc: false
redirect_from:
  - /sections/06-hooks-scripts-and-linters/detail.html
  - /sections/06-hooks-scripts-and-linters/advanced.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Hooks

A hook is a handler that the client runs at a fixed point in a session: before a tool call, after it, when a prompt is sent, before compaction, when the agent tries to stop. It runs whether or not the model remembers any rule, and some hooks can refuse the action. Hooks are layer 3 of the [enforcement ladder]({{ '/sections/06-hooks-scripts-and-linters/enforcement-ladder.html' | relative_url }}): they act at the moment the agent tries something. This page covers the events, the handler types, the data a hook receives and returns, exit codes, registration and debugging. It is part of [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}). The sub-page [Writing a guard]({{ '/sections/06-hooks-scripts-and-linters/writing-a-guard.html' | relative_url }}) covers hooks that refuse actions.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Where hooks fire

![A session timeline: SessionStart, UserPromptSubmit, a repeating loop of PreToolUse, the tool call and PostToolUse, then PreCompact and Stop. UserPromptSubmit, PreToolUse and Stop are marked as able to block.]({{ '/assets/images/06-hooks-scripts-and-linters/hook-lifecycle.svg' | relative_url }})
*One session, left to right. The dashed box repeats for every tool call, so a `PreToolUse` hook runs hundreds of times in a long session.*

Claude Code's hooks reference lists 33 events; Codex has 14. A harness uses a handful of them heavily.

### The events in detail

| Event | Fires | Can block | Typical use |
|---|---|---|---|
| `SessionStart` | Start, resume, after compaction | No | Load state; print reminders |
| `UserPromptSubmit` | A prompt is sent, before the model sees it | Yes | Add context; refuse a prompt |
| `PreToolUse` | Before a tool call runs | Yes | Refuse dangerous commands; check paths |
| `PostToolUse` | After a tool call succeeds | No (the tool already ran) | Format edited files; record changes |
| `PreCompact` | Before the conversation is compacted | Used to save, not to block | Save state the summary might lose |
| `Stop` | The agent tries to finish its turn | Yes | Refuse to stop until a check passes |
| `SubagentStop` | A sub-agent finishes | Yes | Check its result |

#### Choosing the event

| You want to | Use |
|---|---|
| Stop an action before it happens | `PreToolUse` |
| Clean up after an edit | `PostToolUse` |
| Make the agent keep going until tests pass | `Stop` |
| Keep state across a compaction | `PreCompact` to save, `SessionStart` to reload |
| Remind the agent of something on every prompt | `UserPromptSubmit`, sparingly |

## Handler types

A hook can be a shell command, an HTTP call, an MCP tool call, a prompt to a model, or an agent. Command hooks are the most common.

| Type | Runs | Good for | Watch for |
|---|---|---|---|
| Command | A program on your machine | Fast, exact checks | Start-up time on every call |
| HTTP | A request to a URL | A central policy service | Network failures block work |
| MCP tool | A tool on a connected server | Reusing an existing tool | Server must be running |
| Prompt | A short model call | Judgment a regex cannot make | Cost and latency per call |
| Agent | A sub-agent with tools | Checks that need to read files | Much slower; use rarely |

## Input and output

### What a hook receives

A command hook gets the event as JSON on standard input. For a `PreToolUse` event on the shell tool:

```json
{
  "session_id": "abc123",
  "cwd": "/work/app",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": { "command": "git push --force origin main" }
}
```

The hook reads the fields it needs, usually with `jq`, and ignores the rest.

### What a hook returns

#### Exit codes

| Exit code | Meaning in Claude Code | What the agent sees |
|---|---|---|
| 0 | Proceed | Nothing, or the hook's standard output where the event allows it |
| 2 | Stop, don't do this | The action is blocked and the hook's standard error comes back as feedback |
| Other | The hook itself failed | An error is shown; the action proceeds |

#### Structured JSON output

A hook can also print a JSON object on standard output for finer control. On `PreToolUse` it can allow, deny or ask the person:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Force push refused. Push to a new branch and open a pull request."
  }
}
```

| Decision | Effect | Suits |
|---|---|---|
| `allow` | The call runs without a permission prompt | Commands the hook has checked and knows are safe |
| `deny` | The call is blocked and the reason goes to the agent | Commands that are never right |
| `ask` | The person sees a prompt and decides | Commands that are right in some cases |

<details markdown="1">
<summary>Adding context without blocking</summary>

A `SessionStart` or `UserPromptSubmit` hook can return text that is added to the model's context. This is how a harness loads saved state after a compaction, or reminds the agent which task it was on. Every word added this way is paid on every event it fires on, so keep it short and add it only when it applies.

</details>

## Registering a hook

Hooks are registered in a settings file:

| File | Applies to |
|---|---|
| `~/.claude/settings.json` | Every project you open |
| `.claude/settings.json` | Everyone who works in the repository |
| `.claude/settings.local.json` | You, in this repository, not committed |
| A plugin's `hooks/hooks.json` | Everyone who enables the plugin |

Use `$CLAUDE_PROJECT_DIR` in the command path, so the hook is found whatever directory the agent is in:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [{ "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/guards/no-pattern-kill.sh", "timeout": 5 }]
      },
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "\"$CLAUDE_PROJECT_DIR\"/.claude/guards/protected-paths.sh", "timeout": 5 }]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "npx prettier --write \"$(jq -r .tool_input.file_path)\"" }]
      }
    ]
  }
}
```

### Matchers

The matcher picks which tool calls the hook sees.

| Matcher | Sees |
|---|---|
| `Bash` | Every shell command |
| `Edit\|Write` | Every file edit and file write |
| `mcp__github__.*` | Every tool from the MCP server named `github` |
| `*` or empty | Every tool call |

A narrow matcher keeps a slow hook off the calls it does not need to check.

### Timeouts

Each hook takes a `timeout` in seconds.

- Set one on every hook. A hook that hangs on every tool call stops all work.
- A guard that parses a command needs a few seconds; 5 is a common value.
- A hook that runs tests, such as a `Stop` hook, needs longer, sized to the suite it runs.
- Make a timeout visible: print which hook timed out, so the person can find it.

### One manifest

In my harness hooks are registered from one manifest that an install step applies, so a hook is added or removed in one place. A test fails if a hook file exists with no registration, or a registration names a missing file. An illustrative manifest in that shape:

```text
event        matcher      script                     timeout
PreToolUse   Bash         no-pattern-kill            5
PreToolUse   Bash         pipe-status                5
PreToolUse   Edit|Write   evidence-citation          5
PreCompact   *            save-session-state         20
```

## Worked example: keep going until tests pass

A `Stop` hook can refuse to let the agent finish while the tests fail.

### The script

```bash
#!/usr/bin/env bash
# Stop hook: refuse to finish while the test suite fails
input=$(cat)
# Already continuing because of this hook: let the agent stop, so it cannot loop forever
if [[ "$(jq -r '.stop_hook_active' <<<"$input")" == "true" ]]; then
  exit 0
fi
if ! out=$(npm test --silent 2>&1); then
  echo "Tests are failing. Fix them before finishing. Last lines:" >&2
  echo "$out" | tail -n 15 >&2
  exit 2
fi
```

### The session

```text
● The login fix is complete.
  ⎿  Stop hook refused: Tests are failing. Fix them before finishing. Last lines:
     FAIL tests/login.test.ts
       ✕ rejects an expired token (12 ms)

● Reading tests/login.test.ts to see why the expired-token case fails.
  ...
● Fixed the expiry comparison. All 48 tests pass.
```

> A `Stop` hook that can never pass keeps the agent working forever. The script above lets the agent stop on the second attempt, using the `stop_hook_active` field in its input; a retry count or a condition the agent can always reach also works.

## Debugging hooks

| Symptom | Likely cause | Check |
|---|---|---|
| Hook never fires | Matcher does not match the tool name | `/hooks` lists what is registered and from where |
| Dangerous command still runs | Hook exits 1 | Exit 2, or return a deny decision |
| Every tool call is slow | A slow script on a broad matcher | Time the script; narrow the matcher |
| Agent loops on a refusal | The message gives no fix | Name the allowed form in the message |
| Hook works by hand, fails in session | Different working directory or environment | Use `$CLAUDE_PROJECT_DIR` in the path; print `cwd` while testing |

### Testing a hook by hand

```bash
echo '{"tool_name":"Bash","tool_input":{"command":"pkill -f pytest"}}' \
  | .claude/guards/no-pattern-kill.sh; echo "exit=$?"
```

Feed it the JSON the client would send and check the exit code and message. A test file with one such case per rule keeps the guard checked as the harness changes.

## Hooks in Codex

Codex has its own hooks, 14 events at the time of writing, configured in its own settings.

| | Claude Code | Codex |
|---|---|---|
| Events | 33 | 14 |
| Check before a tool runs | Yes | Yes |
| Message back to the agent | Yes | Yes |
| Event names and output fields | Its own | Its own |

A hook script shared by both clients keeps its check in one function and adds a small adapter per client that reads that client's input and writes its output.

## Pitfalls

- A guard that exits 1, which reports an error and lets the command run.
- A broad matcher on a slow script, which slows every tool call.
- No `timeout`, so a hung hook stops all work.
- A refusal with no fix in its message, which gets a retry or a workaround.
- A hook copied from a repository or plugin without reading it. A hook runs as you, with your permissions.

## Sources

- Claude Code hooks, events, handler types, exit codes and JSON output: [Claude Code hooks reference](https://code.claude.com/docs/en/hooks.md), verified 2026-11-09 (partial read of a long page).
- Codex hooks: [Codex hooks documentation](https://learn.chatgpt.com/docs/hooks.md), verified 2026-11-08.
- The manifest is from the author's own harness, described conceptually. The scripts, settings and transcripts are illustrations written for this page. The timeline diagram was drawn for this page.

{% include page_nav.html %}
