---
title: "Code review"
nav_order: 23
has_children: true
has_toc: false
---

# Code review

As of 2026-11-09.

Stop 9 on the talk path, about 2 minutes.

- Agents review agents: an automated review on every change, with a person where a person must approve.
- Cross-family review: a reviewer from another model family does not share the author's blind spots.
- A few practices: have a separate agent verify each finding, fix the acceptance criteria before review, and stop when findings stop being new.
- `/code-review` (alias `/review`) and `/security-review` review a branch or pull request in a separate context for bugs or security risks.

## Pages for this topic

- [Code review in detail]({{ '/sections/13-code-review/detail.html' | relative_url }})
- [Advanced code review topics]({{ '/sections/13-code-review/advanced.html' | relative_url }})
