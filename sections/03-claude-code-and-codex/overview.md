---
title: "Claude Code and Codex basics"
nav_order: 13
has_children: true
has_toc: false
permalink: /sections/03-claude-code-and-codex/
---

{% include page_nav.html %}

# Claude Code and Codex basics

As of 2026-11-09. Slide: [stop 4 on the talk page]({{ '/talk/' | relative_url }}#stop-4-6-minutes), "Claude Code and Codex".

Claude Code (from Anthropic) and Codex (from OpenAI) are coding agents that live in a repository. You start one in a terminal inside a project folder and give it a task in plain language. It reads the code, runs commands, edits files and reports back. Every later section of this site builds on what these two clients give you before you customise anything: a context file, skills, hooks, sub-agents, a goal command and compaction. This overview covers each of those building blocks with an example, and links down to a topic page wherever a block deserves a full article.

![Sequence of one task in Claude Code: you set a goal, the client loads context files into the model, each tool call passes a PreToolUse hook before it runs, an edit triggers a formatter hook, and when the agent tries to stop the goal evaluator sends it back until a passing run and a commit are shown]({{ '/assets/images/03-claude-code-and-codex/one-task-sequence.svg' | relative_url }})

*One task, start to finish. Context files reach the model at launch, and a subdirectory's file loads when the agent first touches that folder. Hooks sit on every tool call: PreToolUse can refuse before anything runs, PostToolUse runs after an edit. The goal is a Stop hook that only reads the conversation, so it sends the agent back until the evidence is printed. Diagram made with archify; the session is an illustration.*

| Phase of the task | Building block at work | Covered in |
|---|---|---|
| Launch | Context files, the skill list, settings | [Context files]({{ '/sections/04-context-files/' | relative_url }}), [Configuration and permissions]({{ '/sections/03-claude-code-and-codex/configuration-and-permissions.html' | relative_url }}) |
| Each tool call | Permission rules and hooks | [Hooks]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) |
| Long waits | Background shells and Monitor | [Background work and headless runs]({{ '/sections/03-claude-code-and-codex/background-work-and-headless-runs.html' | relative_url }}) |
| Trying to stop | `/goal` | [Goal and loop commands]({{ '/sections/03-claude-code-and-codex/goal-and-loop.html' | relative_url }}) |

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What the two clients share

### What both give you

The two clients now have the same building blocks under different names. If you learn one, most of what you learn carries over to the other.

| Building block | Claude Code | Codex | Read more |
|---|---|---|---|
| Context file | `CLAUDE.md` | `AGENTS.md` | [Context files]({{ '/sections/04-context-files/' | relative_url }}) |
| Skills | `.claude/skills/<name>/SKILL.md` | `.agents/skills` | [Skills]({{ '/sections/05-skills-and-plugins/' | relative_url }}) |
| Hooks | 33 events | 14 events | [Hooks]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) |
| Sub-agents | Markdown in `.claude/agents/` | TOML in `.codex/agents/` | [Delegation]({{ '/sections/08-delegation/' | relative_url }}) |
| Keep working until a condition holds | `/goal` | `/goal` | [Goal and loop commands]({{ '/sections/03-claude-code-and-codex/goal-and-loop.html' | relative_url }}) |
| Repeat a prompt on an interval | `/loop` | none | [Goal and loop commands]({{ '/sections/03-claude-code-and-codex/goal-and-loop.html' | relative_url }}) |
| Summarise to free context | `/compact` | `/compact` | [Context window]({{ '/sections/11-context-window/' | relative_url }}) |
| Import another setup | none | `/import` | [Claude Code and Codex compared]({{ '/sections/03-claude-code-and-codex/claude-code-and-codex-compared.html' | relative_url }}) |

### Where they differ

They differ most in defaults and limits, and in how they are configured. Claude Code is configured through JSON settings files and Markdown files under `.claude/`. Codex is configured through a TOML file in its home directory and TOML agent profiles. The full feature grid, the size limits and how to run one repository with both clients are on [Claude Code and Codex compared]({{ '/sections/03-claude-code-and-codex/claude-code-and-codex-compared.html' | relative_url }}).

