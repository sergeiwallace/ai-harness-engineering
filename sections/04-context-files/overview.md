---
title: "Context files"
nav_order: 14
has_children: true
has_toc: false
permalink: /sections/04-context-files/
---

{% include page_nav.html %}

# Context files

As of 2026-11-09. Slide: [stop 4 on the talk page]({{ '/talk/' | relative_url }}#stop-4-6-minutes), "Context files".

A context file is a plain Markdown file of instructions that the client loads into the model's context: `CLAUDE.md` for Claude Code, `AGENTS.md` for Codex and several other tools. The agent sees it at the start of every session: how to build and test, the conventions, the commands to avoid. It is the cheapest part of a harness to write and the first one to add, and it guides behaviour without enforcing anything.

This overview covers, in order:

1. What a context file is for, and where it sits among the layers of a harness.
2. How the files load and unload during a session, in Claude Code and in Codex.
3. How to keep the always-loaded part short, and how to write rules that get followed.
4. What to do when a mistake repeats, how to keep the files current, and how to check what loaded.

Each part links down to a topic page; the [pages in this section](#pages-in-this-section) are listed at the end.

![Sequence of one session: the client loads user, project, local and parent files at launch; reading a file in api loads api/CLAUDE.md; editing a test loads the tests path rule; a shell write loads nothing; compaction re-reads only the root file and unscoped rules, and the nested file returns when api is touched again]({{ '/assets/images/04-context-files/loading-sequence.svg' | relative_url }})

*When each context file enters the window during one session. The project-root `CLAUDE.md` and rules without a `paths:` list survive compaction; subdirectory files and path rules return only when a matching file is touched again. Diagram made with archify.*

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What a context file is

### A first file

```markdown
# Project notes

- Build: `make build`. Test: `make test`. Run one test: `pytest -k <name>`.
- Python 3.12, formatted with ruff. Never edit files under `generated/`; run `make gen`.
- Every bug fix starts with a failing test that reproduces it.
- Database migrations must have a down step.
```

Four lines save the agent from guessing the test command, editing generated code, or fixing a bug without a test. `/init` in Claude Code writes a starter file from the repository, and `/memory` opens the files for editing.

### What it is for

| Belongs in a context file | Belongs somewhere else |
|---|---|
| Commands the agent cannot guess: build, test, run one test | The directory layout; the agent can read it |
| Conventions that differ from the language default | Long procedures; link to them |
| Things never to do, with the reason and the alternative | Rules that must hold every time; enforce them in a hook too |
| A short line explaining each check the system enforces | The fix for a repeated mistake; build it into a workflow, hook or test |
| Misleading facts about the environment | Facts that change weekly |

> A context file is read and interpreted, so it is followed most of the time. Claude Code's docs call `CLAUDE.md` "context, not enforced configuration". When a rule must hold every time, enforce it in a workflow, a [hook]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) or a test, and keep the line as the explanation.

### Where it sits in a harness

A context file guides general behaviour. The model applies it unevenly: less reliably when it juggles several things, switches tasks or works near a full context window. In a systems approach to a harness it is one of four layers, and the second line of defence.

1. Instructions: context files, skills and prompts guide the model and explain the checks.
2. Workflows and tools build the right steps in, so the right path is the default.
3. Hooks, scripts and linters refuse or flag a wrong action at the moment it happens.
4. Validation and tests catch what slips through before a change lands.
   - Layers 2 to 4 do the enforcing.
   - Each time a gap shows up, the fix goes into the lowest layer that can prevent it, and the harness improves in a way that holds.

See [AI harnesses]({{ '/sections/02-ai-harnesses/' | relative_url }}) for the systems approach and [Hooks, scripts and linters]({{ '/sections/06-hooks-scripts-and-linters/' | relative_url }}) for layer 3.

## How they load

### The stack for one working directory

The files that apply when the agent works in `repo/api` differ by client.

- In both, files closer to the working directory are read later.
- Codex documents that the later files take precedence; Claude Code concatenates them, and when two lines contradict each other the model may follow either.
- The sequence at the top of this page shows when each Claude Code file enters the window.

#### Claude Code

| Level | Location | Loads |
|---|---|---|
| Managed policy | `/etc/claude-code/CLAUDE.md` (Linux), `/Library/Application Support/ClaudeCode/CLAUDE.md` (macOS) | At launch, every user on the machine; cannot be excluded |
| User | `~/.claude/CLAUDE.md`, and `~/.claude/rules/` | At launch, every project |
| Project | `CLAUDE.md` or `.claude/CLAUDE.md` at the root | At launch |
| Local | `CLAUDE.local.md` | At launch; personal, not committed |
| Parent directories | `CLAUDE.md` above the working directory | At launch |
| Unscoped rule | `.claude/rules/*.md` without `paths:` | At launch |
| Subdirectory | `CLAUDE.md` below the working directory | When the agent reads, writes or edits a file there; `cat` or `head` on one file counts as a read |
| Path rule | `.claude/rules/*.md` with `paths:` | When the agent reads, writes or edits a matching file |

- The files are concatenated, root first, so instructions nearer the working directory are read last; within a directory, `CLAUDE.local.md` comes after `CLAUDE.md`.
- A file can import another with `@path`; paths resolve from the importing file, up to four hops, and an `@path` inside backticks is left as text.
- Block-level HTML comments (`<!-- maintainer notes -->`) are stripped before the content reaches the model.

#### Codex

Codex reads a global `AGENTS.md` from its home directory, then walks from the project root to the working directory and includes at most one file per directory, checking `AGENTS.override.md` first. It stops at 32 KiB in total by default (`project_doc_max_bytes`).

#### AGENTS.md in Claude Code

| Your repository has | Claude Code reads |
|---|---|
| An `AGENTS.md`, and no `CLAUDE.md` or `CLAUDE.local.md` in the working directory or above | The `AGENTS.md` |
| An `AGENTS.md` and a `CLAUDE.md` | The `CLAUDE.md` files only |
| A `CLAUDE.md` that imports `@AGENTS.md` | Both, through the import |

The Project instructions setting in `/config` changes this, for example to read both files (`claude-md-and-agents-md`).

The full rules, the Codex walk and running both clients on one repository are on [How context files load]({{ '/sections/04-context-files/how-context-files-load.html' | relative_url }}).

### Compaction changes what is loaded

When the window fills and the history is summarised, Claude Code re-reads the project-root `CLAUDE.md` and unscoped rules from disk. Subdirectory files and path rules are summarised away until the agent touches a matching file again.

#### Why a rule disappeared

1. A rule in `api/CLAUDE.md` says "every handler returns a typed error", and the agent follows it all morning.
2. Compaction runs, and the nested file is summarised away.
3. The agent writes a new handler with a shell command (`cat > api/refund.py`), without reading any file in `api/`.
4. The nested file never loads again, and the new handler returns a plain string.

| Fix | What it covers | What it misses |
|---|---|---|
| A check in the test suite or a linter that rejects a handler without a typed error | Every handler, however it was written | Nothing it can detect |
| Move the rule to the root file, if it is short | Survives compaction | Costs context on every turn |
| A path rule on `api/**` | Loads on the next read, write or edit of a file in `api/` | A file created with a shell write |

The check enforces the rule; a line in the context file explains it.

[Context files after compaction]({{ '/sections/04-context-files/after-compaction.html' | relative_url }}) goes through each case.

## Keep the always-loaded file short

### Why size matters

| Cost of a long file | Detail |
|---|---|
| Context on every turn | Every always-loaded line is in the window for every request |
| Uneven following | A long file is followed less reliably than a short one |
| Claude Code guidance | Target under 200 lines per `CLAUDE.md`; Claude Code warns at startup and in `/status` when a file is over the recommended length |
| Claude Code hard limits | A file over 4 MiB is skipped; a subdirectory file or path rule over 40,000 characters (200K window) is cut, and the agent sees only its start |
| Imports | Help organise a file but do not reduce its cost; an imported file loads with the file that imports it |
| Codex limit | Reading stops at 32 KiB in total (`project_doc_max_bytes`) |

Two techniques keep the always-loaded part short: hub and spoke, and path-scoped rules.

### Hub and spoke

1. The context file holds one line per rule, with a trigger and a link to a procedure document.
2. The context file loads every session; the procedures do not.
3. When a task matches a trigger, the agent reads that procedure and follows it.

```markdown
- Before a release, read docs/release.md. Tag from main only, after the full suite passes.
```

One line tells the agent the rule, when to read more and where. The twenty-step release procedure stays out of the context until someone releases.

![Sequence during a release: the hub line is always loaded; when you ask for a release the trigger matches and the agent reads docs/release.md, gets the steps, runs the full suite, writes the changelog and tags main]({{ '/assets/images/04-context-files/hub-and-spoke-sequence.svg' | relative_url }})

*Hub and spoke during a release task. The trigger is in the line itself; a bare list of links gets ignored. Diagram made with archify.*

### Path-scoped rules

A rule file in `.claude/rules/` with a `paths:` list in its frontmatter loads only when the agent touches a matching file, so the rules for tests appear only during work on tests.

```markdown
---
paths: ["tests/**/*.py"]
---
- Test through the public entry point, never a private helper.
- Name tests test_<given>_<when>_<then>.
```

- A path rule loads when the agent reads, writes or edits a matching file; `cat` or `head` on one file counts as a read.
- A file created with a shell write (`cat > file`, a generator) does not load it.
- `paths` takes a YAML list or a comma-separated string, with globs such as `src/**/*.{ts,tsx}`.

> For a procedure used only on some tasks, such as a release, a [skill]({{ '/sections/05-skills-and-plugins/' | relative_url }}) is the better home: it loads only when invoked or matched, where a rule loads on every matching file.

The pattern, its trade-offs and the limits of path rules are on [Hub and spoke and path rules]({{ '/sections/04-context-files/hub-and-spoke-and-path-rules.html' | relative_url }}).

## Write rules that get followed

### Weak and strong rules

| Weak | Strong |
|---|---|
| Be careful with the database. | Never run migrations against production. Use `make migrate-staging`. |
| Write good tests. | Every bug fix starts with a failing test that reproduces the bug. |
| Don't break the build. | Run `make check` before every commit; it must pass. |
| Format code properly. | Use 2-space indentation. |
| Keep files organized. | API handlers live in `src/api/handlers/`. |

- A rule the agent can act on names the action, the condition and the alternative, concretely enough to check.
- Group related rules under headings and bullets; organised sections are followed more reliably than dense paragraphs.
- Two rules that contradict each other get one picked arbitrarily; look for conflicts across the root file, subdirectory files and rules.

### When a mistake repeats

Adding another detailed rule to the context file each time the agent gets something wrong is an anti-pattern: the file grows, and each new line is applied as unevenly as the rest. The fix goes into the system, and the context file gets a short line that explains it.

When to change the harness:

| Signal | What to do |
|---|---|
| You have typed the same correction or the same prompt twice | Consider building it into the harness |
| It happened once, but the context shows it will recur (a step every task needs, a convention the agent cannot see in the code) | Use judgement and fix it the first time |
| It happened once, and a repeat would be costly (a deleted file, a push to the wrong branch, a production command) | Fix it the first time |
| It happened once, in a one-off task | A correction in the conversation is enough |

In each case the fix goes into the system first (workflow, tool, hook, script, linter or test), and the context-file line comes second.

![Sequence: the agent runs the full suite for one test twice; you change make test to run changed tests by default, add a hook that refuses test-all outside a push, and add one explaining line; next week, after a task switch with the window 85% full, the agent tries test-all and the hook refuses it]({{ '/assets/images/04-context-files/correction-to-rule-sequence.svg' | relative_url }})

*A repeated mistake fixed in the system. The workflow change and the hook do the enforcing; the context-file line explains the check. Diagram made with archify; timings are illustrative.*

1. The agent runs the forty-minute end-to-end suite when one test would do, twice in a week.
2. The workflow changes: `make test` runs only the changed tests, and `make test-all` runs the full suite.
3. A PreToolUse hook refuses `make test-all` outside a push and prints the right command.
4. One line in `CLAUDE.md` says so: "`make test` runs the changed tests; the full suite runs at push, and a hook refuses it otherwise."
   - When the agent misses the line, the hook still holds.

[Writing rules that get followed]({{ '/sections/04-context-files/writing-rules.html' | relative_url }}#when-a-mistake-repeats) has the files for each layer.

## A complete example

### The file

A project file for a web service, written for this page, about 30 lines. Every line is one of the kinds above.

```markdown
# Orders service

## Commands
- Install: `make setup`. Start locally: `make run` (port 8080).
- Run one test: `pytest tests/path/test_file.py::test_name -q`.
- Full suite (`make test`, about 12 minutes): only before a push.
- Lint and types: `make check`. It must pass before every commit.

## Conventions
- Python 3.12, ruff formatting. Type hints on every public function.
- Money is integer cents, never floats: floats lose cents in sums.
- Handlers return `Result[T, ApiError]`; never raise from a handler.
- New endpoints get an OpenAPI entry in the same commit.

## Never
- Never edit `generated/`; run `make gen` after changing `schema/`.
- Never run migrations against production. Use `make migrate-staging`.
- Never add a dependency without asking; prefer the standard library.

## Workflow
- Every bug fix starts with a failing test that reproduces it.
- Before a release, read docs/release.md.
- Before writing a migration, read docs/migrations.md.
- Commit messages: `type(scope): summary`, one logical change per commit.

## Known traps
- `tests/integration/` needs `docker compose up db` first; the error says
  "connection refused".
- The `ORDER_TTL` setting is in seconds in config and milliseconds in the cache client.
```

### What each block does

| Block | Kind of line | Why it is there |
|---|---|---|
| Commands | Commands the agent cannot guess | Saves a wrong guess on every task; the times stop it running the slow suite |
| Conventions | Differences from the defaults, with reasons | The reason lets the agent apply the rule to cases it does not name |
| Never | Prohibitions with the alternative | Each says what to do instead; each should also be enforced, for example the migrate script refusing a production target |
| Workflow | Triggers to spokes | The release and migration detail stays out until needed |
| Known traps | Misleading facts about the environment | Each was a real time-waster once; where code can detect one, a setup script or check is the fix and the line explains it |

[Writing rules that get followed]({{ '/sections/04-context-files/writing-rules.html' | relative_url }}) explains how each line was phrased.

## Keep them current

Context files go stale when the mechanisms they describe change, and they grow faster than they are pruned.

1. When you change a mechanism, update the line that describes it in the same commit.
2. Now and then, read the always-loaded file and ask, line by line, whether a recent session needed it.
3. Move detail into spokes; delete lines for problems newer models no longer have.
4. When an agent repeats a mistake, fix it in a workflow, hook or test in the same session, and replace any detailed rule about it with one explaining line.
5. Run `/doctor prompt-audit` in Claude Code: it reports instructions written for older models, references to files or commands that no longer exist, and files that contradict each other, and changes nothing until you ask.

## Context files and auto memory

Claude Code has a second memory, which the agent writes itself.

| | Context files | Auto memory |
|---|---|---|
| Who writes it | You | The agent |
| What it holds | Instructions and rules | Notes of four kinds: about you, your corrections and confirmed approaches, ongoing project facts, and where to find outside references |
| Where | `CLAUDE.md` files and rules | `~/.claude/projects/<project>/memory/`, shared by all worktrees of a repository, local to the machine |
| What loads at start | Each file in full (up to 4 MiB) | The first 200 lines or 25 KB of the `MEMORY.md` index; topic files are read on demand |
| Turn off | | The toggle in `/memory`, `"autoMemoryEnabled": false`, or `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` |

- Asking "remember that the API tests need a local Redis" saves to auto memory; "add this to CLAUDE.md" edits the context file.
- Both are context; neither is enforced. Codex has `/memories` for its own memory.

## Checking what loaded

| Question | How to answer it |
|---|---|
| Which files loaded at launch? | `/context all`, the list under Memory files |
| Did a subdirectory file load? | A `Loaded` line with its path appears in the terminal when it loads |
| Which files and memory locations exist? | `/memory` lists them and opens any one for editing |
| Why did a rule load, and when? | An `InstructionsLoaded` hook can log each file as it loads |
| Did Claude Code read my `AGENTS.md`? | `/memory` lists its path |
| Are any files too long? | Startup warnings, and `/status` |

<!-- IMAGE NEEDED: the Claude Code /context output listing Memory files (CLAUDE.md, rules) with their share of the context window; search: "Claude Code /context command", "Claude Code context window usage" -->

## Examples from my own harness

### Three layers of context

My context has three layers: one file shared by all my projects, one file per repository, and a section for each machine I work on (paths, installed tools, what works and what does not there). Every session on every machine gets the shared rules; each repository adds its own; each machine adds its traps.

| Layer | Holds | Example line (illustrative) |
|---|---|---|
| Shared | How I work everywhere | "Every bug fix starts with a failing test." |
| Repository | This codebase's commands and conventions | "Run the gate script before every push." |
| Machine | This machine's traps | "The system Python is old; use the managed one." |

### A compiled, size-gated shared file

1. Source parts: a base, a part per client, a part per machine.
2. A script joins them and checks the result against a size budget.
   - Over budget: the build fails and nothing is installed.
3. A passing build is written to a new versioned directory and linked into each client's config.
4. A bad edit rolls back by relinking the previous version.

See [Compiled shared context files]({{ '/sections/04-context-files/compiled-shared-context.html' | relative_url }}).

### Procedures as spokes

- Each procedure document is named by one line in the shared file, with the trigger for reading it.
- Path-scoped rules cover tests, documentation and skill authoring.
- The rules that must hold are also enforced by hooks and the test gate; the lines explain them.

| Starting point | When to add the next step |
|---|---|
| One hand-written `CLAUDE.md` per repository | You notice the same lines in every repository |
| A user-level file | You work with two clients or on two machines |
| A compiled shared file | The shared file needs a size budget and a rollback |

## Pages in this section

| Page | What it covers |
|---|---|
| [How context files load]({{ '/sections/04-context-files/how-context-files-load.html' | relative_url }}) | Every level, imports, the Codex walk, both clients on one repository |
| [Context files after compaction]({{ '/sections/04-context-files/after-compaction.html' | relative_url }}) | Sub-topic: what survives, what returns, and how to place a rule so it holds |
| [Writing rules that get followed]({{ '/sections/04-context-files/writing-rules.html' | relative_url }}) | What a context file can and cannot do, what to write, how to phrase it, fixing a repeated mistake in the system, pruning |
| [Hub and spoke and path rules]({{ '/sections/04-context-files/hub-and-spoke-and-path-rules.html' | relative_url }}) | Keeping the always-loaded part small, triggers, path-scoped rules and their limits |
| [Compiled shared context files]({{ '/sections/04-context-files/compiled-shared-context.html' | relative_url }}) | Assembling one file from parts, the size gate, versioned install and rollback |

## Sources

- Claude Code levels, load order, imports, HTML comments, `AGENTS.md` handling, path rules, size limits, auto memory, `/doctor prompt-audit` and troubleshooting: <https://code.claude.com/docs/en/memory.md>, read 2026-11-09.
- Path rules and nested files after compaction: <https://code.claude.com/docs/en/context-window.md>, verified 2026-11-09.
- `/init` and `/memory`: <https://code.claude.com/docs/en/commands.md>, verified 2026-11-09.
- Codex `AGENTS.md`, the global file, overrides and the 32 KiB cap: <https://learn.chatgpt.com/docs/agent-configuration/agents-md>, read 2026-11-09.
- The three layers and the compiled file are from the author's own harness, described conceptually. Example files are illustrations written for this page.

{% include page_nav.html %}
