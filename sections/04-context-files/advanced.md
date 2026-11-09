---
title: "Advanced"
parent: "Context files"
nav_order: 2
---

# Advanced context file topics

As of 2026-11-09.

- How my shared context file is built: a base module, a client-specific part and a section for each machine are joined by a compile script and checked against a size budget. The installer renders the result into a versioned directory and links it, so a bad edit can be rolled back.
