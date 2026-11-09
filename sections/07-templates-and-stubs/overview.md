---
title: "Templates, prompt stubs and injected prompts"
permalink: /sections/07-templates-and-stubs/
nav_order: 17
has_children: true
has_toc: false
last_updated: "2026-11-09"
talk_stop: "4"
stop_minutes: "1.5"
---

{% include page_nav.html %}

# Templates, prompt stubs and injected prompts

In a harness, the person writes a few prompts and agents write all the rest. A lead agent writes a brief for each sub-agent, a coordinator writes a task for each worker session, and a skill writes the prompt for the next skill in a chain. Templates give each of those prompts a fixed shape, so the rules that matter reach every agent word for word. The harness can also type text into a session that is already running, which is what this site calls an injected prompt.

![A sequence across five participants. The person sends a request in exact words to the lead agent, which stores them with a hash. The lead sends a brief filled from the stub to the launch guard, which refuses it because the request block was edited. The lead resends with the block copied unchanged and the guard launches the sub-agent with the brief only. The sub-agent reads and searches the repository, reads thirty files and returns a nine-row table under the cap. The lead re-runs the search itself, gets the same nine callers, and returns the result to the person, checked against the request.]({{ '/assets/images/07-templates-and-stubs/brief-in-flight.png' | relative_url }}){: style="width:100%"}

*Figure 1. A brief in flight, from request to checked result. Built with archify.*

| Idea | One line | Go deeper |
|---|---|---|
| Prompt stub | A fill-in-the-blanks brief with fixed rules already written | [Sub-agent briefs]({{ '/sections/07-templates-and-stubs/briefs.html' | relative_url }}) |
| Document template | The full shape of a design, plan or report | [Document and goal templates]({{ '/sections/07-templates-and-stubs/document-and-goal-templates.html' | relative_url }}) |
| Original-request block | The person's words, unchanged, at the top of every brief | [The original-request block]({{ '/sections/07-templates-and-stubs/original-request-block.html' | relative_url }}) |
| Injected prompt | The harness typing into a live session at the right moment | [Injected prompts]({{ '/sections/07-templates-and-stubs/injected-prompts.html' | relative_url }}) |
| Goal template | A stop condition the goal evaluator can check | [Document and goal templates]({{ '/sections/07-templates-and-stubs/document-and-goal-templates.html' | relative_url }}#goal-templates) |

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Why agents need templates

### The fan-out problem

One request from a person becomes many prompts, and agents write nearly all of them. Each prompt is a chance to leave out a rule, restate the goal slightly wrong, or forget to say what the output should look like. When each prompt is written fresh, no two match, and nobody can check a brief against anything.

A template gives every prompt of one kind the same shape. It does four things:

1. Every brief has the same fields, so none is forgotten.
2. Fixed rules are copied verbatim, so they reach every agent.
3. The receiving agent knows where to look for the output format and its limits.
4. A reviewer, human or agent, can check a brief against its template.

### A sub-agent sees only its brief

A sub-agent starts with an empty context. What it has at launch:

- Its brief, written by the lead for this task.
- Its profile's standing instructions and tool list.
- The project's context files, loaded fresh.

Everything else the parent knows stays with the parent: its conversation, the person's corrections, the rules it picked up during the session.

> Note: writing "follow the project rules" in a brief does not deliver the rules. The sub-agent may never open the file.

#### A systems approach to the gap

A rule reaching a sub-agent through its brief is the weakest layer, because a model applies written instructions inconsistently. The fix for a rule that matters goes into the system, in layers:

1. Instructions: the brief's fixed Boundaries block and the context files.
   - Guides the model; the second line of defence.
2. Workflows and tools: the brief stub itself, filled by a skill so the fixed blocks are always copied.
3. Hooks, scripts and linters: a launch guard that refuses a brief missing its fixed blocks, and a hook that refuses the dangerous command outright.
4. Validation and tests: a test that the guard and hook refuse what they should.

Layers 2 to 4 do the enforcing. Adding one more detailed rule to the brief every time an agent makes a mistake is an anti-pattern: the brief grows, and the rule is still only as reliable as the model's attention.

#### Worked example: the rule that did not travel

1. A parent session knew not to kill processes by name pattern; the rule was in its context.
2. It started a sub-agent with a short brief: "run the tests and fix failures".
3. The sub-agent hit a stuck test run and cleared it with a pattern kill.
   - The kill also matched other sessions' processes.
4. The fix went into the system.
   - A hook now refuses a pattern kill at the moment the agent tries it, whoever the agent is (layer 3, see [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }})).
   - A test checks that the hook refuses it (layer 4).
   - The brief stub's fixed Boundaries block names the rule too, as the second layer (layer 1).

