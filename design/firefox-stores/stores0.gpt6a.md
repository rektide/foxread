---
type: Research
title: Firefox persisted session state, Places history, and the occhre reuse boundary
description: Source-grounded storage and identity evidence for a Rust disk observer, with snapshot-first implementation scope and optional SQLite extraction reuse.
resource: /design/firefox-stores/stores0.gpt6a.md
status: research
generated: { by: "openai/gpt-6-astra#medium", at: "2026-10-14" }
sources:
  - resource: "opencode://ses_ec59e9227ffeL5wyI4SCjAe3QS"
    title: "swift-orchid: Firefox source inspection, persisted identity analysis, and occhre SQLite reuse investigation"
  - resource: "https://github.com/mozilla-firefox/firefox/tree/3934f7874da5d43c3eac71e6f44e0afe79d9071f/browser/components/sessionstore"
    title: "Pinned upstream SessionFile, SessionWriter, SessionSaver, SessionStore, TabState and session schema"
  - resource: "https://github.com/mozilla-firefox/firefox/blob/3934f7874da5d43c3eac71e6f44e0afe79d9071f/toolkit/modules/sessionstore/SessionHistory.sys.mjs"
    title: "Session history serialization and restoration"
  - resource: "https://github.com/mozilla-firefox/firefox/blob/3934f7874da5d43c3eac71e6f44e0afe79d9071f/xpcom/ioutils/IOUtils.h"
    title: "MozLZ4 and encryption framing; corresponding IOUtils.cpp also inspected"
  - resource: "https://github.com/mozilla-firefox/firefox/tree/3934f7874da5d43c3eac71e6f44e0afe79d9071f/toolkit/components/places"
    title: "nsPlacesTables.h, Database.cpp, History.cpp, and nsINavHistoryService.idl"
  - resource: "file:///home/rektide/src/occhre/README.md"
    title: "Current SQLite incremental/tmpfs checkpoint and outstanding live acquisition boundary"
  - resource: "https://github.com/rektide/occhre/tree/9d31b09d8db724ebd7ec4ff774a5f76735428728/crates"
    title: "Local source revision inspected: walpin, SQLite WAL/view/records/changes crates and manifests"
verification:
  method: "Source inspection; no personal browser profiles or databases read; no execution of Firefox or occhre tests"
  limits: "Firefox upstream main revision is pinned, but installed Firefox release compatibility remains to be checked with synthetic fixtures. Reuse conclusions are design recommendations, not an integration proof."
---

# Firefox stores: snapshot-first, history separately

## Findings and scope

**The disk representation of windows and tabs is Session Restore's compressed JSON, not a SQLite window/tab table.** Live browser objects are in memory; the file is a delayed, filtered restoration snapshot. SQLite `places.sqlite` supplies page/visit history and optional enrichment, not the missing exact live identity or lifecycle.

The feasible first slice is decode → typed projection → whole-snapshot comparison → publish an owned generation. Do not make SQLite CDC a prerequisite for that slice. A filesystem watcher only announces that the source may have changed; it cannot recover browser events omitted between writes.

I checked `~/archive/mozilla/` first: it contains `authenticator-rs`, not Firefox. The Firefox evidence below is pinned to upstream `3934f7874da5d43c3eac71e6f44e0afe79d9071f`, fetched during this investigation. This is **upstream main**, not a claim about the operator's installed version. Some schema comments describe Firefox 156 changes. Local occhre was inspected at HEAD `9d31b09d8db724ebd7ec4ff774a5f76735428728`; an unrelated untracked research document was left unread and untouched. Attempted optional nested delegation was refused by the harness depth limit; I performed the SQLite investigation directly.

## Source map

All Firefox links below are immutable revision references:

| Source | Relevant evidence |
|---|---|
| [SessionFile][sessionfile] | `Paths`, startup load ordering, decompression/decryption, version compatibility, final write |
| [SessionWriter][writer] | `#write`: temporary files, recovery backup, shutdown trimming, encryption options |
| [SessionSaver][saver] | Scheduling preferences and `#saveState` filtering |
| [SessionStore][store] | `#generateWindowID`, window initialization, `getCurrentState`, window collection, `#shouldSaveTab` |
| [TabState][tab] | `TabStateData`/`ClosedTabStateData`, collection of `lastAccessed`, container and group properties |
| [SessionHistory][history] | `serializeEntry` and `deserializeEntry`: URLs, titles, IDs, referrer serialization |
| [session.schema.json][schema] | Current on-disk root/window/tab/entry shape; an implementation schema, not a stable third-party API |
| [IOUtils.h][ioheader], [IOUtils.cpp][ioimpl] | MozLZ4 header/block layout and whole-file encryption |
| [nsPlacesTables.h][tables], [Database.cpp][db], [History.cpp][placeshistory] | Page/visit schema, journal setup, visit/referrer insertion |

