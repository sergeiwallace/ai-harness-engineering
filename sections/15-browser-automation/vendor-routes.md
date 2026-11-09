---
title: "Claude in Chrome and computer use"
parent: "Browser automation and computer use"
nav_order: 1
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Claude in Chrome and computer use

Part of [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}).

Claude Code ships two built-in ways to act outside the terminal. Claude in Chrome drives your own Chrome through an extension. Computer use lets the agent see and operate your screen. Both work on your real environment, with your logins and your windows, which is what makes them useful and what separates them from the [CDP route]({{ '/sections/15-browser-automation/cdp-and-playwright.html' | relative_url }}).

## Contents

- [Side by side](#side-by-side)
- [Which tool Claude picks](#which-tool-claude-picks)
- [Claude in Chrome](#claude-in-chrome)
- [Computer use](#computer-use)
- [Choosing between the routes](#choosing-between-the-routes)
- [Pitfalls](#pitfalls)
- [Sources](#sources)

## Side by side

| | Claude in Chrome | Computer use | Your own CDP script |
|---|---|---|---|
| What it drives | tabs in your Chrome, Edge or another Chromium browser | the apps you approve on your screen | a separate Chrome |
| Your logins | shared | shared | none, unless you set them up |
| Visible | yes, a real window | yes, other apps are hidden while it works | no, headless |
| Plans | Pro, Max, Ultra, Team, Enterprise | Pro, Max, Ultra; not Team or Enterprise | any |
| Through Bedrock and other cloud providers | no | no | yes |
| Platform | not supported in WSL | macOS only in the CLI | anywhere Chrome runs |
| Start | `claude --chrome` or `/chrome` | enable `computer-use` in `/mcp` | your script |

## Which tool Claude picks

The computer use documentation describes the order Claude tries tools in, most precise first:

1. An MCP server for the service, if you have one.
2. Bash, if the task is a shell command.
3. Claude in Chrome, if the task is browser work and it is set up.
4. Computer use, when none of those apply.

The same reasoning sits behind the route ladder on the [overview]({{ '/sections/15-browser-automation/#try-the-other-routes-first' | relative_url }}).

## Claude in Chrome

### What it is

From the documentation: "Claude Code integrates with the Claude in Chrome browser extension to give you browser automation capabilities from the CLI or the VS Code extension."

- Tabs: Claude "opens new tabs for browser tasks and shares your browser's login state, so it can access any site you're already signed into."
- Visibility: "Browser actions run in a visible Chrome window in real time."
- Grouping: the tabs Claude opens are collected into a Chrome tab group tied to the session.
- Browsers: Chrome and Edge, and other Chromium browsers such as Brave, Arc, Vivaldi and Opera; not supported in WSL.

### What it can do

| Capability | Example request |
|---|---|
| Live debugging | "Open the dashboard and check the console for errors on load." |
| Design checks | build a page from a mock, then open it to compare |
| Form and flow testing | "Submit the signup form with an invalid email and check the error appears." |
| Logged-in apps | work in a web app you are signed into, without an API connector |
| Data extraction | "Extract name, price and availability for each item and save a CSV." |
| File uploads | attach a local file to an upload field, up to 10 MB per upload |
| Recordings | record a GIF of a flow; it captures everything visible, account details included |

### Starting it

```bash
claude --chrome
```

Inside a session, `/chrome` shows the connection status, manages site permissions, reconnects the extension, and can turn the integration on by default.

### Requirements

- Chrome, Edge or another Chromium-based browser, with the Claude in Chrome extension installed.
- A direct Anthropic plan (Pro, Max, Ultra, Team or Enterprise), signed in with `/login`. An API key or a long-lived token keeps the integration off.
- It is not available through third-party providers such as Amazon Bedrock, Google Cloud's Agent Platform or Microsoft Foundry.

<!-- IMAGE NEEDED: a screenshot of the /chrome status panel in Claude Code showing "Status: Enabled" and "Extension: Installed"; search: "Claude Code /chrome status", "Claude in Chrome setup" -->

### Safety properties worth knowing

#### It pauses on logins

"When Claude encounters a login page or CAPTCHA, it pauses and asks you to handle it manually." Credentials stay with you.

![Sequence diagram. The person asks Claude Code to test the signup form. Claude Code opens a new tab in the person's visible Chrome, which requests the signup page. Inside a critical region for the login page, the web app redirects to sign in, Claude Code tells the person it has paused, and the person signs in by hand. Claude Code then submits a bad email, the web app shows the error, and Claude Code reports a pass.]({{ '/assets/images/15-browser-automation/chrome-login-sequence.svg' | relative_url }}){: style="width:100%"}

*Claude in Chrome pauses at a login and hands the keyboard back to you.*

#### A repository cannot switch it on

A project's settings file cannot enable the integration: "a repository you check out must not be able to connect Claude to your browser." Only you can, with the flag or `/chrome`.

#### Site permissions live in the extension

- The extension decides which sites Claude can browse, click and type on.
- In auto mode, a classifier-approved browser call skips the extension's own per-site check, unless your permission rules deny sites to Claude in Chrome.

#### File uploads are limited

- Claude can upload only files the session may read, so a `Read` deny rule also blocks uploading the file.
- It refuses files with credential-style names or folders, such as `.env`, `.pem`, `.key` or anything under `.ssh`.

### Worked example: checking a form

```text
Open localhost:3000/signup, submit the form with an invalid email,
and tell me whether the error message appears under the field.
```

1. Claude opens a new tab in your browser, in the session's tab group.
2. It fills the form, submits it and reads the page.
3. It reports what it saw, and can save a screenshot to disk if asked.

Because it is your browser, the check runs against whatever you are logged into. That is convenient for an app behind a login, and a reason to keep site permissions narrow on sites where a wrong click has consequences.

### Common problems

| Symptom | Cause | Fix |
|---|---|---|
| "Browser extension is not connected" | the extension is signed in to another account or organisation, or not running | sign the extension in to the same account and organisation as Claude Code; run `/chrome` to reconnect |
| Browser tools stop after a while | the extension's service worker went idle | `/chrome`, then "Reconnect extension" |
| Commands do nothing on a page | a JavaScript alert or confirm dialog is blocking the page | dismiss it by hand and tell Claude to continue |
| Higher context use in every session | Chrome is enabled by default, so its tools always load | connect only when needed, with `--chrome` |

## Computer use

### What it is

Computer use lets Claude open apps, click, type and see your screen.

- Status: a research preview in the CLI on macOS, for Pro, Max and Ultra plans; not available on Team or Enterprise plans.
- Session: interactive only; not available with the `-p` flag.
- Reach: native apps, simulators such as the iOS Simulator, and tools with no API.

### Turning it on

1. Run `/mcp` in an interactive session and find the built-in `computer-use` server, which is disabled by default.
2. Select it and choose Enable. The setting persists per project.
3. On first use, grant two macOS permissions when prompted:
   - Accessibility, so Claude can click, type and scroll;
   - Screen Recording, so Claude can see the screen. macOS may require restarting Claude Code after this one.

### Approving apps

Enabling the server does not give Claude every app. The first time it needs an app in a session, a prompt shows which apps it wants, any extra permissions such as clipboard access, and how many other apps will be hidden. You choose "Allow for this session" or "Deny".

| Warning in the prompt | Applies to |
|---|---|
| Equivalent to shell access | Terminal, iTerm, VS Code, Warp and other terminals and IDEs |
| Can read or write any file | Finder |
| Can change system settings | System Settings |

The level of control also depends on the app: browsers and trading platforms are view-only, terminals and IDEs are click-only, and other apps get full control.

### How it behaves on your screen

| Behaviour | Detail |
|---|---|
| One session at a time | a session takes a lock at its first action and keeps it until the session exits |
| Other apps hidden | only approved apps are visible while Claude works; they are restored when the turn ends |
| Your terminal excluded | the terminal stays visible to you and never appears in Claude's screenshots |
| Screenshots downscaled | a 3456×2234 Retina capture becomes about 1372×887; enlarge small text in the app if Claude cannot read it |
| Stop at any time | `Esc` anywhere aborts the current action; `Ctrl+C` in the terminal also stops it |

<!-- IMAGE NEEDED: the macOS notification "Claude is using your computer · press Esc to stop", or the per-session app approval prompt in the terminal; search: "Claude Code computer use macOS", "Claude is using your computer notification" -->

### Safety

The documentation warns that computer use "runs on your actual desktop with access to the apps you approve", unlike the sandboxed Bash tool. Claude checks each action and flags possible prompt injection from on-screen content. The built-in guardrails:

- per-app approval for the session;
- warnings on apps with shell, file-system or system-settings reach;
- the terminal excluded from screenshots, so on-screen text in your session cannot feed back into the model;
- `Esc` as a global stop, consumed so on-screen content cannot use it to dismiss a dialog;
- the one-session lock.

### What it costs

| Cost | Why |
|---|---|
| Speed | every step is a screenshot, a decision and an action |
| Your screen | it hides your other apps and uses your mouse and keyboard while it runs |
| Exposure | it sees everything in the approved apps |
| Fragility | a moved button or a new dialog changes what it sees |

### Worked example: a layout bug that only appears at some window sizes

```text
The settings modal clips its footer on narrow windows. Resize the app
window down until you can reproduce it, screenshot the clipped state,
then check the CSS for the modal container.
```

1. Claude asks to control the app; you allow it for the session.
2. It resizes the window step by step until the footer clips, and takes a screenshot.
3. It reads the modal's stylesheet in the repository, fixes the height rule, rebuilds and resizes again.
4. It reports both screenshots: clipped before, whole after.

For anything with a web page, an API or a file, use another route.

## Choosing between the routes

Work down the routes in order and stop at the first that answers:

| Question | If yes |
|---|---|
| Is there an API, an MCP server, a file or a CLI? | use it; no browser |
| Is it a native app rather than a web page? | computer use |
| Does it need your logged-in session, with you watching? | Claude in Chrome |
| Otherwise | your own CDP script, background profile |

![Sequence diagram. An agent needing pull request comments first calls the API, which returns 401 because no token is configured. It tries the next rung, the gh CLI, which returns the comments as JSON. The CDP browser is never started, because the CLI answered.]({{ '/assets/images/15-browser-automation/route-ladder-sequence.svg' | relative_url }}){: style="width:100%"}

*An agent working down the ladder. Here the CLI answers, so the browser is never started.*

## Pitfalls

- Your logged-in browser gives the agent the same reach you have. Keep site permissions narrow.
- A GIF recording of a logged-in page captures account details. Review it before sharing.
- A visible browser or screen-driving run interrupts your own work. Prefer the background route for anything long.
- Approving a terminal or IDE to computer use is equivalent to shell access. Approve only what the task needs.
- If you reach Claude only through a cloud provider, plan on the CDP route; both vendor routes need a claude.ai account.

## Sources

- Claude in Chrome: capabilities, requirements, the `--chrome` flag, the project-settings rule, uploads and troubleshooting: [Claude Code docs: Use Claude Code with Chrome](https://code.claude.com/docs/en/chrome.md), verified 2026-11-09.
- Computer use in the CLI: requirements, enabling, app approval, behaviour, safety and the tool order: [Claude Code docs: Let Claude use your computer from the CLI](https://code.claude.com/docs/en/computer-use.md), verified 2026-11-09.

{% include page_nav.html %}
