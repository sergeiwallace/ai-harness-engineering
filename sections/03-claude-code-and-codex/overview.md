---
title: "Claude Code and Codex basics"
nav_order: 13
has_children: true
has_toc: false
permalink: /sections/03-claude-code-and-codex/
last_updated: "2026-11-09"
talk_stop: "4"
stop_minutes: "6"
---

{% include page_nav.html %}

# Claude Code and Codex basics

Claude Code (from Anthropic) and Codex (from OpenAI) are coding agents that work inside a repository.

- You start one in a terminal, in a project folder, and give it a task in plain language.
- It reads the code, runs commands, edits files and reports back.
- Every later section of this site builds on what these two clients provide before you customise anything.

This overview covers, in order:

1. Installing a client and running a first session.
2. The building blocks both clients share: context files, skills, hooks, sub-agents, memory and compaction.
3. Settings and permissions, the commands worth knowing, and how an agent waits on long work.
4. Where the clients run, which features need which sign-in, and how to grow a setup.

Each part links down to a topic page; the [pages in this section](#pages-in-this-section) are listed at the end.

![Sequence of one task in Claude Code: you set a goal, the client loads context files into the model, each tool call passes a PreToolUse hook before it runs, an edit triggers a formatter hook, and when the agent tries to stop the goal evaluator sends it back until a passing run and a commit are shown]({{ '/assets/images/03-claude-code-and-codex/one-task-sequence.svg' | relative_url }})

*One task, start to finish. Context files reach the model at launch, and a subdirectory's file loads when the agent first touches that folder. Hooks sit on every tool call: PreToolUse can refuse before anything runs, PostToolUse runs after an edit. The goal is a Stop hook that reads only the conversation, so it sends the agent back until the evidence is printed. Diagram made with archify; the session is an illustration.*

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

## Your first session

### Install

| Client | Command | Notes |
|---|---|---|
| Claude Code, macOS, Linux, WSL | `curl -fsSL https://claude.ai/install.sh \| bash` | Native installer; updates itself in the background |
| Claude Code, Windows PowerShell | `irm https://claude.ai/install.ps1 \| iex` | Git for Windows is recommended so the agent can use Bash |
| Claude Code, Homebrew | `brew install --cask claude-code` | Stable channel; does not auto-update (`brew upgrade claude-code`) |
| Claude Code, WinGet | `winget install Anthropic.ClaudeCode` | Does not auto-update |
| Codex, npm | `npm install -g @openai/codex` | |
| Codex, Homebrew | `brew install --cask codex` | |

Check the install with `claude --version`; it prints a version followed by `(Claude Code)`.

### Start and sign in

```bash
cd ~/code/my-project
claude            # or: codex
```

| | Claude Code | Codex |
|---|---|---|
| First start | Prompts you to log in through the browser | Choose "Sign in with ChatGPT" |
| Account types | Claude subscription, Claude Console (API credits), or a cloud provider such as Amazon Bedrock | ChatGPT plan, or an API key with extra setup |
| Later sign-in changes | `/login` inside the session | `codex login`, `/logout` |

### First prompts

Ask about the code before you change it:

```text
> what does this project do?
> where is the main entry point?
> add input validation to the user registration form
> what files have I changed?
> commit my changes with a descriptive message
```

- Before an edit or a command it has no permission for, the agent asks; answer yes or no.
- In Claude Code, `Shift+Tab` cycles the permission mode (default, accept edits, plan, and others); see [Configuration and permissions]({{ '/sections/03-claude-code-and-codex/configuration-and-permissions.html' | relative_url }}#permission-modes).
- Type `/` to see the commands and skills available.

<!-- IMAGE NEEDED: a first Claude Code or Codex session in a terminal, showing the prompt, a question about the project and the agent's answer; search: "Claude Code terminal screenshot", "Codex CLI TUI", "Claude Code quickstart" -->

### Useful shell flags

| Command | What it does |
|---|---|
| `claude "fix the build error"` | Starts a session with a first prompt |
| `claude -c` | Continues the most recent conversation in this directory |
| `claude -r` | Picks a previous conversation to resume |
| `claude -p "explain this function"` | Runs one task without the interactive session and exits |

## What the two clients share

### The building blocks by name

The two clients have the same building blocks under different names. Most of what you learn on one carries over to the other.

| Building block | Claude Code | Codex | Read more |
|---|---|---|---|
| Context file | `CLAUDE.md` | `AGENTS.md` | [Context files]({{ '/sections/04-context-files/' | relative_url }}) |
| Skills | `.claude/skills/<name>/SKILL.md` | `.agents/skills` | [Skills]({{ '/sections/05-skills-and-plugins/' | relative_url }}) |
| Hooks | 33 events | 14 events | [Hooks]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) |
| Sub-agents | Markdown in `.claude/agents/` | TOML in `.codex/agents/` | [Delegation]({{ '/sections/08-delegation/' | relative_url }}) |
| Keep working until a condition holds | `/goal` | `/goal` | [Goal and loop commands]({{ '/sections/03-claude-code-and-codex/goal-and-loop.html' | relative_url }}) |
| Repeat a prompt on an interval | `/loop` | none | [Goal and loop commands]({{ '/sections/03-claude-code-and-codex/goal-and-loop.html' | relative_url }}#loop) |
| Summarise to free context | `/compact` | `/compact` | [Context window]({{ '/sections/11-context-window/' | relative_url }}) |
| Write a starter context file | `/init` writes `CLAUDE.md` | `/init` writes `AGENTS.md` | [Context files]({{ '/sections/04-context-files/' | relative_url }}) |
| Import another setup | none | `/import` | [Claude Code and Codex compared]({{ '/sections/03-claude-code-and-codex/claude-code-and-codex-compared.html' | relative_url }}) |

### Where they differ

| | Claude Code | Codex |
|---|---|---|
| Settings format | JSON files at user, project, local and managed levels | TOML configuration in `~/.codex` |
| Context-file limit | Suggestion: under 200 lines per `CLAUDE.md` | Hard cap: 32 KiB in total |
| Path-scoped rules | `.claude/rules/` with `paths:` | not documented |
| Hook events | 33 | 14 |
| Sub-agent profiles | Markdown | TOML, inheriting the sandbox |

The full grid and how to run one repository with both clients are on [Claude Code and Codex compared]({{ '/sections/03-claude-code-and-codex/claude-code-and-codex-compared.html' | relative_url }}).

#### One context file for both

```markdown
<!-- CLAUDE.md -->
@AGENTS.md

## Claude Code only
- Run /code-review before opening a pull request.
```

- Codex reads `AGENTS.md` directly.
- Claude Code reads `CLAUDE.md`, which pulls in `AGENTS.md` through an `@` import and adds its own lines.
- The shared rules live in one file.

## The building blocks in brief

Each block has a section of its own later on this site. Below is what each one is, what it looks like on disk, and the smallest useful example. The blocks differ in how reliably they hold:

| Block | Harness layer | Holds |
|---|---|---|
| Context file | Instructions | Most of the time; less often in a busy or nearly full window |
| Skill | Workflow: builds the steps in | Each time it runs |
| Hook | Encodes how output should be, and enforces at the moment of action | Every time its event fires |
| Sub-agent | Workflow: separates the work | Each time it is called |
| Tests and checks | Validation before a change lands | Every run |

A repeated mistake is fixed in the most reliable layer that can prevent it; the context file explains the fix.

### Context file

A Markdown file of instructions loaded at the start of every session: build and test commands, conventions, things never to do.

```markdown
- Test: `make test`. Run one test: `pytest -k <name>`.
- Never edit files under `generated/`; run `make gen`.
```

- `/init` writes a starter file from the repository; `/memory` opens the files for editing.
- Full treatment: [Context files]({{ '/sections/04-context-files/' | relative_url }}).

### Skills

A skill is a folder with a `SKILL.md`: a name, a description the model reads, and the procedure.

- The model sees only the descriptions until a task matches one, then loads that skill's body, so a large library costs little context.
- Typing `/release` runs the skill below on purpose; asking "cut a release" lets the model pick it.

```markdown
---
name: release
description: Cut a release. Use when asked to release, tag or publish a version.
---
1. Confirm main is up to date and the full suite passes.
2. Update CHANGELOG.md from the commits since the last tag.
3. Tag and push the tag.
```

Full treatment: [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}).

### Hooks

A hook is a command the client runs on an event: before or after a tool call, when you submit a prompt, or when the agent tries to stop. It runs every time its event fires, whatever the model is attending to.

- Proactively, a hook encodes how you want the output to be: formatting, file layout, required sections, conventions.
- Reactively, it stops a mistake at the moment it happens.

| Event | Typical use | Example |
|---|---|---|
| Before a tool call | Refuse something dangerous | Block a pattern kill such as `pkill -f` |
| After a tool call | Shape the output as you want it | Run the formatter on the edited file; check a new doc page has its required sections |
| On stop | Check the work is finished | Refuse to stop while tests fail |
| At session start | Inject current state | Print the open tasks |

Full treatment: [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}).