## Stubs and templates

The words are close, and on this site they mean three different things.

| | Prompt stub | Document template | Goal template |
|---|---|---|---|
| What it shapes | A prompt one agent sends another | A document people and agents read later | A stop condition for `/goal` |
| Read by | One sub-agent, once, at launch | Many readers, many times | The goal evaluator, after every turn |
| Typical kinds | Sub-agent brief, review request, research question | Design, plan, research report, bug report | "Tests pass", "queue empty", "every caller fixed" |
| Fixed parts | Headings and safety rules | Headings and what goes under each | Evidence lines, constraints, a bound |
| Failure it prevents | A rule or the output format left out | A reader not finding the answer | A goal that never ends, or ends too early |

<!-- IMAGE NEEDED: a fill-in-the-blanks form with some fields pre-printed and others left blank; search: fill in the blank form template, blank form illustration, document template icon -->

### Worked example: a brief stub

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

The lead agent fills in the angle-bracket fields. The Boundaries block is fixed text, the same in every brief. Each field, why it exists and how to fill it well is on the [sub-agent briefs]({{ '/sections/07-templates-and-stubs/briefs.html' | relative_url }}) page.

## Where templates live

A template is only useful if the agent that needs it gets it the same way every time. Both tools store reusable prompts as files with argument placeholders.

| Tool | Location | How it is invoked | Placeholders |
|---|---|---|---|
| Claude Code skill | `.claude/skills/<name>/SKILL.md` (project) or `~/.claude/skills/<name>/SKILL.md` (personal) | `/<name> args`, or by Claude when the description matches | `$ARGUMENTS`, `$0`, `$1`, or named `arguments` in frontmatter |
| Claude Code command (older format, still works) | `.claude/commands/<name>.md` | `/<name> args` | Same as skills |
| Codex skill | Codex skills replace custom prompts | By name, or by Codex when it fits | See the Codex skills documentation |
| Codex custom prompt (deprecated) | `~/.codex/prompts/<name>.md` | `/prompts:<name> KEY=value` | `$1` to `$9`, `$ARGUMENTS`, named `$NAME` |

### Worked example: a brief stub as a skill

```markdown
---
name: research-brief
description: Start a read-only research sub-agent on one question.
argument-hint: [question] [start-path]
disable-model-invocation: true
---
## Objective
Answer: $0

## Inputs
Start from $1. Report what you ruled out.

## Output
The answer in two sentences, then a table of sources. At most 300 words.

## Tools and scope
Read and search only.

## Boundaries (fixed)
- Never kill processes by name pattern.
- Never cite a file that is not tracked in git.

## When stuck
Stop after 20 minutes and report what you have, marked partial.
```

1. A person or a lead runs `/research-brief "Which services call old_client.fetch?" src/`.
2. Claude Code replaces `$0` and `$1` with the two arguments.
3. The filled text becomes the brief, with the fixed blocks exactly as written in the file.
4. `disable-model-invocation: true` keeps Claude from loading the skill on its own; it runs only when called by name.

This is the workflow layer of the systems approach: the fixed blocks reach every brief because the tool copies them, not because an agent remembered to.

## The brief

A brief is the prompt that starts a sub-agent. It has six fields, plus the original-request block above them.

- One block is fixed text placed above the fields: the original request, copied from its record.
- One field is fixed text copied from the stub: Boundaries.
- Five fields are filled for the task: Objective, Inputs, Output, Tools and scope, When stuck.

### The six fields at a glance

| Field | The question it answers | If it is missing |
|---|---|---|
| Objective | When am I done? | The agent stops early or never stops |
| Inputs | Where do I start? | It rediscovers what the parent already knew |
| Output | What do I hand back, and how long? | The parent gets five different formats from five agents |
| Tools and scope | What may I touch? | A reader edits files; a writer edits the wrong repository |
| Boundaries | What must I never do? | A rule the parent knew is broken |
| When stuck | What do I do when it goes wrong? | Loops, or silent guessing |

### Worked example: a filled brief, in short

```markdown
## Objective
Find every caller of `parse_date` and list which pass a timezone.
## Output
A Markdown table: file, line, passes timezone (yes or no). At most 40 rows.
## Tools and scope
Read and search only.
## When stuck
If a caller is generated code, list it once with "generated" and move on.
```

