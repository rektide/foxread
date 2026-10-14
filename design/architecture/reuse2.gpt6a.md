---
type: DesignReview
title: Kubeless mechanisms to carry into foxread
description: Parent assessment of the broad kubeless survey, comparison requirements, and proposed crate-boundary adjustment.
resource: /design/architecture/reuse2.gpt6a.md
status: proposed
generated: { by: "openai/gpt-6-astra#high", at: 2026-10-14 }
sources:
  - resource: "opencode://ses_ec59f6862ffeFt6UN2FUOhTXOO"
    title: Commission and review the kubeless survey, inspect probe/API evidence, and integrate reuse recommendations
  - resource: "opencode://ses_ec3c6dcbbffe8ZOeI8z2chI0S2"
    title: G's broad kubeless survey and nine-test external applicability probe
  - resource: /design/kubeless/reuse0.glm53m.md
    title: Coverage map, precise candidate APIs, dependency analysis, and negative findings
  - resource: /design/architecture/layout1.gpt6a.md
    title: Human-selected full-state writes and proposed nine-crate layout
  - resource: "file:///home/rektide/src/kubeless/observation/assessment/src/collection/assess.rs"
    title: Complete-map assessment explicitly asserts complete observation; partial input uses a different entrance
  - resource: "file:///home/rektide/src/kubeless/work/schedule-policy/src/lib.rs"
    title: Caller-clock batching with quiet and maximum intervals
verification:
  method: Read G's report, scratch verification tests and walker, and selected public API implementations; trust the reported nine passing tests without rerunning them.
  limits: Fixture inventories are synthetic, not produced by caminus; no production integration or full-tree coherence proof.
---

# Kubeless mechanisms to carry into foxread

## Main result

The strongest reuse is **output assessment**, followed by keyed change reporting
and batching policy. G's [survey](../kubeless/reuse0.glm53m.md) examined the
workspace's observation, view, work and control families, supporting docs,
integration precedents and test infrastructure. Its coverage map is the detailed
account of that search; it is not merely a search for a crate named diff.

An external scratch workspace exercised the public comparison APIs: nine tests
passed, and the kubeless dependency closure for those comparisons consists of
three standard-library-only crates. The tests use a synthetic desired inventory
and ordinary filesystem fixtures. They establish applicability of the comparison
mechanisms, not a finished caminus integration.

## What to use, and when

| Foxread need | Kubeless mechanism | Recommendation |
|---|---|---|
| Check desired entries against actual output | `kubeless-observation-assessment`, `collection` feature, `Mode::Exact` | First vertical, inside `output/caminus::verify` |
| Report changes between keyed browser observations | `kubeless-view-change::alignment::align` and `ChangeSet` | Later reporting module; do not build a separate diff algorithm |
| Coalesce repeated invalidations without postponing forever | `kubeless-work-schedule-policy::BatchWait` | Evaluate when implementing the runner; std-only, caller supplies the clock |
| Executor-backed keyed scheduling | `kubeless-work-schedule` | Optional if runner needs the additional queue behavior |
| Version-aware observation equality | Scalar assessment `Assessor` | Optional; ordinary equality may suffice initially |

The proposed `core/diff` crate is therefore unnecessary for the first vertical.
Recommend eight initial architectural boundaries instead of nine, keeping change
reporting as a module when used. This is a scope simplification, not evidence
that browser diffs are impossible or never useful.

The full control-driver, in-memory view and custody machinery is not justified
for the initial per-profile single-writer cycle. Kubeless supplies neither the
Firefox parser nor the desired filesystem inventory nor the Watchman adapter.
Those remain concrete integration work. Multiple profiles or durable work
coordination could change the case for heavier components later.

## Why we need comparison but not necessarily domain diff

A browser diff answers **what changed between observations?** Useful for logs,
inspection and consumers, but unnecessary for sending a full value to caminus.
It cannot establish true browser lifecycle when identity or source coverage is
uncertain. The keyed alignment library does not solve Firefox identity matching.

Caminus's write planning answers **which filesystem operations might make this
value current?** That remains the persistence backend's responsibility.

Final assessment answers **does the observed output meet the requested state?**
That is the immediate consumer requirement. `Mode::Exact` can find stale paths,
missing paths and content/kind mismatches when given correct inventories,
including directory entries. It compares supplied evidence; it does not walk
the filesystem, generate encoding, or certify the evidence's completeness.

## Integration conditions

- `complete_entries` declares that both entry sets are complete. Only call it
  after a successful complete acquisition within the declared ownership scope.
  A failed walk can simply fail the cycle. Rich partial diagnostics instead use
  an explicit `Observation` with incomplete closure and `assess_owned`; the probe
  constructs such an observation manually. It does not implement a partial walker.
- The desired inventory must correspond to what caminus encodes. How to obtain
  it remains unresolved. The Kopacetic handoff records the need, not a selected
  backend API or solution.
- Do not promote the scratch walker unchanged. Its lossy string path keys and
  simplified non-directory handling are suitable for its fixtures, not an exact
  inventory of arbitrary existing entries. Preserve native path identity or
  explicitly refuse unsupported names/kinds.
- A complete walk is not an atomic filesystem snapshot. The user accepts in-place
  updates; serialize foxread's writer with its own verification. Any stronger
  claim about concurrent external mutation needs separate evidence/coordination.
- **Unchanged source content does not prove correct output.** Equality can reduce
  reporting noise but must not suppress pending application, failed-cycle retry,
  restart convergence, or explicit verification/repair. Keep the full-value
  application contract primary.
- A successful assessment establishes the checked result under the supplied
  evidence and scope. It is not a permanent guarantee that disk has stayed equal.
  Future jj checkpointing follows the completed application boundary.

## Next implementation boundary

Proceed with the JSON fixture-to-filesystem vertical from
[layout1](layout1.gpt6a.md), using assessment for final comparison once the desired
inventory source is available. Defer the domain-change wrapper to reporting needs.
Choose batching/scheduling mechanisms when adding Watchman and the runner, rather
than importing a control-plane framework into the first dump command.
