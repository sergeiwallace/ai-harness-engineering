---
title: "The context window and tokens"
parent: "Agentic AI fundamentals"
nav_order: 3
---

{% include page_nav.html %}

# The context window and tokens

As of 2026-11-09.

The context window is everything the model can see on one turn. It is the agent's whole world: a rule outside it does not exist, and a rule buried deep inside it competes with everything else. This page explains tokens, works through a real-looking budget, shows why quality drops before the window is full, and covers compaction and caching. The [Agentic AI fundamentals overview]({{ '/sections/01-agentic-ai-fundamentals/' | relative_url }}#the-context-window) is the short version, and [Context window and memory]({{ '/sections/11-context-window/' | relative_url }}) covers management techniques for long runs.

> When an agent ignores a rule, check first whether the rule was in its context at that moment, and how much else was in the context with it.
{: style="border-left: 4px solid #0f766e; background: #f0fdfa; padding: 0.75rem 1rem;"}

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What fills the window

![Diagram: the sources that flow into one turn's context window, system instructions, context files, tool definitions, conversation history, tool results and the newest message, and the window flowing to the model, with compaction replacing old history by a summary]({{ '/assets/images/01-agentic-ai-fundamentals/context-sources.svg' | relative_url }}){: style="width: 100%" }
*Figure: every source the client sends on each turn, and where compaction cuts in.*

| Part | Who puts it there | Changes during a session? |
|---|---|---|
| System instructions | the client | rarely |
| Context files (`CLAUDE.md`, `AGENTS.md`, rules) | the client, from disk | reloaded at start and after compaction |
| Tool, skill and MCP descriptions | the client and its plugins | when servers connect or skills load |
| Conversation history | you and the model | grows every turn |
| Tool results | the client, after each tool runs | grows every turn, usually fastest |
| The newest message | you, or a tool result | every turn |

The first three are a fixed cost paid on every turn. The last three grow, and in coding work the tool results grow fastest.

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

### Rules of thumb

| Content | Rough size |
|---|---|
| One page of prose (500 words) | about 650 tokens |
| A 300-line Python file | 3,000 to 4,000 tokens |
| A verbose test run, 2,000 lines | 25,000 tokens or more |
| A typical project `CLAUDE.md` | 1,000 to 5,000 tokens |

These are planning estimates. Claude Code's `/context` command shows the actual breakdown of the current window.

### Worked example: a morning's budget

<details markdown="1" open>
<summary>Adding up one realistic session</summary>

| Item | Tokens | Running total |
|---|---:|---:|
| System instructions and built-in tools | 15,000 | 15,000 |
| Context files | 4,000 | 19,000 |
| Five MCP servers' tool definitions | 12,000 | 31,000 |
| Reading 12 source files | 40,000 | 71,000 |
| Six test runs, quiet output | 9,000 | 80,000 |
| One verbose test run, by mistake | 30,000 | 110,000 |
| Conversation and the model's own replies | 25,000 | 135,000 |

On a 200,000-token window this session is two thirds full before lunch; on a one-million-token window it has plenty of room. The verbose run alone cost more than all the context files and the quiet runs together. The numbers are illustrative; measure your own with `/context`.
</details>

## Why quality drops before the limit

### Attention is spread across the whole window

A model attends to its whole context on every turn, but not evenly.

- As the window fills, an instruction near the start has more text competing with it.
- Long-context studies find that models recall facts at the beginning and end of a long input better than facts in the middle.
- Accuracy on multi-step tasks falls as irrelevant text grows.
- The size of the effect varies by model and has shrunk with each generation; it has not gone away.

| Placement of a rule | How reliably it is followed late in a long session |
|---|---|
| In a context file, reloaded each session and after compaction | most reliable of the text options |
| Early in the chat, hours ago | least reliable; may also be lost at compaction |
| In the latest message | reliable for that turn |
| In a hook or permission rule | enforced whatever the context holds |

### Symptoms

#### Drift

The agent follows a convention for the first hour and then slips: a test file in the wrong folder, a commit message in the wrong format.

#### Re-asking

The agent reads a file it already read, or asks a question answered earlier, because the earlier answer is far back.

#### Mixing tasks

After two unrelated tasks in one session, the agent applies the first task's assumptions to the second.

> Start a fresh session for unrelated work, and give it the files that task needs.
{: style="border-left: 4px solid #b91c1c; background: #fef2f2; padding: 0.75rem 1rem;"}

## Compaction

### What happens

When the window nears its limit, Claude Code compacts: it asks the model to summarise the conversation so far, replaces the history with that summary, and carries on. You can also run `/compact` yourself, with optional instructions for what to keep.

| Before compaction | After compaction |
|---|---|
| system instructions | system instructions |
| context files | context files, reloaded from disk |
| full history: every message and tool result | a summary of the history |
| recent turns | the most recent turns, depending on the client |

### What survives and what does not

| Survives exactly | Survives only as well as the summary describes it |
|---|---|
| the project-root `CLAUDE.md` and loaded rules | a rule you typed in chat |
| files the agent reads again | a caveat a tool printed once |
| permission rules and hooks | the exact list of tests already run |
| the plan file, if the agent wrote one | why an approach was abandoned |

### Worked example: the rule that vanished

1. Early in a long session you tell the agent: "never edit the generated files under `build/`".
2. Two hours and one compaction later, it edits one; the summary did not carry the chat instruction.
3. The systems fix, in order of strength:
   - a deny rule, enforced on every call whatever the context holds:

     ```json
     { "permissions": { "deny": ["Edit(build/**)"] } }
     ```

   - a line in the project's `CLAUDE.md`, reloaded after compaction, as the second layer that tells the model why.

[Permissions and untrusted input]({{ '/sections/01-agentic-ai-fundamentals/permissions-and-untrusted-input.html' | relative_url }}) covers deny rules and where they stop.

### Compacting well

1. Compact at a natural break, after a task is finished and its state is written down, rather than letting it fire mid-step.
2. Before compacting, have the agent write what is done, what is next and what to watch out for to a plan file.
3. Pass `/compact` a short instruction naming what must survive, such as "keep the list of failing tests and the migration order".
4. Move anything that must survive every compaction into a context file.

## Caching

### Why repeated context is cheaper than it looks

Each turn resends the whole context. Prompt caching makes that much cheaper:

1. The client sends the stable parts first: system instructions, tools, context files.
2. The changing parts come last: history, tool results, the newest message.
3. When the start of a request matches a recent one exactly, the provider reuses its earlier work on that prefix.
4. The reused prefix is billed at a fraction of the normal input price, and the turn starts faster.

### What breaks the cache

| Change | Effect |
|---|---|
| A new message at the end | normal: only the new part is processed in full |
| Editing a context file mid-session | the prefix changes, so the next turn reprocesses it |
| Connecting or disconnecting an MCP server | the tool list changes, same effect |
| A long pause | the cache expires after a few minutes |

For a single developer the cache mostly saves money and time without any attention. For a team running many agents on metered billing, keeping the stable prefix stable is worth a sentence in the team's guidelines.

## A context checklist

- Keep context files short and specific; move rarely needed detail into files the agent reads on demand.
- Load only the MCP servers a project uses.
- Trim tool output: quiet test runs, capped searches, the tail of logs.
- Read only the part of a file you need.
- One task per session where you can; a fresh session for unrelated work.
- Write state to files before compaction, and put rules that must survive it into context files or permission rules.

## Sources

- Context window sizes and compaction behaviour: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-08.
- Prompt caching: <https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching>, not yet verified against the primary source for this page's date.
- Position effects in long contexts: Liu et al., "Lost in the Middle: How Language Models Use Long Contexts", 2023, <https://arxiv.org/abs/2307.03172>, not yet verified against the primary source for this page's date.
- The token sizes and the session budget are illustrative estimates written for this page.

{% include page_nav.html %}
