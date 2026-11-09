---
title: "Templates, prompt stubs and injected prompts"
permalink: /sections/07-templates-and-stubs/
nav_order: 17
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Templates, prompt stubs and injected prompts

As of 2026-11-09. Overview page. Slides: [talk stop 4, "Templates and briefs"]({{ '/talk/' | relative_url }}#stop-4-6-minutes), about 1.5 minutes.

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

The words are close, and on this site they mean different things.

| | Prompt stub | Document template |
|---|---|---|
| What it shapes | A prompt one agent sends another | A document people and agents read later |
| Read by | One sub-agent, once, at launch | Many readers, many times |
| Typical kinds | Sub-agent brief, review request, research question | Design, plan, research report, bug report |
| Fixed parts | Headings and safety rules | Headings and what goes under each |
| Failure it prevents | A rule or the output format left out | A reader not finding the answer |

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

## The brief

A brief is the prompt that starts a sub-agent. A good one answers six questions before the sub-agent has to ask any of them.

- Two of its blocks are fixed text, copied from the stub: the original request and the boundaries.
- Four are filled for the task: objective, inputs, output, and tools and scope.
- One more, When stuck, is filled with a fallback and a stop condition.

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

On this site, an injected prompt is text the harness types into a live session at the right moment.

| Moment | What the harness types |
|---|---|
| After compaction | A reminder of the current task and the rules that matter |
| Session idle with a goal still open | A wake-up that tells it to continue |
| Context window nearly full | `/compact` with a focus line naming the task |
| Message from another session | The message text, with its sender |

> Note: the security attack with the same name, where untrusted content smuggles instructions to a model, is a different thing. It is covered in [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}).

### Why type into the terminal

A built-in command sent as a message between sessions arrives as plain text and never runs. Claude Code's documentation says so for cross-session messages and for scheduled `/loop` fires. A harness that needs a session to run `/compact` at a given moment has to type it into that session's terminal, then check that it was submitted.

![A sequence across five participants. A background job finishes and tells the goal waker. The waker calls the injection library with the session name and text; the library clears the input, types and submits into the terminal pane. The pane still shows the text in the input box, so the library sends submit again, and the session starts a new turn to continue its goal. Later the session's context reaches 81 percent; the waker asks the library to save state and then send /compact with a focus line, and the pane reports that compaction finished.]({{ '/assets/images/07-templates-and-stubs/injection-wake.png' | relative_url }}){: style="width:100%"}

*Waking an idle session, then compacting it, through one library. Built with archify.*

#### Worked example: waking an idle session

```text
[goal active] waiting on background job "full test run"
... session idle ...
[harness] job finished, exit 0
[harness] inject -> session pane: "the background test run finished; continue the goal"
[harness] verify: input box empty, new turn started
[session] Reading the test output...
```

The session wakes and continues without a person. Triggers, the four parts of the library, and the failure modes are on the [injected prompts]({{ '/sections/07-templates-and-stubs/injected-prompts.html' | relative_url }}) page.

## Goal templates

`/goal` keeps a session working until a condition holds, and an evaluator decides after each turn whether it does. The evaluator reads only the conversation, so a condition must name evidence the agent will print.

![A lifecycle from left to right: goal armed, agent works, turn ends, evaluator reads, goal met. From the evaluator, not yet loops back to agent works, and a stated blocker leads to a blocker-reported state. From agent works, a started job leads to an idle state that returns on an injected wake-up.]({{ '/assets/images/07-templates-and-stubs/goal-evaluator.png' | relative_url }}){: style="width:100%"}

*A goal from start to stop. Built with archify.*

| Vague goal | Checkable goal |
|---|---|
| "Fix the tests" | "All tests in `tests/dates/` pass, shown in this session's output, and the change is committed" |
| "Improve the docs" | "Every page under `docs/api/` has an Examples section, listed by a grep in this session" |

More on goals and the other document templates: [Document and goal templates]({{ '/sections/07-templates-and-stubs/document-and-goal-templates.html' | relative_url }}).

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
- One shared prompt-injection library, used by the goal waker, the context supervisor and the compaction path.

To start: one template for the task you delegate most.

## Pitfalls

> Warning: a brief that points at a rule does not deliver the rule to the sub-agent. A request restated by each hand loses parts of itself. A built-in command sent as a message arrives as text. A template that gains a block per incident grows too long to read in full.

## Pages in this section

- [Sub-agent briefs]({{ '/sections/07-templates-and-stubs/briefs.html' | relative_url }}): the six fields in depth, a filled brief end to end, and reviewing a brief.
  - [The original-request block]({{ '/sections/07-templates-and-stubs/original-request-block.html' | relative_url }}): drift, the incident, the hash and the launch guard.
- [Injected prompts]({{ '/sections/07-templates-and-stubs/injected-prompts.html' | relative_url }}): triggers, the library, verification and failure modes.
- [Document and goal templates]({{ '/sections/07-templates-and-stubs/document-and-goal-templates.html' | relative_url }}): research, design and bug templates, and checkable goals.

## Sources

- Commands in cross-session messages arrive as text: <https://code.claude.com/docs/en/cross-session-messaging.md>, verified 2026-11-09.
- Built-in commands in scheduled `/loop` fires arrive as text: <https://code.claude.com/docs/en/scheduled-tasks.md>, verified 2026-11-09.
- `/goal` evaluator reads only the conversation: <https://code.claude.com/docs/en/goal.md>, verified 2026-11-09.
- The stubs, templates, original-request block and injection library are from the author's own harness, described conceptually. The examples are illustrations written for this page.

{% include page_nav.html %}
