---
type: Index
title: foxread research
description: Reading paths into the initial Firefox observer and persistence investigations.
resource: /design/index.md
status: draft
generated: { by: "openai/gpt-6-astra#high", at: 2026-10-14 }
sources:
  - resource: "opencode://ses_ec59f6862ffeFt6UN2FUOhTXOO"
    title: Assemble navigation for the commissioned research
---

# Research index

## Integrated direction

- [Implementation plan](implementation/plan0.gpt6a.md): current work-ready JSON
  scope, eight crate boundaries, three kubeless reuse choices, ticket graph, and
  measurement-backed executor scheduling assessment. Work anchor: `foxr-json`.
- [Kubeless integration assessment](architecture/reuse2.gpt6a.md): parent review
  of the broad survey; recommends output assessment now, optional keyed diff
  later, and eight initial crate boundaries rather than a dedicated diff crate.
- [Domain-shaped crates and verified full-state writes](architecture/layout1.gpt6a.md):
  current revision after human review; nine proposed crates, typed ID prefixes,
  in-place persistence, and desired-versus-actual output verification.
- [Snapshot-first architecture](architecture/synthesis0.gpt6a.md): reconciles the
  investigations and preserves the initial five-crate proposal and decision
  framing; layout and publication direction are refined above.

## Intent

- [Project brief](intent/brief0.gpt6a.md): user goals, the deferred Git extension,
  and the questions the design needs to answer.

## Observation

- [Firefox stores and occhre reuse](firefox-stores/stores0.gpt6a.md): sessionstore
  format and cadence, identity/timestamp limits, Places enrichment, and the
  existing SQLite retention/read/extraction library spine.
- [Watchman integration](watching/watchman0.gpt6a.md): official Rust client,
  notifications as invalidation, and reconnect/read/application boundaries.

## Persistence

- [Kopacetic and caminus investigation](kopacetic-persistence/persistence0.gpt6a.md):
  source-grounded APIs, filesystem layout options, identity and recovery limits,
  and a proposed first one-shot dump.

## Reuse

- [Kubeless reuse verdicts](kubeless/reuse0.glm53m.md): verified
  desired-vs-filesystem comparison through kubeless-observation-assessment,
  neutral observation diff through kubeless-view-change, no backend write-planning
  reuse, and a demoted `core/diff` boundary.
