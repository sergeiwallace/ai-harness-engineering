---
title: "Screenshots as evidence"
parent: "Browser automation and computer use"
nav_order: 3
last_updated: "2026-11-09"
---

{% include page_nav.html %}

# Screenshots as evidence

Part of [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }}).

An agent's report that a UI change worked is a claim. A screenshot of the result is evidence:

- the agent can look at it and compare it with what was asked;
- a reviewer can check it without running anything.

This page covers when to take one, how to compare two, and what a screenshot cannot prove.

## Contents

- [When to take one](#when-to-take-one)
- [The evidence loop](#the-evidence-loop)
- [Taking a useful screenshot](#taking-a-useful-screenshot)
- [Comparing screenshots](#comparing-screenshots)
- [What a screenshot does not prove](#what-a-screenshot-does-not-prove)
- [Worked examples](#worked-examples)
- [Examples from my own harness](#examples-from-my-own-harness)
- [Pitfalls](#pitfalls)

## When to take one

| Change | Screenshot? | What to capture |
|---|---|---|
| Layout, styling, a new UI element | yes | before and after, at each viewport width the change affects |
| Copy or label text | yes | the element, after |
| A bug that shows on screen | yes | the broken state, then the fixed state |
| Backend logic with no visible change | no | a test is the evidence |
| A refactor that should change nothing visible | yes, as a check | a pixel diff of before and after that shows no change |

## The evidence loop

![Sequence diagram. The agent edits the CSS of a local app and asks a headless script to capture the settings page; the script loads it and returns settings.png. Looking at the image, the agent finds a white-on-white button, fixes it and captures again; the button is visible. The agent sends the reviewer a pull request with before and after images.]({{ '/assets/images/15-browser-automation/screenshot-loop-sequence.svg' | relative_url }}){: style="width:100%"}

*A screenshot catches the agent's own mistake before the reviewer sees it.*

![The agent changes the page, takes a headless full-page screenshot, compares it with the intent or a before shot, and reports with the image attached when it matches. When it differs, the agent fixes the page and retakes the screenshot.]({{ '/assets/images/15-browser-automation/screenshot-evidence.svg' | relative_url }}){: style="width:100%"}

*The evidence loop. The comparison step is where the agent catches its own mistakes, before a reviewer does.*

The screenshot works as a check when the agent looks at it, compares it with the intent and fixes what differs. Attached without that step, it is an illustration.

## Taking a useful screenshot

### Fix the conditions

| Setting | Why |
|---|---|
| Viewport size | The same page at 375 and 1280 pixels wide is two different layouts. Name the size. |
| Full page or element | Full page for layout; one element for a detail. |
| Data | Use the same fixture data before and after, or the diff shows data changes, not your change. |
| Animations | Wait for them to finish, or disable them, so two shots of the same page match. |
| Fonts | Wait for web fonts to load; a fallback font shifts every line. |

### Capture the element that changed

```python
page.get_by_role("form", name="Settings").screenshot(path="settings-form.png")
```

A cropped shot of the part that changed is easier to read than a full page where the change is a few pixels.

### Name files for the reader

| Name | Says |
|---|---|
| `settings-375-before.png` | page, viewport width, before the change |
| `settings-375-after.png` | the same page and width, after |
| `screenshot1.png` | nothing |

## Comparing screenshots

### By eye: the agent looks

Multimodal models can read images. The step has three parts:

1. The agent opens both shots.
2. It writes one sentence describing the difference.
3. It compares that sentence with the request.

This catches the gross failures: a missing button, an overlapping label, a blank page, white text on white.

### By pixels: a visual diff

A pixel diff finds small changes the eye misses, and proves that nothing else moved.

```python
from PIL import Image, ImageChops

before = Image.open("settings-375-before.png").convert("RGB")
after = Image.open("settings-375-after.png").convert("RGB")
if before.size != after.size:
    print(f"page size changed: {before.size} -> {after.size}")
else:
    box = ImageChops.difference(before, after).getbbox()
    print("no change" if box is None else f"changed region: {box}")
```

| Output | Reading |
|---|---|
| `no change` | nothing moved; expected for a refactor, a failure for a visual fix |
| `changed region: (16, 120, 359, 610)` | only that rectangle changed; check it covers what you changed and nothing else |
| `page size changed` | the page grew or shrank, often because content moved below the fold |

### In a test: snapshot assertions

Playwright Test can store a reference image and fail when the page drifts from it beyond a threshold:

```js
await expect(page).toHaveScreenshot("settings.png", { maxDiffPixels: 100 });
```

| Step | Command or behaviour |
|---|---|
| First run | no reference exists; the test writes the screenshot as the baseline, which you review and commit |
| Later runs | the test compares against the baseline and fails beyond the threshold |
| An intended visual change | `npx playwright test --update-snapshots`, then review the new images before committing |
| File names | include the browser and platform, such as `settings-chromium-linux.png`, because rendering differs between them |

This turns a one-off check into a layer-4 test that holds on every change:

- Proactive: the baseline encodes how you want the page to look.
- Reactive: the test fails when a change breaks that look.

Run it in the same environment that produced the baseline. See [Testing agent-written code]({{ '/sections/12-testing/' | relative_url }}).

<details markdown="1">
<summary>Aside: why snapshot tests use thresholds</summary>

Two renders of the same page can differ by a few pixels because of font anti-aliasing, GPU differences or sub-pixel layout.

- An exact match fails on that noise.
- A small threshold, such as one percent of pixels, ignores it and still catches a moved button.

</details>

## What a screenshot does not prove

- It does not prove behaviour. A button that renders correctly may do nothing when clicked. Pair the shot with a test that clicks it.
- It does not prove other states. The shot shows one viewport, one user, one set of data.
- It can be stale. A screenshot taken before the last edit proves nothing about the code under review. Take it after the final change, and say when.

## Worked examples

### Example 1: an overflow fix

#### The request

"Labels overflow on narrow screens."

#### What the agent does

It takes `settings-375-before.png`, fixes the CSS, takes `settings-375-after.png`, runs the pixel diff, and sees the changed region covers only the form. It attaches both shots.

#### The report

```text
agent> Fixed label overflow at 375 px: labels now wrap.
agent> Evidence: settings-375-before.png, settings-375-after.png.
agent> Pixel diff: only the form region changed (x 16-359, y 120-610).
```

### Example 2: catching its own mistake

1. An agent adds a "Save" button and is about to report success.
2. The workflow requires a screenshot first, so it captures the page.
3. Looking at the image, it finds the button rendered white on white.
4. It fixes the colour, captures again, and only then reports, with both images.

### Example 3: a claim that a screenshot cannot settle

| Claim | What the screenshot proves | What else is needed |
|---|---|---|
| "The form now validates email addresses" | the error message renders for one input | a test that submits three invalid addresses and checks each is rejected |

## Examples from my own harness

- A screenshot skill captures a page or a window and attaches the image to a report, so evidence travels with the claim.
- Presentation work is checked the same way: each slide is rendered and captured, and the capture is read back before a deck is called done.

To start: build the screenshot into the workflow, for example a script your UI tasks run after every change that captures the page and fails if the image is missing from the pull request. A context-file line asking for screenshots is the second layer.

## Pitfalls

- A screenshot of a logged-in page can show names, emails or account numbers. Use test accounts and fixture data, crop to the element, or mask regions (Playwright's `mask` option; not verified for this page).
- Updating snapshot baselines without looking at the new images turns the regression test into a rubber stamp. Review every changed image.
- A full-page screenshot of a long page is a large image in the agent's context. Capture the element that changed.
- A baseline made on one machine fails on another with different fonts. Generate and compare baselines in the same environment, such as the CI container.

## Sources

- Playwright visual comparisons: baseline creation, `--update-snapshots`, platform-specific file names, `threshold` and `maxDiffPixels`: [Playwright docs: Visual comparisons](https://playwright.dev/docs/test-snapshots), verified 2026-11-09.
- The screenshot skill and the capture checks: from the author's own harness, described conceptually.

{% include page_nav.html %}
