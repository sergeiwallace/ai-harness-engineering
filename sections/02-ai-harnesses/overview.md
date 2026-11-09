---
title: "AI harnesses"
permalink: /sections/02-ai-harnesses/
nav_order: 12
has_children: true
has_toc: false
---

{% include page_nav.html %}

# AI harnesses

As of 2026-11-09. Stop 3 on the talk path, about 3 minutes. Slides for this stop are on the [talk slides page]({{ '/talk/' | relative_url }}), under stop 3.

A harness is the software around a model that gives it safe access to tools, memory and checks, set up to match one person's preferences, workflows and projects. Claude Code and Codex are harnesses out of the box. Harness engineering is the work of extending them: context files that say how you work, skills that hold your procedures, hooks and checks that stop the mistakes you have seen, agent profiles that fix who does what, and memory that carries lessons from one session to the next.

The components sort into five functions around the agent loop: context injection (what the model reads), action (what it can do), control (what decides which actions run and who runs them), observe and verify (what checks the result) and persist (what carries into the next session). This overview walks through each function and each component in it, with what it is, why it matters, a worked example and a snippet. Every component links to a topic page that goes deeper, and some topic pages link further down to sub-topic pages. Two sections near the end tie it together: [a bug fix, step by step](#a-bug-fix-step-by-step) shows every component on one task, and [a systems approach to harness engineering](#a-systems-approach-to-harness-engineering) explains where a fix should go when something goes wrong.

![A layered map of a harness, read top to bottom. Person and policy at the top: the person sends a prompt or a slash command into Skills, and permissions and settings send allow, ask and deny rules into the agent loop. Context injection: context files and the memory index load into the loop at session start, and a skill's steps load on use. In the middle, the agent loop, where the model reasons and decides and the context window is its working memory. Control: every tool call passes the hooks and guards. Action: allowed calls reach the native tools or MCP servers and plugins. Delegation: the Agent tool sends a brief to a sub-agent, which takes its role and tools from an agent profile and edits in its own worktree. Observe, verify, persist: the sub-agent's diff and report go to tests and review, a pass becomes a verified change, the task tracker closes the task, and lessons flow back up into memory for the next session.]({{ '/assets/images/02-ai-harnesses/harness-layers.png' | relative_url }})

*The harness as layers around the agent loop. Green arrows are the main path of one task; red are control points; dashed purple are asynchronous or returning flows. Our own diagram, made with archify. Inspired by Addy Osmani's harness anatomy (<https://addyosmani.com/assets/images/harness-anatomy.jpeg>), Phil Schmid's agent harness diagram (<https://www.philschmid.de/static/blog/agent-harness-2026/agent-harness.jpeg>) and a layered Claude Code architecture map published on Substack (<https://substack-post-media.s3.amazonaws.com/public/images/fd726fee-d916-4c94-8947-adb16f45d08d_2412x1488.jpeg>).*

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## The components at a glance

| Component | Question it answers | In Claude Code | In Codex | Topic page |
|---|---|---|---|---|
| Context files | What does the agent know at the start? | `CLAUDE.md`, `.claude/rules/` | `AGENTS.md` | [Harness context files]({{ '/sections/02-ai-harnesses/context-files.html' | relative_url }}) |
| Templates and briefs | What does a new agent get told? | skills, sub-agent briefs, `--append-system-prompt` | skills, prompt files | [Guides, sensors and enforcement]({{ '/sections/02-ai-harnesses/guides-sensors-and-enforcement.html' | relative_url }}) |
| Native commands and tools | What can it do with no setup? | Read, Edit, Bash, Grep, `/compact` | shell, apply patch, `/status` | [Native commands and tools]({{ '/sections/02-ai-harnesses/native-commands-and-tools.html' | relative_url }}) |
| Skills | What procedures does it follow? | `.claude/skills/<name>/SKILL.md` | `.agents/skills/<name>/SKILL.md` | [Harness skills]({{ '/sections/02-ai-harnesses/skills.html' | relative_url }}) |
| Plugins and MCP servers | What outside services can it reach? | `/plugin`, `.mcp.json` | `[mcp_servers.<id>]` in `config.toml` | [Plugins and MCP servers]({{ '/sections/02-ai-harnesses/plugins-and-mcp-servers.html' | relative_url }}) |
| Permissions and settings | What is it allowed to do unasked? | `settings.json` allow, ask, deny | `approval_policy`, `sandbox_mode` | [Permissions and settings]({{ '/sections/02-ai-harnesses/permissions-and-settings.html' | relative_url }}) |
| Hooks | What runs at fixed points, every time? | `hooks` in `settings.json` | `[hooks]` in `config.toml` | [Harness hooks]({{ '/sections/02-ai-harnesses/hooks.html' | relative_url }}) |
| Scripts, linters and tests | What checks the result? | git hooks, CI, test suites | the same | [Guides, sensors and enforcement]({{ '/sections/02-ai-harnesses/guides-sensors-and-enforcement.html' | relative_url }}) |
| Agent profiles | Who does which part of the work? | `.claude/agents/<name>.md` | roles in the `agents` table of `config.toml` | [Agent profiles]({{ '/sections/02-ai-harnesses/agent-profiles.html' | relative_url }}) |
| Memory | What survives the end of a session? | auto memory, `MEMORY.md` | `AGENTS.md`, files you keep in the repo | [Harness memory and context]({{ '/sections/02-ai-harnesses/memory-and-context.html' | relative_url }}) |
| Context window | How much room is left right now? | `/context`, `/compact` | `/status`, `/compact` | [Harness memory and context]({{ '/sections/02-ai-harnesses/memory-and-context.html' | relative_url }}) |
| Task tracking | What work is open, and who owns it? | an issue tracker the agent can call | the same | [Harness memory and context]({{ '/sections/02-ai-harnesses/memory-and-context.html' | relative_url }}#task-tracking) |

> Every row is something you can add to, configure or replace. The table has no row for the model: by the 2026 definition, the harness is everything except the model.
{: .note }

## What a harness is

### The working definition

The definition at the top of this page is the talk's own. Each part of it names something you build:

- "The software around a model": everything that is not the model's weights, from the agent loop to your own scripts.
- "Safe access to tools": the tool list, plus the permission rules, hooks and sandbox that decide which calls run.
- "Memory": context files at the start of a session, and memory files and a task tracker between sessions.
- "Checks": hooks, linters, tests and review that look at what the agent did.
- "Set up to match one person": a harness fits one way of working, so two good harnesses can look very different.

Birgitta Böckeler of Thoughtworks described the wider 2026 usage: "harness has emerged as a shorthand to mean everything in an AI agent except the model itself". This site uses "harness" for the whole stack and "my harness" for the layer I built on top of Claude Code and Codex.

### The computer analogy

Phil Schmid describes the stack as a computer. The model is the CPU: it does the reasoning and nothing else. The context window is the RAM: fast, limited, and wiped at the end of the session. The harness is the operating system: it decides what is loaded into memory, which devices the program may touch, and what happens when something goes wrong. The agent is the application the person sees.

| Computer | Agent stack | What it means for you |
|---|---|---|
| CPU | model | swap it for a better one; most of the rest stays |
| RAM | context window | everything you load competes for it |
| Operating system | harness | where your rules, tools and checks live |
| Disk | files, git, memory, tracker | the only thing that survives a restart |
| Application | the agent you talk to | what the person experiences |

Addy Osmani's version of the same picture puts it in one line: the model is one chip on the board, and the harness is everything else that makes it useful.

<!-- IMAGE NEEDED: the model as a CPU, the context window as RAM and the harness as the operating system around them; search: agent harness operating system analogy, LLM CPU context window RAM, agent harness diagram -->

### Guides and sensors

Böckeler sorts the parts by when they act.

| | Guides | Sensors |
|---|---|---|
| Act | before the agent acts | after the agent acts |
| Examples | context files, templates, skills, agent profiles | tests, linters, type checks, review, hooks that inspect output |
| Fail by | being ignored or crowded out | being missing, slow or bypassed |

<!-- IMAGE NEEDED: guides acting before the agent acts and sensors acting after, around one agent loop; search: harness engineering guides sensors, feedforward feedback agent, Bockeler harness engineering diagram -->

A harness strong in both steers the agent toward the right action and catches the cases where steering failed. The [guides and sensors]({{ '/sections/02-ai-harnesses/guides-sensors-and-enforcement.html' | relative_url }}#guides-and-sensors) section of the enforcement topic page has a worked example.

### Instructions and enforcement

Some parts are text the model reads and interprets. Others are code that runs the same way every time.

| | Instructions | Enforcement in code |
|---|---|---|
| Examples | context files, skills, prompts | workflows, hooks, scripts, linters, tests |
| Cost to add | minutes | hours |
| Followed | most of the time | every time it runs |
| Fails by | being dropped when the model juggles tasks or the window is full | being wrong, slow or not installed |

Claude Code's docs describe `CLAUDE.md` as "context, not enforced configuration", and point to a `PreToolUse` hook "to block an action regardless of what Claude decides".

![The Swiss cheese model: four slices of cheese in a row, each with holes. A red arrow labelled hazards passes through holes that line up in every slice and comes out as losses. Each slice stands for one layer of defence; a mistake gets through only where the holes in every layer line up.]({{ '/assets/images/02-ai-harnesses/swiss-cheese-model.png' | relative_url }})

*Placeholder image: the Swiss cheese model of layered defence (James Reason). Each layer catches some mistakes; what gets through one layer is caught by the next. Source: <https://commons.wikimedia.org/wiki/File:Swiss_cheese_model_of_accident_causation.png>, Davidmack, CC BY-SA 3.0 (<https://creativecommons.org/licenses/by-sa/3.0/>). Resized for this page.*

The ladder, from the layer that guides to the layers that enforce:

| Layer | What it is | Enforced by | Example from my harness | What it misses, for the next layer to catch |
|---|---|---|---|---|
| 1. Instructions | context files, path rules, skill text, prompts | the model, reading it | "never kill processes by name pattern" in the context file | a model juggling tasks or near a full window drops the line |
| 2. Workflows and tools | skills with fixed steps, scripts, templates, tools that accept only valid input | the program the agent runs | a commit script that builds the message and refuses one without the delegation line | actions the agent takes outside the workflow, such as a raw shell command |
| 3. Hooks, scripts and linters | checks at the moment of action | the harness, every time | a `PreToolUse` guard that refuses `pkill` and names the safe form | spellings the guard does not parse, and mistakes only visible in the finished result |
| 4. Validation and tests | tests, CI, link checkers, review before a change lands | programs and reviewers on the result | a link checker that walks the built site before publish | design and judgment problems no check encodes; those go to a person |

My harness puts each rule as high on this ladder as it can be checked, and keeps the context-file line as a second layer. Rules that cannot be checked from an action, such as design judgment, stay as guidance with a review step behind them. The [systems approach](#a-systems-approach-to-harness-engineering) section below explains the layers, and [Guides, sensors and enforcement]({{ '/sections/02-ai-harnesses/guides-sensors-and-enforcement.html' | relative_url }}#one-rule-moved-into-the-system) follows one rule into the system.


## Context injection

What reaches the model before it acts. These are guides: text the model reads and usually follows.

| Component | Reaches the model | Topic page |
|---|---|---|
| Context files | at session start, path rules on demand | [Harness context files]({{ '/sections/02-ai-harnesses/context-files.html' | relative_url }}) |
| Templates and briefs | when a prompt or sub-agent brief is built | [Guides, sensors and enforcement]({{ '/sections/02-ai-harnesses/guides-sensors-and-enforcement.html' | relative_url }}#templates-as-guides) |
| Skills | description every turn, body on use | [Harness skills]({{ '/sections/02-ai-harnesses/skills.html' | relative_url }}) |

### Context files

#### What they are

A context file is a Markdown file the harness loads into the model's context when a session starts. Claude Code reads `CLAUDE.md`; Codex and many other tools read `AGENTS.md`. Claude Code also reads `AGENTS.md` when a project has no `CLAUDE.md`. The file holds what a new teammate would need on day one:

- how to build and test;
- the conventions that are not obvious from the code;
- a short note of the rules that hooks and tests enforce, so the agent knows why a refusal comes.

Claude Code's docs suggest keeping each `CLAUDE.md` under 200 lines, because longer files cost more context and are followed less closely.

#### Why they matter

The model starts every session with no memory of the last one. Whatever is in the context file is the only standing knowledge it has about you and your project, and it pays for that file in tokens on every turn. Short, specific lines that name a command or a path are followed more consistently than long or general ones.

#### How they layer

![A sequence diagram of context loading over one session. At start the harness loads the user file and the project file into the model's window for the whole session. When the model edits a test file, a matching path rule arrives; when it reads a file under web/, that folder's CLAUDE.md arrives. At compaction the chat is summarised and the files are reloaded.]({{ '/assets/images/02-ai-harnesses/context-loading.png' | relative_url }})

*Which context files reach the model, and when. After compaction the project-root file is re-read from disk; folder files and path rules come back the next time a matching file is touched. Our own diagram, made with archify.*

Files at several levels combine. A user file in `~/.claude/CLAUDE.md` applies to every project you open. A project file is committed and shared with the team. Path rules in `.claude/rules/` load only when the agent touches files that match their pattern, so a rule about test style costs nothing while the agent edits docs. [Context file layering]({{ '/sections/02-ai-harnesses/context-file-layering.html' | relative_url }}) covers precedence and imports.

#### Snippet

```markdown
# Project context

## Build and test
- `make test` runs the full suite; `make test-fast` skips the slow integration tests.
- Run the full suite before every commit.

## Conventions
- Database migrations must be reversible: every `up` has a `down`.
- Never edit files under `generated/`; change the schema and run `make codegen`.

## Mistakes to avoid
- The staging config is not the production config. Never copy values between them.
```

#### Worked example

An agent keeps editing a generated client file, and your changes vanish at the next code generation. The fix goes into the system: a `PreToolUse` hook refuses edits under `generated/` and names `make codegen`, and a test fails when the generated files differ from a fresh run of the generator. One line under "Conventions" stays as a second layer, so the agent expects the refusal and goes to the schema first (see [Hooks](#hooks) below and the [systems approach](#a-systems-approach-to-harness-engineering)).

<details markdown="1">
<summary>How my harness builds its context files</summary>

My global context file is compiled. Each rule lives in a small source module, a build step joins the modules into the files each tool reads, and a size check fails the build when the always-loaded result grows past its budget. A rule that needs more than a line or two moves into a procedure document, and the context file keeps a one-line pointer to it. [Compiling context files]({{ '/sections/02-ai-harnesses/compiling-context-files.html' | relative_url }}) and [Context file size budgets]({{ '/sections/02-ai-harnesses/context-file-budgets.html' | relative_url }}) describe both.

</details>

Topic page: [Harness context files]({{ '/sections/02-ai-harnesses/context-files.html' | relative_url }}). Full section: [Context files]({{ '/sections/04-context-files/' | relative_url }}).

### Templates and briefs

#### What they are

A template is a reusable starting text: a prompt you paste, a brief you hand a sub-agent, a document skeleton the agent fills in. Context files say what is always true; templates say what to do for one kind of task.

#### Why they matter

When one agent hands work to another, the brief is the only context the second agent has. A brief written fresh each time drops something different each time. A template fixes the parts that must never be dropped: the original ask, the scope, the limits, and what to report back.

#### Snippet

```text
## FOUNDING ASK (verbatim, do not paraphrase)
<the user's exact words>

## YOUR PART
You own only <paths>. Never touch files outside them.

## LIMITS (binding)
- No destructive operations. Confirm every deletion against its target.
- Never print secrets, including through a redirect or shell tracing.

## REPORT
Head commit, files changed, checks run and their results, anything unfinished.
```

#### Worked example

A coordinator splits one large documentation job across ten parallel agents. Each brief starts from the same template, so each agent receives the user's exact words, its own folder, the same safety limits and the same report shape. When the user adds a requirement halfway through, the coordinator relays it verbatim to all ten, and every brief already has a place for it. More on templates in [Templates, prompt stubs and injected prompts]({{ '/sections/07-templates-and-stubs/' | relative_url }}).

### Skills

#### What they are

A skill is a folder with a `SKILL.md` file: front matter with a name and a one-line description, then the instructions. The harness lists every skill's name and description in the context. When a task matches, or when you type `/name`, the full body loads. Files the body points to (templates, reference notes, scripts) load only when a step needs them.

![A skill loads in three stages: the name and description always, the SKILL.md body on use, and reference files and scripts per step.]({{ '/assets/images/02-ai-harnesses/skill-loading.svg' | relative_url }})

#### Why they matter

Skills are how a procedure you follow by hand becomes one the agent follows the same way every time, without paying for the full text on every turn. They also give the procedure one place to live, so a fix to it reaches every session.

#### Snippet

```markdown
---
name: bug-fix
description: Diagnose and fix a reproducible defect. Reproduce, find the root cause, freeze a failing regression test, fix, verify.
---

1. Reproduce the bug with the smallest command that shows it. Save the command.
2. Find the root cause. Name the line and why it is wrong.
3. Write a test that fails for that reason. Commit it on its own.
4. Fix the code until the test passes. Run the full suite.
5. Report the cause, the test and the fix.
```

#### Worked example

You keep typing the same five-paragraph prompt whenever a bug comes in, and the agent keeps skipping the regression test. As a skill, the steps are numbered and the test comes before the fix. My harness has skills for landing a change end to end, for fixing bugs, for delegating to another agent and for reviewing code, each grown from a prompt I kept retyping. [Harness skills]({{ '/sections/02-ai-harnesses/skills.html' | relative_url }}) covers authoring, triggering and testing them; [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}) is the full section.

## Action

What the agent can do in the world. Every action goes through a tool, so the tool list is also the list of things a control can watch.

| Component | Gives the agent | Topic page |
|---|---|---|
| Native commands and tools | files, search, shell, web, sub-agents | [Native commands and tools]({{ '/sections/02-ai-harnesses/native-commands-and-tools.html' | relative_url }}) |
| Plugins and MCP servers | outside services and shared packages | [Plugins and MCP servers]({{ '/sections/02-ai-harnesses/plugins-and-mcp-servers.html' | relative_url }}) |

### Native commands and tools

#### What they are

Native tools are what the harness gives the model before you add anything. In Claude Code these include `Read`, `Write` and `Edit` for files, `Bash` for the shell (searches with `find` and `grep` run there on macOS and Linux), `WebFetch` and `WebSearch` for the web, `Agent` for starting a sub-agent, `Skill` for running a skill, and task tools for a session task list. Native commands are the built-in slash commands a person types: `/init`, `/compact`, `/context`, `/memory`, `/agents`, `/hooks`, `/permissions`, `/mcp`, `/model` and others.

#### Why they matter

Most harness extensions are built from native tools. A skill is instructions that call them; a hook watches them; a permission rule names them. Knowing what each one does, and what it costs, tells you what you need to build and what is already there.

#### Snippet

Two native commands, with the kind of output they give (shortened and illustrative):

```text
> /context
  System prompt, system tools, MCP tools, memory files, skills, messages, free space
> /compact keep the open decisions and the test commands
  (the conversation is replaced by a summary that keeps what you named)
```

<!-- IMAGE NEEDED: a screenshot of Claude Code's /context output showing how the window is split; search: claude code /context command screenshot, claude code context usage -->

#### Worked example

Before writing a skill that "summarises the session so far", check `/compact`: it already summarises, and it takes an instruction for what to keep. Before writing a search helper, check that a plain `grep` or `find` through `Bash`, or the `LSP` tool for definitions and references, does not already cover it. The [topic page]({{ '/sections/02-ai-harnesses/native-commands-and-tools.html' | relative_url }}) has a full table of tools and commands, with Codex equivalents. The basics of both tools are in [Claude Code and Codex basics]({{ '/sections/03-claude-code-and-codex/' | relative_url }}).

### Plugins and MCP servers

#### What they are

The Model Context Protocol (MCP) is an open standard for connecting a model to an outside service. An MCP server exposes tools (search the web, query a database, send a message) that the harness lists next to its native tools. A plugin is a package that bundles skills, commands, agent profiles, hooks and MCP server definitions so they install together.

#### Why they matter

They are how a harness reaches past the local machine, and how you share a set of extensions with someone else. They also cost context: each MCP tool's description sits in the context like a native tool's.

#### Snippet

Adding a remote server at project scope writes it to `.mcp.json`, which the team commits:

```text
$ claude mcp add --transport http --scope project docs-search https://mcp.example.com/docs
```

```json
{
  "mcpServers": {
    "docs-search": {
      "type": "http",
      "url": "https://mcp.example.com/docs"
    }
  }
}
```

<!-- IMAGE NEEDED: an MCP client in the harness connected to several MCP servers, each wrapping an outside service; search: model context protocol architecture diagram, MCP client server host diagram -->

#### Worked example

Several sessions on one machine need to send each other messages. An MCP server that wraps a small mailbox gives every session the same send and fetch tools, with no new code in any session. Packaging those tools with the skill that explains when to use them, as a plugin, means a new machine gets both in one install. See [Plugins and MCP servers]({{ '/sections/02-ai-harnesses/plugins-and-mcp-servers.html' | relative_url }}).

## Control

What decides which actions run, who does them, and how much the session can hold. These are code and configuration, so they apply every time.

| Component | Decides | Topic page |
|---|---|---|
| Permissions and settings | which calls run, ask or are refused | [Permissions and settings]({{ '/sections/02-ai-harnesses/permissions-and-settings.html' | relative_url }}) |
| Hooks | what happens at fixed points in the session | [Harness hooks]({{ '/sections/02-ai-harnesses/hooks.html' | relative_url }}) |
| Agent profiles | which role, model and tools a sub-agent gets | [Agent profiles]({{ '/sections/02-ai-harnesses/agent-profiles.html' | relative_url }}) |
| Context window | how much the session holds, and when it compacts | [Harness memory and context]({{ '/sections/02-ai-harnesses/memory-and-context.html' | relative_url }}) |

### Permissions and settings

#### What they are

Settings files configure the harness itself: which model, which hooks, which environment, and which actions the agent may take without asking. Claude Code reads `settings.json` at user, project and local scope, plus a managed policy file an organisation can set. Permission rules come in three lists: `allow` (run without asking), `ask` (prompt the person) and `deny` (refuse).

#### Why they matter

Permissions are the first hard control. A denied action is refused by the harness whatever the model decides. They also set how often you are interrupted, which decides whether you can leave an agent running.

#### Snippet

```json
{
  "permissions": {
    "allow": ["Bash(npm run test *)", "Bash(git diff *)", "Bash(git status)"],
    "ask": ["Bash(git push *)"],
    "deny": ["Read(./.env)", "Read(./secrets/**)"]
  }
}
```

#### Worked example

You approve `npm run test` forty times in one afternoon. An `allow` rule for it removes the prompt. You never want the agent to read the `.env` file, even by accident; a `deny` rule makes that unreachable. Pushing stays on `ask`, because you want to see what leaves the machine. Rules are checked deny first, then ask, then allow, and the first match decides.

<!-- IMAGE NEEDED: a screenshot of a Claude Code permission prompt for a Bash command with its Yes / Yes, and don't ask again / No options; search: claude code permission prompt screenshot, claude code bash approval dialog --> [Permissions and settings]({{ '/sections/02-ai-harnesses/permissions-and-settings.html' | relative_url }}) covers scopes, precedence, modes and sandboxing.

### Hooks

#### What they are

A hook is a command the harness runs at a fixed point in a session: before a tool call, after one, when a prompt is submitted, when the agent tries to stop, before the context is compacted. The hook receives the event as JSON on standard input. It can allow the action, block it with a message the model reads, or add context to the conversation.

![The Claude Code hook lifecycle, simplified, top to bottom. SessionStart once per session injects context. Inside the turn, once per user prompt, UserPromptSubmit validates or blocks the prompt. Inside it, the agentic loop runs once per tool call: PreToolUse can allow, deny, ask the user or rewrite the tool input; the tool executes; PostToolUse reacts to the result, gives feedback or rewrites the output; then the loop repeats. Stop can block the stop until tests pass. SessionEnd once per session does cleanup and logging.]({{ '/assets/images/02-ai-harnesses/hook-lifecycle-ran-the-builder.png' | relative_url }})

*Placeholder image: the Claude Code hook lifecycle. Source: Ran the Builder, <https://ranthebuilder.cloud/blog/agentic-coding-hooks-deterministic-ai-guardrails/>, used with credit.*

#### Why they matter

A hook is code, so it runs every time. It is the step on the ladder where a rule stops depending on the model remembering it. Each one also explains itself at the moment of the mistake, which teaches the model better than a line in a long file.

#### Snippet

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          { "type": "command", "command": "python3 \"$CLAUDE_PROJECT_DIR\"/tools/guards/no_pattern_kill.py" }
        ]
      }
    ]
  }
}
```

#### Worked example

An agent on a shared machine stopped other people's test runs with a pattern kill, though its instructions said not to. A `PreToolUse` hook on Bash now reads the command, refuses any kill by name pattern, and tells the agent how to stop one process by its id. A session that works in its own git worktree gets a similar refusal when it points git at someone else's checkout:

```text
This agent is isolated in the worktree <its worktree>, but this command
redirects git to the shared checkout via -C. Refusing to run it. Run the
equivalent from <its worktree> without the redirect.
```

[Harness hooks]({{ '/sections/02-ai-harnesses/hooks.html' | relative_url }}) is the topic page, with sub-topics on [lifecycle events]({{ '/sections/02-ai-harnesses/hook-lifecycle-events.html' | relative_url }}), [guard hooks]({{ '/sections/02-ai-harnesses/guard-hooks.html' | relative_url }}) and [failure modes]({{ '/sections/02-ai-harnesses/hook-failure-modes.html' | relative_url }}). The full section is [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}).

### Agent profiles

#### What they are

An agent profile fixes a role: its name, a description of when to use it, its model, and the tools it may use. Claude Code reads profiles from `.claude/agents/<name>.md`; Codex declares custom roles in the `agents` table of `config.toml`. The main session starts a sub-agent from a profile, gives it a brief, and receives its summary. A Claude Code sub-agent starts with the project's `CLAUDE.md` files and a git status snapshot, and without the main conversation's history.

![A sequence diagram of one change delegated to sub-agents. The main session briefs a researcher, which returns three rules with sources. It briefs an engineer with the plan, scope and report shape; the engineer writes tests then code in its own worktree, where a guard keeps its edits inside, and reports the head commit, checks and gaps. The main session sends the diff and the original ask to a read-only reviewer from another model family, which returns two findings. The main session fixes one, answers the other, and the change is ready to merge.]({{ '/assets/images/02-ai-harnesses/delegation.png' | relative_url }})

*One change, three sub-agents, each with its own profile and context. Our own diagram, made with archify.*

#### Why they matter

A sub-agent works in its own context, so the main session keeps its room for decisions. A profile makes the role repeatable: a reviewer profile without edit tools cannot "fix" what it was asked to review.

#### Snippet

```markdown
---
name: reviewer
description: Code review specialist. Use for PR reviews and quality audits. Read-only.
tools: Read, Grep, Glob, Bash
model: opus
---

Review the change you are given against the spec you are given. Report each
finding with file, line, severity and a suggested fix. Never edit files.
```

#### Worked example

The main session writes a feature, then starts a reviewer from a different model family with the diff and the original requirements. The reviewer reports two findings; the main session fixes one and answers the other. [Agent profiles]({{ '/sections/02-ai-harnesses/agent-profiles.html' | relative_url }}) has the profile fields and the roles I use; [Agent profiles, sub-agents and delegation]({{ '/sections/08-delegation/' | relative_url }}) is the full section.

### Context window

#### What it is

The context window is the model's working memory for one session: system prompt, tool descriptions, context files, skill summaries, every message and every tool result. When it fills, the harness compacts it into a summary, and detail is lost.

#### Why it matters

Agents follow instructions less consistently as the window fills. Every component above spends some of it. Managing the window means measuring it, writing state down before it fills, and compacting at a time you choose.

#### Worked example

1. My harness has a small command that reports how full the window is, and a rule that any claim about context must quote its reading.
2. When the reading is high, the session writes a resume message before it compacts:

```text
Resume: export-format change, step 4 of 6 (tests pass, docs not written).
Plan: docs/plans/export-format.md. Open decision: keep the old flag or not.
Git: branch feature/export, 3 commits ahead, clean tree.
Next: write the docs page, run the full suite, open the PR.
```

3. The summarised session reads the message and picks up at step 4.

<!-- IMAGE NEEDED: a context window filling up over a session, with system prompt, tools, context files and messages as layers, and the compaction point; search: LLM context window filling diagram, context window compaction diagram, context engineering window -->

 The full treatment is in [Harness memory and context]({{ '/sections/02-ai-harnesses/memory-and-context.html' | relative_url }}#the-context-window) and [Context window and memory]({{ '/sections/11-context-window/' | relative_url }}).

## Observe and verify

What checks the result after the agent acts. These are sensors, and in the [systems approach](#a-systems-approach-to-harness-engineering) they are the last layer of enforcement.

| Check | Runs | Catches |
|---|---|---|
| Linter or type checker | after each edit, from a hook | style and type errors while the edit is fresh |
| Test suite | before commit and in CI | behaviour that broke |
| Purpose-built script | as a skill step or commit command | a missing field, a wrong format |
| Review | before merge | design and spec problems a test cannot see |

### Scripts, linters and tests

#### What they are

Everything that checks the agent's output after it acts: test suites, linters, type checkers, git pre-commit and pre-push hooks, CI pipelines, and small purpose-built scripts that a skill or hook calls.

#### Why they matter

These are the sensors. They catch what the guides failed to prevent, and they work on the result, so they apply whether a person or an agent wrote the change. A script also turns a fragile multi-step instruction into one command the agent cannot get half right.

#### Worked example

My harness requires a commit message to record whether a change was delegated and to whom. As an instruction, agents forgot it. As a commit script that builds the message and refuses an incomplete one, it is never missing.

```text
$ commit-with-provenance --title "docs: add hooks page" --body-file body.txt
refused: the body must say whether the change was delegated, and to whom
```

The same pattern turned "check every link before you publish" into a link checker that walks the built site. See [Guides, sensors and enforcement]({{ '/sections/02-ai-harnesses/guides-sensors-and-enforcement.html' | relative_url }}#scripts-linters-and-tests) and [Testing]({{ '/sections/12-testing/' | relative_url }}).

## Persist

What survives the end of the session, so the next one starts ahead.

| Store | Holds | Read |
|---|---|---|
| Memory files | lessons, each with what happened and the check to run | index at session start |
| Task tracker | open work, owners, notes, history | on demand with a ready-work query |
| Git | the code and every decision recorded in commits | always |

### Memory

#### What it is

Memory is whatever carries from one session to the next: memory files the agent writes and reads, plan documents, the task tracker, and the git history itself. Claude Code keeps an auto memory folder per repository at `~/.claude/projects/<project>/memory/`, shared across its worktrees. The first 200 lines or 25 KB of its `MEMORY.md` index load at the start of each session; topic files load when the agent reads them.

#### Why it matters

Without memory, every session repeats the last session's mistakes. With too much, the index crowds the context and the important lines fall past the part that loads.

#### Snippet

```markdown
# Memory index

One line each: past the read limit drops silently, tail first.

- [Check a merge in a scratch tree first](merge-in-scratch-first.md): a conflict resolved by hand once split a test from its asserts
- [A green push can hide an unrun check](green-push-unrun-check.md): read the verified line beside the exit code
- [Never type into a peer's terminal](never-type-into-peer.md): message it instead
```

<!-- IMAGE NEEDED: one session writing a lesson to memory and the next session reading it at startup; search: agent long-term memory diagram, LLM agent memory across sessions -->

#### Worked example

A session learns that a release tool reports success even when one of its checks was skipped. It writes a lesson file and one index line. The next session reads the line at startup, and when it sees the tool's success message it reads the verified line too. See [Harness memory and context]({{ '/sections/02-ai-harnesses/memory-and-context.html' | relative_url }}).

### Task tracking

#### What it is

A task tracker the agent can read and write: issues with status, priority, dependencies and an owner. I use Beads, an open-source tracker built for coding agents that stores issues with the repository.

```text
$ bd ready
1. [P1] Export format: write docs page
2. [P2] Remove the old export flag (blocked by 1)
$ bd update <id> --claim
```

#### Why it matters

- It answers "what should I work on" without the person retyping it.
- It survives compaction, restarts and handoffs between sessions.
- Dependencies decide what is ready, so two sessions do not start the same blocked task.

<!-- IMAGE NEEDED: a task board with open, in-progress and closed issues and dependency links between them; search: beads issue tracker agents, beads-ui board, kanban dependency graph -->

#### Worked example

1. A coordinator launches three sub-agents for one task.
2. Each launch files its own child issue under the task, with the agent's name in the body.
3. Each issue is closed when that agent's work is collected.
4. One agent crashes. Its issue stays open, so its unfinished work is visible and can be picked up.

See [Harness memory and context]({{ '/sections/02-ai-harnesses/memory-and-context.html' | relative_url }}#task-tracking).

## How the components work together

### A bug fix, step by step

The diagram and the numbered steps below follow the same task: a person asks the agent to fix a login bug that appears when the email field is empty. Each step names the harness part at work, what it does, and what it prevents.

![A sequence diagram of one bug-fix task, top to bottom, across seven lanes: person, harness, model, hooks, tools, sub-agent and tracker. At session start the harness loads CLAUDE.md, rules and MEMORY.md into the model's window, and a SessionStart hook adds git state and open tasks. The person types /bug-fix, the harness adds the skill body, and the model claims the issue in the tracker. The model asks for a pattern kill; the PreToolUse hook denies it and names the safe way. The model edits a test and runs pytest under an allow rule; one test fails, then none. A PostToolUse hook returns a lint report. A reviewer sub-agent with read-only tools gets the diff and the original ask and returns two findings. The model asks before pushing, the person approves, the model closes the issue and adds a lesson, and a Stop hook checks before the turn ends.]({{ '/assets/images/02-ai-harnesses/harness-one-task.png' | relative_url }})

*One bug-fix task in time order. Solid arrows are calls, dashed arrows are returns and events, red marks a control point. Our own diagram, made with archify.*

1. Session start, with [context files]({{ '/sections/02-ai-harnesses/context-files.html' | relative_url }}) and [memory]({{ '/sections/02-ai-harnesses/memory-and-context.html' | relative_url }}) at work.
   - Does: the harness loads `CLAUDE.md`, any matching rules and the `MEMORY.md` index into the window. A `SessionStart` hook adds the git branch and the open tasks.
   - Prevents: the agent guessing the test command, or repeating a mistake an earlier session already recorded.
2. The prompt, with a [skill]({{ '/sections/02-ai-harnesses/skills.html' | relative_url }}) at work.
   - Does: the person types `/bug-fix login fails on empty email`. The harness adds the skill's full body: reproduce, find the cause, write a failing test, fix, verify.
   - Prevents: skipped steps, most often the regression test.
3. Claiming the work, with the [task tracker]({{ '/sections/02-ai-harnesses/memory-and-context.html' | relative_url }}#task-tracking) at work.
   - Does: the agent marks the issue as in progress and its own.
   - Prevents: two sessions fixing the same bug, and work lost if the session dies.
4. A refused command, with a [guard hook]({{ '/sections/02-ai-harnesses/guard-hooks.html' | relative_url }}) at work.
   - Does: the agent tries to stop a stuck test run with `pkill -f pytest`. The `PreToolUse` hook refuses and names the safe form: find the process id and kill that one process.
   - Prevents: stopping other people's test runs on a shared machine.
5. Edit and test, with [native tools]({{ '/sections/02-ai-harnesses/native-commands-and-tools.html' | relative_url }}) and a [permission rule]({{ '/sections/02-ai-harnesses/permissions-and-settings.html' | relative_url }}) at work.
   - Does: the agent edits the test and the code with Edit and runs the suite with Bash. An `allow` rule lets the test command run without a prompt. One test fails, then none.
   - Prevents: interrupting the person for every test run.
6. Lint on the fresh edit, with a `PostToolUse` [hook]({{ '/sections/02-ai-harnesses/hook-lifecycle-events.html' | relative_url }}#posttooluse) at work.
   - Does: after each edit, the hook lints the changed file and returns any errors to the model.
   - Prevents: a commit that fails the lint gate later, after the agent has moved on.
7. A second opinion (optional), with an [agent profile]({{ '/sections/02-ai-harnesses/agent-profiles.html' | relative_url }}) at work.
   - Does: the agent starts a reviewer sub-agent with read-only tools, ideally from another model family, and gives it the diff and the original request. It returns two findings.
   - Prevents: the author's blind spots reaching the merge.
   - Whether to delegate depends on the task. A sub-agent helps with a long search that would crowd the main window, or with a review run in parallel; a small fix like this one can be reviewed in the main session.
8. Asking before the push, with a permission `ask` rule and a [human gate]({{ '/sections/14-human-gates/' | relative_url }}) at work.
   - Does: the push waits until the person approves it.
   - Prevents: anything leaving the machine unseen.
9. Recording the outcome, with the tracker and memory at work.
   - Does: the agent closes the issue and writes one memory line about the cause.
   - Prevents: the next session rediscovering the same cause.
10. Ending the turn, with a `Stop` [hook]({{ '/sections/02-ai-harnesses/hook-lifecycle-events.html' | relative_url }}#stop) at work.
    - Does: before the agent finishes, the hook runs the fast checks and lets the turn end only if they pass.
    - Prevents: a reply that says "fixed" while a check is still red.

| Function | Steps where it acts |
|---|---|
| Context injection | 1, 2 |
| Action | 5 |
| Control | 4, 5, 7, 8, 10 |
| Observe and verify | 5, 6, 7, 10 |
| Persist | 3, 9 |

### A systems approach to harness engineering

Harness engineering treats the model, its instructions, its tools, its checks and the person directing it as one system. When the output is wrong or a gap shows up, the fix goes into the system. The same approach is applied again to each new problem, including by the agents themselves, so the harness improves with use and the output gets better in a way that holds.

The fix should be enforced by the system wherever possible. Context files and instructions guide the model while it works, but the model applies them unevenly. It can miss or drop a rule when it handles several things at once, switches tasks, or works with a nearly full context window. For that reason, adding another detailed rule to a context file each time the model does something wrong is an anti-pattern.

A harness enforces its rules in layers:

1. Instructions in context files, skills and prompts describe the expected behaviour.
   - They shape the model's general behaviour.
   - They are never enough on their own over many sessions.
2. Workflows and tools build the steps into what the agent runs.
   - A skill runs the steps in a fixed order.
   - A tool accepts only valid input.
   - A template gives the agent the structure to fill in.
3. Hooks, scripts and linters refuse or flag an action that breaks a rule at the moment the model tries it.
4. Validation and tests run before a change lands and catch what got past the earlier layers.

Layers 2 to 4 do the enforcing. Layer 1 stays as a second line of defence, so the model's own behaviour matches what the checks require.

Step 4 of the walk-through above shows the approach:

1. An agent stopped other sessions' test runs with a pattern kill, although the context file already said not to.
2. The rule was written down, but nothing in the system enforced it.
3. The fix was a `PreToolUse` guard that refuses pattern kills and names the safe form.
4. Every later session runs with the guard. The context-file rule stays, and the guard's refusal message repeats it at the moment it applies.

#### When to change the harness

A rule of thumb decides most cases: when you type the same correction or the same prompt a second time, change the harness so you do not type it a third.

- A repeated prompt becomes a skill or a template.
- A repeated correction becomes a fix in the system: a workflow step, a hook, a script or a test, with a short context-file line as a second layer.

Use judgement too. Fix it the first time when the context shows it will recur, or when a repeat would be costly or risky.

| Signal | Example | Act after |
|---|---|---|
| the same prompt or correction, typed again | retyping the bug-fix steps | the second time |
| the mistake follows from how the work is set up, so it will come back | an agent edits generated files because nothing marks them | the first time |
| one repeat would be costly or hard to undo | the pattern kill above stopped other sessions' test runs | the first time |

## What a harness costs

Every component that loads into the context costs tokens on every turn.

- HarnessTax (UC Berkeley and Arena, 2026) reported that across 21 model and harness pairs, the harness moved task success by about 2 to 5 points.
- The same work reported that Claude Code cost about twice as much as a minimal four-tool harness on SWE-bench Lite.
- Both figures are not yet verified against the primary source, and benchmark tasks are short, so memory, coordination and approvals never show on them.

In practice:

1. Run `/context` in a fresh session to see the standing cost of your harness.
2. Add a component when it fixes a failure you have seen.
3. Measure again, and remove what does not help.

[Guides, sensors and enforcement]({{ '/sections/02-ai-harnesses/guides-sensors-and-enforcement.html' | relative_url }}#harness-cost) has more on cost and on reading a harness you did not build.

## Pages for this section

| Page | What it covers |
|---|---|
| [Harness context files]({{ '/sections/02-ai-harnesses/context-files.html' | relative_url }}) | What goes in, what stays out, writing rules a model follows |
| [Context file layering]({{ '/sections/02-ai-harnesses/context-file-layering.html' | relative_url }}) | Scopes, precedence, imports, path rules |
| [Context file size budgets]({{ '/sections/02-ai-harnesses/context-file-budgets.html' | relative_url }}) | Why size matters, measuring, trimming |
| [Compiling context files]({{ '/sections/02-ai-harnesses/compiling-context-files.html' | relative_url }}) | Source modules, one build for several tools |
| [Native commands and tools]({{ '/sections/02-ai-harnesses/native-commands-and-tools.html' | relative_url }}) | Every built-in tool and command, with Codex equivalents |
| [Harness skills]({{ '/sections/02-ai-harnesses/skills.html' | relative_url }}) | Anatomy, triggering, authoring, testing |
| [Plugins and MCP servers]({{ '/sections/02-ai-harnesses/plugins-and-mcp-servers.html' | relative_url }}) | Protocol, transports, packaging, cost |
| [Permissions and settings]({{ '/sections/02-ai-harnesses/permissions-and-settings.html' | relative_url }}) | Scopes, rule syntax, modes, sandboxing |
| [Harness hooks]({{ '/sections/02-ai-harnesses/hooks.html' | relative_url }}) | Input, output, exit codes, writing one |
| [Hook lifecycle events]({{ '/sections/02-ai-harnesses/hook-lifecycle-events.html' | relative_url }}) | Each event and what it can do |
| [Guard hooks]({{ '/sections/02-ai-harnesses/guard-hooks.html' | relative_url }}) | Hooks that refuse, and how to write the refusal |
| [Hook failure modes]({{ '/sections/02-ai-harnesses/hook-failure-modes.html' | relative_url }}) | Fail open, fail closed, bypasses, noise |
| [Agent profiles]({{ '/sections/02-ai-harnesses/agent-profiles.html' | relative_url }}) | Fields, roles, model choice |
| [Harness memory and context]({{ '/sections/02-ai-harnesses/memory-and-context.html' | relative_url }}) | Memory files, the window, task tracking |
| [Guides, sensors and enforcement]({{ '/sections/02-ai-harnesses/guides-sensors-and-enforcement.html' | relative_url }}) | Guides and sensors, the ladder, cost, reading a harness |

## Sources

- Böckeler, "Harness engineering": <https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html> (2026-04-02), verified 2026-11-08.
- "context, not enforced configuration": <https://code.claude.com/docs/en/memory.md>, verified 2026-11-08.
- Claude Code hooks, settings, skills and sub-agents: <https://code.claude.com/docs/en/hooks.md>, <https://code.claude.com/docs/en/settings.md>, <https://code.claude.com/docs/en/skills.md>, <https://code.claude.com/docs/en/sub-agents.md>.
- Model Context Protocol: <https://modelcontextprotocol.io/>.
- Beads: <https://github.com/steveyegge/beads>.
- Harness cost measurements: reported by HarnessTax (UC Berkeley and Arena, 2026), <https://harnesstax.github.io/>; not yet verified against the primary source.
- The examples labelled as my harness are from the author's own harness, described conceptually.

{% include page_nav.html %}
