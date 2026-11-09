---
title: "Detail"
parent: "Templates, prompt stubs and injected prompts"
nav_order: 1
---

# Templates and stubs in detail

As of 2026-11-09.

## Purpose

Show how to standardise the prompts agents write for each other, and the text a harness adds to a running session.

## Key points

- In a harness, agents write many of the prompts. A lead agent writes the brief for each sub-agent. A template keeps those briefs consistent.
- A stub is a fill-in-the-blanks brief. A template is the full shape of one kind of document, such as a design or a research report.
- A sub-agent receives none of its parent's rules unless they are in its brief. The template is how the rules reach it.
- Prompt injection, in the sense used here, means the harness sending text into a live session at the right moment: a reminder after compaction, a wake-up for an idle session, a message from another session. The security attack with the same name is covered in [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}).

## Examples from my own harness

- A stub and a template for each kind of document, plus reusable prompt stubs and goal templates.
- A sub-agent brief stub with fixed fields: objective, inputs, output format with a length cap, tools and scope, boundaries, and what to do when stuck. It carries the safety rules word for word, because pointing a sub-agent at a rule does not deliver it.
- A shared prompt-injection library that types a prompt or command into a live terminal session and checks it was submitted. The harness uses it to wake an idle session that has a goal, and to send the compact command when the context is nearly full.

To start: one template for the task you delegate most.

## Pitfalls and lessons

- Briefs grow as each incident adds a block. A long brief costs tokens on every launch, and the important lines get lost among the rest. Prune templates as you prune context files.
- Reflexivity example: briefs now carry the person's original words with a hash, because a paraphrased brief once let an agent record that a requested feature was "not used".

## Sources

From the author's own harness, described conceptually.