### Which files to watch

Paths are relative to an explicitly selected Firefox profile directory:

| Path | Meaning |
|---|---|
| `sessionstore-backups/recovery.jsonlz4` | Latest runtime restoration snapshot |
| `sessionstore-backups/recovery.baklz4` | Previous runtime snapshot, not another current session |
| `sessionstore.jsonlz4` | Clean-shutdown snapshot; moved to `previous.jsonlz4` after startup |
| `sessionstore-backups/previous.jsonlz4` | Previous clean session backup |
| `sessionstore-backups/upgrade.jsonlz4-<build-id>` | Upgrade backup, potentially much older |

Firefox startup's `Paths.loadOrder` is clean → recovery → recovery backup → previous → upgrade. An observer should retain these meanings rather than picking the newest mtime among every backup. While running, recovery is the relevant source. Startup/shutdown transitions require observing both the profile directory and backups directory. `SessionWriter` uses `.tmp` paths and replacement/backup operations: watch directory changes and reopen the selected path, rather than following one inode indefinitely. Clean shutdown removes runtime recovery files only when `performShutdownCleanup` applies (auto-restore disabled) and the write had no error. Recovery files can therefore survive alongside a newer clean snapshot; their existence is not proof Firefox is running.

Missing recovery during shutdown, a failed decode, or an unreadable file is not evidence that every tab closed. Keep the last successful generation and expose source status. A clean-shutdown snapshot is restorable state, not proof those windows are currently open. Offline process liveness is a separate question; stale files alone cannot answer it. Backup fallback must be marked degraded/stale and must not manufacture closures or newer timestamps.

### The JSONLZ4 format and a new compatibility condition

`IOUtils::MozLZ4` documents a nonstandard layout:

1. Eight bytes `mozLz40\0`.
2. Four-byte little-endian decompressed content length.
3. LZ4 compressed block containing JSON text.

It is not JSON Lines and not the standard LZ4 frame format. A bounded decoder should validate the magic, cap input and declared output size, decompress exactly, parse JSON, and inspect the session format/version. Preserve unknown fields only where needed for compatibility; the projection should explicitly select useful fields rather than publishing the entire restoration blob.

At this upstream revision `SessionWriter` optionally compresses **then encrypts**, gated by both `browser.sessionstore.encryption.available` and `.enabled`. `SessionFile` always attempts decryption when reading, including after the preference has been disabled. `IOUtils::Encryption` identifies `mozEnc0\0` followed by a Keystore-encrypted blob. Thus a `.jsonlz4` extension alone does not guarantee externally decodable LZ4. Recognize encrypted input as a distinct unsupported capability initially. No claim is made that encryption is enabled in the installed browser. `Database.cpp` also has storage-encryption integration, another reason to test actual supported versions before promising generic SQLite readers work on every profile.

## Usable fields, identities, and clocks

### Projectable state

| Object | Useful fields | Meaning/caveat |
|---|---|---|
| Root | `version`, `windows`, `selectedWindow`, `session.startTime`, `session.lastUpdate` | Selected window is positional; session times use `Date.now()` (milliseconds since epoch) |
| Window | `tabs`, `selected`, `title`, bounds, `sizemode`, `workspaceID`, groups/split views where supported | Tab ordering and selection are positional, not IDs; title is a window label |
| Tab | `entries`, `index`, `requestedIndex`, `lastAccessed`, `pinned`, `hidden`, `muted`, `userContextId`, `groupId`, `splitViewId`, `image`, optional `canonicalUrl` | `index` is **1-based** current history entry; `requestedIndex` is pending-restore state; lastAccessed is last selection, not opening or navigation |
| History entry | `url`, `title`, `originalURI`, `ID`, `docshellUUID`, `docIdentifier`, optional `navigationKey`/`navigationId`, `referrerInfo`, child frame entries | Current page comes from `entries[index - 1]`; current is not necessarily the last entry because of back/forward navigation |
| Closed tab/window | `closedAt`, `closedId`; closed tab `state`, `pos`, `sourceWindowId`, `title` | Closure metadata belongs to retained undo-close records; not an exhaustive event log |

