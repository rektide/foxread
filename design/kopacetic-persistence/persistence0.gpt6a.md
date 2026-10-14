---
type: Research
title: Firefox observed-state persistence through Kopacetic and Caminus
description: Source-grounded integration boundaries, identity and filesystem choices, recovery semantics, and a first Rust workspace slice.
resource: /design/kopacetic-persistence/persistence0.gpt6a.md
status: draft
generated:
  by: model:openai/gpt-6-astra#medium
  at: '2026-10-14'
sources:
  - resource: opencode://ses_ec59e2b0affeV7AL7VOyAXa2Vo
    title: Independent persistence investigation and proposed integration boundaries commissioned by the parent session
  - resource: https://github.com/rektide/kopacetic/blob/1a3cfbc2f0b436e9f60d0dfb70c81d828fa63a83/crates/kopacetic-caminus/README.md
    title: Backend API, layout, ownership, consistency, and limitations
  - resource: https://github.com/rektide/kopacetic/blob/1a3cfbc2f0b436e9f60d0dfb70c81d828fa63a83/crates/kopacetic-caminus/src/lib.rs
    title: Actual collection operations and identity encoder
  - resource: https://github.com/rektide/kopacetic/blob/1a3cfbc2f0b436e9f60d0dfb70c81d828fa63a83/crates/kopacetic-core/src/domain/mod.rs
    title: EntityIdentity, Entity, and provenance definitions
  - resource: https://github.com/rektide/kopacetic/blob/1a3cfbc2f0b436e9f60d0dfb70c81d828fa63a83/design/path/README.md
    title: Existing path architecture and gated Caminus adoption
  - resource: https://github.com/rektide/facet-caminus/blob/967191746c78fb158b98db06b6304f3a91ef2998/crates/facet-caminus/src/serializer.rs
    title: Active legacy filesystem reconciliation and write reports
  - resource: https://github.com/rektide/facet-caminus/blob/967191746c78fb158b98db06b6304f3a91ef2998/crates/facet-caminus/src/watcher.rs
    title: Watch APIs and notification delivery limitations
  - resource: https://github.com/rektide/facet-caminus/blob/967191746c78fb158b98db06b6304f3a91ef2998/crates/facet-caminus/src/watchman.rs
    title: Actual Watchman protocol lifecycle and omitted recovery signals
  - resource: https://github.com/rektide/facet-caminus/blob/967191746c78fb158b98db06b6304f3a91ef2998/design/path/README.md
    title: Dormant format-v1 and rooted-publication repair gate
verification:
  method: Read local implementation, manifests, README contracts, and prior design indexes; no runtime experiment or compilation.
  scope: Source files correspond to the cited checkout HEADs; both dependency checkouts had unrelated Beads JSONL changes, left untouched.
---

# Firefox observed-state persistence through Kopacetic and Caminus

## Recommendation and input boundary

Start with **one complete, typed profile snapshot stored through
`CaminusStore::put_value`**, with windows and tabs keyed by safe application IDs.
Acquire source snapshots on filesystem invalidation; compare normalized values;
let Caminus compute changed leaves. This gives field-per-file output using the
existing native backend, without inventing a CDC pipeline or modifying either
dependency. A whole-profile value is also the simplest deletion/restart unit.

The key choice still needing human agreement is the visible root: Kopacetic's
native collection root is encoded, not literal `<window>/<tab>`. Within its
`data` root we can supply the desired tree shape. A clean external view or direct
Caminus root is a separate, explicit choice below.

This inquiry assumes another source adapter can deliver a **validated complete
observation for one profile**, plus source revision/fingerprint, observation time,
and field provenance/availability. It does not establish which Firefox files
contain those fields, whether Firefox emits durable tab IDs, or whether creation
and visit times are recoverable. Those questions belong to the independent
Firefox/occhre investigation. No same-wave research was read.

The intended value is the latest observed disk state, which may lag the browser.
Neither notifications nor a snapshot diff prove every intermediate browser event.
An unavailable/invalid source is not an empty profile.

## Existing implementation: exact integration route

Local source homes:

- `/home/rektide/src/kopacetic/crates/kopacetic-core` — identity and reflected
  entity envelope; no Kubernetes needed with default features disabled.
- `/home/rektide/src/kopacetic/crates/kopacetic-caminus` — native collection backend.
- `/home/rektide/src/facet-caminus/crates/facet-caminus` — serializer, parser,
  filesystem diff, watcher, and low-level `Caminus` handle.

`kopacetic-caminus/Cargo.toml:10–17` disables facet-caminus default features and
forwards optional `watchman`. Both projects pin Facet `=0.50.0-rc.5`.
The Kopacetic dependency points at the sibling facet-caminus checkout. An initial
foxread workspace can use local path dependencies to core and the backend and
pin this same Facet version. Commit its lockfile when implementation begins;
the backend README documents compio dependency-version coupling, so do not infer
release reproducibility from a successful local resolution.