The full version, with inputs and boundaries, is on the [briefs page]({{ '/sections/07-templates-and-stubs/briefs.html' | relative_url }}#a-filled-brief-end-to-end).

## Keeping the original request

### The drift problem

Briefs drift as work passes through several hands. Each agent paraphrases the one before, small changes add up, and a requirement can vanish while every check stays green, because no check compares the result with what the person said.

![A sequence across five participants. The person asks for A, B and C, with C using injection. The lead stores the words and hash in the request record. The lead sends a brief with a tidied summary to the launch guard, which compares the hash and refuses it. The lead copies the block unchanged and the guard launches the worker. The worker checks whether its plan keeps C, finds it would drop a named requirement, and tells the lead, which asks the person whether to keep C with injection or change the ask.]({{ '/assets/images/07-templates-and-stubs/request-guard.png' | relative_url }}){: style="width:100%"}

*The original-request block stops a dropped requirement. Built with archify.*

### The fix

The parts and how they work together on one request:

1. The first agent stores the person's words once, with a hash.
   - The stored record is the source; every brief carries a copy.
2. Each brief opens with a fixed block holding those words, unchanged.
   - A sub-agent that starts helpers copies the block again.
3. A launch guard compares each brief's block with the record before the agent starts.
   - A mismatch refuses the launch and says why.
4. The agent checks its plan and its result against the words.
   - A choice that would drop something the person named goes back to the person.

#### Worked example: the block

```markdown
## Original request (verbatim, do not edit)
> add a table of contents on the landing page and the root README.md
record: asks/site-restructure.md  sha256: 9f2c...e41a
```

The incident that led to this, the guard that checks the block, and how agents use it are on [the original-request block]({{ '/sections/07-templates-and-stubs/original-request-block.html' | relative_url }}) page.

## Injected prompts

### What the term means here

On this site, an injected prompt is text that reaches a session that is already running, at a moment the session cannot choose for itself. Claude Code has native routes for most moments; typing into the terminal is the fallback.

| Moment | Native route | Terminal route needed? |
|---|---|---|
| A background shell or sub-agent the session started finishes | Claude Code wakes the idle session with the result, with or without a goal | No |
| Watching that work before it finishes | Optional: the Monitor tool, or a scheduled check such as `/loop` | No |
| A job outside the session ends, such as a CI run | Channels, where available | Yes, where channels are not |
| Another session has news | Cross-session messaging starts a turn in an idle session | Only for sessions without messaging |
| Compaction just happened | A `SessionStart` hook with the `compact` matcher adds a reminder | No |
| The window is nearly full | None: `/compact` sent as a message arrives as text | Yes |

> Note: the security attack with the same name, where untrusted content smuggles instructions to a model, is a different thing. It is covered in [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}).

### Why type into the terminal

- A built-in command sent as a message between sessions arrives as plain text and never runs.
- A built-in command in a scheduled `/loop` fire is passed to Claude as text too.
- Claude Code's documentation says both, and documents no hook that runs a slash command.
- So a harness that needs a session to run `/compact` at a given moment types it into that session's terminal, then checks that it was submitted.

