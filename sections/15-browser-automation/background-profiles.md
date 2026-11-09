---
title: "Background browser profiles"
parent: "Driving a browser over CDP"
grand_parent: "Browser automation and computer use"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Background browser profiles

Part of [Driving a browser over CDP]({{ '/sections/15-browser-automation/cdp-and-playwright.html' | relative_url }}), in [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}).

A background browser profile is a browser the agent owns: its own profile directory, its own debugging port, no window on your screen. It is how an agent can browse for an hour while you keep working, and how two agents can browse at once without fighting over one browser. This page covers why it matters, how to set one up, and how to run several.

## Contents

- [Why not your own browser](#why-not-your-own-browser)
- [Never steal focus](#never-steal-focus)
- [Profile lifecycle](#profile-lifecycle)
- [Several agents at once](#several-agents-at-once)
- [Cleaning up](#cleaning-up)
- [Worked example](#worked-example)
- [Pitfalls](#pitfalls)

## Why not your own browser

| Your own browser | A background profile |
|---|---|
| has every login you have | has only what the task needs |
| shows every tab the agent opens | shows nothing |
| competes with you for the mouse and keyboard | never touches them |
| one per person | one per agent, as many as you need |
| an injected page acts with your reach | an injected page acts with almost none |

The last row connects to [Untrusted page content]({{ '/sections/15-browser-automation/prompt-injection.html' | relative_url }}): a profile with no logins is the cheapest limit on what a misled agent can do.

## Never steal focus

An agent that takes your screen while you use it causes three kinds of damage:

- A keystroke meant for your editor lands in the agent's form.
- A window raised every few seconds makes the machine unusable until the run ends.
- A screen-driving run sees whatever else you have open.

### The rules that prevent it

1. Run headless by default.
2. If a site needs a real window, open it off screen (`--window-position=-32000,-32000`) and never raise it.
3. Never drive the screen with mouse and keyboard tools when a CDP route exists.
4. Reserve screen-driving routes for tasks that cannot be done any other way, and run them when you are not at the machine.

### Build the rules into the launcher

Rules in a context file are applied inconsistently. A launcher script that is the only way agents start a browser applies them every time:

```bash
#!/usr/bin/env bash
# agent-browser: start a background Chrome for one agent task
set -euo pipefail
PROFILE="$(mktemp -d)"
google-chrome --headless=new --no-first-run \
  --user-data-dir="$PROFILE" --remote-debugging-port=0 &
CHROME_PID=$!
trap 'kill "$CHROME_PID"; rm -r "$PROFILE"' EXIT
# Chrome writes the port it picked to DevToolsActivePort in the profile
until [[ -s "$PROFILE/DevToolsActivePort" ]]; do sleep 0.2; done
PORT="$(head -n1 "$PROFILE/DevToolsActivePort")"
echo "{\"pid\": $CHROME_PID, \"port\": $PORT, \"profile\": \"$PROFILE\"}" >> logs/browsers.jsonl
CDP_URL="http://127.0.0.1:$PORT" "$@"
```

| Line | Rule it enforces |
|---|---|
| `--headless=new` | nothing on your screen |
| `mktemp -d` | a fresh profile with no logins |
| `--remote-debugging-port=0` | a free port, so two agents never collide |
| `trap ... EXIT` | the browser and profile are removed even if the task fails |
| `browsers.jsonl` | every browser is recorded, so a stuck one can be found by its pid |

An agent runs `agent-browser python check.py`, and the script reads `CDP_URL`.

## Profile lifecycle

### Fresh per run

```bash
PROFILE="$(mktemp -d)"
google-chrome --headless=new --user-data-dir="$PROFILE" --remote-debugging-port=9333 --no-first-run &
CHROME_PID=$!
# ... run the task ...
kill "$CHROME_PID"
rm -r "$PROFILE"
```

Nothing carries over, so one run cannot affect the next. This is the default for tests, screenshots and scraping public pages.

### Persistent for a login

Some tasks need a site you log into once by hand, such as an internal dashboard. Set up a dedicated automation profile:

1. Create it outside any repository, for example `~/.local/share/agent-browser/dashboard`.
2. Open it once in a visible window and log in by hand.
3. Run every later task headless against the same `--user-data-dir`.
4. Treat the directory like a credential: it holds the session cookies.

### Which to use

| Task | Profile |
|---|---|
| Screenshot of your local app | fresh |
| Scrape a public page | fresh |
| Check a dashboard behind a login | persistent, dedicated |
| Anything touching your personal accounts | neither: do it yourself |

## Several agents at once

Two agents sharing one browser will navigate each other's tabs. Give each its own profile and its own port.

![Sequence diagram in a parallel region. Agent A launches Chrome A on port 9333 with a fresh profile, and Agent B launches Chrome B on port 9334 with a fresh profile. Agent A takes before and after screenshots of the settings page while Agent B reads the pricing page and finds three plans. Each agent then stops its own browser by process id.]({{ '/assets/images/15-browser-automation/parallel-browsers-sequence.svg' | relative_url }}){: style="width:100%"}

*Two agents, two background browsers, no shared state and nothing on the person's screen.*

```bash
# agent A
google-chrome --headless=new --user-data-dir="$(mktemp -d)" --remote-debugging-port=9333 &
# agent B
google-chrome --headless=new --user-data-dir="$(mktemp -d)" --remote-debugging-port=9334 &
```

Two ways to keep ports apart:

| Approach | How | When |
|---|---|---|
| Fixed range | each agent slot owns a port, such as 9333 for slot A and 9334 for slot B | a small, fixed number of agents |
| Port 0 | Chrome picks a free port and writes it to `DevToolsActivePort` in the profile | any number of agents; used by the launcher above |

Either way, the launcher records which agent holds which port and process, so a crashed agent's browser can be found and stopped.

## Cleaning up

A background browser that outlives its agent keeps memory, a port and, for a persistent profile, a live session.

| Do | Avoid | Why |
|---|---|---|
| `kill "$CHROME_PID"`, the process you started | `pkill chrome` or any name pattern | a pattern kill also stops the person's own browser |
| a `trap` that cleans up on exit | cleanup only at the end of the happy path | a failed task leaves the browser running |
| `rm -r "$PROFILE"` for a fresh profile | removing a persistent profile | the persistent one holds a login set up by hand |
| a periodic sweep of `browsers.jsonl` for pids whose agent has ended | leaving orphans until the machine slows | orphans accumulate over a day of runs |

```bash
kill "$CHROME_PID"        # the one you started
```

## Worked example

#### The situation

Two agents work in parallel: one updates the settings page and needs screenshots, the other checks that a public pricing page still lists three plans.

#### The setup

Each agent's launcher starts its own headless Chrome with a fresh profile, on ports 9333 and 9334, and records the process ids.

#### What the person sees

Nothing. They keep editing in their own browser and editor. Ten minutes later two reports arrive: the settings agent with before-and-after screenshots, the pricing agent with "three plans listed; prices unchanged".

#### The cleanup

Each launcher stops its own browser by process id and removes its profile directory.

```text
launcher-a> chrome pid 48211 on :9333 stopped; profile removed
launcher-b> chrome pid 48230 on :9334 stopped; profile removed
```

## Examples from my own harness

- The CDP skill always drives a dedicated profile in the background and never steals focus; that rule is stated before any browser is launched.
- Background processes are stopped by their own process id or process group, never by a name pattern.

To start: write one launcher like the script above and make it the only way your agents start a browser.

## Pitfalls

- A launcher that agents can bypass is a suggestion. Deny direct `google-chrome` and `chromium` calls with a permission rule or a hook so the launcher is the route.
- A persistent profile copied between machines carries its cookies with it. Treat the copy as a credential.
- An off-screen window can still take keyboard focus on some desktops when it opens. Prefer headless, and test the off-screen option on your own machine.
- A crashed launcher without a `trap` leaves the port open and the profile on disk.

## Sources

- Since Chrome 136, the remote debugging switches need `--user-data-dir` pointing at a non-default directory: <https://developer.chrome.com/blog/remote-debugging-port> (2025-03-17), verified 2026-11-09.
- Chrome command-line flags (`--headless`, `--remote-debugging-port=0` and the `DevToolsActivePort` file): <https://developer.chrome.com/docs/chromium/headless>, not yet verified against the primary source for this page.
- The background-profile and focus rules: from the author's own harness, described conceptually.

{% include page_nav.html %}
