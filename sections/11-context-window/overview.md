---
title: "Context window and memory"
nav_order: 21
has_children: true
has_toc: false
permalink: /sections/11-context-window/
---

{% include page_nav.html %}

# Context window and memory

As of 2026-11-09.

A session's context window is everything the model can see at once: your context files, the conversation, every file it read and every command's output. As it fills, an agent follows instructions less reliably, and near the limit the session compacts: it replaces the history with a summary. Whatever was not written down is gone. This section covers how to keep a long session useful through that:

1. Watch how full the window is, with a measurement.
2. Before compacting, write the task, the decisions and the next step to a file.
3. Keep lessons in memory files that load in the next session.
4. Keep the real task list in a tracker outside the session.

![A sequence diagram of one compaction. The context hook reports 58% used to the session. Inside a critical fragment, the session writes a state note and updates the tracker in its files, which answer "saved". The session then runs /compact with a focus, and compaction returns a summary. The session reads its state note back from the files, which return the next step, and reports that it resumed at 9%.]({{ '/assets/images/11-context-window/save-state-and-compact.svg' | relative_url }}){: style="width:100%"}

*Figure: save first, then compact, then read the state back. Made with archify.*

Stop 8 on the talk path, about 2 minutes. The deeper pages:

| Page | What it covers |
|---|---|
| [Compaction and saving state]({{ '/sections/11-context-window/compaction.html' | relative_url }}) | When to compact, what survives, `/compact <focus>`, the state note, the resume message |
| [A context supervisor]({{ '/sections/11-context-window/context-supervisor.html' | relative_url }}) | Hooks that measure the window and save and compact before it is too late |
| [Context in Claude Code and Codex]({{ '/sections/11-context-window/claude-code-and-codex-context.html' | relative_url }}) | Defaults, controls and observations for each tool |
| [Memory files and lessons]({{ '/sections/11-context-window/memory-files.html' | relative_url }}) | Auto memory, an index of lessons, archiving, the task tracker |

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What fills the window

| Source | When it loads | Typical size |
|---|---|---|
| Context files (`CLAUDE.md`, `AGENTS.md`, rules) | Session start; path rules when matching files are touched | Small to medium |
| Auto memory | Session start | Small, if kept short |
| Your messages and the agent's replies | Every turn | Grows steadily |
| File reads | Each time a file is read | Large: a big file can be thousands of tokens |
| Command output | Each command | Large for test runs and logs |
| Sub-agent summaries | When a sub-agent returns | Small: the point of [delegation]({{ '/sections/08-delegation/' | relative_url }}) |

`/context` shows what is filling the context window, by category, with suggestions.

#### Example: where the space went

1. A session has been fixing a test for an hour and runs `/context`.
2. Most of its window is tool output: the full test suite's output, printed five times.
3. It changes how it works.
   - The next test run uses `-q` and prints only failures.
   - The next large search goes to a sub-agent that returns a ten-line summary.

## Quality drops as the window fills

As the context window fills, an agent follows instructions less reliably. Rules stated at the start compete with everything since. Treat this as something to watch, because it changes as models improve.

> My own practice is to compact at about 50 to 60 percent of the context window. Other practitioners report a sweet spot below about 300k tokens (unverified practitioner reports).
{: .note }

### Codex

More in [context in Claude Code and Codex]({{ '/sections/11-context-window/claude-code-and-codex-context.html' | relative_url }}).

- My observation: Codex compacts at about 250k tokens by default, and going past that costs more usage or API spend.
- Unverified: OpenAI's configuration reference says the compaction threshold uses "model defaults" and gives no number.
- Its pricing page says subscription use of one current model "has no long-context surcharge".
- Check current pricing for your plan.

## Compaction

Claude Code compacts automatically near the limit.

| What | Detail |
|---|---|
| Trigger | Automatic near the limit; `/compact` any time |
| Steering | `/compact <focus>` tells the summary what to keep |
| Threshold | Settable with `/autocompact` |
| Reloaded afterwards | The root `CLAUDE.md`, rules without paths, auto memory |
| Everything else | Carried only by the summary |

