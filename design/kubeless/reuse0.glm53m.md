---
type: Research
title: Kubeless reuse for foxread — verified comparison mechanisms and honest diff verdicts
description: Deep survey of ~/src/kubeless (67 crates) against foxread's three comparisons, with executed external-probe evidence for the verification and observation-diff reuse candidates.
resource: /design/kubeless/reuse0.glm53m.md
status: research
generated: { by: "zai/glm-5.3#max", at: 2026-10-14 }
sources:
  - resource: "opencode://ses_ec3c6dcbbffe8ZOeI8z2chI0S2"
    title: This session's survey, external probe construction, and diff verdicts
  - resource: "opencode://ses_ec59f6862ffeFt6UN2FUOhTXOO"
    title: Parent commission of the kubeless reuse investigation and foxread direction
  - resource: "file:///home/rektide/src/kubeless/view/change/src/lib.rs"
    title: Neutral change storage, checked constructors, and property laws
  - resource: "file:///home/rektide/src/kubeless/view/change/src/alignment.rs"
    title: Pure key-union alignment kernel with duplicate custody
  - resource: "file:///home/rektide/src/kubeless/observation/assessment/src/collection/mod.rs"
    title: Scoped collection profile: Observation, Expectation, Mode, Fact knowledge
  - resource: "file:///home/rektide/src/kubeless/observation/assessment/src/collection/input.rs"
    title: Fact/Expected construction and checked authored expectations
  - resource: "file:///home/rektide/src/kubeless/observation/assessment/src/collection/report.rs"
    title: Satisfaction, gaps, closure, transitions, and explicit lowering
  - resource: "file:///home/rektide/src/kubeless/observation/assessment/src/collection/assess.rs"
    title: complete_entries/assess signatures and indexed union walk
  - resource: "file:///home/rektide/src/kubeless/observation/assessment/md/collection.md"
    title: Rich collection assessment contract and 729-pair reference test claim
  - resource: "file:///home/rektide/src/kubeless/kubeless.md"
    title: Substrate matrix, semantics that must not be collapsed, current limitations
  - resource: "file:///home/rektide/src/kubeless/doc/information-model.md"
    title: Fact vocabulary, key roles, non-collapse laws, seam table
  - resource: "file:///home/rektide/src/kubeless/doc/reconciliation/maintained-projection.md"
    title: Maintained native projection composition precedent
  - resource: "file:///home/rektide/src/kubeless/doc/consumer/kopacetic-akri.md"
    title: Existing kopacetic↔kubeless integration and its scope
  - resource: "file:///home/rektide/src/kubeless/probes/observation/collection-assessment/README.md"
    title: Independent std-only consumer precedent for assessment
  - resource: "file:///home/rektide/src/foxread/design/architecture/layout1.gpt6a.md"
    title: Current foxread direction: nine crates, full-value writes, verify-what-landed
  - resource: "file:///home/rektide/src/foxread/design/kopacetic-persistence/persistence0.gpt6a.md"
    title: Caminus API, write-report limits, and watcher evidence
  - resource: "file:///home/rektide/src/kopacetic/design/consumers/foxread/handoff0.gpt6a.md"
    title: Consumer needs: know what really landed, including obsolete directories
verification:
  method: >-
    Source inspection of kubeless at jj change syyznkst (git 629424e3, empty
    working copy above) plus an executed external applicability probe in
    foxread scratch (.test-agent/kubeless-reuse) using public path dependencies.
  commands:
    - "cargo test --manifest-path .test-agent/kubeless-reuse/Cargo.toml  -> 9 passed, 0 failed (4 domain-diff, 5 verify)"
    - "cargo tree -e dev + Cargo.lock  ->  kubeless closure is exactly 3 std-only crates, zero external dependencies"
  limits: >-
    Probe covers comparison 1 and 3 shapes with a synthetic desired inventory
    and synthetic materialization; it is not a caminus integration. Its
    partial-coverage test uses a manually constructed Observation (closure
    Missing), not a partial-fact walker, and does not assert the exact
    Satisfaction variant (API-deduced). The fixture walker is lossy for
    non-UTF-8 names and treats all non-directory entries as ordinary files.
    Kubeless tests were read, not rerun, except the scratch probe; kubeless
    0.1.0 is workspace-internal and unpublished, so revision pinning is
    mandatory.