Actual call chain (`kopacetic-caminus/src/lib.rs`):

1. `CaminusStore::at(root)` constructs a synchronous handle without I/O (52–58).
2. `entity_path(&EntityIdentity)` computes the private tree root (82–89).
3. `put_value(&identity, &snapshot)` calls `FilesystemDeser::write` (116–123).
4. `get_value::<ProfileSnapshot>(&identity)` returns `Option<T>` (134–143).
5. `delete(&identity)` recursively removes a whole entity (150–177).
6. `entity(&identity)` exposes a configured low-level `Caminus` (96–105).

`put(&Entity<T>)` writes the full `identity/provenance/payload` envelope;
`put_value` writes exactly the supplied reflected value, without an envelope.
Prefer the latter for deliberate visible fields. Ordinary standalone application
types deriving `Facet` are sufficient; Kopium, kube, Splay, and Turso are not
prerequisites. Core's provenance enum has only Standalone and Kubernetes variants;
Firefox observation provenance belongs in the application value.

Conceptual integration (proposed types, not compiled):

```rust,ignore
let identity = EntityIdentity {
    api_version: Some("foxread/v1".into()),
    kind: "ProfileSnapshot".into(),
    namespace: None,
    name: profile_key.clone(),
};
let store = CaminusStore::at(output);
let report = store.put_value(&identity, &snapshot)?;
let root = store.entity_path(&identity);
```

All collection methods are blocking. A dedicated single-writer worker is a clean
boundary if acquisition uses an async runtime. Enabling compio does not turn them
into async methods. `Caminus::at` starts with `commit: None` (`store.rs:70–78`);
ordinary collection writes do not commit. Runtime Git checkpoints remain outside
this commission.

## Document, entity, and node identity

There are three distinct identities:

1. **Source observation**: profile, source revision/fingerprint, observation
   time, and coverage. It identifies evidence, not a durable tab.
2. **Kopacetic entity**: `api_version`, `kind`, `namespace`, `name`
   (`core/src/domain/mod.rs:3–20`). It selects one persisted document/tree.
3. **Structural node**: field name, map key, or sequence index within that value.
   Caminus does not automatically allocate persistent IDs to these nodes.

Do not key tabs by URL or title: duplicate URLs are normal and both fields mutate.
Do not treat vector position as identity: insert/reorder shifts every subsequent
index. Use a map keyed by application ID and persist `position` separately for
ordering. Scope IDs to a profile and, when necessary, a source-session epoch.

If the source proves stable IDs, retain their raw form as values and encode them
as safe keys. If it provides only positions, use explicitly snapshot-local IDs
in the first slice; do not imply stability across snapshots. A later matching
algorithm needs persisted allocation state, ambiguous-match handling, and tests
for duplicate tabs, moves, and restore. Matching by URL alone is insufficient.
Identity quality is a gate for claiming meaningful tab-open/tab-close deltas.

Suggested leaf fields are `id`, `url`, `title`, `position`, supported source
timestamps with explicit units, `referrer`, and observation metadata. Keep
`first_observed_at` distinct from `opened_at`; a history visit time does not by
itself establish when this tab opened. Use optional/availability-qualified values
rather than fabricated timestamps or empty-string sentinels. Facet support for
foreign time/URL types requires deliberate adaptation; integer timestamps plus
unit-documented schema, or canonical strings, avoid that initial dependency.

Legacy parser/serializer representation needs a round-trip fixture for nullable
text: `None` serializes to `null`, and strings are raw file content
(`serializer.rs:209–229`; `parser.rs:1–12`). Do not assume arbitrary option/string
combinations are lossless without testing literal `"null"`, empty strings, and
newlines. An explicit tagged availability enum is an alternative if needed, at
the cost of extra directory components.

## Filesystem mapping choices

Kopacetic encodes framed identity bytes as lowercase hex, in chunks of at most
120 characters, ending in `data` (`lib.rs:191–224`, `82–89`). This protects identity
components, **not payload map keys**. Active Caminus uses raw map/field names.
Choose bounded lowercase ASCII ID keys such as `w-000001` and `t-000001`, or a
reversible bounded encoding of actual IDs. URLs and titles stay in file contents.
Arbitrary metadata keys must not be directly mapped into directories.

### A. One profile entity, hierarchical payload — recommended first

```text
<store>/v1/<encoded-profile-identity>/data/
  schema_version
  profile_id
  observation/...
  windows/<window-key>/
    id
    position
    tabs/<tab-key>/
      id
      url
      title
      position
      ...supported metadata...
```