### Sub-agents

A sub-agent profile is a named role with its own instructions, tools and model.

1. The main session hands the sub-agent a task.
2. The sub-agent works in its own context window.
3. It returns a result, so the main window holds the answer and leaves out the search.

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

Two mechanisms carry knowledge past the limits of one window.

| Mechanism | What it does | Commands |
|---|---|---|
| Auto memory | The client keeps notes it writes for itself and reads in later sessions | `/memory` (Claude Code), `/memories` (Codex) |
| Compaction | Summarises the conversation when the window fills, so the session can continue | `/compact <focus>`, `/autocompact` |

```text
> /context
  ⎿  Messages: 71%   Free: 9%
> /compact keep the failing test name, the root cause and the plan's next step
```

- After compaction the project-root `CLAUDE.md` is read again from disk; folder-level files return only when the agent touches that folder ([Context files after compaction]({{ '/sections/04-context-files/after-compaction.html' | relative_url }})).
- Full treatment: [Context window and memory]({{ '/sections/11-context-window/' | relative_url }}).

<details markdown="1">
<summary>Which block to reach for</summary>

| The problem | Block |
|---|---|
| The agent does not know a fact or a command | Context file line |
| The agent does a multi-step task differently each time | Skill |
| The agent breaks a rule it was told | Hook, plus a context-file line explaining it |
| The agent's window fills with search results | Sub-agent |
| A long session forgets earlier decisions | Compaction with a focus, or a memory note |

