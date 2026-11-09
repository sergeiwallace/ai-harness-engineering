---
title: "Background browser profiles"
parent: "Driving a browser over CDP"
grand_parent: "Browser automation and computer use"
nav_order: 1
---

{% include page_nav.html %}

# Background browser profiles

As of 2026-11-09.

Part of [Driving a browser over CDP]({{ '/sections/15-browser-automation/cdp-and-playwright.html' | relative_url }}), in [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}).

A background browser profile is a browser the agent owns: its own profile directory, its own debugging port, no window on your screen. It is how an agent can browse for an hour while you keep working, and how two agents can browse at once without fighting over one browser. This page covers why it matters, how to set one up, and how to run several.

## Contents

- [Why not your own browser](#why-not-your-own-browser)
- [Never steal focus](#never-steal-focus)
- [Profile lifecycle](#profile-lifecycle)
- [Several agents at once](#several-agents-at-once)
- [Cleaning up](#cleaning-up)
- [Worked example](#worked-example)

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

An agent that opens a window, raises it, or moves the pointer takes your screen while you are using it. A keystroke you meant for your editor lands in the agent's form. A run that steals focus every few seconds makes the machine unusable until it ends.

### The rules that prevent it

1. Run headless by default.
2. If a site needs a real window, open it off screen (`--window-position=-32000,-32000`) and never raise it.
3. Never drive the screen with mouse and keyboard tools when a CDP route exists.
4. Reserve screen-driving routes for tasks that cannot be done any other way, and run them when you are not at the machine.

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

Some tasks need a site you log into once by hand: an internal dashboard, a vendor console. Keep a named automation profile for it, separate from your own, stored outside any repository. Log in once in a visible window, then run headless against the same profile. Treat the profile directory like a credential: it holds the session cookies.

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

Pick ports from a range reserved for automation and have the launcher record which agent holds which, so a crashed agent's browser can be found and stopped.

## Cleaning up

A background browser that outlives its agent keeps memory, a port and, for a persistent profile, a live session. Clean up by the process id you started, never by a name pattern: a pattern kill can stop the person's own browser along with the agent's.

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

To start: add `--headless=new --user-data-dir="$(mktemp -d)"` to every browser your agents launch.

## Sources

- Chrome command-line flags (`--headless`, `--user-data-dir`, `--remote-debugging-port`): <https://developer.chrome.com/docs/chromium/headless>, not yet verified against the primary source for this page.
- The background-profile and focus rules: from the author's own harness, described conceptually.

{% include page_nav.html %}