Allow empty entries and absent/out-of-range current indexes explicitly. `userTypedValue` is input or a pending load, not guaranteed committed page URL. Canonical URL differs from navigated URL. Subframe `children` are not additional top-level tabs. Closed windows, saved groups, last-session/deferred restoration state must not be merged indiscriminately into currently projected `windows`.

`referrerInfo` is serialized via `E10SUtils.serializeReferrerInfo`, not a plain URL field. `deserializeEntry` also accepts legacy `referrer`/`referrerPolicy`. Initially represent modern referrer information as unavailable or opaque versioned metadata; do not pretend it is directly usable URL text. Session history entries in the inspected serializer/schema have no general visit timestamp or tab-open timestamp.

### Stable live window/tab identity is not supplied by this format

This is the primary modeling constraint, not just a parser detail:

* `SessionStore` keeps live window IDs in an internal `WeakMap`. `#generateWindowID()` returns `"window" + #nextWindowID++`; the counter starts at zero. The serialized window object initialized at lines 1732–1750 has no corresponding ordinary live `id` field.
* `__lastSessionWindowID` is optional restoration bookkeeping, copied only when the separate last-session map has an entry (lines 6028–6033). It is not a guaranteed current window identity.
* `TabState.#collectBaseTabData` starts with `{entries: [], lastAccessed: tab.lastAccessed}` and collects state, not a public stable tab ID. `browser.permanentKey` is an in-memory cache key; the closed-tab finalization path deletes its temporary copy. WebExtension `tabs.Tab.id`/`windows.Window.id` should not be invented from these files.
* `entry.ID` is a history/frame entry identifier. The restore code explicitly remaps it to a new unique ID because the prior session's ID may conflict. `docIdentifier` is a bfcache identifier. `docshellUUID` can be a useful matching hint, but is serialized on history entries, not a contractual unique live-tab ID. Restoring/duplicating state and absent entries undermine treating it as an infallible key.

Consequences by operation:

| Operation | What disk snapshots permit |
|---|---|
| Tab reorder | Array position changes. A path keyed by position changes referent; matching history/context can suggest continuity but duplicate tabs make it ambiguous. |
| Tab navigation/back/forward | Active entry changes; URL and entry ID are unsuitable tab identity. Overlapping history can support a heuristic, not prove all cases. |
| Tab move to another window | Window+position changes. Global candidate matching is necessary for a heuristic; remove+add is a valid observed structural change, not proof of close+open. |
| Window reorder/focus | Positional `selectedWindow`/window list does not create persistent identity. |
| Restart/restore | Runtime window counter restarts; entry IDs may be remapped; optional restoration identifiers do not cover all windows/tabs. Retained state may restore later or not at all. |
| Duplicate tabs/identical windows | Multiple equal candidates defeat URL/hash-based unique identity. |

**Recommended initial contract:** publish generation-scoped window/tab coordinates and their order, with a source generation/content digest. These coordinates identify observations, not immortal browser objects. If consumers need persistent handles, assign foxread-owned IDs with an explicit match confidence/ambiguity policy, and distinguish `observed-added` from `opened`. A separate browser API integration would be needed to improve live identity and event completeness; disk-only parsing cannot resolve these ambiguities reliably.

Timestamp names should be unambiguous: `source_session_started_at`, `source_snapshot_at`, `last_selected_at`, `observed_at`, `first_observed_at`, and optionally actual `closed_at` when present. **Never label `first_observed_at` as `opened_at`.** First observation can occur long after a tab opened, after restore, or after the observer restarted. Likewise a page's last Places visit is not the navigation time of a particular tab.

## Cadence and completeness

`SessionSaver` defaults to a 15,000 ms minimum active write interval, 3,600,000 ms idle interval, and idle detection after 180 seconds. `runDelayed` defaults to at least another 2,000 ms and dispatches via `requestIdleCallback`. These are scheduling rules, **not a 15-second freshness SLA**. Preferences, pending writes, main-thread scheduling, asynchronous content-state collection and I/O failures all affect freshness.

