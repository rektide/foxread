---
type: Project
title: foxread
description: A proposed multi-crate Rust observer of Firefox disk state, projected through kopacetic and caminus.
resource: /README.md
status: planning
generated: { by: "openai/gpt-6-astra#high", at: 2026-10-14 }
sources:
  - resource: "opencode://ses_ec59f6862ffeFt6UN2FUOhTXOO"
    title: Commission and integrate Firefox storage, Watchman, and kopacetic persistence research
  - resource: /design/intent/brief0.gpt6a.md
    title: Initial goals and open design questions
  - resource: /design/architecture/synthesis0.gpt6a.md
    title: Integrated findings and proposed snapshot-first architecture
  - resource: /design/architecture/layout1.gpt6a.md
    title: Human-reviewed full-state write direction and domain-shaped crate proposal
  - resource: /design/implementation/plan0.gpt6a.md
    title: Work-ready JSON implementation plan and dependency-linked tickets
---

# foxread

Observe Firefox's persisted windows and tabs with Rust, follow changes with
Watchman where practical, and maintain a field-per-file representation through
kopacetic's caminus backend.

The project is preparing for implementation. The intended output includes supported IDs,
URLs, titles, timestamps, and referrer information, with source coverage made
explicit. Git checkpoints are a later extension.

Start with the [project brief](design/intent/brief0.gpt6a.md) and
[research index](design/index.md).

## Implementation direction

Firefox's windows/tabs are persisted as compressed JSON session snapshots;
SQLite Places history is a separate enrichment source. Start with snapshot
decoding and one profile value written through `CaminusStore::put_value`, followed
by exact assessment of the resulting filesystem tree.

The [implementation plan](design/implementation/plan0.gpt6a.md), anchored by
`foxr-json`, defines eight crate boundaries under `core/`, `firefox/`, `observe/`,
`output/`, and `app/`. Kubeless provides output assessment, optional keyed change
reporting, and scheduling mechanisms; scalar Assessor is deferred.
The selected direction is a full desired-value write in place, followed by
comparison against the actual filesystem. IDs use `win_<id>` and `tab_<id>`;
cross-snapshot identity remains a separate modeling question.

The [initial synthesis](design/architecture/synthesis0.gpt6a.md) preserves the
research integration. SQLite/WAL extraction design now proceeds separately in
`~/src/walcap`; foxread's current focus is the JSON sessionstore path.