</details>

## Configuration in brief

### Settings files and precedence

Claude Code reads settings from several files. When two set the same key, the higher one wins.

| Precedence | File | Who it affects |
|---|---|---|
| 1, highest | Managed settings | Everyone your organization deploys it to |
| 2 | `--settings` on the command line | One session; writes no file |
| 3 | `.claude/settings.local.json` | You, in one project; kept out of git |
| 4 | `.claude/settings.json` | The team, once committed |
| 5, lowest | `~/.claude/settings.json` | You, in every project |

- List keys such as `permissions.allow` merge across files: each file adds its entries.
- Committed `allow` rules wait until each person trusts the folder; `deny` and `ask` rules apply at once.
- `/status` lists which files loaded.

### What a settings file holds

A settings file holds permissions, hooks, environment variables and the default model. This one lets the agent run the tests without asking, asks before any push, blocks reading the secrets file, and formats every edited file:

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

A slash command is something you type in the session, such as `/compact`. Both clients ship with dozens; these come up throughout this site, and the rest are grouped by purpose on [Native commands]({{ '/sections/03-claude-code-and-codex/native-commands.html' | relative_url }}).

| Command | What it does | When you reach for it | Codex |
|---|---|---|---|
| `/goal <condition>` | Keeps the session working until a separate small model judges the condition met | "Keep going until the tests pass and the branch is pushed" | `/goal` |
| `/loop` | Repeats a prompt on an interval; recurring tasks expire after 7 days | "Check the deploy every five minutes" | none |
| `/compact <focus>` | Summarises the session to free context, steered by the focus text | The window is filling and you want to keep two facts | `/compact` |
| `/context` | Shows what fills the context window | Before a long task | `/status` shows token usage |
| `/model`, `/effort` | Choose the model and the reasoning effort | A hard task, or a cheap one | `/model` |
| `/rc` | Controls the session from another device; needs a claude.ai sign-in | You walk away from the desk | none |

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

A forty-minute test suite does not have to block the session.

1. The agent starts the command with `run_in_background: true` and gets a handle at once.
2. It keeps working on something else, or the session goes idle.
3. When the command finishes, its result wakes the session as a new turn. No goal, loop or polling is needed; a sub-agent's result arrives the same way.
4. To react before it finishes, the agent can attach the Monitor tool to the output, filtered to the lines that matter.
   - A failure line lets it start on the failing test while the suite finishes.
   - Monitor or a scheduled check also covers work the session did not start, such as another process's log.

