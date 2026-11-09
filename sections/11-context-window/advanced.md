---
title: "Advanced"
parent: "Context window and memory"
nav_order: 2
---

{% include page_nav.html %}

# Advanced context window topics

As of 2026-11-09.

- My context supervisor: hooks read how full the context is and, past a threshold, run save-state and then send `/compact` into the session through the prompt-injection library.
- `/autocompact` sets the threshold for automatic compaction in Claude Code.

{% include page_nav.html %}
