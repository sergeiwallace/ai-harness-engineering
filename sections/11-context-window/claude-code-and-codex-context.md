---
title: "Context in Claude Code and Codex"
parent: "Compaction and saving state"
grand_parent: "Context window and memory"
nav_order: 2
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Context in Claude Code and Codex

A sub-topic of [compaction and saving state]({{ '/sections/11-context-window/compaction.html' | relative_url }}):

- Claude Code and Codex handle a filling context window in similar ways, with different commands, settings and defaults.
- This page compares what each one documents, what I have observed, and what to check for your own plan.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Side by side

| | Claude Code | Codex |
|---|---|---|
| Automatic compaction | Near the limit; the point depends on the model and configuration | At `model_auto_compact_token_limit`; unset "uses model defaults", with no number given |
| Moving the threshold | `/autocompact <tokens>`, such as `/autocompact 500k` | `model_auto_compact_token_limit` in `config.toml` |
| Other automatic compaction | Idle compaction: a setting that compacts a conversation left idle | `model_post_turn_compact_threshold_percent`: compacts after a final response once the context reaches that percentage (0 or unset turns it off) |
| Manual compaction | `/compact <focus>` steers the summary | `/compact`: "Summarize the visible chat to free tokens" |
| Changing the summary prompt | The focus text on `/compact` | `compact_prompt`, or `experimental_compact_prompt_file` |
| Summarising part of a conversation | `/rewind`, then "Summarize from here" or "up to here" | `/recap` summarises for you without compacting |
| Seeing how full it is | `/context`, by category, with suggestions | `/status`, including remaining context capacity |
| Starting fresh | `/clear` | `/new`, or `/clear` |
| Context file | `CLAUDE.md` (can read `AGENTS.md` in its place) | `AGENTS.md`, up to `project_doc_max_bytes` |
| What reloads after compaction | Root `CLAUDE.md`, rules without `paths:`, auto memory, skill descriptions, up to five recent files | Not documented on the pages checked here |
| Long windows | 1M tokens on Fable models, Sonnet 5 and later, Haiku 5.5, Opus 4.6 and later, Sonnet 4.6, as a `[1m]` variant where the plan offers it | `model_context_window` reports the active model's window |

## Observations, marked as such

> My observation is that Codex compacts at about 250k tokens by default, and that going past that costs more usage or API spend. Unverified: OpenAI's configuration reference says the compaction threshold uses "model defaults" and gives no number, and its pricing page says subscription use of one current model "has no long-context surcharge". Check current pricing for your plan.
{: .warning }

- My own practice is to compact at about 50 to 60 percent of the context window in either tool.
- Other practitioners report a sweet spot below about 300k tokens (unverified practitioner reports).

## Setting the threshold

### Claude Code

```text
/autocompact 500k
```

### Codex

```toml
# ~/.codex/config.toml
model_auto_compact_token_limit = 250000
model_post_turn_compact_threshold_percent = 60
```

Both lines are examples; pick numbers for your model's window and your own compaction point.

## Bigger windows

A one-million-token window delays compaction. The drop in instruction-following as the window fills still applies, and compaction works the same way at the larger limit.

| Session | Fill | What it holds |
|---|---|---|
| A: 1M-token window | 600k tokens | Mostly tool output from long test runs |
| B: 200k-token window | 150k tokens | Mostly the same kind of output |

Both sessions are past my compaction point, and A has more to summarise when it compacts.

#### Example: checking the window you got

A session can fall back to a smaller context window than expected, for example when a long-context option is not available on your account or route.

1. Before planning a long run, check the limit the session reports.
   - Claude Code: `/context`.
   - Codex: `/status`.
2. Compare it with the window you selected.
3. If it is smaller, plan the compaction point for the smaller window.

## Working across both

![A sequence diagram. Claude Code writes a state note and updates the tracker in the repository's state files. You ask Codex to continue. Codex reads the state note, gets the next step back, updates the state note, and reports progress to you.]({{ '/assets/images/11-context-window/tool-handover.svg' | relative_url }}){: style="width:100%"}

*Figure: one task continued across two tools through shared state files. Made with archify.*

When Claude Code and Codex work on the same repository:

- Their context files differ: `CLAUDE.md` and `AGENTS.md`. Claude Code can read `AGENTS.md` in place of `CLAUDE.md`, so one file can serve both.
- The files that survive compaction can be shared: the state note, the memory or lesson files, and the task tracker.
- Write those as plain Markdown in the repository or a notes folder, and either tool can resume from them.

## Sources

- Claude Code: what survives compaction, `/autocompact`, `/rewind`, 1M-token models: [Explore the context window (Claude Code docs)](https://code.claude.com/docs/en/context-window.md), verified 2026-11-09.
- Claude Code reading `AGENTS.md`: [How Claude remembers your project (Claude Code docs)](https://code.claude.com/docs/en/memory.md), verified 2026-11-09.
- Codex configuration keys (`model_auto_compact_token_limit`, `model_post_turn_compact_threshold_percent`, `compact_prompt`, `model_context_window`, `project_doc_max_bytes`): [Codex configuration reference (OpenAI)](https://learn.chatgpt.com/docs/config-file/config-reference.md), verified 2026-11-09.
- Codex slash commands (`/compact`, `/status`, `/new`, `/clear`, `/recap`): [Codex CLI slash commands (OpenAI)](https://learn.chatgpt.com/docs/cli/slash-commands.md), verified 2026-11-09.
- Codex pricing: [Codex pricing (OpenAI)](https://learn.chatgpt.com/docs/pricing.md), fetched 2026-11-08.

{% include page_nav.html %}
