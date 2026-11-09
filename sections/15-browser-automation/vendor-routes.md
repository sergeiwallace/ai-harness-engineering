---
title: "Claude in Chrome and computer use"
parent: "Browser automation and computer use"
nav_order: 1
---

{% include page_nav.html %}

# Claude in Chrome and computer use

As of 2026-11-09.

Part of [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}).

Claude Code ships two built-in ways to act outside the terminal. Claude in Chrome drives your own Chrome through an extension. Computer use lets the agent see and operate your screen. Both work on your real environment, with your logins and your windows, which is what makes them useful and what makes them different from the [CDP route]({{ '/sections/15-browser-automation/cdp-and-playwright.html' | relative_url }}).

## Contents

- [Side by side](#side-by-side)
- [Claude in Chrome](#claude-in-chrome)
- [Computer use](#computer-use)
- [Choosing between the routes](#choosing-between-the-routes)
- [Pitfalls](#pitfalls)

## Side by side

| | Claude in Chrome | Computer use | Your own CDP script |
|---|---|---|---|
| What it drives | tabs in your Chrome or Edge | your whole screen | a separate Chrome |
| Your logins | shared | shared | none, unless you set them up |
| Visible | yes, a real window | yes, it uses your screen | no, headless |
| Provider | direct Anthropic plan | direct Anthropic plan, preview | any |
| Platform | Chrome, Edge, other Chromium browsers; not WSL | macOS, research preview | anywhere Chrome runs |

## Claude in Chrome

### What it is

From the documentation: "Claude Code integrates with the Claude in Chrome browser extension to give you browser automation capabilities from the CLI or the VS Code extension." Claude "opens new tabs for browser tasks and shares your browser's login state, so it can access any site you're already signed into. Browser actions run in a visible Chrome window in real time."

### Starting it

```bash
claude --chrome
```

Inside a session, `/chrome` shows the connection status, manages site permissions, reconnects the extension, and can turn the integration on by default.

### Requirements

- Chrome, Edge or another Chromium-based browser with the extension installed.
- A direct Anthropic plan (Pro, Max, Ultra, Team or Enterprise), signed in with `/login`.
- It is not available through third-party providers such as Amazon Bedrock, Google Cloud's Agent Platform or Microsoft Foundry.

### Safety properties worth knowing

#### It pauses on logins

"When Claude encounters a login page or CAPTCHA, it pauses and asks you to handle it manually." Credentials stay with you.

![Sequence diagram. The person asks Claude Code to test the signup form. Claude Code opens a new tab in the person's visible Chrome, which requests the signup page. Inside a critical region for the login page, the web app redirects to sign in, Claude Code tells the person it has paused, and the person signs in by hand. Claude Code then submits a bad email, the web app shows the error, and Claude Code reports a pass.]({{ '/assets/images/15-browser-automation/chrome-login-sequence.svg' | relative_url }}){: style="width:100%"}

*Claude in Chrome pauses at a login and hands the keyboard back to you.*

#### A repository cannot switch it on

A project's settings file cannot enable the integration: "a repository you check out must not be able to connect Claude to your browser." Only you can, with the flag or `/chrome`.

#### Site permissions live in the extension

The extension decides which sites Claude can browse, click and type on.

### Worked example: checking a form

```text
Open localhost:3000/signup, submit the form with an invalid email,
and tell me whether the error message appears under the field.
```

Claude opens a tab in your browser, fills the form, submits it and reports what it saw. Because it is your browser, the check runs against whatever you are logged into, which is convenient for an app behind a login and a reason to be careful on sites where a wrong click has consequences.

> Enabling Chrome by default loads the browser tools into every session, which uses context. The documentation suggests connecting only when needed.

## Computer use

### What it is

Computer use lets the agent take screenshots of your screen and use the mouse and keyboard. In the Claude Code CLI it is a research preview on macOS for Pro, Max and Ultra plans. It reaches what no other route reaches: native desktop apps, system dialogs, tools with no web or API surface.

### What it costs

Every step is a screenshot, a decision and an action, so it is the slowest route. It occupies your screen while it runs. And it sees whatever is on that screen, including windows unrelated to the task.

### Worked example: a native app with no API

A design tool stores its export settings only in a preferences window. No CLI or file route exists. Computer use opens the window, sets the export format and confirms with a screenshot. This is the case the route exists for; for anything with a web page or a file, use another route.

## Choosing between the routes

Work down the routes in order and stop at the first that answers:

| Question | If yes |
|---|---|
| Is there an API, a file or a CLI? | use it; no browser |
| Is it a native app rather than a web page? | computer use |
| Does it need your logged-in session, with you watching? | Claude in Chrome |
| Otherwise | your own CDP script, background profile |

![Sequence diagram. An agent needing pull request comments first calls the API, which returns 401 because no token is configured. It tries the next rung, the gh CLI, which returns the comments as JSON. The CDP browser is never started, because the CLI answered.]({{ '/assets/images/15-browser-automation/route-ladder-sequence.svg' | relative_url }}){: style="width:100%"}

*An agent working down the ladder. Here the CLI answers, so the browser is never started.*

## Pitfalls

- Your logged-in browser gives the agent the same reach you have. Keep site permissions narrow.
- A visible browser or screen-driving run interrupts your own work. Prefer the background route for anything long.
- If you reach Claude through a cloud provider, plan on the CDP route; the vendor routes need a direct plan.

## Sources

- Claude in Chrome, including requirements, the `--chrome` flag and the project-settings rule: <https://code.claude.com/docs/en/chrome.md>, verified 2026-11-09.
- Computer use: <https://code.claude.com/docs/en/computer-use.md>, verified 2026-11-08.

{% include page_nav.html %}
