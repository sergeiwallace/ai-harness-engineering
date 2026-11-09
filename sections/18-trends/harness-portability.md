---
title: "Moving a harness between tools"
parent: "Trends and model changes"
nav_order: 2
last_updated: "2026-11-09"
talk_stop: "13"
---

{% include page_nav.html %}

# Moving a harness between tools

Until recently, a harness built for one coding agent stayed with that agent. Now:

- Skills share a format.
- Context files have converged on a few names.
- Codex can import a Claude Code setup.

Some parts still need rework when they move. This page sorts the parts of a harness by how well they travel, and walks through a move from Claude Code to Codex.

![Sequence diagram of moving a Claude Code setup to Codex. You run /import in Codex, which reads the existing setup and carries the context into AGENTS.md and the skills over in the shared format; you then compare hook events and approval settings, rewrite each hook for the target, run three known tasks and compare the results with Claude Code.]({{ '/assets/images/18-trends/portability-sequence.svg' | relative_url }}){: style="width:100%"}

*Figure 1. A move from Claude Code to Codex: the import carries context and skills, and you check the rest by hand. Diagram made with archify.*

## What travels and what does not

| Part of the harness | Layer | How well it travels | Why |
|---|---|---|---|
| Context file content | 1 | Well | It is plain text; only the file name differs |
| Skills | 1 and 2 | Well | Agent Skills is a shared format built on `SKILL.md` |
| Templates and stubs | 2 | Well | Plain files the agent reads |
| Scripts and linters | 3 | Well | They run outside the agent |
| Tests and merge gates | 4 | Well | They live in the repository and the CI |
| Hooks | 3 | Needs rework | Event names, input format and configuration differ by tool |
| Approval and sandbox settings | 3 | Needs rework | Each tool has its own model of permissions |
| Sub-agent profiles | 2 | Needs rework | Profile files and their fields differ by tool |
| Habits and procedures | All | Well | They live in you and in your documents |

- Parts that live in plain files or run outside the agent move as they are.
- Parts that plug into the client's own event loop need rewriting.

For the [systems approach]({{ '/sections/17-building-a-harness/' | relative_url }}#1-a-systems-approach-to-building-a-harness) this has a practical consequence: put each check in a script or a test where you can, and keep the hook as a thin trigger, so the enforcement survives a change of tool.

## Context files

### One content, two names

Claude Code reads `CLAUDE.md` and Codex reads `AGENTS.md`; the content can be the same. Two common ways to keep one source:

#### Option 1: point one at the other

```markdown
<!-- CLAUDE.md -->
@AGENTS.md

## Claude Code only
- Hooks in this repository run the formatter after each edit.
```

Claude Code reads the `@` line as an import, so it gets the shared rules plus the few lines that apply only to it.

#### Option 2: generate both from one source

Keep the rules in one source file and generate both context files from it. This is more work to set up and pays off when the files differ in more than a few lines.

See [Context files]({{ '/sections/04-context-files/' | relative_url }}) for what goes in either file.

## Skills

A skill is a folder with a `SKILL.md` file:

```markdown
---
name: review-changes
description: Review the uncommitted changes for bugs, missing tests and unclear names.
---

1. Run `git diff` and read every changed file in full.
2. ...
```

Because the format is shared, the same folder works in tools that support Agent Skills. After a move, check:

- Steps that call a tool-specific feature, such as a Claude Code slash command.
- Scripts the skill runs, and whether the target tool's sandbox allows them.
- Hooks the skill relies on, which need rewriting for the target.

## Hooks

### Why hooks need rework

A hook is code that the client runs at a named point in its own loop, such as before a tool call or when the agent stops. Three things differ between clients:

1. The names of those points, and which ones exist.
2. The input the hook receives, such as the JSON fields describing the tool call.
3. How the client reads the result: exit codes, output streams and their meaning.

Codex has hooks too, but a Claude Code hook script and its settings entry do not run there unchanged.

### What to keep

Split each hook into two parts:

1. A plain script that holds the check.
   - It takes a file or a command as its argument.
   - It exits non-zero on failure, with a message that says what to do instead.
   - It lives in the repository, so it moves with the code and can also run in CI.
2. A thin wrapper per client.
   - It reads that client's hook input and passes the file or command to the script.
   - It is the only part you rewrite when you move to another tool.

```bash
#!/usr/bin/env bash
# check-no-force-push.sh: the logic, usable from any client or from CI
cmd="$1"
if [[ "$cmd" =~ git[[:space:]]+push.*(--force|[[:space:]]-f([[:space:]]|$)) ]]; then
  echo "Force push is blocked. Push a new branch and open a pull request instead." >&2
  exit 2
fi
```

> Note: a check that also runs in CI is enforced even when the agent runs in a client with no hooks at all, such as a chat app. See [Desktop, Web & Mobile Apps]({{ '/sections/19-chat-and-desktop-clients/' | relative_url }}).

## A worked move

1. Run `/import` in Codex and let it read the Claude Code setup.
2. Open the resulting `AGENTS.md` and compare it with your `CLAUDE.md`. Remove anything that only applies to Claude Code.
3. List your Claude Code hooks. For each, write down the check it performs, then wire that check into Codex's hook or approval settings.
4. Compare the approval rules. Make sure every action you had set to "ask" or "deny" is covered.
5. Run three tasks you know well and compare the results with Claude Code. Look first at the tasks where a hook used to fire.

## Examples from my own harness

| Part of my harness | Shared or per tool |
|---|---|
| Context-file content | Shared, rendered into each tool's file name |
| Skills, templates and scripts | Shared |
| Hooks, approvals and agent profiles | Per tool, as thin layers |

Because the checks live in plain scripts, most guards move to a new client by adding a wrapper.

To start: take one hook you rely on, move its logic into a plain script, and make the hook call the script.

## Sources

- Codex `/import`: [Codex docs: Developer commands](https://learn.chatgpt.com/docs/developer-commands.md?surface=cli), verified 2026-11-08.
- Codex `AGENTS.md`: [Codex docs: AGENTS.md](https://learn.chatgpt.com/docs/agent-configuration/agents-md), not re-checked for this page.
- Codex hooks: [Codex docs: Hooks](https://learn.chatgpt.com/docs/hooks.md), not re-checked for this page.
- Claude Code memory files and `@` imports: [Claude Code docs: Memory](https://code.claude.com/docs/en/memory.md), not re-checked for this page.

{% include page_nav.html %}
