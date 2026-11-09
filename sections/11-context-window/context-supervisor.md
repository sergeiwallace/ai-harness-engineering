---
title: "A context supervisor"
parent: "Compaction and saving state"
grand_parent: "Context window and memory"
nav_order: 1
redirect_from:
  - /sections/11-context-window/advanced.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# A context supervisor

A sub-topic of [compaction and saving state]({{ '/sections/11-context-window/compaction.html' | relative_url }}):

- A context supervisor is a small piece of automation that watches how full a session's context is and, past a threshold, saves state and compacts, so a long autonomous run does not depend on the agent noticing in time.
- This page covers the built-in options to try first, the parts of a supervisor, how they fit together, a minimal hook, and the failure modes.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Why automate it

| Without a supervisor | With a supervisor |
|---|---|
| The agent has to notice its own fill level mid-task | A hook measures it after every tool call |
| Automatic compaction fires near the limit, often without a save | The save runs at a set threshold, then the compaction |
| The routine varies run to run | The same steps every time |

A written instruction to "save state at 55%" competes with the task for the agent's attention. A hook applies it the same way every time, which is the systems approach to this problem.

## Built-in options first

Claude Code's own settings cover much of what a supervisor does. Try them before building one.

| Need | Built-in option |
|---|---|
| Compact earlier than near the limit | `/autocompact` with a token count, such as `/autocompact 500k` on a 1M-token window |
| Bring the state back after any compaction | A `SessionStart` hook with the `compact` matcher that prints the state note ([compaction and saving state]({{ '/sections/11-context-window/compaction.html' | relative_url }}#a-sessionstart-hook-on-compact)) |
| Remind the agent to save before the threshold | A `PostToolUse` hook that adds a note to Claude's context (below) |

A full supervisor adds one thing these do not: it makes sure the save finishes before the compaction starts, and checks afterwards that the compaction happened.

## The parts

| Part | Job |
|---|---|
| A measurement | Reports the real context percentage of a session |
| A hook | Runs the measurement at a regular point, such as after each tool call or at the end of a turn |
| Bands | Map the percentage to an action: nothing, a reminder, save now, compact now |
| A save step | Writes the state note and resume message |
| An injection step | Sends `/compact` into the session |

### Bands

| Fill level | Action |
|---|---|
| Below 40% | Nothing |
| 40 to 50% | Tell the agent to finish the current step and save soon |
| 50 to 60% | Save state now |
| Above 60% | Save if not saved, then compact |

The numbers follow my own practice of compacting at about 50 to 60 percent; tune them to your models.

## How it fits together

![A sequence diagram. The hook warns the session at 45% and asks it to save at 55%. The session writes a state note and the files confirm. In an optional fragment the hook asks the injector to compact, the injector types /compact into the session, and the hook reads 9% afterwards. The session reads its state back.]({{ '/assets/images/11-context-window/context-supervisor.svg' | relative_url }}){: style="width:100%"}

*Figure: a supervisor saves first, compacts second, and checks the fill level dropped. Made with archify.*

#### Step by step

1. A hook fires after a tool call and reads the session's context percentage.
2. At 55 percent it adds a note to the agent's context: "save state now".
3. The agent runs its save-state routine: state note, memory lessons, tracker.
4. At the next check, with state saved and the fill above 60 percent, the supervisor sends `/compact` into the session through the prompt-injection library.
5. After compaction the resume message tells the session what to read first.

### Why `/compact` has to be typed in

- Slash commands such as `/compact` run only when typed at the session's prompt.
  - A `SendMessage` message containing `/compact` arrives as plain text and does not run.
  - A `/loop` prompt that is a built-in command also reaches Claude as plain text.
- So the supervisor types the command into the session's terminal, the way a person would.
- It goes through a small library that:
  - targets the right session by name,
  - refuses to type into a pane a person is using,
  - and lets the supervisor check afterwards that the fill level dropped.

[Templates and stubs]({{ '/sections/07-templates-and-stubs/' | relative_url }}) covers injected prompts.

## A minimal hook

A sketch of the reminder step as a `PostToolUse` hook:

- A `PostToolUse` hook reaches Claude through JSON output: `hookSpecificOutput.additionalContext` is added to Claude's context next to the tool result.
- How you read the percentage depends on your setup; this one calls a command that prints it.

```bash
#!/usr/bin/env bash
# PostToolUse hook: remind the agent to save state when context is high.
set -euo pipefail
pct="$(context-percent 2>/dev/null || echo unavailable)"
if [[ "$pct" == unavailable ]]; then
  exit 0            # no reading: say nothing rather than guess
fi
if (( pct >= 55 )); then
  jq -n --arg msg "Context is at ${pct}%. Finish this step, save state, then compact." \
    '{hookSpecificOutput: {hookEventName: "PostToolUse", additionalContext: $msg}}'
fi
```

Registered for every tool in `.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      { "matcher": "*",
        "hooks": [ { "type": "command", "command": "tools/context-reminder.sh" } ] }
    ]
  }
}
```

## Failure modes

| Failure | Sign | Response |
|---|---|---|
| The compaction never fires | The fill level did not drop after the injection | Retry, then raise it |
| The state saved mid-step | The note describes a half-finished action | Finish the current step before saving (the middle band) |
| Over-eager reminders | A reminder on every tool call fills the window | Remind once per band |

#### Compaction that never fires

- Sign: the fill level did not drop after the injection.
- Causes: the session was mid-turn, or the terminal was not where the library expected.
- Response: retry once after the turn ends, then raise it to a person or the coordinator.

#### Saving the wrong state

- Sign: the note describes a half-finished action.
- Response: ask the agent to finish its current step before saving, which is what the middle band is for.

#### Over-eager reminders

- Sign: a reminder on every tool call above 40 percent fills the window with reminders.
- Response: remind once per band, and record in a small state file which band was last announced.

## Examples from my own harness

- My context supervisor: hooks read how full the context is and, past a threshold, run save-state and then send `/compact` into the session through the prompt-injection library.
- A separate command reports the actual context percentage and the band it is in.
  - A session quotes that reading before it talks about saving or compacting.

To start: a hook that prints a reminder at one threshold.

## Sources

- Hooks, `PostToolUse` and `additionalContext`, `SessionStart`: [Hooks reference (Claude Code docs)](https://code.claude.com/docs/en/hooks.md) and [Automate actions with hooks (Claude Code docs)](https://code.claude.com/docs/en/hooks-guide.md), verified 2026-11-09.
- `/autocompact` and what survives compaction: [Explore the context window (Claude Code docs)](https://code.claude.com/docs/en/context-window.md), verified 2026-11-09.
- Commands in cross-session messages arrive as text: [Message your other Claude Code sessions (Claude Code docs)](https://code.claude.com/docs/en/cross-session-messaging.md), verified 2026-11-09.
- Built-in commands in `/loop` prompts arrive as text: [Run prompts on a schedule (Claude Code docs)](https://code.claude.com/docs/en/scheduled-tasks.md), verified 2026-11-09.

{% include page_nav.html %}