![A sequence across five participants. A background job that runs outside the session finishes and tells the waker, labelled goal waker in the drawing. The waker calls the injection library with the session name and text; the library clears the input, types and submits into the terminal pane. The pane still shows the text in the input box, so the library sends submit again, and the session starts a new turn to continue its goal. Later the session's context reaches 81 percent; the waker asks the library to save state and then send /compact with a focus line, and the pane reports that compaction finished.]({{ '/assets/images/07-templates-and-stubs/injection-wake.png' | relative_url }}){: style="width:100%"}

*The terminal route: waking a session after a job that ran outside it, then compacting it. Built with archify.*

#### Worked example: waking an idle session

A session has a goal and is waiting on a CI pipeline, which runs outside the session, so Claude Code does not see it end.

```text
[goal active] waiting on CI pipeline "full test run"
... session idle ...
[harness] CI finished, exit 0
[harness] inject -> session pane: "the CI test run finished; read its result and continue the goal"
[harness] verify: input box empty, new turn started
[session] Reading the CI output...
```

The four parts of the library, the native routes in detail and the failure modes are on the [injected prompts]({{ '/sections/07-templates-and-stubs/injected-prompts.html' | relative_url }}) page.

## Goal templates

`/goal` keeps a session working until a condition holds. After each turn, a small fast model reads the conversation and returns a verdict: not yet met, met, or impossible.

```text
/goal all tests in tests/dates pass and the lint step is clean
/goal          # status
/goal clear    # remove
```

- The evaluator reads only the conversation and calls no tools, so a condition must name evidence the agent will print.
- One goal per session; the condition can be up to 4,000 characters.
- A clause such as "or stop after 20 turns" bounds the run.

![A lifecycle from left to right: goal armed, agent works, turn ends, evaluator reads, goal met. From the evaluator, not yet loops back to agent works, and a stated blocker leads to a blocker-reported state. From agent works, a started job leads to an idle state that returns on an injected wake-up.]({{ '/assets/images/07-templates-and-stubs/goal-evaluator.png' | relative_url }}){: style="width:100%"}

*A goal from start to stop. The wake-up arrow is the case of a job the session did not start; a background shell or sub-agent the session started wakes it on its own, with or without a goal. Built with archify.*

| Vague goal | Checkable goal |
|---|---|
| "Fix the tests" | "All tests in `tests/dates/` pass, shown in this session's output, and the change is committed" |
| "Improve the docs" | "Every page under `docs/api/` has an Examples section, listed by a grep in this session" |

More on goals, their commands and edge cases, and the document templates: [Document and goal templates]({{ '/sections/07-templates-and-stubs/document-and-goal-templates.html' | relative_url }}#goal-templates).

## Keeping templates small

Briefs grow, because each incident adds a block. A long brief costs tokens on every launch, and the important lines get lost among the rest. Prune templates the way you prune context files: merge blocks that cover the same failure, and move rarely needed detail into a procedure the brief links to.

<details markdown="1">
<summary>A pruning pass, step by step</summary>

1. List every fixed block in the template with the incident that added it.
2. Merge blocks that guard the same failure into one sentence.
3. Move a block whose failure is now refused by a hook out of the brief; the hook enforces it.
4. Move rarely needed detail into a linked procedure, and keep one line saying when to read it.
5. Re-run a recent delegation with the pruned template and compare the result.

</details>

## Examples from my own harness

- A stub and a template for each kind of document I write often, plus prompt stubs for briefs and goal templates for long runs.
- The brief stub above, with its safety rules copied word for word, and a launch guard that refuses a brief missing them.
- The original-request block, stored with a hash and checked at launch.
- One shared prompt-injection library, used by the waker for external jobs, the context supervisor and the compaction path.

To start: one template for the task you delegate most.

## Pitfalls

| Pitfall | What happens | Fix in the system |
|---|---|---|
| A brief points at a rule | The sub-agent never reads the rule | Enforce the rule with a hook; name it in Boundaries as the second layer |
| Each hand restates the request | A requirement drops out with every check green | The original-request block and its launch guard |
| A command is sent as a message | `/compact` arrives as text and nothing compacts | Native route first; the terminal route with a verify step for commands |
| A goal names no evidence | The evaluator cannot tell it is met, and the session keeps running | A goal template with evidence lines and a bound |
| A template gains a block per incident | The brief grows too long to read in full | Prune; move enforced rules into hooks and tests |

## Pages in this section

- [Sub-agent briefs]({{ '/sections/07-templates-and-stubs/briefs.html' | relative_url }}): the six fields in depth, a filled brief end to end, and reviewing a brief.
  - [The original-request block]({{ '/sections/07-templates-and-stubs/original-request-block.html' | relative_url }}): drift, the incident, the hash and the launch guard.
- [Injected prompts]({{ '/sections/07-templates-and-stubs/injected-prompts.html' | relative_url }}): triggers, the library, verification and failure modes.
- [Document and goal templates]({{ '/sections/07-templates-and-stubs/document-and-goal-templates.html' | relative_url }}): research, design and bug templates, and checkable goals.

## Sources

- Commands in cross-session messages arrive as text: <https://code.claude.com/docs/en/cross-session-messaging.md>, verified 2026-11-09.
- Built-in commands in scheduled `/loop` fires arrive as text: <https://code.claude.com/docs/en/scheduled-tasks.md>, verified 2026-11-09.
- `/goal` commands, verdicts and check-ins: <https://code.claude.com/docs/en/goal.md>, verified 2026-11-09.
- Skills, `.claude/commands/`, `$ARGUMENTS` and `$N` placeholders, `disable-model-invocation`, `argument-hint`: <https://code.claude.com/docs/en/skills.md>, verified 2026-11-09.
- Codex custom prompts, their placeholders and their deprecation in favour of skills: <https://learn.chatgpt.com/docs/custom-prompts>, verified 2026-11-09.
- `SessionStart` hook with the `compact` matcher: <https://code.claude.com/docs/en/hooks.md>, verified 2026-11-09.
- Channels: <https://code.claude.com/docs/en/channels.md>, verified 2026-11-09.
- The stubs, templates, original-request block and injection library are from the author's own harness, described conceptually. The examples are illustrations written for this page.

{% include page_nav.html %}
