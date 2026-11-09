---
title: "Context window and memory"
nav_order: 21
has_children: true
has_toc: false
permalink: /sections/11-context-window/
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Context window and memory

A session's context window is everything the model can see at once: your context files, the conversation, every file it read and every command's output. As it fills, two things happen:

- The agent follows instructions less reliably.
- Near the limit, the session compacts: it replaces the history with a summary, and whatever was not written down is gone.

Keeping a long session useful comes down to four habits, and the tools that enforce them:

1. Measure how full the window is.
2. Before compacting, write the task, the decisions and the next step to a file.
3. Keep lessons in memory files that load in the next session.
4. Keep the real task list in a tracker outside the session.

![A sequence diagram of one compaction. The context hook reports 58% used to the session. Inside a critical fragment, the session writes a state note and updates the tracker in its files, which answer "saved". The session then runs /compact with a focus, and compaction returns a summary. The session reads its state note back from the files, which return the next step, and reports that it resumed at 9%.]({{ '/assets/images/11-context-window/save-state-and-compact.svg' | relative_url }}){: style="width:100%"}

*Figure: save first, then compact, then read the state back. Made with archify.*

Stop 8 on the talk path, about 2 minutes.

Pages in this section, in reading order:

- This overview: what fills the window, why quality drops, compaction, memory and the tracker.
- [Compaction and saving state]({{ '/sections/11-context-window/compaction.html' | relative_url }}): when to compact, what survives in detail, steering the summary, the state note and resuming.
  - [A context supervisor]({{ '/sections/11-context-window/context-supervisor.html' | relative_url }}): hooks that measure the window, save and compact on a threshold.
  - [Context in Claude Code and Codex]({{ '/sections/11-context-window/claude-code-and-codex-context.html' | relative_url }}): defaults, controls and observations for each tool.
- [Memory files and lessons]({{ '/sections/11-context-window/memory-files.html' | relative_url }}): auto memory, writing a lesson, the index and its limit, and where a fix belongs.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What fills the window

| Source | When it loads | Size |
|---|---|---|
| `CLAUDE.md` (or `AGENTS.md`) and rules without `paths:` | Before you type anything | Small to medium; the docs suggest under 200 lines per `CLAUDE.md` |
| Auto memory | Before you type: the first 200 lines or 25KB of `MEMORY.md` | Small, if kept to an index |
| MCP tool names and skill descriptions | Before you type | Small per item; adds up with many servers and skills |
| Rules with `paths:` and nested `CLAUDE.md` files | When Claude reads or edits a matching file | Small to medium |
| Your messages and the agent's replies | Every turn | Grows steadily |
| File reads | Each read | Large: one big file can be thousands of tokens |
| Command and hook output | Each command, each hook | Large for test runs and logs |
| Sub-agent results | When a sub-agent returns | Small: the summary only, which is the point of [delegation]({{ '/sections/08-delegation/' | relative_url }}) |

`/context` shows your live usage by category, with suggestions, including which `CLAUDE.md` and memory files loaded. `/memory` opens those files.

<!-- IMAGE NEEDED: Claude Code /context output showing context usage by category; search: claude code /context command, context window usage breakdown, claude code context visualization -->

#### Example: where the space went

1. A session has been fixing a test for an hour and runs `/context`.
2. Most of its window is tool output: the full test suite's output, printed five times.
3. It changes how it works.
   - The next test run uses `-q` and prints only failures.
   - The next large search goes to a sub-agent that returns a ten-line summary.

### Keeping it small

| Habit | Command or mechanism |
|---|---|
| Start fresh between unrelated tasks | `/clear` |
| Keep large reads out of the main window | A sub-agent for search and research |
| Print less | Quiet flags on tests and builds; `tail` on logs |
| Summarise part of the conversation | `/rewind`, then "Summarize from here" or "Summarize up to here" |
| Compact on your terms | `/compact <focus>` before a long new task |

## Quality drops as the window fills

As the context window fills, an agent follows instructions less reliably: rules stated at the start compete with everything since. Treat this as something to watch, because it changes as models improve.

> My own practice is to compact at about 50 to 60 percent of the context window. Other practitioners report a sweet spot below about 300k tokens (unverified practitioner reports).
{: .note }

| Window | Models (Claude Code docs) |
|---|---|
| 1 million tokens | Fable models, Sonnet 5 and later, Haiku 5.5, Opus 4.6 and later, Sonnet 4.6, selected as a `[1m]` variant where your plan offers it |
| Smaller | Other models, and plans without extended context |

A larger window delays compaction; compaction works the same way at the larger limit.

### Codex

- My observation: Codex compacts at about 250k tokens by default, and going past that costs more usage or API spend.
- Unverified: OpenAI's configuration reference says the compaction threshold uses "model defaults" and gives no number.
- Its pricing page says subscription use of one current model "has no long-context surcharge".
- Check current pricing for your plan. More in [context in Claude Code and Codex]({{ '/sections/11-context-window/claude-code-and-codex-context.html' | relative_url }}).

## Compaction

Claude Code compacts automatically as the window nears its limit; `/compact` compacts on request.

| Control | What it does |
|---|---|
| `/compact <focus>` | Compacts now, steering the summary toward what you name |
| `/autocompact 500k` | Sets how full the window gets before automatic compaction, as a token count |
| Idle compaction | A setting that also compacts a conversation left idle |

### What survives

| Content | After compaction |
|---|---|
| Project-root `CLAUDE.md`, rules without `paths:`, auto memory | Re-injected from disk |
| Rules with `paths:`, nested `CLAUDE.md` | Reloaded only when a matching file is touched again |
| Files Claude read or edited | Up to five re-read, most recently modified first; a file over 5,000 tokens comes back as a path only |
| Invoked skill bodies | Re-injected, capped at 5,000 tokens each and 25,000 in total |
| Background commands and sub-agents | Keep running |
| `SessionStart` hooks matching `compact` | Run, and their output is added |
| Everything else in the conversation | Carried only by the summary |

