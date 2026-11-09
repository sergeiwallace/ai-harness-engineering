---
title: "Context file size budgets"
parent: "Harness context files"
grand_parent: "AI harnesses"
nav_order: 2
---

{% include page_nav.html %}

# Context file size budgets

As of 2026-11-09. A sub-topic of [Harness context files]({{ '/sections/02-ai-harnesses/context-files.html' | relative_url }}), in [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}).

Every line of an always-loaded context file is paid for on every turn of every session. A size budget turns "keep it short" into a number a check can enforce.

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What size costs

### Tokens on every turn

The context file sits in the context from the first turn to the last. A file of a few thousand tokens is a few thousand tokens of every request, in every session, on every machine. Prompt caching lowers the price of repeated text, and it does not lower the space it takes.

### Attention

The larger cost is attention. A model given many rules follows each one less reliably than a model given a few. Research on long contexts finds that text in the middle of a long prompt is used less well than text at the start or end. A rule buried in a long file is a rule that will sometimes be missed.

### Crowding

What the context file takes, the task cannot use. A session that starts with a large standing brief reaches its compaction point sooner and loses more detail when it gets there.

| Part of the window at startup | Share (illustrative) |
|---|---|
| System prompt and tool descriptions | about 30% |
| Context files | about 25% |
| Skill and MCP summaries | about 15% |
| Room left for the task | about 30% |

The shares are illustrative. Run `/context` in a fresh session to see your own.

![A sequence diagram of context loading over one session: the user and project files load at start and stay all session, path rules and folder files arrive on demand, and all of them reload after compaction.]({{ '/assets/images/02-ai-harnesses/context-loading.png' | relative_url }})

*Everything in the "loaded at start" part of this diagram counts against the budget on every turn. Our own diagram, made with archify.*

## Measuring

### What to count

Count what the model receives, not what you wrote. That means the file after imports are expanded, plus every layer that loads at startup. Characters divided by four is a usable estimate of tokens for English text; a tokenizer gives the exact figure.

### Where to look

```text
> /context
```

Claude Code's `/context` command shows how the window is split between the system prompt, tools, memory files, skills and messages. Run it at the start of a fresh session to see the standing cost of your harness before any task.

## Setting a budget

### Two thresholds

A budget with two levels works better than one hard limit.

| Level | Meaning | Action |
|---|---|---|
| Warn | the file is approaching its budget | schedule a trim; the build passes |
| Fail | the file is over its budget | the build stops until something is trimmed or moved |

### A budget for each file

Give each always-loaded file its own budget. A total budget lets one file grow by squeezing another, and the squeezed file is often the one with the newer and more relevant rules.

## The trim ladder

When a file is over budget, work down this list in order. Each step keeps the rule available and only moves where it lives.

1. Remove lines that repeat another layer.
2. Remove lines the model follows without being told.
3. Shorten: one sentence per rule, the reason in a clause.
4. Move detail to a procedure document and keep a one-line pointer.
5. Move file-specific rules to [path rules]({{ '/sections/02-ai-harnesses/context-file-layering.html' | relative_url }}#path-rules).
6. Move procedures to [skills]({{ '/sections/02-ai-harnesses/skills.html' | relative_url }}).
7. Move rules that must always hold to [hooks]({{ '/sections/02-ai-harnesses/hooks.html' | relative_url }}), and keep a short line saying the hook exists.
8. Only then remove a rule, and record that you did.

> Before removing any line, check it is not the only place the rule is written down. A trim that deletes a rule's only copy loses the rule.
{: .warning }

## A worked example

### The situation

A global context file has grown past its fail threshold. The build stops.

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

The eleven lines moved to a path rule that loads only on test files. The context file keeps one line saying where they went. Every session that never touches a test saves those lines; every session that does gets them at the right moment.

### The check that the rule survived

My harness pairs every trim with a traceability check: for each removed line, the build looks for the rule's new home and refuses the trim if it finds none. A rule can move down the ladder; it cannot vanish by accident.

## Related pages

- [Compiling context files]({{ '/sections/02-ai-harnesses/compiling-context-files.html' | relative_url }}): where the size check runs.
- [Harness memory and context]({{ '/sections/02-ai-harnesses/memory-and-context.html' | relative_url }}): the window the budget protects.

## Sources

- Claude Code `/context`: <https://code.claude.com/docs/en/slash-commands.md>.
- Liu et al., "Lost in the Middle: How Language Models Use Long Contexts" (2023): <https://arxiv.org/abs/2307.03172>.
- The two-threshold gate, trim ladder and traceability check are from the author's own harness, described conceptually.

{% include page_nav.html %}
