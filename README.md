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
  - resource: /design/architecture/synthesis0.gpt6a.md
    title: Integrated findings and proposed snapshot-first architecture
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

## Proposed starting point

Firefox's windows/tabs are persisted as compressed JSON session snapshots;
SQLite Places history is a separate enrichment source. Start with snapshot
decode/diff and one profile value written through `CaminusStore::put_value`.

The [architecture synthesis](design/architecture/synthesis0.gpt6a.md) proposes five
Rust crates and the first end-to-end slice. The main choices are observation-scoped
versus heuristic persistent IDs, in-place versus generation-based publication,
and the visible filesystem layout. Disk snapshots do not supply reliable stable
live tab IDs, and the current caminus backend writes in place.