One write supplies all membership, so removed windows/tabs become stale paths
within the same owned tree. No collection enumeration or separate entity index
is required. Costs: full reflection/diff planning for each changed observation;
moving a tab between windows changes its path even if its `id` remains stable.

An exact `<window>/<tab>/` suffix is possible using a root
`BTreeMap<WindowKey, BTreeMap<TabKey, Tab>>` as `put_value`'s value, but then window
and profile metadata need another location. The explicit `windows/.../tabs/...`
shape preserves those domains cleanly. These are visible-format choices, not
limitations of the collection identity API.

### B. One entity per window or tab

Separate `Tab` entities can remain stable across window moves, with membership
stored as references or fields. Updates inspect less data per entity. However,
the encoded roots are not a navigable window/tab hierarchy; collection `list`,
`decode_path`, atomic batches, and collection watchers do not exist. A durable
application inventory is then necessary to know what to delete after restart.
Window entities are a useful middle ground, but still need an inventory of windows.

### C. Human-facing export root

Direct `facet_caminus::Caminus::at(export_root).write(&tree)` gives a literal root
chosen by foxread, but bypasses Kopacetic's identity-addressed store wrapper.
Alternatively, keep A canonical and expose its `entity_path` through CLI output
or a deliberately managed convenience link outside the owned tree. A second
materialized export introduces another projection and another recovery boundary.
Do not silently maintain both just to hide the encoded address. Choose C only
if a literal top-level path outweighs using the collection API directly.

## Snapshot, changes, deletion, and restart semantics

`FilesystemSerializer::finalize` plans a diff before execution. Byte-identical
leaves are left untouched; stale non-dot entries are pruned. Empty containers are
represented by directories. `WriteReport` contains relative written/removed
files and created/removed directories (`serializer.rs:111–162`, `191–201`,
`239–260`). It is a filesystem report, not a domain tab-event log.

Important implementation detail: the report is built **from the plan before
execution**, and the synchronous executor ignores `remove_file`/`remove_dir`
errors (`serializer.rs:123`, `144–149`). Therefore a successful report is not
proof that every stale path was actually deleted. If exact mirror verification
matters, check the resulting owned paths, including absence; typed readback alone
may miss surplus fields. This is a more specific limit than the README's
high-level idempotence language.

Recommended ingestion state machine:

1. Subscribe to source invalidations, then acquire a complete initial snapshot;
   drain any queued invalidations and reacquire if needed.
2. On a relevant invalidation, debounce/coalesce and read a fresh complete source
   snapshot using the source adapter's consistency checks.
3. Normalize into a candidate, validate coverage and identity, compare semantic
   content with the last accepted observation. Avoid changing every tab's
   timestamp merely because the source was reread.
4. Persist the complete candidate through one serialized writer. Only advance the
   in-memory accepted baseline after success; retain enough candidate state to
   retry a partial write idempotently.
5. On acquisition failure, retain the last valid export and report stale status.
   On startup or explicit rescan, reacquire full authoritative state rather than
   treating the output as the sole source of truth.

The user requested maintained windows/tabs, so removal of absent members can be
part of that explicitly scoped mirror operation. Absence means deletion only
after a successful **complete** profile observation. An unavailable source,
partial source read, or disappearance of a watched input file is not evidence
that every tab closed. Do not add unrelated cleanup or retention policy.

For A, full replay repairs a partially reconciled export once valid source state
is available again, and can remove stale members without remembering the previous
snapshot. If the source remains unavailable, the previous output may itself be
partial: expose that uncertainty rather than claim a recovered snapshot.
For B, a failed write followed by a missing inventory update can orphan entities;
there is no supported backend scan to recover them. This is a principal reason
to defer per-tab entities.

Current backend guarantees are **idempotent best-effort in-place reconciliation**,
not atomic publication or crash durability. There is no writer lock, fsync
protocol, generation swap, read snapshot, or cross-entity transaction. Serialize
writers and treat live readers as potentially seeing mixed generations.
If coherent concurrent reads become required, design a separate generation
publication protocol with pinned-reader semantics; a marker file or debouncing
alone does not provide that guarantee.

Deletion of an entire profile uses `CaminusStore::delete` only as an explicit
operation. It removes dot metadata too and bypasses Caminus report/history policy.
Routine profile unavailability must not trigger it.

## Watchman: reusable notification surface, not durable CDC

The existing `FilesystemWatcher` can watch any existing directory and return
`FileChange` batches; it does not require the directory contents to be a Caminus
tree for `wait_changes`/`poll_changes`. That makes it a plausible initial source
invalidation adapter through Kopacetic's re-export. Its `sync`/`reload` methods,
however, parse a Caminus-serialized typed tree and cannot decode Firefox files.
The Firefox decoder remains a separate module.

