---
type: DesignBrief
title: Firefox disk state into kopacetic
description: The initial foxread commission, desired capabilities, and questions the research must resolve.
resource: /design/intent/brief0.gpt6a.md
status: draft
generated: { by: "openai/gpt-6-astra#high", at: 2026-10-14 }
sources:
  - resource: "opencode://ses_ec59f6862ffeFt6UN2FUOhTXOO"
    title: Human commission for multi-crate Rust Firefox observation and Astra medium investigations
---

# Firefox disk state into kopacetic

## Desired outcome

Build a multi-crate Rust program that observes Firefox browser state from disk,
preferably with Watchman, and maintains a windows-and-tabs representation in
kopacetic using its caminus filesystem backend. Change capture may use source
CDC or successive dumps and differences, according to what Firefox actually
persists.

The human's tentative filesystem picture is `<window>/<tab>/` containing useful
fields such as ID, URL, title, opened/visited times, and referrer. This is an
exploration of persistence shape, rather than a settled schema or a requirement
to invent fields Firefox does not supply.

The sibling repositories are `~/src/kopacetic` and `~/src/occhre`. Recent occhre
work reading OpenCode's SQLite is a reuse precedent; a reusable SQLite
library/toolkit is a possible outcome. If SQLite is involved, its reader should
be Rust, and WAL following is specifically worth investigating.

Git checkpoints in kopacetic are a desired later extension. The current
commission defers their implementation.

## Commissioned investigations

- Astra medium: establish Firefox's actual stores, their contents and update
  behavior. Where SQLite is relevant, investigate occhre's implementation and
  its reusable boundary, with a separate SQLite investigation if useful.
- Astra medium: investigate the more advanced kopacetic/caminus integration,
  filesystem shape, identities, and persistence behavior.
- Parent integration: connect these findings into a proposed Rust architecture
  and a coherent first implementation slice. Inspect Watchman's official Rust
  API independently.

The initial repository contains empty `README.md` and `foxread.ts` files plus a
JavaScript package manifest. There is no existing Rust implementation to extend.

## Questions the design needs to answer

1. **Observation coverage:** Which files describe open windows/tabs versus
   historical visits? What browser activity never reaches the observed files?
2. **Identity:** Which IDs survive reorder, navigation, window moves, restart,
   and session restore? Where would foxread need its own IDs, and how should
   uncertain correspondence be represented?
3. **Fields:** Which timestamps and references have an exact source meaning?
   Keep observation time distinct from actual tab-open time, and a URL's global
   visit history distinct from a visit attributable to a particular tab.
4. **Projection:** Are fields separate files, one tab document, or a mix? Is
   `<window>/<tab>` the identity-bearing store or a navigational view of a
   separately identified tab?
5. **Application:** How are changes applied and persisted through kopacetic?
   What can a reader observe during a partial batch or after a crash?
6. **Closure:** What represents a tab no longer present in a valid source
   snapshot? Explore closure records, retained documents, and explicit removal
   rather than implicitly adding a pruning policy.
7. **Restart:** What state must survive so a restarted observer does not assign
   unrelated identities or claim unseen events occurred?
8. **Reuse:** Which occhre components are already generic and tested, which are
   tied to OpenCode's schema, and what is merely proposed work?

## Expected research payoff

The investigations should replace guessed storage and API details with cited
evidence, expose the choices that affect the shape of the program, and identify
the smallest useful end-to-end implementation. Source inspection and design
documents are evidence for that choice; they are not a running integration.