---

# Kubeless reuse for foxread

Kubeless is a soft fork of kube-rs whose control-plane machinery has been
opened into 67 independently usable, mostly std-only `kubeless-*` crates, with
Kubernetes reconstructed on top as one composition. Foxread needs almost none
of the control-plane. What it needs — two honest comparisons out of three —
already exists there as small, heavily tested, Kubernetes-free crates, and
this research verified the reuse from an external workspace rather than
inferring it from crate names.

## Summary of verdicts

The [current foxread direction](../architecture/layout1.gpt6a.md) already
separates three comparisons. Kubeless answers them unevenly:

| # | Comparison | Needed now? | Kubeless answer | Verdict |
|---|---|---|---|---|
| 1 | Domain observation diff (prior normalized browser state vs candidate) | Optional (reporting, tests); a plain equality gate suffices for the write loop | `kubeless-view-change` `alignment::align` + `ChangeSet` | **Reuse as a thin module; do not write a diff algorithm.** Defer to slice 2 |
| 2 | Backend desired-vs-current write planning | No — caminus plans writes internally from the full reflected value | `MutationBatch` targets keyed in-memory materializers, not caminus | **No reuse. Nothing here for foxread** |
| 3 | Desired tree vs actual filesystem after writing (incl. stale dirs) | **Yes — the human-selected "know what really landed" boundary** | `kubeless-observation-assessment` `collection` feature: `complete_entries` / `assess` with `Mode::Exact` | **Reuse. Executed probe evidence below; this is the strongest finding** |

So: does foxread really need a diff? It needs exactly one comparison to start
— the post-write verification — and that one is a *requirement check* (Exact
mode), not a structural diff. The structural domain diff is real but optional
and smaller than a crate: it is key-union alignment over foxread-chosen keys,
which kubeless already implements. **Recommendation: drop `core/diff` as a
proposed crate; keep a `diff` module (likely in `core/model` or
`observe/runner`) that wraps `kubeless-view-change`, created when slice 2
lands. Verification lives in `output/caminus`'s `verify` module built on
`kubeless-observation-assessment`.**

## Coverage map

Evidence that this survey looked broadly, not only at "diff":

| Area | Examined | Result |
|---|---|---|
| Workspace/manifests | Root `Cargo.toml` (67 crates, 7 domains + kube stack), README "choose an entrance" table | Lower mechanisms accept application keys/values; no Kubernetes below `kubeless-kube-*` |
| Guide/substrate | `kubeless.md`: substrate matrix (std-only / executor-free / Tokio / K8s columns), "semantics that must not be collapsed", "current limitations" | Confirms candidate crates are std-only; honest limits lists |
| Observation family | READMEs of all 14 children; source+tests for `model`, `assessment` (collection), `session`; sources for `order`, `projector`, `backoff` | Assessment collection = verification machinery; session = cursor/paged sources (poor fit); backoff = retry decisions |
| View family | READMEs of all 11; full source+tests of `view/change` (lib, alignment, neutral/alignment tests); `memory`, `impact`, `publication` READMEs | view-change = the diff algebra; memory/cache/publication = in-memory/lifecycle machinery foxread doesn't need |
| Work family | READMEs of all 8; `schedule-policy` and `schedule` detail | schedule-policy (std, caller clock) and schedule (Tokio) = coalescing/debounce for the runner |
| Control family | READMEs + one-line purposes of all 28; `driver`, `operation`, `settlement` in depth | Oriented to keyed concurrent control loops and conditional writes — overkill for a single-writer mirror; settlement is the conceptual match for verify-after-write but aimed at remote retirement |
| Operation/catalog/runtime | `operation` README + Cargo (std-only); `catalog` README; `runtime` overview | Conditional-write retry (conflict-reread) and capability descriptors — not foxread-shaped |
| Docs | `doc/information-model.md` (fact vocabulary, key roles, non-collapse laws), `doc/reconciliation/` (README, maintained-projection, maintenance refs), `doc/consumer/kopacetic-akri.md` | Information model is high-value design guidance; maintained-projection is a runnable precedent of the foxread shape (source → artifact → verify) at much heavier weight |
| Probes/tests | `probes/observation/collection-assessment` (std-only external consumer), `probes/dependency-profiles/profiles.tsv`, `probes/runtime/level-projection` README, `view/change` tests (6+6+7 incl. proptest reference folds), assessment `tests/collection.rs` (26 tests, 729-pair reference claim) | Testing patterns to imitate: independent workspaces, property tests with reference oracles |
| Kopacetic integration | `doc/consumer/kopacetic-akri.md`; `~/src/kopacetic` Cargo manifests (`kopacetic-core` optional `kubeless-kube-core`; `kopacetic-akri` `maintained`/`timed`/`handler` features compose kubeless control/observation/work crates) | Precedent: kopacetic already path-depends on kubeless behind optional features |
| Status | `design/state-reconciliation/implementation/README.md`, CHANGELOG (inherited 4.2.0 upstream line), README posture notes | Implemented mechanisms distinguished from open inquiry; packages workspace-internal, not on crates.io |
| Filesystem/watch in kubeless | `rg 'notify|watchman|inotify|std::fs|walkdir'` across library crates | None in the reusable crates (only kube client config loading and probe-local I/O). **Watch integration remains foxread-owned** |

