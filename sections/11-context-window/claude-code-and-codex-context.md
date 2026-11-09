---
title: "Context in Claude Code and Codex"
parent: "Compaction and saving state"
grand_parent: "Context window and memory"
nav_order: 2
---

{% include page_nav.html %}

# Context in Claude Code and Codex

As of 2026-11-09.

A sub-topic of [compaction and saving state]({{ '/sections/11-context-window/compaction.html' | relative_url }}). The two coding agents handle a filling context window in similar ways with different defaults and different controls. This page compares what each documents, what I have observed, and what to check for your own plan.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Side by side

| | Claude Code | Codex |
|---|---|---|
| Automatic compaction | Near the limit | At a threshold the configuration reference describes as "model defaults", with no number given |
| Manual compaction | `/compact <focus>` steers the summary | A manual compaction command exists; check your version |
| Setting the threshold | `/autocompact` sets it | A configuration setting |
| What reloads after compaction | Root `CLAUDE.md`, rules without paths, auto memory | Its context file, `AGENTS.md`, is read again at session start; check what reloads after compaction in your version |
| Seeing what fills the window | `/context`, by category, with suggestions | Check your version's status output |
| Long windows | One-million-token models are documented | Depends on the model |

Sources for the Claude Code column: the context-window page in the Claude Code docs, verified 2026-11-08. The Codex column is thinner because its documentation gives fewer numbers.

## Observations, marked as such

> My observation is that Codex compacts at about 250k tokens by default, and that going past that costs more usage or API spend. Unverified: OpenAI's configuration reference says the compaction threshold uses "model defaults" and gives no number, and its pricing page says subscription use of one current model "has no long-context surcharge". Check current pricing for your plan.
{: .warning }

My own practice is to compact at about 50 to 60 percent of the context window in either tool. Other practitioners report a sweet spot below about 300k tokens (unverified practitioner reports).

## Bigger windows

A one-million-token window delays compaction; it does not remove the drop in instruction-following as the window fills. A session at 600k tokens of mostly tool output has the same problem as one at 150k on a smaller model, with more to summarise when it finally compacts.

#### Example: checking the window you got

A session can fall back to a smaller context window than expected, for example when a long-context option is not available on your account or route. Before planning a long run around a large window, check the limit the session reports, not the one you selected. An illustrative reading, not real output:

```text
> /context
  Context window: 200k tokens   (expected 1M: long-context not active)
  Used: 41% ...
```

## Working across both

![A sequence diagram. Claude Code writes a state note and updates the tracker in the repository's state files. You ask Codex to continue. Codex reads the state note, gets the next step back, updates the state note, and reports progress to you.]({{ '/assets/images/11-context-window/tool-handover.svg' | relative_url }}){: style="width:100%"}

*Figure: one task continued across two tools through shared state files. Made with archify.*

When Claude Code and Codex work on the same repository, their context files differ (`CLAUDE.md` and `AGENTS.md`), but the files that survive compaction can be shared: the state note, the memory files and the task tracker. Write them as plain Markdown in the repository or a notes folder, and either agent can resume from them.

## Sources

- Claude Code compaction, `/compact`, what reloads, one-million-token models: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-08.
- Codex compaction threshold: <https://learn.chatgpt.com/docs/config-file/config-reference.md>, fetched 2026-11-08 (no default number given).
- Codex pricing: <https://learn.chatgpt.com/docs/pricing.md>, fetched 2026-11-08.

{% include page_nav.html %}
