---
type: Research
title: Watchman as foxread's invalidation source
description: Rust subscription APIs, restart semantics, and the boundary between file notifications and browser-state changes.
resource: /design/watching/watchman0.gpt6a.md
status: draft
generated: { by: "openai/gpt-6-astra#high", at: 2026-10-14 }
sources:
  - resource: "opencode://ses_ec59f6862ffeFt6UN2FUOhTXOO"
    title: Inspect Watchman's Rust integration and connect its observation boundary to the integrated architecture
  - resource: /design/architecture/synthesis0.gpt6a.md
    title: Integration with Firefox sessionstore and kopacetic source research
  - resource: https://github.com/facebook/watchman/blob/923b0935155590be54c0fc052fdca0201f8ebc4b/website/docs/cmd/subscribe.md
    title: Subscription lifecycle, initial results, and settling
  - resource: https://github.com/facebook/watchman/blob/923b0935155590be54c0fc052fdca0201f8ebc4b/website/docs/cmd/watch-project.md
    title: Root resolution and relative-path requirements
  - resource: https://github.com/facebook/watchman/blob/923b0935155590be54c0fc052fdca0201f8ebc4b/website/docs/file-query.md
    title: Fresh-instance semantics
  - resource: https://github.com/facebook/watchman/blob/923b0935155590be54c0fc052fdca0201f8ebc4b/watchman/rust/watchman_client/src/lib.rs
    title: Rust connection and subscription implementation
  - resource: https://github.com/facebook/watchman/blob/923b0935155590be54c0fc052fdca0201f8ebc4b/watchman/rust/watchman_client/src/pdu.rs
    title: Rust query and subscription request/result types
---

# Watchman as foxread's invalidation source

## Situation and recommendation

The user prefers Watchman for a Rust program following Firefox's files. Use
Watchman to tell an adapter **its source might have changed**. Let the Firefox
adapter establish a readable snapshot and derive browser changes. A filesystem
clock neither identifies a browser transaction nor preserves overwritten file
contents.

The archived Watchman checkout at `~/archive/facebook/watchman`, revision
`923b0935155590be54c0fc052fdca0201f8ebc4b`, includes the official
`watchman_client` Rust crate (manifest version `0.9.0`). `command -v watchman`
found no executable on this session's PATH. This is source inspection, not a
daemon integration test; deployment needs a daemon/socket or executable.

## Concrete Rust interface

The upstream [subscription example](https://github.com/facebook/watchman/blob/923b0935155590be54c0fc052fdca0201f8ebc4b/watchman/rust/watchman_client/examples/subscribe.rs)
uses:

1. `Connector::new().connect().await`.
2. `client.resolve_root(CanonicalPath::canonicalize(profile)?).await`.
3. `client.subscribe::<Fields>(&resolved, SubscribeRequest { ... }).await`.
4. Repeated `subscription.next().await`.

`SubscriptionData` distinguishes `FilesChanged`, `Canceled`, and state
enter/leave notifications. `FilesChanged` carries a `QueryResult` with optional
files, a clock, and `is_fresh_instance`. The returned subscription initiation
response is an acknowledgment with a clock, not the file-content baseline.

Use a filename filter over the source adapter's declared dependency paths.
Request `name` and `exists`; avoid an `exists == true` filter that would discard
deletions. For a SQLite adapter, the dependency set must include its database and
WAL-related paths; the Firefox investigation determines the complete set. Watch
the directory so atomic file replacement remains observable.

`watch-project` may select an ancestor as the root. Honor its resolved root and
relative path, and inspect the actual watched scope. A subscription filter
restricts delivered matches, not necessarily the underlying crawl. Root markers
and global `enforce_root_files` configuration can affect profile watching.

## Proposed observation loop

1. Establish the subscription before reading an initial source snapshot; keep
   consuming notifications while reads/projection work proceed.
2. Set a dirty flag on matching changes. Coalesce work while retaining a
   generation/count so a notification during a read schedules another read.
3. Read and validate the source using its own consistency rules. Watchman's
   settle period is useful batching, not proof that a writer has committed.
4. Derive a normalized snapshot; compare it to the last successfully applied
   snapshot. A repeated notification with identical content produces no domain
   change.
5. Apply the output batch. Advance the applied snapshot only after the output
   boundary succeeds. Preserve a pending retry if reading or applying failed.
6. On cancellation/disconnection, reestablish the watch and obtain a fresh
   baseline. A fresh-instance response also triggers a complete source read.

Initially, always re-read on process startup. A saved Watchman clock is an
optimization; it cannot substitute for the durable source/application state.
If clocks are persisted later, never record a notification as successfully
applied before the resulting output has succeeded. A newer source snapshot can
include writes beyond the delivered clock; re-reading those on the next wakeup
is harmless when application is idempotent.

Source disappearance or decoding failure is an unavailable observation, not an
empty browser. Do not infer all tabs closed from a missing session file. A valid
empty snapshot is a different result.

The Rust client's internal subscription queue is unbounded in this inspected
version (`Subscription.responses`). Keep the receiver draining into a bounded
dirty-state representation rather than making it await expensive projection
work for each packet.

## What this can and cannot report

This loop can report differences between successfully observed persisted states.
It cannot recover a tab that opened and closed between Firefox's disk writes,
nor all intermediate versions of a file overwritten during a disconnection.
SQLite WAL following, if selected, needs separate commit/generation semantics;
Watchman notifications are only its wakeup source too.

An explicit polling watcher could share the same invalidation interface if
useful. It should be a visible backend choice, so a failed Watchman setup is not
silently mistaken for an active subscription.

## Verification targets for implementation

- Source replacement, deletion/reappearance, and a malformed intermediate read.
- An event arriving during a read or a slow output batch.
- Repeated identical snapshots producing no repeated domain changes.
- Disconnect and fresh-instance recovery without interpreting unavailable
  source data as closure.
- A wider resolved root with correct profile-relative matching.

These are behavioral test targets, not claims of tests already run.

## Integration

The [architecture synthesis](../architecture/synthesis0.gpt6a.md) joins this
notification boundary to the actual Firefox snapshot format and the existing
kopacetic sink. It recommends the official client for source watching because
the inspected caminus wrapper drops some recovery signals.
