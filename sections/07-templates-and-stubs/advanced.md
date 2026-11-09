---
title: "Advanced"
parent: "Templates, prompt stubs and injected prompts"
nav_order: 2
---

# Advanced templates and stubs topics

As of 2026-11-09.

- The original-request block in detail: the person's words are stored with a hash and copied as a fixed block at the top of every brief. The launch guard refuses a brief whose block cannot be validated against the stored record.
- The prompt-injection library's interface: a session target, a payload, an injector, and a verify step that confirms the text was submitted.
