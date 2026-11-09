---
title: "Templates, prompt stubs and injected prompts"
nav_order: 17
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Templates, prompt stubs and injected prompts

As of 2026-11-09.

Stop 4 on the talk path, about 1.5 minutes. [Templates and stubs in detail]({{ '/sections/07-templates-and-stubs/detail.html' | relative_url }}) has the brief fields, the original-request block and the injection library.

## Agents write most of the prompts

In a harness, the person writes a few prompts and agents write the rest. A lead agent writes the brief for each sub-agent it starts. A coordinator writes the task for each worker session. A skill writes the prompt for the next skill in a chain. If each of those prompts is written fresh, each one differs, and the rules that matter get left out of some of them.

Templates fix this. A template is the fixed shape of one kind of prompt or document, with the fields to fill in and the rules that must always be present.

### Stubs and templates

- A stub is a fill-in-the-blanks brief: the headings and fixed rules are there; the agent fills in the task.
- A template is the full shape of one kind of document, such as a design, a research report or a bug report, with a description of what goes under each heading.

### Example: a sub-agent brief stub

```markdown
## Objective
<one sentence: what done looks like>

## Inputs
<files, links, data the agent starts from>

## Output
<exact format, where to write it, length cap>

## Tools and scope
<which tools, which directories, read-only or not>

## Boundaries
- Never delete files outside your worktree.
- Never push to main.

## When stuck
<what to try, and when to stop and report>
```

The lead agent fills in the angle-bracket fields. The boundaries are fixed text, copied into every brief.

## A sub-agent sees only its brief

A sub-agent starts with its own empty context. It receives none of its parent's rules unless they are in its brief. Writing "follow the project rules" does not deliver the rules; the sub-agent may never open the file. The template is how the rules reach it, word for word.

### Example: the rule that did not travel

A parent session knows not to kill processes by pattern; it is in the parent's context file. It starts a sub-agent with a short brief that says "run the tests and fix failures". The sub-agent, with no such rule, clears a stuck test run with a pattern kill. Putting the rule in the brief template, and later in a hook, closed the gap.

## Keeping the original request

Briefs drift as work passes through several hands. Each agent paraphrases the one before. The fix is to carry the person's original words, unchanged, at the top of every brief, so each agent can check its work against what was asked. See [deep dive: the original-request block]({{ '/sections/07-templates-and-stubs/detail.html' | relative_url }}#the-original-request-block).

## Injected prompts

The harness can also send text into a session that is already running: a reminder after compaction, a wake-up for an idle session that has a goal, a message from another session. In this site, prompt injection means that: the harness typing into a live session at the right moment. The security attack with the same name is covered in [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}).

### Example: waking an idle session

A session with a `/goal` finishes a step, waits on a background job, and goes idle. When the job ends, the harness types "the job finished; continue the goal" into that session's terminal and checks that the text was submitted. The session wakes and continues without a person.

## Pages for this topic

- [Templates and stubs in detail]({{ '/sections/07-templates-and-stubs/detail.html' | relative_url }}): stubs and document templates, the brief fields, the original-request block, injected prompts and the library behind them, goal templates, and keeping templates small.

{% include page_nav.html %}