### Write state before compacting

The summary keeps the gist and drops specifics: the exact error, the file and line, the decision made forty minutes ago and why. Write those to a file first.

```markdown
# State: export timeout fix (2026-11-09 14:20)

Task: nightly export times out after last week's migration.
Done: cause found, the index on rows.created_at was dropped.
Decided: restore the index in a new migration; old migrations already ran in production.
Next: write test_export_uses_index, then the migration, then run the suite.
Open: does the archive table need the same index? Ask before changing it.
```

### Make the resume automatic

A `SessionStart` hook with the `compact` matcher runs after every compaction and adds its output to the new context. Pointing it at the state note means the resume does not depend on the agent remembering to read it.

```json
{
  "hooks": {
    "SessionStart": [
      { "matcher": "compact",
        "hooks": [ { "type": "command", "command": "cat notes/state.md" } ] }
    ]
  }
}
```

[Compaction and saving state]({{ '/sections/11-context-window/compaction.html' | relative_url }}) covers the note, the resume and `/compact <focus>` in depth; [a context supervisor]({{ '/sections/11-context-window/context-supervisor.html' | relative_url }}) automates the save.

## Memory between sessions

Two mechanisms carry knowledge across sessions in Claude Code:

| | `CLAUDE.md` files | Auto memory |
|---|---|---|
| Who writes it | You | Claude |
| Contains | Instructions and rules | Learnings: your preferences, corrections, project facts, where to find things |
| Scope | Project, user or organisation | One repository, shared across its worktrees, on one machine |
| Loads | Every session, in full | Every session: the first 200 lines or 25KB of `MEMORY.md`; topic files on demand |

The Claude Code docs say both are treated as context, and point to a `PreToolUse` hook for anything that must be blocked regardless of what Claude decides.

![A sequence diagram. Session 1 writes a lesson file and adds one line to the index. Days later session 2 loads the index at start, gets the pointer, reads the lesson file and gets the story behind the rule.]({{ '/assets/images/11-context-window/lesson-reaches-later-session.svg' | relative_url }}){: style="width:100%"}

*Figure: a lesson travels from one session to a later one through a short index. Made with archify.*

#### Example: a lesson

```markdown
# Count a message as delivered only when the target replies

What happened: a send timed out, I re-sent through the fallback, and the
target got the task twice. The first send had arrived.
Rule: treat a timeout as unknown. Ask the target before re-sending.
Applies to: any channel with a timeout.
Enforced by: the send tool refuses a second send of the same task id.
```

- Recording the failure lets the agent apply the general idea to a new case.
- The "Enforced by" line points to the fix in the system: the lesson guides, and the tool or hook makes the mistake impossible.
- The same hooks and checks can also enforce how you want the output to be from the start. See [where a fix belongs]({{ '/sections/11-context-window/memory-files.html' | relative_url }}#where-a-fix-belongs).

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
| `/context` | Live context usage by category, with suggestions |
| `/compact <focus>` | Compacts now, steering the summary |
| `/autocompact <tokens>` | Sets the automatic compaction threshold, such as `/autocompact 500k` |
| `/clear` | Starts a fresh conversation for unrelated work |
| `/rewind` | Rewinds, or summarises part of the conversation |
| `/memory` | Opens `CLAUDE.md` and auto memory files; toggles auto memory |

## Examples from my own harness

- A save-state skill that writes the current state before compaction, in a fixed resume-message format.
  - It records the task, decisions, git state, what is next, and any feedback from me in that session.
- A memory index with one line per lesson and one file per lesson, each recording the failure behind it.
- A command that reports the actual context percentage, and hooks that act on it ([a context supervisor]({{ '/sections/11-context-window/context-supervisor.html' | relative_url }})).
- A task tracker as the record of all work; the task panel in the terminal only displays it.

To start: one notes file the agent updates before each compaction, and a `SessionStart` hook with the `compact` matcher that prints it.

## Pitfalls and lessons

| Pitfall | What helps |
|---|---|
| A memory index past its load limit | `MEMORY.md` beyond 200 lines or 25KB is not loaded at start; keep one line per entry and archive old ones |
| A rule with `paths:` that must hold after compaction | Drop the `paths:` frontmatter, or move the rule to the root `CLAUDE.md` |
| A long skill truncated after compaction | Put its most important instructions near the top of `SKILL.md` |
| A smaller window than you expected | Check the limit the session reports; a long-context option may not be active on your plan |
| A guess at "how full" | Read the number with `/context` or a hook |

## Sources

- What fills the window, what survives compaction, `/autocompact`, `/rewind`, 1M-token models: [Explore the context window (Claude Code docs)](https://code.claude.com/docs/en/context-window.md), verified 2026-11-09.
- `CLAUDE.md` and auto memory, load limits: [How Claude remembers your project (Claude Code docs)](https://code.claude.com/docs/en/memory.md), verified 2026-11-09.
- `SessionStart` hook with the `compact` matcher: [Automate actions with hooks (Claude Code docs)](https://code.claude.com/docs/en/hooks-guide.md), verified 2026-11-09.
- Codex compaction threshold: [Codex configuration reference (OpenAI)](https://learn.chatgpt.com/docs/config-file/config-reference.md), fetched 2026-11-08 (no default number given).
- Codex pricing: [Codex pricing (OpenAI)](https://learn.chatgpt.com/docs/pricing.md), fetched 2026-11-08.

{% include page_nav.html %}