```text
● Bash(python -m pytest tests -q > /tmp/run.log 2>&1; echo "EXIT $?" >> /tmp/run.log)
  run_in_background: true

● Monitor(command: tail -f /tmp/run.log | grep --line-buffered -E "FAILED|EXIT")
  ⎿  FAILED tests/test_sync.py::test_lock_release
  ⎿  EXIT 1
```

Codex lists its background terminals with `/ps` and stops them with `/stop`. [Background work and headless runs]({{ '/sections/03-claude-code-and-codex/background-work-and-headless-runs.html' | relative_url }}) covers background shells, Monitor patterns and their trade-offs.

### Headless runs

`claude -p "<prompt>"` runs one task without the interactive session and prints the result, which is how the client runs in CI.

- `--bare` skips hooks, skills, plugins and `CLAUDE.md`, so a job behaves the same on every machine.
- Leave `--bare` off for jobs that edit or push, so your hooks still apply.

```yaml
- name: Summarise the failing tests
  run: claude -p --bare "Read test-results.xml and list each failing test with a one-line cause" > summary.txt
```

## Where they run and what needs a sign-in

### Surfaces

Both clients run in a terminal, in editors, and as desktop or web apps. Claude Code "runs the same underlying engine everywhere", and the terminal, the editor extensions and the desktop app read the same settings files.

| Surface | Good for | What carries over |
|---|---|---|
| Terminal | Long tasks, several sessions side by side, scripting | Everything |
| Editor extension | Work beside the code you are reading | Same engine and settings as the terminal |
| Desktop and web | Starting or reviewing work away from a terminal | A cloud session runs on a fresh clone and does not read your user or local settings |
| Headless (`claude -p`) | CI jobs and scripts | Skills and hooks run; terminal-only commands do not |

### One setup, two surfaces

1. You add a formatter hook to `.claude/settings.json` while working in the terminal.
2. You open the same project in the editor extension.
3. The hook runs there too, because both surfaces read the same file.
   - A cloud session would also run it once the file is committed.
   - A hook kept only in `~/.claude/settings.json` would not reach a cloud session.

### Sign-in routes

Some features need a claude.ai sign-in and do not work with API keys or through cloud providers such as Amazon Bedrock. If you use Claude Code through work, check which route your account uses before you rely on a feature.

| Feature | With a claude.ai sign-in | On a cloud provider |
|---|---|---|
| `/goal`, dynamic workflows, auto mode | Yes | Yes |
| Same-machine session messaging | Yes | Yes, from v2.1.248 |
| Remote Control (`/rc`) | Yes | No |
| Claude in Chrome, computer use | Yes | No |
| `/advisor` | Yes | No; Anthropic API only |

The full table is on [Claude Code and Codex compared]({{ '/sections/03-claude-code-and-codex/claude-code-and-codex-compared.html' | relative_url }}#availability-on-cloud-providers).

## Growing a setup

1. Run one client in one real project with the defaults.
2. Give it real tasks for a few days and note each time it goes wrong.
3. Write a short [context file]({{ '/sections/04-context-files/' | relative_url }}) with the commands and conventions the agent cannot guess.
4. When a procedure repeats, make it a [skill]({{ '/sections/05-skills-and-plugins/' | relative_url }}), so the steps are built in.
5. Encode how you want the output to be (formatting, structure, required sections) in a formatter, linter or hook, from the start.
6. When a mistake repeats, fix it in the system.
   - A workflow or script that makes the right path the default.
   - A [hook]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) that refuses the wrong action.
   - A test that catches it before a change lands.
   - Then one context-file line that explains the check.
7. Add one piece at a time, so a new failure has one suspect.

## Common pitfalls

| Pitfall | What happens | What to do |
|---|---|---|
| Many features turned on at once | A failure has several possible causes | Add one piece at a time |
| Scripting against a command name | Names and meanings change between releases (`/review`, `/simplify` both changed) | Check the commands reference for your installed version |
| A feature that needs a claude.ai sign-in | It is missing on a cloud provider, sometimes without an error | Say in each procedure which sign-in route it assumes |
| A hook only in your user settings | It does not run in a cloud session | Commit hooks the team relies on to `.claude/settings.json` |
| `--bare` on a job that edits | Your safety hooks do not run | Use `--bare` only for read-only jobs |
| An unfiltered monitor | The window fills with log lines | Filter to failures and exit codes |