### Write state before compacting

The summary is written by a model in a hurry. It keeps the gist and drops the specifics: the exact error, the file and line, the decision you made forty minutes ago and why. Write those to a file first.

```markdown
# State: export timeout fix (2026-11-09 14:20)

Task: nightly export times out after last week's migration.
Done: cause found, the index on rows.created_at was dropped.
Decided: restore the index in a new migration, not edit the old one.
Next: write test_export_uses_index, then the migration, then run the suite.
Open: does the archive table need the same index? Ask before changing it.
```

After compaction the session reads this note and carries on as if nothing happened. [Compaction and saving state]({{ '/sections/11-context-window/compaction.html' | relative_url }}) covers the note, the resume message and `/compact <focus>` in depth.

## Memory between sessions

Memory files carry lessons between sessions.

- A good lesson file says what to do, what went wrong that taught it, and how to recognise the case.
- The index that loads every session stays short; each lesson has its own file.
- A lesson guides the model. For a mistake that matters or repeats, the fix goes into the system as well: a workflow step, a hook or a test. See [where a fix belongs]({{ '/sections/11-context-window/memory-files.html' | relative_url }}#where-a-fix-belongs).

![A sequence diagram. Session 1 writes a lesson file and adds one line to the index. Days later session 2 loads the index at start, gets the pointer, reads the lesson file and gets the story behind the rule.]({{ '/assets/images/11-context-window/lesson-reaches-later-session.svg' | relative_url }}){: style="width:100%"}

*Figure: a lesson travels from one session to a later one through a short index. Made with archify.*

#### Example: a lesson

```markdown
# Count a message as delivered only when the target replies

What happened: a send timed out, I re-sent through the fallback, and the
target got the task twice. The first send had arrived.
Rule: treat a timeout as unknown. Ask the target before re-sending.
Applies to: any channel with a timeout.
```

Recording the failure behind the rule lets the agent apply the general idea, not only the one phrase. See [memory files and lessons]({{ '/sections/11-context-window/memory-files.html' | relative_url }}).

## The task tracker

A task tracker outside the session holds the real list of work.

| Where the list lives | At compaction | Visible to other sessions |
|---|---|---|
| In the conversation | Summarised, often garbled | No |
| In the terminal's task panel | Displayed only; the panel shows what the tracker says | No |
| In a tracker (a file or a service) | Untouched; read back after | Yes |

When a new request arrives mid-task, file it in the tracker and finish the current step.

## Commands

| Command | What it does |
|---|---|
| `/context` | Shows what is filling the context window, by category, with suggestions |
| `/compact <focus>` | Compacts now, steering the summary toward the focus |
| `/autocompact` | Sets the threshold for automatic compaction in Claude Code |

## Examples from my own harness

- A save-state skill that writes the current state before compaction, in a fixed resume-message format.
  - It records the task, decisions, git state, what is next, and any feedback from me in that session.
- A memory index with one line per lesson and one file per lesson, each recording the failure behind it.
- A command that reports the actual context percentage, and hooks that act on it ([a context supervisor]({{ '/sections/11-context-window/context-supervisor.html' | relative_url }})).
- A task tracker as the record of all work; the task panel in the terminal only displays it.

To start: one notes file the agent updates before each compaction.

## Pitfalls and lessons

> An index that grows past what loads drops its oldest lines without a warning. Keep indexes short and archive old entries.
{: .warning }

A session can fall back to a smaller context window than expected; check the limit you actually got. A guess at "how full" is not a measurement: read the number.

## Sources

- Compaction, `/compact`, what reloads, one-million-token models: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-08.
- Codex compaction threshold: <https://learn.chatgpt.com/docs/config-file/config-reference.md>, fetched 2026-11-08 (no default number given).
- Codex pricing: <https://learn.chatgpt.com/docs/pricing.md>, fetched 2026-11-08.

{% include page_nav.html %}