#### One context file for both

```markdown
<!-- CLAUDE.md -->
@AGENTS.md

## Claude Code only
- Run /code-review before opening a pull request.
```

Codex reads `AGENTS.md` directly. Claude Code reads `CLAUDE.md`, which pulls in `AGENTS.md` through an `@` import and adds its own lines. The shared rules live in one file.

## The building blocks in brief

Each block has a section of its own later on this site. This is what each one is, what it looks like on disk, and the smallest useful example.

### Context file

A Markdown file of instructions loaded at the start of every session: build and test commands, conventions, things never to do.

```markdown
- Test: `make test`. Run one test: `pytest -k <name>`.
- Never edit files under `generated/`; run `make gen`.
```

Full treatment: [Context files]({{ '/sections/04-context-files/' | relative_url }}).

### Skills

A skill is a folder with a `SKILL.md`: a name, a description the model reads, and the procedure. The model sees only the descriptions until a task matches one, then loads that skill's body, so a large library costs little context.

```markdown
---
name: release
description: Cut a release. Use when asked to release, tag or publish a version.
---
1. Confirm main is up to date and the full suite passes.
2. Update CHANGELOG.md from the commits since the last tag.
3. Tag and push the tag.
```

Typing `/release` runs it on purpose; asking "cut a release" lets the model pick it. Full treatment: [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}).

### Hooks

A hook is a command the client runs on an event, such as before or after a tool call, when you submit a prompt, or when the agent tries to stop. Unlike a context-file line, a hook runs every time.

| Event | Typical use | Example |
|---|---|---|
| Before a tool call | Refuse something dangerous | Block a pattern kill such as `pkill -f` |
| After a tool call | Clean up after an edit | Run the formatter on the edited file |
| On stop | Check the work is finished | Refuse to stop while tests fail |
| At session start | Inject current state | Print the open tasks |

Full treatment: [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}).

### Sub-agents

A sub-agent profile is a named role with its own instructions, tools and model. The main session hands it a task; it works in its own context and returns a result, so the main window holds the answer and not the search.

```markdown
---
name: reviewer
description: Reviews a diff for correctness bugs. Use after a change is ready.
tools: Read, Grep, Glob, Bash
---
Read the diff, find bugs that change behaviour, and report each with the file,
the line and a failing input.
```

Claude Code keeps these as Markdown in `.claude/agents/`; Codex keeps them as TOML in `.codex/agents/`. Full treatment: [Agent profiles, sub-agents and delegation]({{ '/sections/08-delegation/' | relative_url }}).

### Memory and compaction

Two mechanisms carry knowledge across the limits of one window.

| Mechanism | What it does | Commands |
|---|---|---|
| Auto memory | The client keeps notes it writes for itself and reads in later sessions | `/memory` (Claude Code), `/memories` (Codex) |
| Compaction | Summarises the conversation when the window fills, so the session can continue | `/compact <focus>`, `/autocompact` |

```text
> /context
  ⎿  Messages: 71%   Free: 9%
> /compact keep the failing test name, the root cause and the plan's next step
```

After compaction the project-root `CLAUDE.md` is read again from disk; folder-level files return only when the agent touches that folder ([Context files after compaction]({{ '/sections/04-context-files/after-compaction.html' | relative_url }})). Full treatment: [Context window and memory]({{ '/sections/11-context-window/' | relative_url }}).

<details markdown="1">
<summary>Which block to reach for</summary>

| The problem | Block |
|---|---|
| The agent does not know a fact or a command | Context file line |
| The agent does a multi-step task differently each time | Skill |
| The agent breaks a rule it was told | Hook |
| The agent's window fills with search results | Sub-agent |
| A long session forgets earlier decisions | Compaction with a focus, or a memory note |

</details>

## Where they run

### Surfaces