Inspected revisions: kubeless at jj change `syyznkst` (git `629424e3`);
foxread at jj change `xusyyknt` (git `45a303fa`). Both working copies clean.

## The three comparisons in detail

### 1. Domain observation diff — real but small, and kubeless already owns it

`kubeless-view-change` (`view/change/src/lib.rs`, `src/alignment.rs`) is a
zero-dependency std crate providing:

- `Transition<K, B, A>` — one keyed row; `before`/`after` are `Option<B>`/`Option<A>`
  where **`None` means known absence, never unknown** (lib.rs doc header and
  `Transition` docs, lines 231–253). Heterogeneous sides; no `Clone` bounds.
- `ChangeSet::try_from_transitions` (Eq+Hash) / `try_from_transitions_ordered`
  (Ord only) — checked uniqueness; duplicates return **every input row** with
  positions (`Duplicate`, lib.rs 325–394).
- `ChangeSetBuilder::record` — sequential first-before/final-after coalescing law.
- `alignment::align(left, right)` — key-union alignment into
  `Aligned::{Matched, LeftOnly, RightOnly}`; duplicate rejection returns both
  full inputs (`AlignmentError`, alignment.rs 89–167). "Equality is not
  tested; matched equal payloads remain full occupancy rows" — the equality
  policy is an explicit caller `retain` (doc example, alignment.rs 127–141).
- `Affected`/`AffectedBuilder` — unique first-touch keys.

How foxread would call it: project the normalized snapshot into
`BTreeMap<TabId, Tab>` (and per-window), `align(prior.iter(), next.iter())`,
convert with `Aligned::into_transition`, check with `ChangeSet::try_from_transitions`,
`retain(|row| row.before != row.after)`. The scratch probe
(`.test-agent/kubeless-reuse/tests/domain_diff.rs`, 4 passing tests)
demonstrates exactly this and pins the honest behaviors:

- **Stable keys:** reorder is a `position` value change on one key; navigation
  is a value change; add/remove are membership rows with known absence.
- **Positional keys:** a swap looks like two value changes — churn, not a
  lifecycle event. The kernel cannot express "swap"; that is foxread's honesty
  requirement working as intended, not a limitation to engineer around.
- **Duplicate IDs** (an allocator bug) are refused with both entries retained —
  a real safety property for the `win_<id>`/`tab_<id>` allocator.

