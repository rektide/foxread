---
type: Design
title: Domain-shaped crates and verified full-state writes
description: Revise foxread's crate layout around Firefox acquisition, interpretation, observation, and filesystem output after human review.
resource: /design/architecture/layout1.gpt6a.md
status: proposed
generated: { by: "openai/gpt-6-astra#high", at: 2026-10-14 }
sources:
  - resource: "opencode://ses_ec59f6862ffeFt6UN2FUOhTXOO"
    title: Human selection of in-place full-state writes and typed IDs; revised crate walkthrough and walcap delegation
  - resource: /design/architecture/synthesis0.gpt6a.md
    title: Predecessor synthesis and source evidence
  - resource: "file:///home/rektide/src/facet-caminus/crates/facet-caminus/src/serializer.rs"
    title: Desired tree, private diff planner, stale-path pruning, and report construction inspected during revision
  - resource: "file:///home/rektide/src/facet-caminus/crates/facet-caminus/src/manifest.rs"
    title: Existing private metadata/hash cache and its non-receipt semantics
  - resource: "file:///home/rektide/src/facet-caminus/crates/facet-caminus/src/lib.rs"
    title: Full-value write API and existing optional jj/git commit hook
---

# Domain-shaped crates and verified full-state writes

## Direction after review

The user has read the initial synthesis and generally agrees with it, selecting:

- Submit the entire desired value to caminus each cycle; let persistence decide
  which filesystem operations are necessary.
- In-place updates are acceptable. Track intended output and compare with actual
  disk state after writing, including obsolete directories.
- Use `win_<id>` and `tab_<id>` as type-bearing IDs and path keys.
- Keep the current foxread effort on JSON sessionstore. Standalone SQLite/WAL
  extraction design and tickets now belong to `~/src/walcap`.
- Organize crates under useful top-level domains, without redundant `foxread-`
  directory prefixes. Explain the responsibilities before implementing.

This supersedes the five-crate layout proposal in the
[initial synthesis](synthesis0.gpt6a.md). The source-format and identity evidence
there remains applicable. A later jj snapshot follows a completed output cycle;
this revision does not implement automatic checkpointing.

## Proposed nine-crate workspace

```text
core/
  model/
  diff/
firefox/
  sessionstore/
  profile/
  project/
observe/
  watch/
  runner/
output/
  caminus/
app/
  cli/
```

Each leaf is a crate directory with its own `Cargo.toml` and `src/`. Cargo package
names can retain the `foxread-` namespace to avoid dependency-name collisions;
the directory and executable names need not repeat it. These are proposed
boundaries with independent contracts, not a target number of packages.

### Core: what an observation means

**`core/model`** owns normalized profiles, windows, tabs, history entries where
selected, typed ID wrappers, ordering, source provenance and field availability.
It is independent of Firefox's JSON layout, file watching, and kopacetic. An ID
and a position remain separate fields even if an initial allocator uses positions.

**`core/diff`** compares normalized observations. It produces structural additions,
removals and field changes with old/new observation references. It does not read
directories or dictate write operations. It must not turn ambiguous structural
replacement into an asserted tab-open/tab-close event. This is useful independently
for comparing two dumps, testing projection, and reporting watch results.

### Firefox: acquisition and interpretation

**`firefox/sessionstore`** is a pure bytes-to-source-data library. It owns bounded
MozLZ4 decoding, plain JSON fixture input, encrypted-input classification, format
compatibility, and source-shaped Rust structures. It understands the serialized
format without selecting profiles or consulting the filesystem.

**`firefox/profile`** owns environmental acquisition: explicit input/profile paths,
source-file roles, reading candidates, and relevant watched paths. It distinguishes
runtime recovery, clean shutdown, backup fallback and unavailable input. Start with
explicit paths; discovery can enter here later without changing the decoder.
This split isolates lifecycle/race tests from pure codec tests.

**`firefox/project`** translates source-shaped data into the core model. It chooses
the active entry using Firefox's 1-based index, distinguishes current tabs from
undo-close records, names timestamps honestly, and assigns observation coordinates.
Projection policy is separate from format compatibility: successfully parsing a
new Firefox field does not automatically expose it in the persisted model.

### Observe: triggers and progress

**`observe/watch`** adapts the official Watchman client into invalidation,
rescan-required and disconnected signals. It owns watch-root/path handling and
notification lifecycle, not Firefox parsing. Its contract permits a visible
polling backend later; there is no need to implement every backend initially.

