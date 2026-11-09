---
title: "Driving a browser over CDP"
parent: "Browser automation and computer use"
nav_order: 2
has_children: true
has_toc: false
redirect_from:
  - /sections/15-browser-automation/detail.html
---

{% include page_nav.html %}

# Driving a browser over CDP

As of 2026-11-09.

Part of [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}). Sub-page: [Background browser profiles]({{ '/sections/15-browser-automation/background-profiles.html' | relative_url }}).

The Chrome DevTools Protocol (CDP) is the interface that Chrome's own developer tools use to inspect and control a page. Any program that speaks it can drive Chrome: open pages, click, type, read the page, run JavaScript and take screenshots. Playwright and Puppeteer are libraries built on it. For an agent, driving a browser over CDP is the route that works with any model provider, runs in the background, and keeps the agent's browsing apart from yours.

## Contents

- [How it fits together](#how-it-fits-together)
- [Starting Chrome for automation](#starting-chrome-for-automation)
- [Talking to it](#talking-to-it)
- [Writing automation that does not break](#writing-automation-that-does-not-break)
- [A complete worked example](#a-complete-worked-example)
- [Examples from my own harness](#examples-from-my-own-harness)
- [Pitfalls](#pitfalls)

## How it fits together

![The agent writes and runs a script; the script speaks CDP to a Chrome with its own profile on a local port; the page's text and screenshots come back as files the agent reads; the person's own browser is untouched.]({{ '/assets/images/15-browser-automation/cdp-setup.svg' | relative_url }}){: style="width:100%"}

*Data and control flow: the agent drives Chrome only through a script, and reads results from files.*

Three parts work together on each browser task:

1. The agent writes or calls a script.
   - The script can be reviewed, rerun and reused.
2. The script drives Chrome over CDP.
   - Chrome runs with its own profile, headless or off screen.
3. The results come back as files.
   - Page text, extracted data, screenshots and console logs.
   - The agent's context holds a short summary and the file paths.

![Sequence diagram. The agent asks a Playwright script to check the settings page. The script launches a headless Chrome with its own profile and, in a loop that waits for the heading, navigates; Chrome requests the page, which returns HTML including hidden text. The script finds the Export button, captures a screenshot, receives PNG bytes, and returns "present" and a file path to the agent.]({{ '/assets/images/15-browser-automation/background-check-sequence.svg' | relative_url }}){: style="width:100%"}

*One check, call by call. The agent sees only the last arrow: a short result and a file path.*

## Starting Chrome for automation

### The flags

| Flag | Why |
|---|---|
| `--remote-debugging-port=9222` | opens the CDP endpoint on a local port |
| `--user-data-dir=<dir>` | a separate profile, so the agent never sees your cookies or history; recent Chrome versions refuse remote debugging on the default profile |
| `--headless=new` | no window at all; omit it and position the window off screen if a site behaves differently headless |
| `--no-first-run` | skips the welcome screens in a fresh profile |

```bash
PROFILE="$(mktemp -d)"
google-chrome \
  --remote-debugging-port=9222 \
  --user-data-dir="$PROFILE" \
  --headless=new \
  --no-first-run &
curl -s http://127.0.0.1:9222/json/version
```

The last command returns a JSON object with the browser version and a `webSocketDebuggerUrl`, which proves the endpoint is up.

### Dedicated profile: fresh or persistent

[Background browser profiles]({{ '/sections/15-browser-automation/background-profiles.html' | relative_url }}) goes deeper on profiles, focus and running several agents at once.

A fresh profile per run is the safest: nothing carries over, so one run cannot leak into the next. A persistent automation profile is useful when a task needs a login that you set up once by hand, for example an internal dashboard. Keep it separate from your own profile either way, and keep its directory out of any repository.

> Bind the debugging port to localhost only, which is Chrome's default. Anyone who can reach the port can control the browser and read every page in it.

## Talking to it

### With Playwright

Playwright can launch its own browser or attach to one you started:

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.connect_over_cdp("http://127.0.0.1:9222")
    context = browser.contexts[0] if browser.contexts else browser.new_context()
    page = context.new_page()
    page.goto("https://example.com", wait_until="domcontentloaded")
    print(page.title())
    page.screenshot(path="example.png")
```

### With raw CDP

Raw CDP is a JSON message protocol over a WebSocket. It is worth knowing because it is what every library sends underneath, and because a few things are easier raw:

```json
{"id": 1, "method": "Page.navigate", "params": {"url": "https://example.com"}}
{"id": 2, "method": "Runtime.evaluate", "params": {"expression": "document.title"}}
{"id": 3, "method": "Page.captureScreenshot", "params": {"format": "png"}}
```

These go to a page's own WebSocket, which `http://127.0.0.1:9222/json` lists for each open tab. Each reply carries the same `id` with a `result`; the screenshot comes back as base64.

### Which to use

| Need | Use |
|---|---|
| Clicking, typing, waiting for elements | Playwright |
| A screenshot or a page's text, nothing else | either; raw CDP has no dependency |
| Network interception, performance traces | Playwright, or raw CDP domains such as `Network` |
| Tests that a team runs in CI | Playwright Test |

## Writing automation that does not break

### Select elements by role and label

A selector like `div:nth-child(3) > button` breaks when the layout changes. A selector by role and name survives redesigns, because it names what the person sees:

```python
page.get_by_role("button", name="Save").click()
page.get_by_label("Email").fill("test@example.com")
```

### Wait for a condition

`time.sleep(3)` is too long on a fast day and too short on a slow one. Wait for the thing you need:

```python
page.get_by_role("heading", name="Settings").wait_for()
```

### Set a time limit on everything

A browser step that hangs holds the agent's turn open. Build the limits into the script:

| Limit | Playwright setting | Example |
|---|---|---|
| Every action and wait | `page.set_default_timeout(ms)` | `page.set_default_timeout(10_000)` |
| Navigation | `page.set_default_navigation_timeout(ms)` | `page.set_default_navigation_timeout(30_000)` |
| The whole script | the caller's timeout | `timeout 120 python check.py` |

On a timeout the script exits non-zero with a message the agent can act on, for example `timeout waiting for heading "Settings" after 10 s`.

### Write results to files

A full page of HTML in the agent's context is expensive and mostly noise. Split the output:

- Standard output: one or two summary lines, such as `export button: present`.
- Files: `page.txt` for the text, `shot.png` for the screenshot, `console.log` for browser errors.
- Exit code: 0 for a pass, 1 for a failed check, 2 for a script error.

## A complete worked example

#### The task

Check that the settings page of a local app shows the new "Export" button, and capture proof.

#### The script the agent writes

```python
import sys
from playwright.sync_api import sync_playwright

URL = "http://localhost:3000/settings"

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    page = browser.new_page(viewport={"width": 1280, "height": 800})
    page.set_default_timeout(10_000)
    page.goto(URL)
    button = page.get_by_role("button", name="Export")
    found = button.count() > 0
    page.screenshot(path="settings-export.png", full_page=True)
    browser.close()

print("export button:", "present" if found else "MISSING")
sys.exit(0 if found else 1)
```

#### What the agent reports

```text
agent> Ran the check against localhost:3000/settings.
agent> export button: present
agent> Screenshot: settings-export.png (attached). The button sits under
agent> "Data", right of "Import", as the design asked.
```

## Examples from my own harness

- One CDP skill is the entry point for every browser task. It starts with the non-browser routes, then drives a dedicated profile in the background so it never steals focus from my screen.
- A small library of reusable scripts sits behind the skill, so agents call a tested script instead of writing a new scraper each time.

To start: save the screenshot script above, point it at your local app, and ask the agent to run it after every front-end change.

## Pitfalls

- Using your own profile gives the agent your logged-in sessions everywhere. Use a separate one.
- Headless pages sometimes render differently, and some sites block headless browsers. Try an off-screen window before concluding a page is broken.
- A script that prints the whole page into the agent's context wastes it. Print a summary; write the rest to a file.

## Sources

- Chrome DevTools Protocol: <https://chromedevtools.github.io/devtools-protocol/>, not yet verified against the primary source for this page.
- Playwright `connect_over_cdp` and locators: <https://playwright.dev/python/docs/api/class-browsertype>, not yet verified against the primary source for this page.
- The CDP skill and script library: from the author's own harness, described conceptually.

{% include page_nav.html %}
