---
title: "Mods and function hooks"
parent: "Skills, commands and plugins"
nav_order: 4
last_updated: "2026-11-09"
talk_stop: "4"
stop_minutes: "1.5"
checked_against: "Claude Code mods documentation, read 2026-11-09 (the mods reference covers the CLI and Desktop app as of v2.1.290)"
---

{% include page_nav.html %}

# Mods and function hooks

A mod is a plugin whose hooks module registers function hooks: JavaScript or TypeScript functions that Claude Code loads into its own process and calls when an event happens.

- Events include a tool call, a submitted prompt, the end of a turn, and part of the interface being drawn.
- A function hook can watch the event, change it, or answer it in Claude Code's place.
- Changing the interface, with panes, a band above the prompt, toasts or new slash commands, is one common use; a mod that only guards tool calls or adds context to prompts draws nothing.

This page is part of [Skills, commands and plugins]({{ '/sections/05-skills-and-plugins/' | relative_url }}), next to [Plugins, marketplaces and mods]({{ '/sections/05-skills-and-plugins/plugins-and-mods.html' | relative_url }}), and it complements the shell-based [hooks in section 6]({{ '/sections/06-hooks-scripts-and-linters/hooks.html' | relative_url }}).

## Contents
{: .no_toc .text-delta }

1. TOC
{:toc}

## What a mod is

### The parts

| Part | What it is |
|---|---|
| Plugin | A mod is an ordinary plugin with a manifest at `.claude-plugin/plugin.json`. Mods add no required manifest fields |
| `hooks/hooks.json` | Its `modules` key points to one hooks module, as in `"modules": ["./register.js"]`. That key is what makes the plugin a mod. The same file can also hold ordinary settings hooks under `hooks` |
| Hooks module | An ES module, written as `.js`, `.mjs`, `.cjs`, `.jsx`, `.ts`, `.mts`, `.cts` or `.tsx`, that exports `register(on, options)`. No build step: Claude Code loads `.js` and `.ts` files directly |
| Function hook | Each call to `on(event, [matcher], handler)` inside `register` registers one hook |
| Mods API | The `$` argument every hook receives. A hook reaches outside its own code only through it: `$.ui`, `$.command`, `$.tool`, `$.session`, `$.fs`, `$.process`, `$.http`, `$.model` and other namespaces |

Claude Code's documentation uses "hook" for a mod's handler and "settings hook" for the shell command, HTTP request, prompt or agent hooks configured in a settings file. During early access the mod kind was switched on with an environment variable named `CLAUDE_CODE_ENABLE_FUNCTION_HOOKS`; v2.1.287 and later ignore it.

### Versions and where mods run

| Where | Minimum version | Hooks run | What the mod draws appears |
|---|---|---|---|
| `claude` in a terminal | v2.1.287 | Yes | Yes |
| Code tab of the Desktop app | v2.1.286 | Yes | Yes, except terminal-only elements |
| VS Code extension chat panel | Extension 2.1.299 for drawing | Yes | Yes |
| `claude -p` and the Agent SDK | | Yes | No |
| A cloud session | | Yes, for a plugin that reaches it | No |
| A WSL session in the Desktop app | | No | No |

Some of Claude Code's own features are built-in mods, which shows the range: `cc-plugin-agents-md` loads `AGENTS.md` as project instructions and draws nothing, while `cc-plugin-diff` draws the `/diff` pane.

### The hook function

```javascript
on('tool.call', { tool: 'Bash' }, async ($, e, next) => {
  // $    the mods API
  // e    the event's input, deeply frozen; e.tool, e.command for Bash
  // next the rest of the chain: other mods' hooks, then Claude Code itself
  return next(e)
})
```

What the handler does with `next` decides what kind of hook it is:

1. Observe: do some work and `return next(e)`, or `await next(e)` first to act after the event.
2. Rewrite: call `next` with a changed copy, as in `next({ ...e, text: e.text.trim() })`. Assigning to `e` throws.
3. Answer: return a result without calling `next`, such as `{ deny: 'reason' }`, so later mods and Claude Code's own behaviour do not run.

