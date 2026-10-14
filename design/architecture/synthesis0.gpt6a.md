---
type: Design
title: Snapshot-first Firefox observation through kopacetic
description: Integrate Firefox store evidence, occhre reuse, Watchman lifecycle, and caminus persistence into a proposed multi-crate starting point.
resource: /design/architecture/synthesis0.gpt6a.md
status: proposed
generated: { by: "openai/gpt-6-astra#high", at: 2026-10-14 }
sources:
  - resource: "opencode://ses_ec59f6862ffeFt6UN2FUOhTXOO"
    title: Commission independent research, inspect integration boundaries, and synthesize the initial architecture
  - resource: /design/intent/brief0.gpt6a.md
    title: User goals and unresolved persistence questions
  - resource: /design/firefox-stores/stores0.gpt6a.md
    title: Firefox formats, identity and timing limits, Places, and occhre's reuse boundary
  - resource: /design/kopacetic-persistence/persistence0.gpt6a.md
    title: Native backend API, layout options, write behavior, and watcher limitations
  - resource: /design/watching/watchman0.gpt6a.md
    title: Official Rust Watchman client and observation-loop design
  - resource: https://github.com/rektide/kopacetic/blob/1a3cfbc2f0b436e9f60d0dfb70c81d828fa63a83/crates/kopacetic-caminus/src/lib.rs
    title: Parent inspection of put_value, entity_path, and deletion boundaries
  - resource: https://github.com/rektide/facet-caminus/blob/967191746c78fb158b98db06b6304f3a91ef2998/crates/facet-caminus/src/serializer.rs
    title: Parent inspection of in-place execution and planned write reports
  - resource: https://github.com/rektide/occhre/blob/9d31b09d8db724ebd7ec4ff774a5f76735428728/README.md
    title: Parent inspection of implemented incremental extraction and unfinished live-retention join
verification:
  method: Research synthesis and selected local API/source inspection; no program implemented or compiled.
  limits: Firefox evidence is pinned upstream main, not a verified installed-version profile; persistence behavior needs fixture-driven integration tests.
---

# Snapshot-first Firefox observation through kopacetic

## Recommendation

Build the first vertical around **sessionstore snapshot → typed observation →
kopacetic/caminus field tree**, then watch and compare observations. Add SQLite
Places enrichment separately. This follows the user's dump/diff option while
keeping occhre reuse available for the part that really is SQLite.

The research has resolved the storage guess but exposed two genuine design
choices: persistent tab identity cannot be recovered reliably from these files,
and the existing filesystem backend does not publish atomic generations.

## What the independent investigations establish

| Concern | Evidence and consequence |
|---|---|
| Open windows/tabs | Firefox writes MozLZ4-compressed JSON sessionstore snapshots, primarily runtime `sessionstore-backups/recovery.jsonlz4`. They are delayed restoration state. |
| SQLite | `places.sqlite` stores pages and visits. It supplies visit times and partial referrer relationships, not stable live tab identity. |
| Cadence | Upstream defaults include a 15-second active minimum interval and an hour idle interval. Snapshot differences cannot report activity that never reached disk. |
| Identity | No general persistent live window/tab ID is serialized. History-entry IDs, URL, title, and array position are not substitutes. |
| Filesystem sink | `CaminusStore::put_value` accepts a Facet-reflected value. It maps fields to files under an encoded entity root and updates changed leaves in place. |
| Notifications | Official `watchman_client` exposes clocks, fresh instances, and disconnects. The existing caminus wrapper drops some of that information. |
| SQLite reuse | occhre already has retention, WAL, read-view, row-reader, and net-change crates. Its incremental extractor exists; the general live acquisition/retention join remains unfinished. |

The Firefox report's strongest contribution is identifying what the source can
actually mean. The persistence report establishes the usable API and its failure
boundaries. Their suggested first slices differ on immutable versus in-place
publication; that is an architectural choice, not two descriptions of the same
backend capability.

## Proposed workspace

Use five crates with dependency direction toward a source-independent model:

```text
crates/
  foxread-model/          # observations, coordinates, availability, structural diff
  foxread-sessionstore/   # file roles, bounded MozLZ4 decoder, source normalization
  foxread-observe/        # invalidation, accepted baseline, retries, watch lifecycle
  foxread-caminus/        # reflected output schema, profile entity and sink
  foxread/                # CLI and composition
```

`foxread-sessionstore` and `foxread-caminus` consume model types; neither owns the
other. `foxread-observe` coordinates observations and sink completion through
narrow interfaces. The CLI selects/configures adapters. Later
`foxread-places` can add Rust SQLite reads without changing sessionstore parsing.

A smaller four-crate alternative merges observation scheduling into a broader
`foxread-firefox` source crate; this is viable if there is only one source. Prefer
five here because Places and a reusable SQLite observation path are already
desired directions. Avoid adding a generic SQLite crate before using occhre's
existing boundaries in a controlled experiment.

## Filesystem and identity proposal

Start with one profile entity and a versioned payload:

```text
<store>/v1/<encoded-profile-identity>/data/
  schema_version
  profile_id
  observation/
    source_generation
    source_kind
    source_snapshot_at
    observed_at
    coverage
  windows/w-0000/
    position
    title
    tabs/t-0000/
      position
      url
      title
      last_selected_at
      pinned
      ...supported fields...
```