Both clients run in a terminal, in editors, and as desktop or web apps. Claude Code "runs the same underlying engine everywhere", and the terminal, the editor extensions and the desktop app read the same settings files, so a setup you build in the terminal also applies in the editor.

| Surface | Good for | What carries over |
|---|---|---|
| Terminal | Long tasks, several sessions side by side, scripting | Everything |
| Editor extension | Work beside the code you are reading | Same engine and settings as the terminal |
| Desktop and web | Starting or reviewing work away from a terminal | A cloud session runs on a fresh clone and does not read your user or local settings |
| Headless (`claude -p`) | CI jobs and scripts | Skills and hooks run; terminal-only commands do not |

> A cloud session is a different machine. It reads the repository's committed files and your organization's server-managed settings, and nothing from your own home directory. A hook that only exists in `~/.claude/settings.json` does not run there.

### One setup, two surfaces

You add a formatter hook to `.claude/settings.json` while working in the terminal. The next time you open the same project in the editor extension, the hook runs there too, because both read the same file. The topic page [Configuration and permissions]({{ '/sections/03-claude-code-and-codex/configuration-and-permissions.html' | relative_url }}) covers which file reaches which surface.

## Configuration in brief

### The layers

Claude Code reads settings from several files, and when two set the same key the higher one wins. The order, highest first, is managed settings (your organization's policy), the command line (`--settings`, one session), project local (`.claude/settings.local.json`, you in one project), shared project (`.claude/settings.json`, committed for the team) and user (`~/.claude/settings.json`, you everywhere). List keys such as `permissions.allow` merge across files: each file adds its entries to the list.

### What a settings file holds

A settings file holds permissions, hooks, environment variables and the default model. This one lets the agent run the tests without asking, asks before any push, and formats every edited file:

```json
{
  "permissions": {
    "allow": ["Bash(npm test *)"],
    "ask": ["Bash(git push *)"],
    "deny": ["Read(./.env)"]
  },
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "jq -r '.tool_input.file_path' | xargs npx prettier --write" }]
      }
    ]
  }
}
```

![Sequence of three tool calls: make test matches an allow rule and runs; git push matches an ask rule, you answer yes and it runs; reading .env matches a deny rule and is refused, so the agent asks you for the one value it needs]({{ '/assets/images/03-claude-code-and-codex/permission-check-sequence.svg' | relative_url }})

*Three tool calls through those rules. Rules from every file merge, and a local allow cannot outrank a project ask. Diagram made with archify.*

The permission rules, the modes and the order in which allow, ask and deny combine are on [Configuration and permissions]({{ '/sections/03-claude-code-and-codex/configuration-and-permissions.html' | relative_url }}).

## Commands worth knowing by name

A slash command is something you type in the session, such as `/compact`. Claude Code ships with dozens. Four come up throughout this site, and the rest are grouped by purpose on [Native commands]({{ '/sections/03-claude-code-and-codex/native-commands.html' | relative_url }}).

| Command | What it does | When you reach for it |
|---|---|---|
| `/goal <condition>` | Keeps the session working until a separate small model judges the condition met | "Keep going until the tests pass and the branch is pushed" |
| `/loop` | Repeats a prompt on an interval; recurring tasks expire after 7 days | "Check the deploy every five minutes" |
| `/compact <focus>` | Summarises the session to free context, steered by the focus text | The window is filling and you want to keep two facts |
| `/rc` | Controls the session from another device; needs a claude.ai sign-in | You walk away from the desk |

### How /goal works

![Sequence of two goal checks: the agent edits and tries to stop with a claim, the evaluator answers not met because no test output follows the edit; the agent runs the test and the suite and commits, tries to stop again, and the evaluator answers met]({{ '/assets/images/03-claude-code-and-codex/goal-checks-sequence.svg' | relative_url }})

*Two goal checks. A claim is refused; printed evidence is accepted. Diagram made with archify.*

1. You type `/goal` with a condition.
2. The agent works: reads, edits, runs commands.
3. The agent tries to stop, and the goal's Stop hook runs a small evaluator model.
   - The evaluator has no tools; it reads only the conversation.
   - Not met: the agent is sent back with the reason.
   - Met: the session stops.
4. Steps 2 and 3 repeat until the condition is met.

A condition the evaluator can judge names evidence the agent will print, such as a test summary or a commit id. [Goal and loop commands]({{ '/sections/03-claude-code-and-codex/goal-and-loop.html' | relative_url }}) covers how to write conditions that do not stop too early or run forever.

### An unattended fix

```text
> /goal the flaky test in tests/test_sync.py passes 20 runs in a row
  and the fix is committed

● Bash(for i in $(seq 20); do python -m pytest tests/test_sync.py -q || break; done)
  ⎿  3 passed, then 1 failed: timeout waiting for lock

  ... the agent finds a missing release in a cleanup path, fixes it,
      reruns 20 times, commits ...

  ⎿  Goal met: 20 consecutive passes shown, commit a1b2c3d created.
```

Each time the agent tried to stop before the twentieth pass, the evaluator sent it back. The excerpt is an illustration written for this page.

## Waiting without polling

### Background commands and Monitor

Without these, an agent that starts a forty-minute test suite checks on it every few turns and spends context on "still running". With them:

1. The agent starts the command with `run_in_background: true` and gets a handle at once.
2. It attaches the Monitor tool to the output, filtered to the lines that matter.
3. It keeps working on something else.
4. Each matching line arrives as a message the moment it is printed.
   - A failure line lets it start on the failing test while the suite finishes.
   - The exit line tells it the run is over.

```text
● Bash(python -m pytest tests -q > /tmp/run.log 2>&1; echo "EXIT $?" >> /tmp/run.log)
  run_in_background: true

● Monitor(command: tail -f /tmp/run.log | grep --line-buffered -E "FAILED|EXIT")
  ⎿  FAILED tests/test_sync.py::test_lock_release
  ⎿  EXIT 1
```

> Every line a monitor feeds back costs context. Filter it to the lines that should change what the agent does next, such as failures and the exit code.

[Background work and headless runs]({{ '/sections/03-claude-code-and-codex/background-work-and-headless-runs.html' | relative_url }}) covers background shells, Monitor patterns and their trade-offs.

### Headless runs

`claude -p "<prompt>"` runs one task without the interactive terminal and prints the result, which is how the client runs in CI. `--bare` skips hooks, skills, plugins and `CLAUDE.md`, so a job behaves the same on every machine.

```yaml
- name: Summarise the failing tests
  run: claude -p --bare "Read test-results.xml and list each failing test with a one-line cause" > summary.txt
```

## Sign-in and availability

Some features need a claude.ai sign-in and do not work with API keys or through cloud providers such as Amazon Bedrock. If you use Claude Code through work, check which route your account uses before you rely on a feature.

| Feature | With a claude.ai sign-in | On a cloud provider |
|---|---|---|
| `/goal`, dynamic workflows, auto mode | Yes | Yes |
| Same-machine session messaging | Yes | Yes, from v2.1.248 |
| Remote Control (`/rc`) | Yes | No |
| Claude in Chrome, computer use | Yes | No |
| `/advisor` | Yes | No; Anthropic API only |

The full table is on [Claude Code and Codex compared]({{ '/sections/03-claude-code-and-codex/claude-code-and-codex-compared.html' | relative_url }}#availability-on-cloud-providers).

## How to start

1. Install one client and run it in one real project with the defaults.
2. Give it real tasks for a few days and note each time it goes wrong.
3. Write a short [context file]({{ '/sections/04-context-files/' | relative_url }}) with the commands and conventions the agent cannot guess.
4. When a procedure repeats, make it a [skill]({{ '/sections/05-skills-and-plugins/' | relative_url }}), so the steps are built in.
5. When a mistake repeats, fix it in the system.
   - A workflow or script that makes the right path the default.
   - A [hook]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) that refuses the wrong action.
   - A test that catches it before a change lands.
   - Then one context-file line that explains the check.

Add one piece at a time. Turning on many features at once makes a failure hard to trace.

> A context-file line on its own is applied unevenly, especially when the model is busy, switches tasks or is near a full window. Adding a new detailed rule for each mistake is an anti-pattern; [Context files]({{ '/sections/04-context-files/' | relative_url }}#where-it-sits-in-a-harness) shows the layers.

## Examples from my own harness

### One install for every machine

I work on several machines. One install step puts the same context files, skills, hooks and agent profiles on each of them, so a new session on any machine starts with the same setup. The shared context file is assembled from parts (a base, a part per client, a part per machine), rendered into a versioned directory and linked into place, so a bad edit rolls back in one step. [Compiled shared context files]({{ '/sections/04-context-files/compiled-shared-context.html' | relative_url }}) describes the pattern.

![Sequence: an author adds a lesson to a source part, the compiler fails the build when the joined file is over its size budget, a second try moves the detail to a spoke and passes, the new version is linked into every client, and a bad edit is undone by relinking the previous version]({{ '/assets/images/04-context-files/compiled-file-sequence.svg' | relative_url }})

*Adding one lesson to the compiled shared file. A failed build installs nothing; rollback changes one link. Diagram made with archify.*

### Codex as a second family

My harness drives most work from Claude Code and calls Codex through a small wrapper with one configuration per role (reviewer, researcher, implementer). I use it mainly for review: a change written with one model family is reviewed with the other. That is a choice for my setup; a fresh-context review with the same family is another. See [Code review]({{ '/sections/13-code-review/' | relative_url }}).

## Topic pages

| Page | What it covers |
|---|---|
| [Claude Code and Codex compared]({{ '/sections/03-claude-code-and-codex/claude-code-and-codex-compared.html' | relative_url }}) | Feature grid, limits and defaults, configuration formats, running both on one repository, availability by sign-in route |
| [Native commands]({{ '/sections/03-claude-code-and-codex/native-commands.html' | relative_url }}) | The built-in commands grouped by purpose, with a worked session |
| [Goal and loop commands]({{ '/sections/03-claude-code-and-codex/goal-and-loop.html' | relative_url }}) | Sub-topic of Native commands: how `/goal` and `/loop` work, writing conditions, failure modes |
| [Configuration and permissions]({{ '/sections/03-claude-code-and-codex/configuration-and-permissions.html' | relative_url }}) | Settings files, precedence, permission rules and modes, Codex configuration |
| [Background work and headless runs]({{ '/sections/03-claude-code-and-codex/background-work-and-headless-runs.html' | relative_url }}) | Background shells, the Monitor tool, `claude -p`, `--bare` and CI |

## Sources

- Commands and features: <https://code.claude.com/docs/en/commands.md>, <https://code.claude.com/docs/en/goal.md>, <https://code.claude.com/docs/en/scheduled-tasks.md>, <https://code.claude.com/docs/en/remote-control.md>, verified 2026-11-08 and 2026-11-09.
- Settings files and precedence: <https://code.claude.com/docs/en/settings.md>, read 2026-11-09.
- Surfaces: <https://code.claude.com/docs/en/platforms.md>, verified 2026-11-08.
- Monitor and headless runs: <https://code.claude.com/docs/en/tools-reference>, <https://code.claude.com/docs/en/headless.md>, verified 2026-11-02 and 2026-11-09.
- Codex: <https://learn.chatgpt.com/docs/developer-commands.md?surface=cli>, <https://learn.chatgpt.com/docs/hooks.md>, <https://learn.chatgpt.com/docs/agent-configuration/subagents.md>, <https://learn.chatgpt.com/docs/build-skills.md>, verified 2026-11-08.
- The install pattern is from the author's own harness, described conceptually. Example files and session excerpts are illustrations written for this page.

{% include page_nav.html %}