`watch_watchman` is Unix/feature gated; `watch_auto` tries Watchman and then notify
(`watcher.rs:193–215`). `backend()` exposes the selected backend. Use an explicit
CLI policy so “Watchman required” fails on unavailable setup and “auto” reports
fallback. Watch the stable containing source directory so replacement of an input
file can invalidate the snapshot. Keep output outside the watched source tree.

Source inspection reveals recovery limits beyond the method documentation:

- `watchman.rs:225–243` subscribes from a newly acquired clock; it has no public
  durable cursor/resume interface.
- `Pdu` parses `is_fresh_instance`, but `forward` only forwards file entries,
  dropping clock and fresh-instance status; empty lists emit nothing (181–200).
- Socket/protocol error ends the reader thread without a reconnection loop or an
  explicit error event (250–260).
- `wait_changes` treats timeout and disconnected channel alike as an empty batch;
  notify callback errors are discarded (`watcher.rs:131`, `239–247`).

Consequently this API is suitable as a **best-effort wakeup**, not the sole
correctness mechanism for a continuously maintained mirror. A periodic full
source check and subscription recreation can bound missed updates in an initial
adapter; expose the poll fallback interval/policy. A stronger source watcher
should surface `Invalidated`, `RescanRequired`, and `Disconnected` explicitly,
with clock handling and reconnect behavior. That can be a foxread adapter or a
separately commissioned upstream enhancement. CDC should wait for evidence of an
actual reliable record-change source; these file events do not supply one.

## Rust workspace boundaries and implementation slice

Two plausible domain-grouped layouts:

**Compact four-crate workspace**

```text
crates/foxread-model/          # normalized observations, IDs, semantic diff
crates/foxread-firefox/        # source acquisition, parsing, invalidation adapter
crates/foxread-persistence/    # Facet schema, Kopacetic identity, Caminus sink
crates/foxread-cli/            # configuration, dump/watch/rescan orchestration
```

**Capability-separated workspace**, if watch supervision is reusable:

```text
crates/model/foxread-model/
crates/source/foxread-firefox/
crates/source/foxread-watch/       # Watchman/notify lifecycle and rescan signals
crates/persistence/foxread-caminus/
crates/app/foxread/               # binary and coordinator
```

Prefer the compact layout initially. The model should not depend on a watcher or
storage backend. The source emits observations and coverage, the persistence
adapter owns its versioned reflected schema, and the coordinator owns retries and
when absence authorizes mirror removal. For a small first schema, deriving Facet
on model types is simpler; a separate persisted DTO becomes worthwhile when
source/domain evolution must not silently migrate the filesystem format.

**Best next coherent slice:** a one-shot `dump` from a validated fixture or source
snapshot into one profile entity, printing the actual `entity_path` and write
report. Settle profile identity, source ID fidelity, nullable text representation,
and the visible field layout in that slice. Meaningful verification:

- typed write/read round trip, including supported metadata and nullable edge cases;
- identical second write yields no changes and preserves unchanged leaf mtimes;
- URL/title update changes the expected leaves without changing tab identity;
- reordered tabs preserve keyed paths; duplicate URLs stay distinct;
- complete snapshot removal eliminates the old owned tab/window paths;
- incomplete/invalid acquisition leaves the last export untouched;
- startup full replay repairs an intentionally interrupted/partial output.

Only then add a `watch` coordinator around the same acquisition and publication
operations, with Watchman preference and missed-event recovery. This slice tests
the advanced Kopacetic integration before committing to a notification strategy.

## Connections to existing work and remaining decisions

- [Kopacetic path architecture](https://github.com/rektide/kopacetic/blob/1a3cfbc2f0b436e9f60d0dfb70c81d828fa63a83/design/path/README.md)
  is precedent for keeping logical paths, optional Splay, and native backends
  separate. Foxread does not need to build an intermediate Splay model.
- [Caminus path-format checkpoints](https://github.com/rektide/facet-caminus/blob/967191746c78fb158b98db06b6304f3a91ef2998/design/path/README.md)
  constrain this proposal: v1 codec and rooted I/O work exist but remain dormant
  in the active serializer. Do not assume their protections are already enabled.
- [Backend consistency and ownership](https://github.com/rektide/kopacetic/blob/1a3cfbc2f0b436e9f60d0dfb70c81d828fa63a83/crates/kopacetic-caminus/README.md#semantics)
  ground the single-writer, whole-profile choice and explicit deletion boundary.

Decisions for integration after source research: exact versus logical window/tab
hierarchy; stable versus snapshot-local IDs; authoritative coverage definition;
which timestamps/referrer fields have evidence; whether best-effort live reading
is sufficient; and whether Watchman is required or preferred. None requires
adding Git checkpoints now.