Here `w-0000`/`t-0000` are **coordinates within an observation**, not stable IDs.
Their full reference includes profile and generation. If an `id` leaf is exposed,
it must identify this observation coordinate, with that scope documented. Reorder
can change the referent at a path. Exposing a structural replacement is honest;
labeling it a tab close/open would not be.

Keep generation identity distinct from semantic equality: source bytes/timestamps
may change without a meaningful projected tab change. Domain diffs should compare
selected content, not manufacture changes from every reread's `observed_at`.
Keep supported source times separately: last selection is not opening, observation
is not navigation, and a URL-level visit is not proven to belong to this tab.

Use safe bounded application keys for paths; store URLs/titles as values.
Unknown fields should not become arbitrary filesystem components. Prototype the
optional-string representation against caminus, including literal `null`, empty
strings and newlines, before choosing the persisted schema.

The exact `<window>/<tab>/` spelling is possible as a nested map, but explicit
`windows/.../tabs/...` makes room for profile/window metadata. Print the actual
entity data path. A literal friendly root is a separate export or layout choice;
the current kopacetic wrapper adds the encoded entity address above the payload.

### If stable handles matter immediately

A foxread-owned identity ledger could match across snapshots using history,
container, and structural context. It needs explicit ambiguous/unmatched results,
restart persistence, and tests for identical duplicate tabs and window moves.
It cannot turn uncertain evidence into exact browser lifecycle events. Exact live
identity would call for an additional browser-side/API source. That changes the
disk-only premise and should be chosen deliberately.

## Publication and watching

Recommend an explicit **current observed-state projection** for the first slice,
using the existing in-place sink. A successful complete source observation can
replace membership in that owned tree; making this a current mirror also means
opting into caminus's existing stale-field removal. Source failure must preserve
the last accepted observation, not request an empty-tree write.

Readers may see mixed generations during a write. The serializer builds its
report from a plan and ignores some removal errors; do not equate its report with
verified final membership. The integration test must inspect expected removals.
A failed application retains the candidate for replay and does not advance the
accepted baseline. In-place failure can still leave partial output; keeping an
in-memory baseline does not undo that.

The Firefox research proposes immutable generation publication as a stronger
alternative. It is worthwhile if coherent concurrent reads or retained snapshots
are required, but needs a staging/completion protocol and a selected-generation
pointer. A unique directory alone is not atomic publication. Do not imply the
current `put_value` supplies that protocol, nor quietly add retention/pruning.

For watching, prefer the official Rust Watchman client directly for source
invalidation, retaining fresh-instance/cancellation/error signals. Establish the
subscription before reading the baseline, drain notifications into dirty state,
and reacquire when an event races a read. On reconnect or fresh instance, read
the source again. Parsing and application run separately from notification drain.
No Watchman executable was found on this session's PATH; daemon availability is
an implementation setup item.

Begin with an explicit input file for deterministic dumps. Profile mode must
preserve the roles of recovery, clean-shutdown and older backups, rather than
selecting arbitrary newest mtimes. Clean-shutdown data represents restorable
state, not proof of open windows. Missing/unreadable/encrypted input is a source
status, not a closure event. The research identifies upstream `mozEnc0\0`
encryption support; initial decoding should distinguish that from malformed LZ4.

## SQLite and extraction path

For optional Places enrichment, first use Rust SQLite reads in a short read-only
transaction. SQLite should interpret main and WAL together. Do not independently
copy live main/WAL files or treat a visit-ID watermark as complete CDC. Keep
page/visit identities and microsecond visit times distinct from sessionstore's
observation coordinates and millisecond times.

The occhre reuse path is documented in the
[Firefox investigation](../firefox-stores/stores0.gpt6a.md#occhre-actual-reusable-architecture-and-remaining-obligations):
`walpin → waltail → walview → sqlite-records → sqlite-changes`, with transport
coupling through `fch-arrow`. The useful extraction proposal is a shared
transport-independent SQLite value/key/schema/change model, followed by existing
retention/read/extraction crates. Reuse them before copying correctness-sensitive
WAL machinery. A WAL pin affects checkpoint retention; a file watcher is not a
substitute. Publication acknowledgement and permission to advance retention are
separate contracts.

## First implementation and the choices to settle

The next coherent implementation is a fixture-backed one-shot dump across the
five crates: explicit file → bounded decoder → typed observation → one profile
entity. Verify back navigation's 1-based index, duplicate tabs, absent histories,
supported metadata, unsupported encryption, nullable strings, idempotent writes,
and expected field removal. Then add `watch` using exactly that read/apply path
and verify recovery after failed reads, failed writes and disconnection.

Before that implementation, select:

1. **Identity:** observation-scoped coordinates initially (recommended), or an
   explicitly heuristic persistent-ID ledger now.
2. **Publication:** current in-place profile projection (recommended first), or
   immutable generations with a deliberate publication protocol.
3. **Visible layout:** native entity root plus `windows/.../tabs/...`
   (recommended), or a separately designed literal root/export.

These are proposals awaiting direction, not completed functionality. Git
checkpoints remain the deferred extension from the original brief.
