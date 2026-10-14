---
type: Project
title: foxread
description: A proposed multi-crate Rust observer of Firefox disk state, projected through kopacetic and caminus.
resource: /README.md
status: research
generated: { by: "openai/gpt-6-astra#high", at: 2026-10-14 }
sources:
  - resource: "opencode://ses_ec59f6862ffeFt6UN2FUOhTXOO"
    title: Commission and integrate Firefox storage, Watchman, and kopacetic persistence research
  - resource: /design/intent/brief0.gpt6a.md
    title: Initial goals and open design questions
---

# foxread

Observe Firefox's persisted windows and tabs with Rust, follow changes with
Watchman where practical, and maintain a field-per-file representation through
kopacetic's caminus backend.

The project is in research/design. The intended output includes supported IDs,
URLs, titles, timestamps, and referrer information, with source coverage made
explicit. Git checkpoints are a later extension.

Start with the [project brief](design/intent/brief0.gpt6a.md) and
[research index](design/index.md).