The matcher, the optional second argument, is an object compared with the event's fields: a value, an array of allowed values, or a regular expression, as in `{ tool: /^mcp__github__/ }`.

### The events

| Family | Events | What a hook can return |
|---|---|---|
| Tools | `tool.call`, `tool.check`, `tool.describe` | `next(e)`, `{ deny }` or `{ result }` on `tool.call`; `{ decision: 'allow' \| 'ask' \| 'deny' }` on `tool.check`; a new `{ description }` on `tool.describe` |
| Prompts | `prompt.submit`, `prompt.section`, `prompt.context`, `prompt.attachment`, `prompt.mention`, `skill.prompt` and others | Rewritten `text`, added `context`, `{ drop }` to stop a prompt, replaced or omitted system prompt sections |
| Commands and configuration | `command.run`, `command.describe`, `config.set`, `settings.read` | Command output `{ text }`, changed values, `{ deny }` |
| Turns | `turn.start`, `turn.step`, `turn.complete` | A different `model` for one request; `{ text }` shown under the answer |
| Session | `session.start`, `session.end`, `session.compact`, `session.measure`, `session.send`, `session.receive` and others | `{ skip }` to stop a compaction; delivery decisions for messages between sessions |
| Subagents | `agent.offer`, `agent.spawn` | `{ isOffered: false }`, a different `model`, `{ deny }` |
| Interface | `ui.render`, `ui.press`, `ui.input`, `ui.select`, `ui.close` and others | An element tree for a pane or band, or a changed Claude Code element |
| Settings hook events | `classic.<Event>`, such as `classic.Stop` | `e` is the JSON a settings hook would read on standard input |

