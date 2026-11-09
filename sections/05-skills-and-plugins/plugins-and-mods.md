---
title: "Plugins, marketplaces and mods"
parent: "Skills, commands and plugins"
nav_order: 3
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Plugins, marketplaces and mods

A plugin packages skills, sub-agent profiles, hooks and MCP servers that are meant to be used together, so they install as one unit at one version. This page covers the plugin layout and manifest, marketplaces, what a plugin costs in context, mods, and a pattern for sharing code across the skills in a plugin. It is part of [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What a plugin is

The docs define a plugin as "a directory of skills, agents, hooks, MCP servers, or other components that Claude Code installs and loads as one unit", and say to use one "when you want several skills, subagents, hooks, or MCP servers packaged as one unit". A plugin can hold:

- Skills, in `skills/`.
- Sub-agent profiles, in `agents/`.
- Hook registrations, in `hooks/hooks.json`.
- MCP servers, in `.mcp.json`.

Because the hook ships with the skill, a plugin is also how a team ships the enforcement for a procedure along with the procedure: install the review skill and the lint hook arrives with it.

### When to make one

| Situation | Plugin? | Why |
|---|---|---|
| One skill for one repository | No | A project skill in `.claude/skills/` is enough |
| A skill plus the hook that guards it | Yes | They must arrive together |
| The same parts in many repositories | Yes | One install, one version, everywhere |
| Parts for a whole team | Yes, in a marketplace | New people install once |

## The layout

A plugin is a folder with a manifest in `.claude-plugin/` and its parts beside it:

```text
review-kit/
├── .claude-plugin/
│   └── plugin.json            # manifest
├── skills/
│   └── review/SKILL.md
├── agents/
│   └── security-reviewer.md   # sub-agent profile
├── hooks/
│   └── hooks.json             # hook registrations
└── .mcp.json                  # MCP servers the plugin starts
```

### The manifest

```json
{
  "name": "review-kit",
  "version": "1.4.0",
  "description": "Code review skill, a security reviewer sub-agent and a lint-before-commit hook.",
  "author": { "name": "Platform team" }
}
```

- `name` is the field the manifest must have; the other fields describe the plugin.
- Raise the version on every change you ship, so people can tell which copy they have.
- Only `plugin.json` goes inside `.claude-plugin/`. The `skills/`, `agents/` and `hooks/` folders sit at the plugin's root; parts placed inside `.claude-plugin/` are not found.

### How the parts load

![Sequence diagram with four lanes: New team member, Client, Marketplace repo and Context window. The person adds the marketplace example-org/platform-tools; the client reads marketplace.json and finds two plugins. The person installs review-kit; the client fetches version 1.4.0 with one skill, one agent profile and one hook, and says to run /reload-plugins. From then on the client puts the review skill and reviewer descriptions into the context window on every turn, and the lint hook runs before each commit as code.]({{ '/assets/images/05-skills-and-plugins/plugin-install-sequence.png' | relative_url }})
*Installing a plugin from a team marketplace, and what it costs on every turn afterwards.*

## Marketplaces

A marketplace is a catalogue of plugins that people add once and install from.

- A git repository with a marketplace file listing the plugins is enough.
- A team can host its own, next to its code.
- A plugin's version in the marketplace is what everyone installs, so a release is one change in one place.

### Worked example: a team marketplace

#### The catalogue

The marketplace file is `.claude-plugin/marketplace.json` at the root of the marketplace repository:

```json
{
  "name": "platform-tools",
  "owner": { "name": "Platform team" },
  "plugins": [
    { "name": "review-kit", "source": "./plugins/review-kit" },
    { "name": "release-kit", "source": "./plugins/release-kit" }
  ]
}
```

#### A new team member's session

```text
> /plugin marketplace add example-org/platform-tools
  Added marketplace platform-tools (2 plugins).

> /plugin install review-kit@platform-tools
  Installed review-kit 1.4.0: 1 skill, 1 agent, 1 hook.

> /reload-plugins
  Reloaded 1 plugin.
```

After these commands the new person has the same review skill, reviewer profile and lint hook, at the same version, as the rest of the team.

### Managing plugins

| Command | Does |
|---|---|
| `/plugin` | Lists, installs, enables and disables plugins |
| `/reload-plugins` | Applies plugin changes without a restart |
| `/skill-doctor` | Shows what each skill costs in context and how often it is used |

Where a plugin loads:

| Surface | Loads |
|---|---|
| Claude Code on your machine | Every enabled plugin and all its parts |
| A cloud session | Not your local plugins |
| claude.ai and Cowork | The same plugin format, with a different set of components |

## The context cost

Every enabled plugin costs context on every turn:

- Each model-invocable skill and agent keeps its name and description loaded.
- Each MCP tool adds its name and schema.
- A hook that injects text adds it on every event it fires on.

### Worked example: an audit

| Plugin | Skills | Agents | MCP tools | Used last month | Action |
|---|---|---|---|---|---|
| review-kit | 1 | 1 | 0 | Daily | Keep |
| release-kit | 3 | 0 | 0 | Weekly | Keep |
| docs-helper | 6 | 2 | 12 | Never | Disable |
| old-linter | 2 | 0 | 0 | Never | Uninstall |

Disabling the two unused plugins removes eight skill descriptions, two agent descriptions and twelve tool schemas from every turn.

## Mods

A mod is a plugin whose hooks module registers function hooks: JavaScript or TypeScript functions that Claude Code loads into its own process and calls on events such as tool calls, prompts and turns.

- A function hook can watch an event, change it, or answer it in Claude Code's place, such as refusing a command or adding context to a prompt.
- Changing the interface, with a pane, a band above the prompt, a toast or a new slash command, is one common use; a mod that only guards tool calls draws nothing.
- Mods need Claude Code v2.1.287 or later in the terminal.

[Mods and function hooks]({{ '/sections/05-skills-and-plugins/mods-and-function-hooks.html' | relative_url }}) covers the event contract, the comparison with shell hooks, building and debugging a mod, and the context indicator from my harness.

## Shared code across skills

### The problem

Skills often include small command-line tools. Written one per skill, they diverge:

- Each parses options its own way.
- Each loads settings its own way.
- Each has its own test helpers.
- A bug fixed in one stays in the others.

### Plugin modules

One pattern I use is a plugin whose skills, MCP servers and hooks all import one shared library, so the code is written once. I call this a plugin module. That is my own name; Claude Code has no such term. The nearest vendor term is the hooks module above.

![Sequence diagram with five lanes: Developer, Shared library, Skill A tool, Skill B tool and MCP server. Two skill tools call settings.load and both get a wrong path on a second profile. The developer fixes the profile lookup in the library once and adds a test, which passes. After the next install, Skill A, Skill B and the MCP server each call settings.load and get the correct path, with no change to any of them.]({{ '/assets/images/05-skills-and-plugins/shared-library-fix-sequence.png' | relative_url }})
*A settings bug fixed once in the shared library. Every tool, hook and server that imports it gets the fix on the next install.*

#### Worked example: a skill tool on the shared library

```python
# A skill's tool: only its own logic, everything else from the library
from toolkit import entry, options, settings

@entry.command(help="List open issues older than N days.")
@options.flag("--days", type=int, default=14)
def main(days):
    cfg = settings.load()                 # same settings file as every tool
    for issue in cfg.tracker.open_issues(older_than=days):
        print(issue.id, issue.age_days, issue.title)
```

- The tool has seven lines of its own.
- Option parsing, settings, error output and exit codes come from the library, so every tool behaves the same.
- A fix in the library reaches every tool on the next install.
- The library's test helpers give every tool the same way to run against a temporary settings file.

<details markdown="1">
<summary>A core plugin for two clients</summary>

In my harness one core plugin serves the same read-only tools to Claude Code and to Codex through an MCP server, so both clients query the same knowledge store the same way. The server is built on the same shared library as the skill tools. A tool added to the server is available in both clients at once.

</details>

## Pitfalls

- A plugin that bundles everything costs context in every session that enables it.
- A plugin version that never changes leaves people unsure which copy they run.
- Copying a helper into each skill multiplies every bug in it.
- A plugin's hooks and injected commands run with your permissions. Read a plugin from outside your team before installing it, as you would a script.

## Sources

- Plugins, marketplaces, the hooks module and mods: [Claude Code plugins overview](https://code.claude.com/docs/en/plugins/overview.md) and [Claude Code mods overview](https://code.claude.com/docs/en/plugins/mods/overview.md), verified 2026-11-08.
- `/plugin`, `/reload-plugins`, `/skill-doctor`: [Claude Code commands reference](https://code.claude.com/docs/en/commands.md), verified 2026-11-09.
- The shared library, core plugin and context pane are from the author's own harness, described conceptually. The example plugin, marketplace, audit and tool code are illustrations written for this page.

{% include page_nav.html %}
