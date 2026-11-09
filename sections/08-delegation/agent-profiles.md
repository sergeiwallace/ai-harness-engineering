---
title: "Agent profiles"
parent: "Agent profiles, sub-agents and delegation"
nav_order: 1
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Agent profiles

As of 2026-11-09. Topic page in [Agent profiles, sub-agents and delegation]({{ '/sections/08-delegation/' | relative_url }}). Slides: [talk stop 6, "Sub-agents and agent profiles"]({{ '/talk/' | relative_url }}#stop-6-4-minutes).

An agent profile is a file that turns a general agent into a role: it names the role, says when to use it, lists the tools it may call, picks its model, and carries its standing instructions. A brief describes one task. A profile describes the role across every task it is used for. This page covers the fields, how the runtime picks and enforces a profile, how to design a role set, and how to record which profile did which work. Choosing the model for each role has its own sub-page: [model choice]({{ '/sections/08-delegation/model-choice.html' | relative_url }}).

![A sequence across five participants. The lead asks the agent runtime for a reviewer for a diff. The runtime reads the reviewer profile file and gets tools Read, Grep and Glob and model opus, then starts the reviewer with those tools only. The reviewer reads the changed files, then tries to fix a typo; the runtime answers that there is no such tool. The reviewer returns findings with file, line, severity and fix to the lead.]({{ '/assets/images/08-delegation/profile-launch.png' | relative_url }}){: style="width:100%"}

*Figure 1. Pick, load, enforce: a reviewer profile in use. Built with archify.*

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## Where profiles live

| Tool | Location | Format |
|---|---|---|
| Claude Code, one project | `.claude/agents/` in the repository | Markdown with YAML frontmatter |
| Claude Code, every project | `~/.claude/agents/` | Markdown with YAML frontmatter |
| Codex | `.codex/agents/` | TOML |

A project profile is checked into the repository, so everyone who works on it gets the same roles. A user profile follows one person across projects. When two profiles share a name, Claude Code uses the one from the higher-priority location:

| Priority | Location | Scope |
|---|---|---|
| 1 (highest) | Managed settings | Organization-wide |
| 2 | `--agents` command-line flag | Current session |
| 3 | `.claude/agents/` | Current project |
| 4 | `~/.claude/agents/` | All your projects |
| 5 (lowest) | A plugin's `agents/` directory | Where the plugin is enabled |

Only `name` and `description` are required. Claude Code ignores a field it does not recognise without reporting an error, so a misspelled field silently does nothing.

## The fields

Fields fall into four groups:

- Identity: `name`, `description`.
- Capability: `tools`, `disallowedTools`, `permissionMode`, `isolation`.
- Model: `model`, `effort`.
- Other: `maxTurns`, `skills`, `mcpServers`, `hooks`, `memory`, `background`, `color` and more.

Only `name` and `description` are required. An unrecognised field is ignored without an error, so a misspelled field does nothing.

### Identity

#### `name`

- How the role is called, by a person or by another agent: "start a `reviewer`".
- Short, a noun, unique across the locations in the table above.

```yaml
name: reviewer
```

#### `description`

- When to use this role.
- The runtime reads every profile's description and matches it against the task.
  - So this field decides which role starts when nobody names one.

| Weak description | Strong description |
|---|---|
| "Helpful reviewer agent" | "Reviews a diff for bugs and missing tests. Read-only. Use after any code change, before merge." |
| "Does research" | "Answers one factual question from docs and code, with sources. Read-only. Not for design decisions." |

### Capability

These fields set what the agent can do.

| Field | Sets | Example |
|---|---|---|
| `tools` | The tools the agent may call | `Read, Grep, Glob` |
| `disallowedTools` | Tools removed from the inherited or listed set | `Write, Edit` |
| `permissionMode` | How approval prompts are handled | `acceptEdits` |
| `isolation` | Whether it runs in its own git worktree | `worktree` |

#### `tools`

- The tools the role may call.
- A tool not listed is unavailable to that agent, so "please do not edit" becomes "cannot edit".
- The runtime enforces it, which makes it the right place for a limit that matters for safety.

```yaml
tools: Read, Grep, Glob
```

#### `disallowedTools`

- Tools removed from the inherited or listed set, in the same format as `tools`.
- Useful when a role needs almost every tool but must never use a few.

```yaml
# inherits every tool except file writes
disallowedTools: Write, Edit
```

#### `permissionMode`

How the role handles actions that would normally ask for approval.

| Value | Typical use |
|---|---|
| `default` | Ask as the session would |
| `acceptEdits` | A writing role in its own worktree |
| `plan` | A role that plans and does not act |
| `dontAsk`, `auto`, `bypassPermissions` | Further documented modes; see the sub-agents docs before using them |

#### `isolation`

- `worktree` runs the agent in a temporary git worktree.
- The worktree is branched by default from the repository's default branch, so its edits never touch the lead's checkout.
- A worktree in which the agent made no changes is removed automatically.
- See [worktree isolation]({{ '/sections/08-delegation/worktree-isolation.html' | relative_url }}).

### Model

Two fields set how the agent thinks: which model, and how much reasoning effort it spends.

| Field | Values | Example |
|---|---|---|
| `model` | An alias, a full model ID, or `inherit` | `opus` |
| `effort` | A reasoning effort level | `high` |

#### `model`

- An alias: `opus`, `sonnet`, `haiku` or `fable`.
- A full model ID.
- `inherit`, to use the lead's model.
- Omitted: Claude Code picks by its own model order.

#### `effort`

- The reasoning effort for the role, set separately from the model.
- A mid-tier model on a hard bug can run at high effort; a strong model summarising short reports can run at low effort.
- See [model choice]({{ '/sections/08-delegation/model-choice.html' | relative_url }}#effort-is-separate-from-tier).

### Other fields

| Field | What it sets |
|---|---|
| `maxTurns` | A cap on how many turns the agent may take |
| `skills` | Skills the agent can use |
| `mcpServers` | MCP servers available to the agent |
| `hooks` | Hooks that apply while the agent runs |
| `memory` | Memory the agent reads and keeps |
| `background` | Whether the agent runs in the background |
| `color` | Its colour in the interface |

The full list, with exact semantics, is in the sub-agents documentation linked under Sources.

### Body

The Markdown under the frontmatter is the role's standing instructions. It is the same for every launch.

| Goes in the body | Goes in the brief |
|---|---|
| The output format the role always uses | What to work on this time |
| What the role always ignores, such as style the linter enforces | Which files to start from |
| What it always checks before reporting | The length cap for this answer |

> Note: the body is the instruction layer. A limit that matters for safety is more reliable in `tools`, where the runtime enforces it.

## Worked example: two profiles side by side

<details markdown="1" open>
<summary>reviewer.md</summary>

```markdown
---
name: reviewer
description: Reviews a diff for bugs and missing tests. Read-only. Use after any code change, before merge.
tools: Read, Grep, Glob
model: opus
---
Review the diff you are given. Report each finding with file, line, severity
and a one-line fix. Do not suggest style changes the linter already enforces.
If you find nothing, say so and list what you checked.
```

</details>

<details markdown="1" open>
<summary>engineer.md</summary>

```markdown
---
name: engineer
description: Implements a change described in a brief, with tests, in its own worktree.
tools: Read, Grep, Glob, Edit, Write, Bash
model: sonnet
isolation: worktree
---
Write the failing test first. Commit each logical step. Report the commit ids
and the exact test command with its last line of output.
```

</details>

| | reviewer | engineer |
|---|---|---|
| Can edit files | No | Yes |
| Runs in its own worktree | Not needed | Yes |
| Model | Strongest | Mid |
| Output | Findings table | Commits and a test line |

## How the runtime uses a profile

### Picking

```text
lead> Review the change on branch agent-a/fix-parser before I merge it.
runtime: matching task against profile descriptions...
runtime: starting "reviewer" (Reviews a diff for bugs and missing tests. Read-only.)
```

The lead can name a profile, or the runtime picks one from the descriptions. In the second case, the choice depends on how each description is written.

### Loading

1. The runtime reads the profile's frontmatter.
   - `tools`, `model`, `permissionMode` and `isolation` set up the launch.
2. It starts a fresh context with the profile's body as standing instructions.
3. It adds the brief as the task.
4. It exposes only the listed tools to the new agent.

### Enforcing

A tool not in the list is not available to call, so figure 1's reviewer gets "no such tool" when it tries to edit. No sentence in a brief can add a tool to a profile.

> Important: when a brief and a profile disagree about tools, the profile applies. A tool missing from the profile is unavailable whatever the brief says, so safety limits are safer in the profile.

## Designing a role set

### Start from what goes wrong

Each role exists to prevent a failure. One way to design a role set is around those failures.

| Failure | Role that prevents it | How |
|---|---|---|
| A reviewer edits the code and hides its own findings | Read-only reviewer | No write tools |
| A researcher changes files nobody asked to change | Read-only researcher | No write tools |
| Two writers collide in one checkout | Engineer | `isolation: worktree` |
| An agent settles a design question it should not | Architect on the strongest model | Decisions go to that role |

### An example role set

| Role | Purpose | Write access | May start sub-agents |
|---|---|---|---|
| Researcher | Answer one question with sources | No | Search helpers, one level |
| Analyst | Extract and summarise data | No | No |
| Planner | Split work into tasks | Docs only | No |
| Architect | Design and decide | Docs only | Researchers |
| Engineer | Implement a brief | Yes, in a worktree | No |
| Reviewer | Find bugs in a diff | No | No |
| Auditor | Check a plan or result against its sources | Its report only | No |

### Keep authority and tools in agreement

A profile can say one thing in its body and allow another in its tools. A build check catches that.

1. Read every profile's description, body and tool list.
2. For each role described as read-only, fail if any write tool is listed.
3. For each role allowed to start sub-agents, fail if the tool that starts them is missing.
4. For each role not allowed to, fail if that tool is present.

```text
profile check: reviewer.md says "Read-only" but tools include Edit
FAIL: 1 profile disagrees with its stated authority
```

## Recording who did what

When a change goes wrong, the first question is which agent wrote it and under which brief. Commit trailers answer it.

```text
fix(parser): accept leap-day dates

Agent-Family: claude
Agent-Role: engineer
Agent-Model: <model id>
Agent-Effort: high
Agent-Worktree: <worktree name>
```

The trailer names are illustrative. Any consistent set works, as long as every delegated commit carries one.

## Pitfalls

| Pitfall | Effect | Fix |
|---|---|---|
| Vague description | The wrong role is started | Say what it does, what it does not, and when to use it |
| Limits in the brief only | A brief can be written without them | Put safety limits in `tools` |
| One giant profile | Every agent can do everything | Split by the failure each role prevents |
| Body repeats the brief | Two sources disagree | Body for the role, brief for the task |

## Examples from my own harness

- Profiles by role: analyst, architect, auditor, engineer, planner, researcher and reviewer.
  - Read-only roles have no write tools.
- Each profile states whether that role may start sub-agents.
  - A check fails the build if a profile's stated authority and its tools disagree.
- Each delegated commit carries trailers naming the family, role, model, effort, profile and worktree.

## Related pages

- [Model choice]({{ '/sections/08-delegation/model-choice.html' | relative_url }}), the `model` field in depth.
- [Sub-agent briefs]({{ '/sections/07-templates-and-stubs/briefs.html' | relative_url }}), the per-task half of the pair.
- [Worktree isolation]({{ '/sections/08-delegation/worktree-isolation.html' | relative_url }}), what `isolation: worktree` does.

## Sources

- Sub-agent profiles, locations, fields and `isolation`: <https://code.claude.com/docs/en/sub-agents.md>, verified 2026-11-09.
- Codex sub-agents: <https://learn.chatgpt.com/docs/agent-configuration/subagents.md>, verified 2026-11-08.
- The role set, the authority check and the trailers are from the author's own harness, described conceptually. The other examples are illustrations written for this page.

{% include page_nav.html %}