The full list, with every field, is in the [mods reference](#references); the types file Claude Code writes beside a mod in development is the version-exact source.

## Function hooks and shell hooks

| | Function hook (mod) | Command hook | HTTP hook | Prompt hook | Agent hook |
|---|---|---|---|---|---|
| Process model | A function inside Claude Code's process | A new process for each event | A request to a URL | A model call | A sub-agent with tools |
| Latency | No process start per event (no published figure) | Process start on every event | Network round trip | Model latency | Slowest of the five |
| State between events | Module variables shared by the mod's hooks; `$.store` shared across sessions; variables reset on reload | None, unless the script writes files | Whatever the server keeps | None | None |
| Can change | Tool calls, prompts, commands, turns, settings, what the interface draws | Whether a call or prompt goes ahead, a tool call's input and result, added context | The same as a command hook | The same, decided by a model | The same, decided by an agent |
| Can draw | Yes | No | No | No | No |
| Failure | Throw, timeout or wrong shape before `next`: skipped, the event goes on (fails open) unless a `.catch` handler answers | Exit 2 blocks; another non-zero exit shows an error and the action proceeds | See the hooks reference | See the hooks reference | See the hooks reference |
| Time limit | 10 seconds of the hook's own execution, 50 ms for `prompt.edit`; 1 second for a `.catch` handler | The `timeout` you register | The `timeout` you register | The `timeout` you register | The `timeout` you register |
| Written in | JavaScript or TypeScript | Any language | Any server | A prompt | A prompt and tools |
| Trust | Runs with your permissions, not sandboxed; sees every prompt and tool call | Runs with your permissions | Sends event data to the server | Sends event data to a model | Runs tools |
| Use it for | Interface, rewriting events, state across events, commands that run instantly | A guard or check with a script you already have | A central policy service | A judgment a pattern cannot make | A check that must read files |

Where the two kinds meet in one tool call:

1. `PreToolUse` hooks from managed settings run first. If one blocks, no mod's `tool.call` hook runs.
2. Mods' `tool.call` hooks run, in order: the built-in guard and organisation mods, then mods you installed, then `appendPlugins` mods, then other built-in mods.
3. When the last mod calls `next`, the other `PreToolUse` hooks, from user and project settings and plugins, run as part of Claude Code's own behaviour.
4. `tool.check` decides allow, ask or deny, before any permission prompt.

A mod that answers `tool.call` without calling `next` keeps the step 3 hooks from running.

## Mods that change behaviour

A mod with no interface at all can guard, rewrite and inform. These are the same jobs a [guard]({{ '/sections/06-hooks-scripts-and-linters/writing-a-guard.html' | relative_url }}) does in a shell script, with access to state and the mods API.

| Job | Event | Return |
|---|---|---|
| Refuse a command | `tool.call` | `{ deny: 'reason Claude can act on' }` |
| Change a command's input | `tool.call` | `next({ ...e, command: newCommand })` |
| Answer a tool call without running it | `tool.call` | `{ result }` in the tool's own result shape |
| Decide permission from live state | `tool.check` | `{ decision: 'deny', reason }` |
| Add context to a prompt | `prompt.submit` | `next({ ...e, context: [...(e.context ?? []), line] })` |
| Stop a prompt | `prompt.submit` | `{ drop: 'reason for the person' }` |
| Log without starting a turn | any | `$.ui.log('text')`, then `next(e)` |

### Worked example: a guard that fails closed

```javascript
// hooks/register.js
const FORCE = /\bgit\s+push\b.*(--force\b|\s-f\b)/

async function guard($, e, next) {
  if (FORCE.test(e.command)) {
    return { deny: 'Force pushes are refused. Push to a new branch and open a pull request.' }
  }
  // Run git log without a pager, so its output comes straight back to the agent
  if (/^git log\b/.test(e.command)) {
    return next({ ...e, command: e.command.replace(/^git log/, 'git --no-pager log') })
  }
  return next(e)
}

export function register(on) {
  on('tool.call', { tool: 'Bash' }, guard).catch(async ($, e, next) => {
    // The guard threw or timed out; refuse unless it had already let the call through
    if (next.called) return next(e)
    return { deny: 'The command guard failed, so the command was not run: ' + next.error.kind }
  })

  on('prompt.submit', async ($, e, next) => {
    if (!/\bPR\b|pull request/i.test(e.text)) return next(e)
    const git = await $.process.run(['git', 'branch', '--show-current'])
    if (git.exitCode !== 0) return next(e)
    return next({ ...e, context: [...(e.context ?? []), 'Current branch: ' + git.stdout.trim()] })
  })
}
```

The example does three things:

1. It refuses a force push with a message that names the allowed form, so Claude's next call is a push to a new branch.
2. It rewrites `git log` to `git --no-pager log` before the command runs.
3. When a prompt mentions a pull request, it adds the current branch as context only Claude reads.

Without the `.catch` handler, a guard that throws lets the force push through, because a failed hook is skipped. `claude plugin validate` prints `gating hook without .catch: tool.call` for a hook in that state.

## Mods that change the interface

| What a mod can draw or change | How |
|---|---|
| A pane beside the transcript | `$.ui.open` and a `ui.render` hook for the `Pane` site |
| A band above the prompt | A `ui.render` hook for the `AbovePrompt` site |
| Claude Code's own elements: the spinner, tool call rows, the question dialog, the hint line | A `ui.render` hook on that site that rewrites its props or replaces it |
| Buttons, text fields, selects, Markdown, code, images, raster grids | Elements returned in the tree; `ui.press`, `ui.input` and `ui.select` hooks handle use |
| Toasts, notices, status and transcript lines | `$.ui.toast`, `$.ui.notice`, `$.ui.status`, `$.ui.log` |
| A slash command that runs at once, with no Claude turn | `$.command.register` at `session.start`, and a `command.run` hook |
| A line under each answer | `turn.complete` returning `{ text }` |

A mod cannot restyle the permission prompt. A custom status line is a separate, older feature configured in settings (see the [status line documentation](#references)); a mod can show similar information from inside the client.

### Worked example: a context indicator

In my harness a mod shows how full the context window is, after every answer, so the person decides when to run `/compact` from a measured number. A minimal version of the same idea:

```javascript
// hooks/register.js: show context use under each answer and add /context-now
async function usageLine($) {
  const { context } = await $.session.usage()
  return 'context ' + Math.round(context.percent) + '% (' + context.tokens + ' of ' + context.window + ' tokens)'
}

export function register(on) {
  on('session.start', async ($, e, next) => {
    await $.command.register({ name: 'context-now', description: 'Show how full the context window is' })
    return next(e)
  })

  on('turn.complete', async ($, e, next) => {
    // A subagent's turn also fires this event; show the line for the main conversation only
    if (e.agentId) return next(e)
    return { text: await usageLine($) }
  })

  on('command.run', { command: 'context-now' }, async ($) => ({ text: await usageLine($) }))
}
```

1. `session.start` registers `/context-now`, which runs instantly, even while Claude is working.
2. `turn.complete` returns a line that appears under each main-conversation answer.
3. Both read `$.session.usage().context`, which holds `tokens`, `window` and `percent`.

The reference does not say whether `percent` is a fraction or a number from 0 to 100; the example assumes 0 to 100. Check the types file for your build.

<!-- IMAGE NEEDED: a Claude Code terminal session with a mod-drawn pane or a line under the answer showing context-window usage; search: claude code mods pane, claude code mod context usage, claude code token-weather mod -->

The sample `token-weather` mod in Anthropic's mods playground draws a forecast of the context window above the prompt; its source shows the band and element API in full.

## Building, installing and debugging

### Folder layout

```text
context-indicator/
├── .claude-plugin/
│   └── plugin.json          # name, version, description, author
├── hooks/
│   ├── hooks.json           # { "modules": ["./register.js"] }
│   └── register.js          # the hooks module
└── tests/
    └── context-indicator.test.ts
```

```json
{
  "name": "context-indicator",
  "version": "0.1.0",
  "description": "Shows context-window use under each answer and adds /context-now",
  "author": { "name": "Your Name" }
}
```

### The development loop

| Step | Command | What it does |
|---|---|---|
| 1. Load for one session | `claude --plugin-dir ./context-indicator` | Loads the directory without installing it and reloads the hooks module when a file is saved |
| 2. Confirm it loaded | `/plugin` | A dim line under the tabs reads `1 mod active · context-indicator` |
| 3. Check what Claude Code reads | `claude plugin validate ./context-indicator` | Lists `hooks:` (events and matchers) and `calls:` (mods API methods), and errors such as a misspelled event |
| 4. Run tests | `claude plugin test` | Runs `*.test.ts` files with no session, sign-in or network |
| 5. Reload an installed copy | `/reload-plugins` | Picks up a plugin installed or updated from the shell |
| 6. Share it | `/plugin install context-indicator@your-marketplace` | Installs from a marketplace; raise the version for each release, because installed copies are cached by version |

Claude Code also writes TypeScript declarations into the mod's `.claude-plugin/types/` while it is loaded with `--plugin-dir`. They describe the events, methods and elements of the running version and take precedence over any page, this one included.

### Rules static analysis enforces

- Write each mods API call in full, as `$.store.get('notes')`; assigning `$` or a namespace to a variable fails validation.
- Write each event name as a string literal in the `on` call.
- Use top-level `import` declarations of files inside the plugin, or `claude-code` for types; no `require` and no dynamic `import()`.
- Register an event once per matcher; two `session.start` hooks with no matcher fail to load.

### A test

```typescript
// tests/guard.test.ts
import { expect, test } from 'claude-code/testing'

test('force push is refused', async ($, on) => {
  on('tool.call', () => ({ result: 'ok' }))   // stand in for the real tool
  const answer = await $.tool.call({ tool: 'Bash', command: 'git push --force origin main' })
  expect(answer.deny).toContain('Force pushes are refused')
})
```

The shape of `answer` for a denied call is inferred from the events reference; check it against the types for your build.

### When a mod does nothing

| Symptom | Check |
|---|---|
| Not named in `/plugin` | The `modules` path in `hooks/hooks.json`; whether mods are disabled by `--safe-mode`, `--bare`, `disableAllHooks` or managed settings |
| One hook never runs | `claude plugin validate` and its `hooks:` line; a misspelled event is reported as `"tool.calls" is not an event` |
| A hook is skipped | The transcript or debug log line `<mod>: tool.call hook skipped: threw ...` |
| Count resets | A reload runs `register` again; keep values in `$.store` |

## Trust and security

A mod is code that runs with your permissions inside Claude Code.

- It can read and write your files, start processes, make network requests, read environment variables and settings, see every prompt and tool call, rewrite them, and call a model on your plan.
- Mods are not sandboxed. Sandboxing isolates the Bash commands Claude runs; a process a mod starts runs outside it.
- With managed settings or a Team or Enterprise login, a mod's approval does not get a call past your `ask` rules or `PreToolUse` hooks by default. Without them, a mod can approve a call an `ask` rule would prompt for.
- Before installing one, run `claude plugin validate` on its directory and read the `hooks:` and `calls:` lines.
- Administrators control mods with managed settings such as `allowManagedModsOnly`, `prependPlugins` and `appendPlugins`.

## Pitfalls

- A guard with no `.catch` handler fails open: a throw or a timeout lets the call through.
- Time awaiting your own promise counts against the 10-second limit; time inside a mods API call such as `$.ui.ask` does not.
- Module variables reset on every reload. Keep anything that must survive in `$.store`, which is shared by every session on the machine and limited to 4 MiB of JSON.
- `$.fs.write` is not atomic; another process can read a partly written file.
- Text a `prompt.section`, `prompt.context` or `skill.prompt` hook changes between requests invalidates the prompt cache.
- A drawing mod shows nothing in `claude -p`, the Agent SDK or a cloud session; fall back to a transcript line or a command's text there.
- Editing an installed copy does nothing until its version is raised and it is installed again; develop against the directory with `--plugin-dir`.

## Related

- [Plugins, marketplaces and mods]({{ '/sections/05-skills-and-plugins/plugins-and-mods.html' | relative_url }}): packaging and distributing the plugin a mod lives in.
- [Hooks]({{ '/sections/06-hooks-scripts-and-linters/hooks.html' | relative_url }}): settings hooks, events and exit codes.
- [MCP Apps]({{ '/sections/19-chat-and-desktop-clients/mcp-apps.html' | relative_url }}): interactive UI in Claude web and desktop, the chat-app counterpart to interface mods.

## References

- [Mods overview](https://code.claude.com/docs/en/plugins/mods/overview.md), Claude Code documentation: definition, where mods run, versions, trust, built-in mods, comparison with settings hooks, skills and MCP servers. Read 2026-11-09.
- [Create a mod](https://code.claude.com/docs/en/plugins/mods/create.md): folder layout, manifest, `hooks.json`, the development loop, types, validation rules and tests. Read 2026-11-09.
- [Mods reference](https://code.claude.com/docs/en/plugins/mods/reference.md): the hook function, every event and return shape, mods API methods, render sites, elements and limits. Read 2026-11-09.
- [React to events with a mod](https://code.claude.com/docs/en/plugins/mods/events.md): observe, rewrite and answer; matchers; ordering with settings hooks; failure and `.catch`. Read 2026-11-09.
- [Draw in the interface with a mod](https://code.claude.com/docs/en/plugins/mods/interface.md): panes, the band, elements and state. Not re-read for this page.
- [Claude Code hooks reference](https://code.claude.com/docs/en/hooks.md): settings hook handler types, exit codes and timeouts.
- [Customize your status line](https://code.claude.com/docs/en/statusline.md): the settings-based status line.
- [Mods playground](https://github.com/anthropics/claude-code-playground/tree/main/claude-code/mods), Anthropic: sample mods including `token-weather`, `blast-radius` and `replay-theater`, shared without support.
- The context indicator is from the author's own harness, described conceptually; the code on this page is an illustration written for it.

{% include page_nav.html %}
