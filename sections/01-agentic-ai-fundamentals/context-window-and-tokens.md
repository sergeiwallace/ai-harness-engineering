---
title: "The context window and tokens"
parent: "Agentic AI fundamentals"
nav_order: 3
---

{% include page_nav.html %}

# The context window and tokens

As of 2026-11-09.

The context window is everything the model can see on one turn. A rule outside it does not exist for the model, and a rule buried deep inside it competes with everything else. The [Agentic AI fundamentals overview]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }}#the-context-window) is the short version, and [Context window and memory]({{ '/sections/11-context-window/' | relative_url }}) covers techniques for long runs. This page covers:

- what Claude Code puts in the window, and when;
- tokens, window sizes and a worked budget;
- why quality drops before the window is full;
- compaction: what survives it and how to compact well;
- prompt caching: why resending the context is cheaper than it looks, and what breaks the cache.

> When an agent ignores a rule, check first whether the rule was in its context at that moment, and how much else was in the context with it.
{: style="border-left: 4px solid #0f766e; background: #f0fdfa; padding: 0.75rem 1rem;"}

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What fills the window

![Diagram: the sources that flow into one turn's context window, system instructions, context files, tool definitions, conversation history, tool results and the newest message, and the window flowing to the model, with compaction replacing old history by a summary]({{ '/assets/images/01-agentic-ai-fundamentals/context-sources.svg' | relative_url }}){: style="width: 100%" }
*Figure: every source the client sends on each turn, and where compaction cuts in.*

### Before you type anything

| Part | What it is | Notes |
|---|---|---|
| System prompt | Claude Code's core instructions and built-in tool definitions | changes only when the set of loaded tools changes |
| `CLAUDE.md` files | instructions at four levels: managed policy, user (`~/.claude/CLAUDE.md`), project (`./CLAUDE.md`) and local (`./CLAUDE.local.md`), plus those in parent directories | files in subdirectories load on demand; `AGENTS.md` can load in place of `CLAUDE.md`; [Context files]({{ '/sections/04-context-files/' | relative_url }}) covers them |
| Auto memory | notes Claude writes about your corrections and preferences | the first 200 lines or 25 KB load |
| Skill descriptions | one entry per skill, so the model knows when to use it | the skill body loads only when the skill runs |
| MCP tool names | with tool search on, names and server instructions only | full definitions load on demand |

### As the session runs

| Part | When it enters | Grows? |
|---|---|---|
| Your messages and the model's replies | every turn | yes |
| Tool results | after each tool call | yes, usually fastest |
| Path-scoped rules and nested `CLAUDE.md` files | when the agent reads or edits a file they cover | once per trigger |
| Hook output, such as a `PostToolUse` hook's message | when the hook fires | yes |
| Sub-agent results | when a sub-agent finishes | the summary only, which keeps its large reads out of your window |

`/context` shows the live breakdown for your session, including which `CLAUDE.md` and memory files loaded, with suggestions for reducing it.

## Tokens

### What a token is

Models read text as tokens: pieces of words, punctuation and whitespace.

- Common English words are often one token each.
- Rare words, long code identifiers and non-English text split into several.
- For English prose a rough rule is four characters, or three quarters of a word, per token.

An illustrative split (the exact boundaries depend on the model's tokenizer):

```text
"The parser rejects February 29."  ->  The | parser | rejects | February | 29 | .
"parse_date_with_timezone"         ->  parse | _date | _with | _timezone
```

### Window sizes

- Several current Claude models support a one-million-token window: Fable models, Sonnet 5 and later, Haiku 5.5, and Opus 4.6 and later.
- In Claude Code the larger window is selected as a `[1m]` model variant, where your plan allows it.
- Compaction works the same way at the larger size; `/autocompact` sets how full the window gets before it runs.
- `/context` shows your session's actual size and use.

### Rules of thumb

| Content | Rough size |
|---|---|
| One page of prose (500 words) | about 650 tokens |
| A 300-line Python file | 3,000 to 4,000 tokens |
| One `Bash` result at Claude Code's default inline limit | about 30,000 characters, roughly 7,500 tokens |
| A typical project `CLAUDE.md` | 1,000 to 5,000 tokens |

These are planning estimates; `/context` gives the real numbers.

### Worked example: a morning's budget

<details markdown="1" open>
<summary>Adding up one realistic session</summary>

| Item | Tokens | Running total |
|---|---:|---:|
| System prompt and built-in tools | 15,000 | 15,000 |
| `CLAUDE.md` and auto memory | 5,000 | 20,000 |
| Five MCP servers, tool names only (tool search on) | 1,000 | 21,000 |
| Reading 12 source files | 40,000 | 61,000 |
| Six quiet test runs | 9,000 | 70,000 |
| Four verbose test runs, each near the inline limit | 30,000 | 100,000 |
| Conversation and the model's own replies | 25,000 | 125,000 |

On a 200,000-token window this session is well over half full before lunch; on a one-million-token window it has plenty of room. The four verbose runs cost more than the context files, the MCP servers and the quiet runs together. The numbers are illustrative.
</details>

## Why quality drops before the limit

### Attention is spread across the whole window

A model attends to its whole context on every turn, but not evenly.

- As the window fills, an instruction near the start has more text competing with it.
- Long-context studies find that models recall facts at the beginning and end of a long input better than facts in the middle.
- Accuracy on multi-step tasks falls as irrelevant text grows.
- The size of the effect varies by model and has shrunk with each generation; it has not gone away.

Claude Code's own documentation says it treats `CLAUDE.md` and auto memory "as context, not enforced configuration", and points to a `PreToolUse` hook for anything that must be blocked regardless. That gives an order of reliability:

| Where a rule lives | How reliably it holds late in a long session |
|---|---|
| A hook or permission rule | enforced whatever the context holds |
| The project-root `CLAUDE.md` | reloaded after compaction; followed most of the time |
| A path-scoped rule | loaded with its files; summarized away at compaction until those files are read again |
| Early in the chat, hours ago | least reliable; may be lost at compaction |

### Symptoms

| Symptom | What it looks like | What to do |
|---|---|---|
| Drift | the agent follows a convention for an hour, then a test file lands in the wrong folder | move the convention into a check, such as a lint rule on file locations |
| Re-asking | it reads a file it already read, or asks a question answered earlier | compact with a focus, or start fresh with a handoff note |
| Mixing tasks | after two unrelated tasks, it applies the first task's assumptions to the second | `/clear` between unrelated tasks |

> Start a fresh session for unrelated work, and give it the files that task needs. `/clear` starts a new conversation and keeps project memory.
{: style="border-left: 4px solid #b91c1c; background: #fef2f2; padding: 0.75rem 1rem;"}

## Compaction

### What happens

When the window nears its limit, Claude Code compacts automatically:

1. It sends the conversation to the model with an instruction to summarise it.
2. It replaces the message history with that summary.
3. It reloads what it can from disk, as in the table below, and the session continues.

You can also compact yourself, at a moment you choose.

### What survives

| Content | After compaction |
|---|---|
| System prompt | still applies |
| Project-root `CLAUDE.md` and unscoped rules | re-injected from disk |
| Auto memory | re-injected from disk |
| The plan written in plan mode | re-injected from disk |
| Skill descriptions | rebuilt and re-injected |
| Skills that ran in the session | re-injected, up to 5,000 tokens each and 25,000 in total; the oldest drop first |
| Files the agent read or edited | the five most recently modified are read again; a file over 5,000 tokens comes back as a path only |
| Path-scoped rules and nested `CLAUDE.md` files | summarized away; reloaded when the agent next touches a file they cover |
| Context that hooks added earlier | summarized with the rest |
| Background commands and sub-agents | keep running |
| A rule you typed in chat, a caveat a tool printed once | only as well as the summary describes them |

Permission rules and hooks are not in the context at all, so compaction does not affect them.

### Worked example: the rule that vanished

1. Early in a long session you tell the agent: "never edit the generated files under `build/`".
2. Two hours and one compaction later, it edits one; the summary did not carry the chat instruction.
3. The systems fix, in order of strength:
   - a deny rule, enforced on every call whatever the context holds:

     ```json
     { "permissions": { "deny": ["Edit(build/**)"] } }
     ```

   - a line in the project-root `CLAUDE.md`, reloaded after compaction, as the second layer that tells the model why.
4. A version of the trap to know about: had the line been in a rule file with `paths:` frontmatter, it would also have been summarized away. A rule that must survive compaction belongs in the project-root `CLAUDE.md` or a rule with no `paths:`.

[Permissions and untrusted input]({{ '/sections/01-agentic-ai-fundamentals/permissions-and-untrusted-input.html' | relative_url }}) covers deny rules and where they stop.

### Compacting well

| Command | What it does | When to use it |
|---|---|---|
| `/compact focus on the auth bug fix` | compacts now, keeping what you name | at a natural break, before a long new task |
| `/rewind`, then "Summarize from here" or "Summarize up to here" | summarizes part of the conversation | when one stretch of the session is no longer needed |
| `/autocompact 500k` | sets how full the window gets before automatic compaction | on a large window, to compact earlier |
| `/clear` | starts a new conversation and keeps project memory | switching to unrelated work |
| delegate to a sub-agent | keeps large reads in the sub-agent's window | research that reads many files |

A routine for long tasks:

1. Before compacting, have the agent write what is done, what is next and what to watch out for to a plan file.
2. Compact at the break with `/compact` and a focus that names what must survive, such as "keep the list of failing tests and the migration order".
3. Put anything that must survive every compaction in the project-root `CLAUDE.md`, or have a `SessionStart` hook with the `compact` matcher add it back after each compaction.

## Caching

### Why repeated context is cheaper than it looks

Each turn resends the whole context. Prompt caching makes that much cheaper:

1. Claude Code orders each request so that what rarely changes comes first: the system prompt and tool definitions, then `CLAUDE.md`, auto memory and unscoped rules, then the conversation.
2. The API matches the start of each request against content it processed recently. The match is exact, so a change anywhere in the prefix reprocesses everything after it.
3. The matched prefix is billed at the lower cached-token rate, and the turn starts faster.
4. Only the newest part of the request is processed in full.

### What breaks the cache, and what does not

| Action | Effect on the cache |
|---|---|
| Switching models with `/model` | breaks it: each model has its own cache |
| Changing effort level | breaks it on most models; Opus 5.5, Sonnet 5.5, Haiku 5.5 and Fable 5.1 keep it with an API key or subscription |
| Turning fast mode on or off | breaks it: each speed has its own cache |
| Compacting | replaces the conversation part; the system prompt can stay cached |
| Upgrading Claude Code | the first session afterwards builds its cache from the top |
| Connecting or removing an MCP server | no effect with tool search on, the default; breaks it when tools load up front |
| Editing `CLAUDE.md` mid-session | no effect, and the edit does not apply until `/clear`, `/compact` or a restart |
| Changing permission mode, running skills, `/rewind`, spawning a sub-agent | no effect |

### How long the cache lasts

| Billing | Main conversation | Sub-agents, compaction and other requests |
|---|---|---|
| Claude subscription, within plan usage | one hour | five minutes |
| API key, usage credits or a cloud provider | five minutes | five minutes |

The `promptCacheTtl` setting chooses `5m` or `1h` for the main conversation. `/usage` shows a `Prompt cache (main)` line with the session's hit ratio and, where Claude Code can tell, the likely cause of the last miss.

### Worked example: a slow turn after lunch

1. You work for an hour with a warm cache, so each turn reads most of its input from the cache.
2. You leave for 40 minutes on API-key billing, with a five-minute TTL.
3. The first turn back reprocesses the whole conversation at the full input rate, and is visibly slower.
4. You then switch from one model to another to "save money", which reprocesses everything again on the new model.

Picking the model and effort at the start of a session, and compacting at breaks, avoids both costs.

## A context checklist

- Keep `CLAUDE.md` short and specific; move rarely needed detail into files the agent reads on demand.
- Put rules that must hold into hooks and permission rules; keep the `CLAUDE.md` line as the explanation.
- Load only the MCP servers a project uses.
- Trim tool output: quiet test runs, capped searches, the tail of logs.
- Read only the part of a file you need.
- One task per session where you can; `/clear` between unrelated tasks.
- Write state to a plan file before compaction, and compact at breaks with a focus.

## Sources

- What loads at start and during a session, what survives compaction, `/compact`, `/rewind`, `/autocompact`, `/clear`, `/context`, and one-million-token windows: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-09.
- `CLAUDE.md` and auto memory loading, the 200-line and 25 KB limit, and "context, not enforced configuration": <https://code.claude.com/docs/en/memory.md>, verified 2026-11-09.
- Prompt caching in Claude Code: request layers, what invalidates the cache, TTLs, `promptCacheTtl` and `/usage`: <https://code.claude.com/docs/en/prompt-caching.md>, verified 2026-11-09.
- `Bash` output limits: <https://code.claude.com/docs/en/tools-reference.md>, verified 2026-11-09.
- Position effects in long contexts: Liu et al., "Lost in the Middle: How Language Models Use Long Contexts", 2023, <https://arxiv.org/abs/2307.03172>, not yet verified against the primary source for this page's date.
- The token sizes and the session budget are illustrative estimates written for this page.

{% include page_nav.html %}