What kubeless deliberately does **not** do (per its own docs, and confirmed in
source): sequence/array alignment (foxread's projector must key arrays first —
which is precisely foxread's identity policy question, correctly kept
foxread-owned); value equality (caller-selected); and history ("net rows do
not reconstruct intermediate or external history", lib.rs 5–7) — matching
foxread's rule that structural replacement must not be labeled tab open/close.

Cost/benefit versus a local implementation: the crate is std-only, ~700 lines
with proptest reference-oracle properties (lib.rs 498–708) and 12 integration
tests. A local rewrite would duplicate a small but subtle evidence algebra
(known-absence discipline, duplicate custody, first-touch order) that is
already proven. The only dependency cost is the pinned path/git revision
(see adoption costs).

Verdict: **reuse as a module wrapper; defer until the reporting slice.** Do
not create `core/diff` as an algorithm crate. For the write loop's "did the
observation change" gate, a plain `PartialEq` on the normalized model — or
kubeless's scalar `Assessor` fingerprint (see below) — suffices and is cheaper.

### 2. Backend desired-vs-current write planning — not foxread's problem

The human-selected direction submits the **whole desired value** through
`CaminusStore::put_value`; caminus's serializer plans changed leaves, stale
pruning, and in-place execution internally
([persistence research](../kopacetic-persistence/persistence0.gpt6a.md);
[layout1](../architecture/layout1.gpt6a.md)). Kubeless's planning-adjacent
cranes do not intersect this:

- `view-memory`/`view`/`view-cache` — atomic **in-memory** keyed
  materialization with anchors and generations. Foxread's sink is a filesystem
  backend with its own engine; adding an in-memory view would be a second
  materializer, not a plan.
- `MutationBatch` — instructions to keyed materializers; caminus consumes
  reflected values, not mutation batches. Assessment's `mutations_with`
  lowering is explicitly "not a universal source edit protocol"
  (collection.md 149–154).
- `operation` — bounded conditional-write retry with conflict rereads, aimed
  at read-modify-write conflict domains (HTTP `resourceVersion` in the kube
  adapter). Foxread's writes are unconditional full values into a tree it
  owns; there is no conflict classification to feed it.

Nothing here beats "pass the full value to caminus." This comparison stays
entirely inside caminus. (Exposing the serializer's desired tree upstream is a
kopacetic-side extension proposed by
[layout1](../architecture/layout1.gpt6a.md); the
[foxread handoff](file:///home/rektide/src/kopacetic/design/consumers/foxread/handoff0.gpt6a.md)
deliberately stops at needs — "what evidence can a caller rely on" — without
prescribing a solution.)

### 3. Desired vs final output verification — the strongest reuse

This is the comparison foxread cannot skip: "know what really landed ...
including removal of obsolete directories" (handoff, "Know whether application
really finished"). `kubeless-observation-assessment` with its `collection`
feature is almost purpose-built for it, and the probe proves the comparison
mechanics from an external workspace — against a **synthetic** desired
inventory and synthetic materialization. It is not an end-to-end caminus
integration; caminus supplies neither side of the probe's inputs.

The API (`observation/assessment/src/collection/`):

- `complete_entries::<K: Ord, V: Eq>(actual, desired, Mode)` (assess.rs 117–131)
  — both sides plain entry iterators; returns a `Report` retaining **all**
  original inputs, rows, gaps, and classification. `Mode::Exact` requires
  that every coordinate not required present is **absent** — stale entries
  become known-unmet exclusion rows.
- `Observation<S,B,K,V,D>` with independent `Fact { membership, value }`
  knowledge: `Evidence::{Known, Missing, Unsupported}` — three-valued, so
  "field not serialized" ≠ "not supported" ≠ "known absent"; and a `closure`
  claim where `Missing` means incomplete enumeration (input.rs 107–176).
- `Expectation::try_new` — checked authored requirements
  (`Expected::present/absent/member/omitted`) refusing duplicates and
  contradictions with all inputs retained.
- `Report`: `satisfaction()` ∈ {Satisfied, Unmet, Unestablished,
  UnmetAndUnestablished} (report.rs 9–40); `known_unmet()`;
  `transitions()`/`change_set()` — "Unknown is never `None` occupancy"
  (report.rs 241–280); `gaps()` with typed attachments; `closure_check()`;
  `mutations_with` explicit lowering.

Executed evidence (`.test-agent/kubeless-reuse/tests/verify.rs`, 5 passing
tests, std-only closure of exactly `kubeless-observation-assessment` →
`kubeless-observation-model` + `kubeless-view-change`, zero external crates):

| Scenario (probe test) | Result through `complete_entries(..., Mode::Exact)` |
|---|---|
| Exact materialization, empty `tabs` dir | `is_satisfied()`, zero known-unmet; empty container survives as an owned directory on both sides |
| Stale `windows/win_0009/**` tree | `!is_satisfied()`; every stale path known-unmet (including intermediate dirs); `change_set_ordered()` yields removal transitions for repair |
| Deleted file + changed title | Exactly those two keys known-unmet, separately |
| Scalar → directory (wrong kind) | The path **and** its unexpected child are known-unmet |
| Incomplete enumeration, via a **manually constructed** `Observation` with `closure: Missing` and `assess_owned` (not produced by the walker or `complete_entries`) | `!is_satisfied()`; the unobserved key is reported as a gap; **no transition fabricates its removal** |

That last row is the property foxread would most likely get wrong in a local
implementation: an enumeration that could not finish must not look like
"everything else absent = clean", and unknown membership must never become a
known-absence transition. Three honesty notes on it:

- **Library behavior, under stated inputs.** The crate guarantees this given
  an honestly declared incomplete enumeration. It cannot prevent a caller
  from falsely marking `closure: Known` over a partial walk — correct
  verification depends on foxread's walker reporting coverage honestly.
- **The partial route is deliberate, not automatic.** `complete_entries`
  asserts complete enumeration (`Observation::complete` ⇒ closure `Known`)
  and cannot produce partial gaps from walk errors. The probe's walker is
  fail-closed — it returns `Result` and walk errors propagate, aborting
  verification with no report at all (never a fabricated clean). Reporting
  partial coverage instead is a separate route: construct the `Observation`
  with `closure: Missing` (as that test does) and call `assess_owned`, which
  yields gap/unestablished-family results rather than false conclusions.
  Fail-closed is the sensible default for the small owned output tree.
- **The exact `Satisfaction::Unestablished` variant is API-deduced, not
  asserted.** The executed assertions are `!is_satisfied()`, a membership gap
  for the unobserved key, and no removal transition. The variant follows
  from `Report`'s construction (`Unmet` needs a known mismatch;
  `report.rs` `from_flags`) and none exists when the observed entry matches
  and the other coordinate is merely unobserved — but that is reasoning over
  the API, not an executed check.

What stays foxread-owned:

1. The **desired inventory source** — it must come from the same encoding
   that supplies the write (layout1's requirement). Options remain: caminus
   exposing its planned tree upstream (preferred), a scratch re-encode through
   the same serializer, or a foxread reflection walk. Assessment is agnostic:
   it consumes `(relative path, entry kind+content)` pairs from any of these.
2. The **owned-tree walker**. The probe's ~30-line walker is a **fixture, not
   code to lift unchanged into production**: it skips dot entries (preserving
   `.caminus/` metadata) and records directories (including empty ones) and
   file contents, but it builds its map keys with `to_string_lossy` — twice,
   on the filename and on the joined relative path — and reads every
   non-directory entry as an ordinary file. A production verifier must
   preserve native path identity (or refuse unrepresentable names) and
   classify unsupported entry kinds — symlinks and other non-regular entries —
   as wrong-kind/unknown rather than silently treating them as ordinary
   files. The ASCII fixture's success says nothing about arbitrary existing
   filesystem entries. Walk errors propagate as `Result::Err` (the
   fail-closed route above).
3. The **cycle receipt**: candidate/source identity + owned root +
   `satisfaction()` + report rows; a completed cycle requires `Satisfied`.
   Per layout1, the future jj checkpoint hangs off this boundary — this is
   kubeless's "acknowledgment is not settlement" law (information-model.md)
   made concrete for foxread.

Costs: `collection` adds only the two std-only leaves (Cargo.toml 14–18);
default scalar profile has **no normal dependencies at all**. Values need
`Eq`; keys need `Ord` (or `Eq+Hash` via `complete_entries_hashed`) — paths and
`enum Entry { File(bytes|digest), Dir }` fit directly.

## Ranked actionable opportunities

1. **`kubeless-observation-assessment` (collection) — output verification**
   (first vertical, `output/caminus` `verify` module). Evidence above. The
   only open input is the desired-inventory source, which is foxread/kopacetic
   work regardless.
2. **`kubeless-view-change` — neutral diff vocabulary** (slice 2, module in
   `core/model` or `observe/runner`; also the report shape for a future
   `foxread diff` command and for test assertions about projection).
3. **Scalar `Assessor` gate** (same crate, default features): `Assessor` +
   `Projected { basis, fingerprint }` → `Comparison::{Equal, Different,
   BaselineUnknown, IncompatibleBasis, ProjectionUnavailable}`
   (assessment/src/lib.rs 13–99). This is foxread's "only write when the
   normalized observation changed" gate, with `basis` cleanly encoding the
   projection version — an `IncompatibleBasis` after a foxread schema change
   forcing a rewrite rather than silently comparing incomparable values.
   Cheap, std-only, no history retained. **Scope: an optimization only.** It
   may skip a redundant write when the output is already known-current, or
   drive semantic-change reporting. It must never suppress pending retries,
   restart-time verification, or explicit repair: after a restart or any
   incomplete cycle, verify (and if needed resubmit the full value)
   regardless of source equality. Full-value convergence remains the primary
   mechanism, per the human-selected direction.
4. **`kubeless-work-schedule-policy` (std) / `work-schedule` (Tokio) — runner
   coalescing** (slice 2+). `BatchWait::open(now, BatchOptions { quiet,
   maximum })` implements min(latest-observation+quiet, opened+maximum) with
   checked arithmetic and caller-supplied clock — exactly the watcher-drain
   debounce the [watching design](../watching/watchman0.gpt6a.md) describes
   ("coalesce work while retaining a generation/count"). If foxread's runner
   is Tokio (likely, since `watchman_client` is async), `work-schedule`'s
   `Schedule` adds equal-key latest-payload merging; otherwise the leaf needs
   no runtime at all.
5. **`kubeless-observation-backoff` — read-failure pacing** (optional). Core
   decides `Wait`/`GiveUp` from classified failures and caller `Instant`s
   (Kubernetes-free restatement of watcher backoff, incl. progress-reset and
   far-apart decay). Adds `observation-session` (futures+thiserror) to the
   graph. Worth it only if the runner wants a policy crate instead of a
   `count++ → sleep` loop; the classification input is foxread's own error
   taxonomy.
6. **Design guidance, not code:** `doc/information-model.md`'s fact table
   (key/value/revision/cursor/evidence/generation/incarnation) and key-role
   separation (lookup vs coalescing vs execution vs impact identities) should
   inform `core/model`'s vocabulary and the future `core/identity` ledger —
   notably "recreation at the same key is a new incarnation", which is
   exactly Firefox-restart semantics for `win_0001`. Free to adopt as
   thinking; no dependency.

## Negative findings — looks applicable, isn't

- **`observation-session`** — executor-free paged snapshot + resumable-change
  sessions over caller-supplied ops/cursors. Foxread's source is "watchman
  ping → reread whole file"; there are no pages, no cursors, no steady-state
  change stream. Using it would wrap a file reread in a protocol built for
  LIST/WATCH. The session's cursor discipline ("volatile resume position,
  not acceptance") is good guidance; the crate isn't the vehicle.
- **`observation-reflection`/`reflector`/`relation`/`view`/`view-cache`** —
  candidate/commit reflection engines into lifecycle-aware in-memory views.
  Foxread's "last accepted observation" is one field on the runner; a
  Generation-counter view machinery solves a problem foxread doesn't have
  until it grows subscriber fan-out.
- **`control-driver`/`control-controller`/`work-custody`/`retained-actions`**
  — multi-key concurrent control loops, retained custody, prepared/retained
  action hosts. The [maintained-projection composition](
  file:///home/rektide/src/kubeless/doc/reconciliation/maintained-projection.md)
  is a runnable precedent of foxread's *shape* (source capture → evaluate →
  checked acceptance → receipts → successors), and `kopacetic-akri` composes
  it for Kubernetes publication (`doc/consumer/kopacetic-akri.md`). But
  foxread has one profile, one writer, one output tree: single-writer
  serialized application per layout1. Adopting the host would import
  custody/permit/wake machinery for no second key. Revisit only if foxread
  grows multi-profile concurrency or durable cross-restart work ledgers.
- **`operation`** — see comparison 2. **`control-settlement`** —
  acknowledgment-to-settlement composition is conceptually foxread's
  write-then-verify, but its entrance (`acknowledge_then_settle`) targets
  submitted effects with identity predicates; foxread's settlement is simply
  comparison 3 against a tree it can fully enumerate. The crate's law
  ("acknowledgment is not settlement") is the lesson; the code isn't needed.
- **`catalog`**, **`observation-order`**, **`observation-projector`**,
  **`view-predicate`/`wait`/`filter`/`dispatch`/`query`/`publication`** —
  capability descriptors, declared ordering comparators, borrowed projection
  traits, TTL predicate gates, condition waits, stream filters, weak
  notifications, snapshot scans, bounded broadcast. Each solves a problem
  foxread doesn't currently have (foxread's order is a value field, not a
  comparator regime; projection is domain-specific; there are no
  subscribers). None recommended.
- **No filesystem/watch support anywhere** in the reusable crates (grep
  evidence in coverage map). `facet-caminus`'s watcher limits are already
  documented in foxread's persistence research; the official `watchman_client`
  path stands.

## Integration sketch (grounded in executed probe code)

`output/caminus::verify`, adapted from the passing probe:

```rust,ignore
use kubeless_observation_assessment::collection::{complete_entries, Mode};
// Fail-closed route (probe walker semantics): a walk error aborts with the
// error and no report is produced. complete_entries asserts complete
// enumeration (Observation::complete => closure Known); it cannot produce
// partial gaps from walk errors.
let walked = walk_owned(&store.entity_path(&identity))?;   // foxread-owned walker
let desired = desired_inventory(&snapshot)?;              // same encoding as the write
let report = complete_entries(walked.iter(), desired.iter(), Mode::Exact)?;
if report.is_satisfied() {
    // completed cycle: record receipt (candidate digest + owned root), checkpoint later
} else {
    // incomplete: retain candidate for repair; report.known_unmet() names
    // missing / wrong-content / wrong-kind / stale paths. Repair is full-value
    // resubmission, not transition replay.
}
// Deliberate partial-coverage route (only if reporting partials instead of
// failing outright): construct Observation { closure: Missing, observed
// facts } and call assess_owned — gaps()/unestablished-family results.
```

Domain diff module (slice 2), from the probe:

```rust,ignore
use kubeless_view_change::{ChangeSet, alignment::{align, Aligned}};
let rows = align(prior.tabs().map(|(k,v)|(k.as_str(), v)),
                next.tabs().map(|(k,v)|(k.as_str(), v)))?;   // duplicate IDs refused
let mut changes = ChangeSet::try_from_transitions(
    rows.into_iter().map(Aligned::into_transition))?;        // None = known absence
changes.retain(|row| row.before != row.after);               // explicit equality policy
```

## Boundary revision

Against [layout1's nine crates](../architecture/layout1.gpt6a.md) ("proposed
boundaries, not requirements to preserve"):

- **`core/diff` → demote to a module** (in `core/model`, or `observe/runner`
  when it feeds reporting) wrapping `kubeless-view-change`. Rationale: the
  mechanism is keyed alignment foxread calls with its own keys/equality; a
  crate boundary implies an algorithm worth owning, and the honest answer is
  that there isn't one — the remaining foxread-owned piece (array→key policy)
  already lives in `firefox/project` + `core/model`'s allocator.
- **`output/caminus` gains a `verify` module** as sketched; its inventory
  type is the natural shared seam with the runner's cycle receipt.
- Everything else in the nine-crate layout stands unaffected; kubeless reuse
  replaces *internal implementation*, not foxread's domain boundaries.

## Adoption costs and risks

- **Not on crates.io.** Workspace-internal, 0.1.0 lockstep, soft-fork cadence.
  Foxread must use a path or Git dependency **pinned to a reviewed revision**
  (README "Use from a checkout"). Precedent exists: kopacetic already
  path-depends on kubeless behind optional features.
- **MSRV/edition:** edition 2024, rust-version 1.89 (workspace manifest).
  Current nightly (1.101) is fine; foxread should record the same floor.
- **Churn risk is contained:** the foxread-touched surface is
  `complete_entries`/`assess*`/`Observation`/`Expectation`/`align`/
  `ChangeSet` — small, documented, and under active-but-stable contracts with
  independent probes. If kubeless churns, a local fallback for this surface
  is mechanical (the probe doubles as a migration test).
- **The scratch probe is disposable evidence**: `.test-agent/kubeless-reuse/`
  is gitignored; its findings are promoted here. When foxread's real
  workspace exists, lift the inventory types and these tests into
  `output/caminus` — and write a production walker (native path identity,
  entry-kind classification) rather than reusing the lossy fixture walker
  unchanged.

## Unanswered questions

1. **Desired-inventory source** — caminus exposing the serializer's desired
   tree (the preferred upstream extension **proposed by layout1**; the
   kopacetic handoff intentionally records needs only, not solutions)
   vs scratch re-encode vs independent reflection walk. Determines whether
   verification compares content bytes or digests; digests shrink memory but
   must come from the same encoding.
2. **Does the domain diff need assessment's partial-fact machinery?** If
   reporting should say "title: unsupported in this Firefox version" vs
   "title: missing", `Fact`/`Evidence` is the right vocabulary — but that is
   heavier than slice 2 needs and can be introduced without breaking the
   align-based start.
3. **Runner timing substrate** — Tokio `work-schedule` (matches the async
   watchman client) vs std `schedule-policy` leaf with a hand-rolled loop.
   Decide when the runner slice is built.
4. **Backoff** — adopt `observation-backoff` with foxread's own failure
   classification, or keep a local counter until a second consumer exists
   (kubeless's own law: "interfaces are earned by a second consumer").
5. **Kubeless pin policy** — which revision foxread pins, and who re-reviews
   it; kubeless is in active development with an explicit human-gate culture.

## Recommended first slice

Unchanged from layout1's first vertical, now with named mechanisms:

1. `firefox/sessionstore` fixture → `firefox/project` → `core/model`
   (typed IDs, honest timestamps).
2. `output/caminus`: schema + `apply` via `CaminusStore::put_value` (full
   value, in place).
3. `output/caminus::verify` via `kubeless-observation-assessment`
   (`collection`, `Mode::Exact`) + foxread walker; completed-cycle receipt
   only on `Satisfied`.
4. `app/cli` one-shot `dump` printing `entity_path`, write report, and the
   verification report's known-unmet/gap rows.

Slice 2 then adds `observe/watch` (watchman_client) and `observe/runner`
(coalescing via `schedule-policy`/`work-schedule`; equality gate via scalar
`Assessor` or model `PartialEq`; domain diff via `view-change` when reporting
is wanted).

## Cross-references

- [Layout1 direction](../architecture/layout1.gpt6a.md) — the three
  comparisons and the verify-what-landed contract this research implements.
- [Persistence investigation](../kopacetic-persistence/persistence0.gpt6a.md)
  — caminus write/report limits that make comparison 3 mandatory.
- [Kopacetic foxread handoff](file:///home/rektide/src/kopacetic/design/consumers/foxread/handoff0.gpt6a.md)
  — consumer-side framing of the same verification need.
- Kubeless: [view-change](file:///home/rektide/src/kubeless/view/change/README.md),
  [assessment collection guide](file:///home/rektide/src/kubeless/observation/assessment/md/collection.md),
  [information model](file:///home/rektide/src/kubeless/doc/information-model.md),
  [maintained-projection precedent](file:///home/rektide/src/kubeless/doc/reconciliation/maintained-projection.md),
  [kopacetic-akri integration](file:///home/rektide/src/kubeless/doc/consumer/kopacetic-akri.md).
- Scratch applicability probe: `.test-agent/kubeless-reuse/` (gitignored;
  commands and results in frontmatter `verification`).