The saver skips permanent private browsing, filters private windows/tabs, and calls `keepOnlyWorthSavingTabs`. At the inspected revision ordinary `about:blank`/`about:newtab` entries can be persisted; do not mistakenly generalize the stricter undo-close filter to all open tabs. Empty/unworthy and private-browsing entries can be removed. Shutdown trims back/forward history according to serialization preferences. Closed-tab/window lists are bounded restoration mechanisms.

Therefore snapshot diff yields **net observed state changes**, not full CDC of UI activity. A tab can open and close, or navigate several times, between persisted snapshots without producing a corresponding observable lifecycle. Watchman cannot improve Firefox's persistence cadence. The filesystem projection should surface the source timestamp and completeness mode (`persisted-session-snapshot`) alongside data.

## Places SQLite: useful enrichment, not a tab registry

`moz_places` supplies page `id`, `url`, `title`, `guid`, `visit_count`, `last_visit_date`, and other URL-level fields. `moz_historyvisits` supplies visit `id`, `place_id`, `visit_date`, `visit_type`, `from_visit`, plus version-dependent fields such as `source` and `triggeringPlaceId`.

* Visit dates are PRTime, microseconds since the Unix epoch; sessionstore uses milliseconds. Preserve precision/unit when normalizing.
* `moz_places.id` and `guid` identify page records, not tabs. `moz_historyvisits.id` identifies visits, not tabs. Namespace them by source/profile incarnation.
* `from_visit` links to another visit; resolve its page via a self join and `moz_places`. `History.cpp::FetchReferrerInfo` discards referrer associations outside its recent-event condition, so zero/missing does not mean no real-world referrer.
* The schema's `session` column must not be mistaken for a live browser-window/session identifier: `History.cpp` inserts literal `0` into it in the inspected path.
* Page title is current URL-level metadata, not necessarily historical title-at-visit. History expiration/deletion, disabled recording, private browsing and never-loaded restore tabs prevent treating this as complete tab activity. URL joining can enrich observations but does not prove a particular tab caused a particular visit.

For an initial optional history adapter, open `places.sqlite` through SQLite read-only, run related queries in one short read transaction, and let SQLite interpret main/WAL together. `Database.cpp::SetupDurability` requests WAL and configures autocheckpoint; fallback journal modes remain possible. Watch the database directory and relevant `places.sqlite`, `places.sqlite-wal`, `places.sqlite-shm` lifecycle, not just main-file mtime. Main may remain unchanged while commits accumulate in WAL. A notification on `-shm` is not itself an application row change.

Do not copy only the live main file, or independently copy main and WAL and call the result a coherent snapshot. Use SQLite's backup API for an owned database copy, or materialize the selected projection while a read transaction holds a consistent view. Do not use `immutable=1` against a changing Firefox database. Keep reads bounded: long reader transactions can inhibit checkpoint progress and grow WAL. `PRAGMA data_version` on a retained connection can help skip queries, but it is an invalidation hint, not a changelog. A visit-ID watermark only handles a subset of inserts and misses page edits/deletions and source resets.

## occhre: actual reusable architecture and remaining obligations

The source is already separated into a useful library spine. Local absolute references below identify inspected files; the frontmatter pins the common revision.

| Crate / path under `/home/rektide/src/occhre` | Current responsibility | Foxread relevance |
|---|---|---|
| `walpin` — `crates/walpin` | Read-only SQLite citizen-reader connections acquire read marks; staggered handoff and validated ack keep history retained | Needed for dependable historical page access, but affects checkpoint retention; not a passive notification backend |
| `waltail` — `crates/sqlite/wal/src/tailer.rs` | Classify source/WAL generation, verify salts and rolling checksums, split commits, guard reads, spool FCHW1 windows | Reusable WAL observation, with explicit reset/no-WAL/torn-tail cases |
| `walview` — `crates/sqlite/view` | Position-scoped page overlays and `NativeRead` contract tying a selected commit to covering retention | Required to interpret pages against the right main-file baseline; raw changed pages alone are insufficient |
| `sqlite-records` — `crates/sqlite/records` | `RowReader`, experimental direct decoder, view-backed Turso, independent capped native Turso readers | Existing row decoding; avoid inventing another SQLite parser |
| `sqlite-changes` — `crates/sqlite/changes` | Full keyed state staging and snapshot import/export; new incremental `RowidIndex` path | Net row changes, staged P→Q ownership and explicit schema/generation limits |
| `fch-arrow` — `crates/interchange/arrow` | SQLite values, table/key/row-change model plus Arrow planning/emission | Existing semantic coupling even if foxread does not want Arrow transport |
| `fch-sync` / sinks | Coordination, delivery, outbox, downstream receipts | More machinery than first sessionstore projection needs |