**`observe/runner`** owns the read/project/apply/verify cycle: initial baseline,
coalesced invalidations, serialized application, retries, last accepted state and
pending candidate. It consumes small acquisition/output interfaces. Keep them
close to this consumer rather than introducing a generic plugin framework.
It emits a completed-cycle result only after verification; that is the future
checkpoint boundary. Slow persistence must not stop the watcher draining events.

### Output: the materialized tree

**`output/caminus`** owns the versioned reflected filesystem schema, kopacetic
profile entity, full-value writes, desired inventory, and post-write verification.
It maps typed IDs to paths and exposes missing, unexpected, wrong-kind and
wrong-content entries. Its job includes the actual owned directory set, not only
successful file writes. The runner sees a checked result rather than trusting
`WriteReport` as proof of final state.

Start with `schema`, `inventory`, `apply`, and `verify` modules here. A generic
tree-planning API belongs in caminus if it is useful across callers; do not invent
a second serializer in foxread simply to make inventory collection another crate.

### App: composition

**`app/cli`** provides the `foxread` binary, configuration, diagnostics, and command
composition: one-shot dump first, then watch and explicit verify/diff commands as
their libraries become usable. It wires concrete adapters; it does not own parsing
or lifecycle rules. Unit-testable functionality stays in the libraries above.

## Full-state persistence contract

There are three different comparisons:

1. **Observation difference:** prior normalized browser state versus candidate.
2. **Write plan:** desired encoded tree versus disk before application.
3. **Verification difference:** the same desired tree versus disk after application.

Pass the full value to caminus even though it can avoid rewriting identical files.
Domain differences explain the browser observation; they are not the writer's
source of truth. After restart, a full desired value can repair output without
replaying the entire history of domain changes.

The desired inventory needs relative paths, entry kinds including empty
directories, and expected file contents or content digests. It must come from
the **same encoding** that supplies the write. The re-inspected caminus code
already constructs desired file/directory sets internally, but does not expose
a public desired-tree/plan API. Its existing `.caminus/manifest` is a private
size/mtime/hash optimization cache, not a verified completion record.

The preferred upstream extension is to expose the serializer's desired tree so
planning, execution and verification share it. That API is proposed, not present.
For a prototype, materializing an expected tree in scratch through the same
serializer is an alternative, but doubles I/O and should not become the long-term
architecture. Typed readback alone cannot detect every stale extra path.

Existing caminus code **does** plan stale-file and deepest-first directory removal.
The concrete gap is that some removal errors are discarded and reports are built
from the plan. Verification should detect leftover old `win_*`/`tab_*` trees and
return an incomplete cycle, retaining the candidate for repair. Limit comparison
to the declared owned non-dot tree; preserve the backend's reserved dot metadata.
Check contents rather than using only cached size/mtime as final evidence.

Capture an output receipt with candidate/source identity, owned root, desired
inventory and verification result. A durable last-verified receipt can support
inspection/restart, but is evidence of a past check, not proof the tree has not
subsequently changed. A fresh verification handles that distinction.

Existing caminus has an optional jj/git commit hook, but it runs on write completion.
For this design, future checkpointing should occur after the stronger verification
boundary instead. With a single output writer, a completed cycle gives a sensible
snapshot to commit. Concurrent external edits would require coordination to claim
that the commit exactly matches the verified inventory.

## IDs and remaining growth

Use the same type-bearing value in the map key and ID leaf, for example
`win_0001` and `tab_0017`. Tab allocation can be profile-wide within an observation
so a short tab ID does not repeat in each window. The containing observation still
supplies the scope: adding a prefix does not establish cross-snapshot continuity.
The exact allocator and persistent-matching policy remain open.

If stable foxread-owned handles are selected, a tenth crate **`core/identity`** can
own the matching ledger, confidence and ambiguity results. Firefox-specific match
hints would come from projection. Mere formatting/allocation remains in model and
projection until that responsibility exists. A separate output-audit crate is
justified only once the inventory protocol is actually shared beyond caminus.

An alternative top-level vocabulary is `core/`, `input/firefox/`, `observe/`,
`output/`, `app/`, retaining the same leaf contracts. Prefer `firefox/` now because
it names a concrete domain and groups acquisition, codec and interpretation
without forcing them into extract/transform/load categories.

## First vertical

Implement bytes/fixture → sessionstore → projection → model → full caminus write
→ disk verification through the CLI. Add core diff and watch/runner behavior as
the second coherent slice. The workspace map can be agreed now without filling
every proposed crate with scaffolding before its behavior exists.
