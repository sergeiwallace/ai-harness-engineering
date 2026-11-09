---
title: "A context supervisor"
parent: "Compaction and saving state"
grand_parent: "Context window and memory"
nav_order: 1
redirect_from:
  - /sections/11-context-window/advanced.html
---

{% include page_nav.html %}

# A context supervisor

As of 2026-11-09.

A sub-topic of [compaction and saving state]({{ '/sections/11-context-window/compaction.html' | relative_url }}). A context supervisor is a small piece of automation that watches how full a session's context is and, past a threshold, saves state and compacts, so a long autonomous run does not depend on the agent noticing in time.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Why automate it

An agent deep in a task does not stop to check its own context. By the time the automatic compaction near the limit fires, it has spent a long stretch at the fill level where it follows instructions less reliably, and it compacts without having saved state. A supervisor moves the save and the compaction earlier, and makes them happen the same way every time.

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

Slash commands such as `/compact` are typed into the session's prompt; a hook cannot call them as a function. So the supervisor types the command into the terminal, the same way a person would, through a small library that targets the right session and refuses to type into a pane a person is using. [Templates and stubs]({{ '/sections/07-templates-and-stubs/' | relative_url }}) covers injected prompts.

## A minimal hook

A sketch of the measurement step as a Claude Code hook script. The way you read the percentage depends on your setup; this one calls a command that prints it.

```bash
#!/usr/bin/env bash
# PostToolUse hook: remind the agent to save state when context is high.
set -euo pipefail
pct="$(context-percent 2>/dev/null || echo unavailable)"
if [[ "$pct" == unavailable ]]; then
  exit 0            # no reading: say nothing rather than guess
fi
if (( pct >= 55 )); then
  echo "Context is at ${pct}%. Finish this step, save state, then compact."
fi
```

> When the measurement is unavailable, the hook says nothing. A guess dressed as a reading is worse than no reading.
{: .note }

## Failure modes

#### Compaction that never fires

The injection can fail: the session is busy, the terminal is not where expected. The supervisor should notice that the fill level did not drop and retry or raise it, not assume success.

#### Saving the wrong state

If the save runs mid-step, the note describes a half-finished action. Ask the agent to finish its current step before saving, which is what the middle band is for.

#### Over-eager reminders

A reminder on every tool call above 40 percent fills the context with reminders. Remind once per band.

## Examples from my own harness

My context supervisor: hooks read how full the context is and, past a threshold, run save-state and then send `/compact` into the session through the prompt-injection library. A separate command reports the actual context percentage and the band it is in, so a session quotes a reading, not a guess, before it talks about saving or compacting.

To start: a hook that prints a reminder at one threshold, and a habit of saving state when you see it.

## Sources

- Hooks: <https://code.claude.com/docs/en/hooks.md>.
- Compaction and `/compact`: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-08.

{% include page_nav.html %}