`waltail::Tailer::poll` explicitly returns `NoWal`, `Idle`, `Committed`, or `Reset`. It opens paths anew, validates the generation and committed prefix, and spools before returning. Its guard narrows races; it does not create retention. WAL is a page-image journal, not a row/event stream. Checkpoint/reset can invalidate a bystander's history. The coalesced FCHW1 spool is **not** a native SQLite WAL prefix and does not by itself reconstruct every intermediate database state.

`walview::NativeRead` borrows a held pin, checks the main-fallback floor, and exposes a selected committed endpoint. Native Turso must recover exactly to that cap. A historical cap over today's mutable main file after restart is not durable replay. `sqlite-records` currently pins Turso `=0.8.0-pre.11` behind its `turso` feature, with `turso_core` `fs`/`conn_raw_api`; both reader and changes crates depend on `fch-arrow`.

The newest incremental path in `sqlite-changes/src/incremental/mod.rs` is concrete code, not merely a proposal:

1. Bootstrap `RowidIndex` structural ownership of allowed rowid btrees and overflow chains.
2. Build `ChangedPages` from validated contiguous committed windows exactly covering P→Q in one WAL generation.
3. Stage topology changes, nominate affected keys and compare native P/Q rows. Visitor output is provisional until preparation succeeds.
4. Retain operation descriptors and sparse structural journal; reread Q upserts when emitting.
5. Install the prepared journal after the caller's publication boundary, independently of downstream acknowledgement.

It refuses unsupported schema/profile/root/generation changes, WITHOUT ROWID in this incremental route, and resource limits rather than silently scan-falling back. Full-state comparison still exists separately. Its output is interval **net upserts/deletes**, not every SQL statement or browser visit event.

The current README explicitly says the end-to-end live admission/capture-owner retention join remains work. Conservative stale-tail/spill/no-WAL refusals remain active; tested generated-source extraction/outbox composition is not proof that arbitrary live Firefox profiles are admitted. Do not carry a claim of production-ready Firefox CDC from the existence of these crates.

### Reuse and possible extraction

Prefer a version-pinned dependency on the existing occhre crates for a later controlled experiment; copying them into foxread would duplicate the correctness-sensitive retention machinery. First validate a generated Firefox-like database with ordinary rowid tables, then a deliberately created test profile/version. Real Places schemas can contain features that cause an engine open failure even outside the selected table allowlist.

A useful eventual extracted toolkit could retain these domain boundaries:

* SQLite source identity + retention + committed WAL positions.
* Position-scoped page/row reading.
* Typed SQLite value/key/schema/net-change model.
* Full-state and changed-page incremental extraction, with owned continuation state.

The clearest extraction seam is to move the transport-independent `SqliteValue`, `TableDef`, keys and `RowChange` out of `fch-arrow` into a small shared model crate, then let Arrow and a Caminus-facing adapter consume that model. This is a proposal requiring occhre owner agreement; no extraction is necessary for the first foxread slice. Keep native source retention and filesystem publication receipts distinct. A published snapshot does not by itself establish that a live WAL pin can be safely advanced.

## Proposed foxread crate boundaries and first slice

These are proposed domain boundaries, not authorization to implement every crate:

| Crate | Owns |
|---|---|
| `foxread-model` | Projection types, source provenance, observation coordinates, timestamps and uncertainty |
| `foxread-sessionstore` | Candidate selection, bounded MozLZ4 decode, version-tolerant parsing and normalization |
| `foxread-observe` | Accepted snapshot generations, diff policy, watch invalidation/rescan scheduling and retry state |
| `foxread-fs` | Mapping an owned generation to the verified kopacetic/caminus interface; no Firefox parsing |
| `foxread` | CLI/config/runtime composition |
| Later `foxread-places` | Explicit read-only SQL projection/enrichment, independently replaceable by an occhre-backed source |

