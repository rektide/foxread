---
type: ImplementationPlan
title: JSON session snapshots to verified filesystem state
description: Work-ready foxread scope, crate responsibilities, kubeless reuse decisions, and dependency-linked implementation tickets.
resource: /design/implementation/plan0.gpt6a.md
status: proposed
generated: { by: "openai/gpt-6-astra#high", at: 2026-10-14 }
extensions:
  ticket: foxr-json
sources:
  - resource: "opencode://ses_ec59f6862ffeFt6UN2FUOhTXOO"
    title: Human endorsement of domain structure and kubeless findings; commission implementation plan and tickets, defer scalar Assessor, assess executor-backed scheduling
  - resource: /design/architecture/layout1.gpt6a.md
    title: Domain-shaped crates, full-state writes, and typed ID direction
  - resource: /design/architecture/reuse2.gpt6a.md
    title: Parent assessment of output verification, optional domain diff, and batching reuse
  - resource: /design/kubeless/reuse0.glm53m.md
    title: Broad kubeless survey, applicability tests, and corrected evidence limits
  - resource: /design/firefox-stores/stores0.gpt6a.md
    title: Sessionstore format, file roles, source limits and meaningful fixtures
  - resource: "file:///home/rektide/src/kopacetic/design/consumers/foxread/handoff0.gpt6a.md"
    title: Consumer needs and unresolved output-completion questions, without prescribing the backend solution
  - resource: "file:///home/rektide/src/kubeless/work/schedule/README.md"
    title: Executor-backed keyed admission, latest-payload merging, checked refusals and timing guarantees
---

# JSON session snapshots to verified filesystem state

## Working direction

Build the JSON path now. The first useful result is a Rust command that reads an
explicit sessionstore input, projects windows/tabs, submits the whole desired
value through Kopacetic/Caminus, and reports whether the actual owned tree matches.
Watching then maintains that same behavior.

The implementation anchor is **`foxr-json`**. This plan brings together the
research and human feedback; earlier independent reports remain evidence rather
than separate competing work plans. The implementation has not started.

### Decisions carried forward

- Full desired values go to persistence. In-place updates are acceptable; stale
  owned entries belong in final-state comparison.
- IDs and directory keys use `win_<id>` and `tab_<id>`. For the first slice,
  IDs are explicitly scoped to an observation; persistent matching is separate.
  Allocate tab coordinates uniquely within the profile observation and preserve
  ordering as data. The qualified reference includes profile and generation.
- Use one profile entity initially. The logical payload groups `windows` and
  `tabs`; retain the native encoded Kopacetic entity root and print its location.
- Use **kubeless collection assessment** for output verification.
- Use **kubeless keyed alignment/change sets** when adding observation reporting.
  A separate `core/diff` crate is unnecessary.
- Reuse **kubeless batching/scheduling** rather than inventing timing laws. Assess
  the executor-backed queue with actual write/verification measurements; favor
  adopting it if its keyed admission behavior fits.
- Leave the scalar **Assessor** out. Ordinary equality is sufficient for optional
  content comparison, and cannot replace output verification or pending repairs.
- SQLite/WAL belongs to the separate walcap effort. Future jj checkpointing can
  consume a completed output cycle; it is not part of these implementation tickets.

## Workspace and dependency shape

```text
core/model/             normalized observations, typed IDs, provenance
firefox/sessionstore/   bounded JSON/MozLZ4 codec and source-shaped types
firefox/profile/        explicit profile/file acquisition and source roles
firefox/project/        source interpretation and model projection
observe/watch/          Watchman invalidation and connection lifecycle
observe/runner/         pending work, serialized apply/verify, retries
output/caminus/         persisted schema, full-value output and verification
app/cli/               foxread command composition
```

`project` joins sessionstore and model. Profile acquisition owns filesystem
selection without putting I/O in the decoder. Output consumes the model and
backend; the runner coordinates through small interfaces and the CLI selects
concrete components. Add the keyed-diff reporting module alongside its consumer,
keeping the foundational model free of unused reporting machinery.

Crate directories omit the redundant project prefix; Cargo packages may retain
it. Introduce working code with each crate rather than scaffolding empty packages.
Retain existing repository files until their replacement/removal is deliberate.

## The first output contract

A successful cycle means a valid source observation was projected, its complete
value was submitted, and the owned output was checked against the expected
encoding. Byte-identical content need not be physically rewritten. Failures leave
the candidate eligible for replay and do not become browser closure events.

The main unresolved integration input is **how we obtain the desired encoded
inventory**. Resolve it in `foxr-json-inventory` with the owning projects. The
Kopacetic handoff supplies needs, not a chosen API. Foxread must not build a second
independent serializer merely to calculate expectations.

The inventory/representation agreement covers relative path identity, entry kinds,
empty containers, reserved metadata, and exact optional-text semantics including
empty strings and literal `null`. It must make stale files/directories observable.
This work can proceed alongside the decoder using small synthetic values.

