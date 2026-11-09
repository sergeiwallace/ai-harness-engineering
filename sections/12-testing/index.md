---
title: "Testing agent-written code"
nav_order: 22
has_children: true
has_toc: false
---

{% include page_nav.html %}

# Testing agent-written code

As of 2026-11-09.

Stop 9 on the talk path, about 2 minutes.

- Test what the code is for at its real entry point, including end-to-end runs.
- Write the failing test first, and watch for agents editing tests to make them pass.
- One line on browser checks: an agent can take a screenshot of what it built and compare it with the intended result (see [Browser automation and computer use]({{ '/sections/15-browser-automation/' | relative_url }})).

## Pages for this topic

- [Testing in detail]({{ '/sections/12-testing/detail.html' | relative_url }})
- [Advanced testing topics]({{ '/sections/12-testing/advanced.html' | relative_url }})

{% include page_nav.html %}