Watchman should enter `foxread-observe` through a small invalidation stream abstraction. The parent investigation owns its verified client API and filesystem publishing APIs; this research does not assert uninspected kopacetic/caminus calls. Source capture and consistency remain the store adapter's responsibility.

**Next feasible implementation slice:** accept an explicit profile or input-file path; parse a generated unencrypted JSONLZ4 fixture; project windows/tabs with generation-local coordinates, correct active-entry indexing and honest timestamps; publish one immutable owned generation through the selected FS adapter; then add directory invalidation and whole-snapshot comparison. Keep last-good data on transient read/parse failures and annotate unavailable/encrypted sources. Begin with observation diffs rather than ambiguous stable-tab matching. This exercises the useful end-to-end contract without waiting for Places CDC.

Meaningful fixtures should cover back navigation (`index` not last), duplicate URLs, reorder/move ambiguity, missing histories, closed-vs-open groups, startup/shutdown file roles, unsupported encryption, truncated/oversized compressed input, and restart provenance. Optional SQLite work should separately prove coherent WAL reads, deletion handling, source replacement, and explicit reset/rebaseline semantics with synthetic data.

## Cross-references

* The sibling watching investigation at `design/watching/watchman0.gpt6a.md` owns Watchman client capability and invalidation handling. Its filename/scope were supplied by the parent; its same-wave contents were not read for this independent research.
* [occhre records README](file:///home/rektide/src/occhre/crates/sqlite/records/README.md) grounds engine/read-view capabilities and limitations; [changes README](file:///home/rektide/src/occhre/crates/sqlite/changes/README.md) explains staged net-state ownership. These are reuse evidence, not a Firefox integration claim.
* [occhre WAL README](file:///home/rektide/src/occhre/crates/sqlite/wal/README.md), [view README](file:///home/rektide/src/occhre/crates/sqlite/view/README.md), and [walpin README](file:///home/rektide/src/occhre/crates/walpin/README.md) separate observation, historical readability and retention. That distinction is why SQLite dump/diff is the simpler optional initial adapter.

[sessionfile]: https://github.com/mozilla-firefox/firefox/blob/3934f7874da5d43c3eac71e6f44e0afe79d9071f/browser/components/sessionstore/SessionFile.sys.mjs
[writer]: https://github.com/mozilla-firefox/firefox/blob/3934f7874da5d43c3eac71e6f44e0afe79d9071f/browser/components/sessionstore/SessionWriter.sys.mjs
[saver]: https://github.com/mozilla-firefox/firefox/blob/3934f7874da5d43c3eac71e6f44e0afe79d9071f/browser/components/sessionstore/SessionSaver.sys.mjs
[store]: https://github.com/mozilla-firefox/firefox/blob/3934f7874da5d43c3eac71e6f44e0afe79d9071f/browser/components/sessionstore/SessionStore.sys.mjs
[tab]: https://github.com/mozilla-firefox/firefox/blob/3934f7874da5d43c3eac71e6f44e0afe79d9071f/browser/components/sessionstore/TabState.sys.mjs
[history]: https://github.com/mozilla-firefox/firefox/blob/3934f7874da5d43c3eac71e6f44e0afe79d9071f/toolkit/modules/sessionstore/SessionHistory.sys.mjs
[schema]: https://github.com/mozilla-firefox/firefox/blob/3934f7874da5d43c3eac71e6f44e0afe79d9071f/browser/components/sessionstore/session.schema.json
[ioheader]: https://github.com/mozilla-firefox/firefox/blob/3934f7874da5d43c3eac71e6f44e0afe79d9071f/xpcom/ioutils/IOUtils.h
[ioimpl]: https://github.com/mozilla-firefox/firefox/blob/3934f7874da5d43c3eac71e6f44e0afe79d9071f/xpcom/ioutils/IOUtils.cpp
[tables]: https://github.com/mozilla-firefox/firefox/blob/3934f7874da5d43c3eac71e6f44e0afe79d9071f/toolkit/components/places/nsPlacesTables.h
[db]: https://github.com/mozilla-firefox/firefox/blob/3934f7874da5d43c3eac71e6f44e0afe79d9071f/toolkit/components/places/Database.cpp
[placeshistory]: https://github.com/mozilla-firefox/firefox/blob/3934f7874da5d43c3eac71e6f44e0afe79d9071f/toolkit/components/places/History.cpp