For the initial verifier, use a complete fail-closed walk: read/enumeration failure
is an incomplete cycle, not a fabricated complete inventory. Feed only a completed
walk and complete desired inventory to collection `Mode::Exact`. Partial evidence
reports are optional later. Preserve native path identity or explicitly reject
unsupported entries; the research's ASCII fixture walker is not production code.

Verification runs after foxread's own writer has finished. The result is scoped
evidence, not atomic publication or a promise against subsequent external edits.
Print useful mismatches and return failure when the cycle cannot establish the
requested state. Keep source status, attempted output and verified output distinct.

## Work graph

All children belong explicitly to epic `foxr-json`. These are capability slices,
not one ticket per crate and not time estimates.

| Ticket | Outcome | Prerequisites |
|---|---|---|
| `foxr-json-model` | Rust workspace and normalized observation/ID contract | — |
| `foxr-json-sessionstore` | Bounded unencrypted JSON/MozLZ4 decoding with honest failures | model |
| `foxr-json-project` | Correct Firefox-to-model projection | model, sessionstore |
| `foxr-json-inventory` | Agreed, exercised output schema and desired-inventory boundary | — |
| `foxr-json-output` | Full-value caminus application and exact final-tree assessment | project, inventory |
| `foxr-json-dump` | Useful explicit-input one-shot command and end-to-end fixtures | output |
| `foxr-json-profile` | Profile file-role selection and source availability | sessionstore |
| `foxr-json-watch` | Watchman invalidation, fresh-instance and disconnect handling | model |
| `foxr-json-measure` | Stage costs and slow-write/burst workload evidence | dump |
| `foxr-json-scheduling` | Assess executor-backed keyed scheduling and select runner policy | measure |
| `foxr-json-runner` | Continuously converge with retry and in-flight invalidation handling | output, profile, watch, scheduling |
| `foxr-json-reporting` | Optional keyed observation-change reporting | project |

The workspace bootstrap is included in `model`; sessionstore/watch depend on that
ticket for integration plumbing, not because their pure contracts require browser
model types. Reporting is lower priority and blocks neither dump nor watching.

## Executor-backed scheduling assessment

The user's inclination to adopt it is sensible: Watchman's official client already
uses Tokio, and `kubeless-work-schedule` supplies keyed coalescing with latest-work
merging and bounded debounce. Avoid making low latency in one tiny fixture the
sole reason to omit a useful component.

Nevertheless measure the real boundaries: decode/project, encode/plan/write, full
filesystem walk, assessment, and total cycle. Include first write, identical
rewrite, sparse edits, removed windows, and deliberately slow output. Record
fixture sizes, machine/filesystem context, stage distributions and output costs.
Do not make an unmeasured performance guarantee or an arbitrary threshold.

Then exercise policy-only and executor-backed admission against those timings:

- Repeated invalidations for one profile collapse without losing a notice received
  during application; work after that application still runs when needed.
- Continuous arrivals cannot indefinitely restart an earned batching wait.
- Failed application remains retryable even when the source has not changed.
- Profile is the coalescing key; individual tab identity does not determine the
  scheduling unit for a whole-profile write.
- Multiple profile keys, if evaluated, remain independent; writes to the same
  owned root never overlap. This experiment does not require multi-profile CLI scope.
- Close/disconnect/refusal behavior is explicit. Timer arithmetic refusal retains
  the work; a canceled waiter does not imply a blocking write has stopped.

`Schedule` is timed admission, not a filesystem execution engine. Its maximum
batch delay bounds batching, not queueing behind slow I/O or execution duration.
The runner still owns dirty/in-flight state and serialized writes. Prefer queuing
an invalidation/request to reacquire current state, rather than accidentally
discarding a failed output obligation when merging snapshot payloads.

The ticket's result is a recorded choice with a small executable comparison and
selected configuration/merge semantics. If executor-backed scheduling fits,
adopt it in the runner ticket; there is no extra approval ritual. If rejected,
show the concrete mismatch and use the shared policy leaf instead.

## Verification and readiness

Meaningful checks cover current-entry back navigation, duplicate URLs, absent
histories, open-versus-closed collections, malformed/oversized/encrypted inputs,
null/empty text, idempotent writes, stale directories, wrong-kind entries,
interrupted output and restart convergence. Watch tests add fresh instances,
disconnects and notices racing slow writes. Use synthetic browser data.

Reuse the research's public-API comparison fixtures as evidence and adapt them to
the real output contract. Run appropriate nextest, clippy and formatting checks
for each implemented slice; measure performance only after a functioning dump.

**Ready to start:** model/workspace and the output-inventory investigation. The
decoder follows immediately. The desired-inventory boundary is the only open
backend question blocking an honestly verified end-to-end dump; it does not block
the JSON work. No further broad research wave is needed before those tickets.