## Examples from my own harness

### One install for every machine

I work on several machines. One install step puts the same context files, skills, hooks and agent profiles on each of them.

1. The shared context file is assembled from parts: a base, a part per client, a part per machine.
2. It is checked against a size budget and rendered into a versioned directory.
3. The install links it into each client's config location.
4. A bad edit rolls back by relinking the previous version.

[Compiled shared context files]({{ '/sections/04-context-files/compiled-shared-context.html' | relative_url }}) describes the pattern.

![Sequence: an author adds a lesson to a source part, the compiler fails the build when the joined file is over its size budget, a second try moves the detail to a spoke and passes, the new version is linked into every client, and a bad edit is undone by relinking the previous version]({{ '/assets/images/04-context-files/compiled-file-sequence.svg' | relative_url }})

*Adding one lesson to the compiled shared file. A failed build installs nothing; rollback changes one link. Diagram made with archify.*

### Codex as a second family

My harness drives most work from Claude Code and calls Codex through a small wrapper.

- One configuration per role: reviewer, researcher, implementer.
- The main use is review: a change written with one model family is reviewed with the other.
  - The reviewer gets the diff and the spec only.
  - That is a choice for my setup; a fresh-context review with the same family is another.

See [Code review]({{ '/sections/13-code-review/' | relative_url }}).

## Pages in this section

| Page | What it covers |
|---|---|
| [Claude Code and Codex compared]({{ '/sections/03-claude-code-and-codex/claude-code-and-codex-compared.html' | relative_url }}) | Feature grid, limits and defaults, configuration formats, running both on one repository, availability by sign-in route |
| [Native commands]({{ '/sections/03-claude-code-and-codex/native-commands.html' | relative_url }}) | The built-in commands grouped by purpose, with a worked session |
| [Goal and loop commands]({{ '/sections/03-claude-code-and-codex/goal-and-loop.html' | relative_url }}) | Sub-topic of Native commands: how `/goal` and `/loop` work, writing conditions, failure modes |
| [Configuration and permissions]({{ '/sections/03-claude-code-and-codex/configuration-and-permissions.html' | relative_url }}) | Settings files, precedence, permission rules and modes, Codex configuration |
| [Background work and headless runs]({{ '/sections/03-claude-code-and-codex/background-work-and-headless-runs.html' | relative_url }}) | Background shells, the Monitor tool, `claude -p`, `--bare` and CI |

## Sources

- Install, first session, shell flags and permission-mode switching: [Claude Code: quickstart](https://code.claude.com/docs/en/quickstart.md), read 2026-11-09.
- Commands and features: [Claude Code: commands reference](https://code.claude.com/docs/en/commands.md), [Claude Code: keep Claude working toward a goal](https://code.claude.com/docs/en/goal.md), [Claude Code: run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks.md), [Claude Code: Remote Control](https://code.claude.com/docs/en/remote-control.md), verified 2026-11-08 and 2026-11-09.
- Settings files and precedence: [Claude Code: settings files and precedence](https://code.claude.com/docs/en/settings.md), read 2026-11-09.
- Surfaces: [Claude Code: platforms](https://code.claude.com/docs/en/platforms.md), verified 2026-11-08.
- Monitor and headless runs: [Claude Code: tools reference](https://code.claude.com/docs/en/tools-reference), [Claude Code: headless mode](https://code.claude.com/docs/en/headless.md), verified 2026-11-02 and 2026-11-09.
- Codex install and sign-in: [the Codex CLI README on GitHub](https://github.com/openai/codex), read 2026-11-09. Codex commands (including `/init`, `/model`, `/ps`, `/stop`, `/status`): [Codex: CLI slash commands](https://learn.chatgpt.com/docs/developer-commands.md?surface=cli), read 2026-11-09. Hooks, sub-agents, skills: [Codex: hooks](https://learn.chatgpt.com/docs/hooks.md), [Codex: subagents](https://learn.chatgpt.com/docs/agent-configuration/subagents.md), [Codex: build skills](https://learn.chatgpt.com/docs/build-skills.md), verified 2026-11-08.
- The install pattern is from the author's own harness, described conceptually. Example files and session excerpts are illustrations written for this page.

{% include page_nav.html %}
