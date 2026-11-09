---
title: "Context file size budgets"
parent: "Harness context files"
grand_parent: "What is a harness?"
nav_order: 2
redirect_from:
  - /sections/02-ai-harnesses/context-file-budgets.html
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Context file size budgets

Every line of an always-loaded context file is paid for on every turn of every session. A size budget turns "keep it short" into a number a check can enforce.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What size costs

### Tokens on every turn

The context file sits in the context from the first turn to the last.

- A file of a few thousand tokens is a few thousand tokens of every request, in every session, on every machine.
- Prompt caching lowers the price of repeated text, but it does not lower the space the text takes in the window.

### Attention

The larger cost is attention.

- A model given many rules follows each one less reliably than a model given a few.
- Research on long contexts finds that text in the middle of a long prompt is used less well than text at the start or end (Liu et al., 2023).
- A rule buried in a long file is a rule that will sometimes be missed, which is why rules that must hold go into hooks and tests.

### Crowding

What the context file takes, the task cannot use. A session that starts with a large standing brief reaches its compaction point sooner and loses more detail when it gets there.

| Part of the window at startup | Share (illustrative) |
|---|---|
| System prompt and tool descriptions | about 30% |
| Context files | about 25% |
| Skill and MCP summaries | about 15% |
| Room left for the task | about 30% |

The shares are illustrative. Run `/context` in a fresh session to see your own.

![A sequence diagram of context loading over one session: the user and project files load at start and stay all session, path rules and folder files arrive on demand, and after compaction the project-root file is re-read while folder files and path rules wait for the next matching file.]({{ '/assets/images/02-ai-harnesses/context-loading.png' | relative_url }})

*Everything in the "loaded at start" part of this diagram counts against the budget on every turn. Our own diagram, made with archify.*

## Measuring

### What to count

Count what the model receives.

1. The file after `@` imports are expanded.
2. Every layer that loads at startup: user, project, and any always-on rules.
3. Characters divided by four gives a usable token estimate for English; a tokenizer gives the exact figure.

```text
$ wc -c CLAUDE.md docs/conventions/testing.md
  9120 CLAUDE.md
  3480 docs/conventions/testing.md      (imported)
 12600 total                            (about 3,150 tokens)
```

### Where to look

```text
> /context
```

Claude Code's `/context` command shows how the window is split between the system prompt, tools, memory files, skills and messages. Run it at the start of a fresh session to see the standing cost of your harness before any task.

## Setting a budget

### Limits the tools already set

Start from the numbers the vendors publish.

| Limit | Value | Source |
|---|---|---|
| Suggested size of one `CLAUDE.md` | under 200 lines | Claude Code memory docs |
| `MEMORY.md` loaded at startup | first 200 lines or 25 KB | Claude Code memory docs |
| A `CLAUDE.md` Claude Code skips entirely | over 4 MiB | Claude Code memory docs |
| `AGENTS.md` text Codex reads | 32 KiB by default (`project_doc_max_bytes`) | Codex `AGENTS.md` docs |
| A skill's description in the listing | cut at 1,536 characters | Claude Code skills docs |

Your own budget is usually tighter than these. The limits say only when text stops loading; a file well under them can still be followed poorly.

### Two thresholds

A budget with two levels works better than one hard limit.

| Level | Meaning | Action |
|---|---|---|
| Warn | the file is approaching its budget | schedule a trim; the build passes |
| Fail | the file is over its budget | the build stops until something is trimmed or moved |

### A budget for each file

Give each always-loaded file its own budget.

- A total budget lets one file grow by squeezing another.
- The squeezed file is often the one with the newer and more relevant rules.
- Per-file budgets make the owner of each file responsible for its size.

### What a file budget misses

A per-file budget counts the files you write. Other parts of the harness also load on every turn.

| Standing cost | How to keep it down |
|---|---|
| tool and MCP tool descriptions | connect only the servers a project needs; Claude Code's MCP tool search defers loading tool definitions until they are needed |
| skill and agent descriptions | keep each one short and put the trigger first; enable only the plugins you use |
| text added by hooks | add context only when its condition holds |
| rules without `paths` | give each rule a `paths` pattern unless it applies everywhere |

Measure the total, files and everything else, with `/context` in a fresh session.

## The trim ladder

When a file is over budget, work down this list in order. Each step keeps the rule available and only moves where it lives.

1. Remove lines that repeat another layer.
2. Remove lines the model follows without being told.
3. Shorten: one sentence per rule, the reason in a clause.
4. Move detail to a procedure document and keep a one-line pointer.
5. Move file-specific rules to [path rules]({{ '/sections/02-what-is-a-harness/context-file-layering.html' | relative_url }}#path-rules).
6. Move procedures to [skills]({{ '/sections/02-what-is-a-harness/skills.html' | relative_url }}).
7. Move rules that must always hold to [hooks]({{ '/sections/02-what-is-a-harness/hooks.html' | relative_url }}), and keep a short line saying the hook exists.
8. Only then remove a rule, and record that you did.

## A worked example

### The situation

A global context file has grown past its fail threshold, and the build stops.

- The most recent additions were a block of test rules.
- The test rules only matter when the agent edits tests.
- A path rule for test files does not exist yet.

### The trim

```diff
- ## Testing
- Tests must test behaviour at the real entry point. Do not test private
- helpers unless the helper has its own contract. Use fixtures for setup.
- Never mock the module under test. Name tests after the behaviour...
- (eleven more lines)
+ ## Testing
+ Test standards load from the test path rule on test files only.
```

The eleven lines moved to a path rule that loads only on test files.

- The context file keeps one line saying where they went.
- Every session that never touches a test saves those lines; every session that does gets them at the right moment.

### The check that the rule survived

My harness pairs every trim with a traceability check.

1. For each removed line, the build looks for the rule's new home: a path rule, a procedure document, a skill or a hook.
2. If it finds none, it refuses the trim.
3. A rule can move down the ladder; it cannot vanish by accident.

## Related pages

- [Compiling context files]({{ '/sections/02-what-is-a-harness/compiling-context-files.html' | relative_url }}): where the size check runs.
- [Harness memory and context]({{ '/sections/02-what-is-a-harness/memory-and-context.html' | relative_url }}): the window the budget protects.

## Sources

- [Claude Code `/context`](https://code.claude.com/docs/en/slash-commands.md).
- Claude Code size guidance and limits: [memory docs](https://code.claude.com/docs/en/memory.md) and [skills docs](https://code.claude.com/docs/en/skills.md), verified 2026-11-09.
- [Codex `project_doc_max_bytes`](https://learn.chatgpt.com/docs/agent-configuration/agents-md), verified 2026-11-09.
- [Liu et al., "Lost in the Middle: How Language Models Use Long Contexts" (2023)](https://arxiv.org/abs/2307.03172).
- The two-threshold gate, trim ladder and traceability check are from the author's own harness, described conceptually.

{% include page_nav.html %}
