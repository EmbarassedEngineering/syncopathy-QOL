# syncopathy-QOL — Project Continuity

> Read this at the start of every session. Update after every meaningful change.  
> Last updated: 2026-03-26

---

## What Is This Project?

A Flutter desktop (Windows) QOL companion for syncopathy playback. It manages a video/funscript media library, controls playback, and provides filtering/search/categorization tools.

---

## Current State (as of 2026-03-26)

### Active Conductor Track
- **shortcuts** (`conductor/tracks/shortcuts/plan.md`) — ✅ PHASE 1 COMPLETE
  - Replaced `Focus`/`FocusNode` approach with `HardwareKeyboard.instance.addHandler`
  - `_isEditingText()` guard prevents shortcuts firing inside text fields
  - `VideoPlayer` accessed via `getIt<VideoPlayer>()` — no context needed in handler
  - Configurable shortcuts remain as Phase 2 (backlog)

### Recent Significant Changes
- `media_manager.dart` was fully refactored (2026-03-26):
  - Decomposed 190-line `refreshVideos` into 5 focused methods
  - Fixed missing `await` on `deleteVideo` DB call
  - Fixed concurrent `_allVideos` mutation inside `Future.wait`
  - `saveFavorite`/`saveDislike` now return `Future<void>`
  - Null guards added throughout category/delete methods
  - `_getVideoMetadata` simplified to `_getVideoDuration` (no cache param)

### Open Issues / Mistakes & Learnings

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| `GlobalShortcuts` unreliable | Uses `FocusNode`; any widget gaining focus breaks shortcuts | Replace with `HardwareKeyboard.addHandler` |
| `saveFavorite`/`saveDislike` had `void` return | Fire-and-forget async — errors invisible | Fixed: now `Future<void>` |
| Missing `await` on DB delete | `deleteVideo` had unawaited `DatabaseHelper().deleteVideo()` | Fixed with await + null guard |
| Concurrent list mutation | 8 parallel closures wrote to `_allVideos` simultaneously | Fixed: futures return `Video?`, merged after `Future.wait` |

---

## Architecture Notes

### Stack
- Flutter (Windows desktop target)
- SQLite via `DatabaseHelper` (singleton pattern — **always `await` DB calls**)
- `signals_flutter` for reactive UI (`Signal<T>`)
- `async_locks` Semaphore for concurrency control
- `provider` for DI (IoC via `lib/ioc.dart`)

### Key Subsystems

| Subsystem | Files | Notes |
|-----------|-------|-------|
| Media Library | `lib/media_library/` | 15 dart files; `media_manager.dart` is the core I/O layer |
| Player | `lib/player/` | `VideoPlayer` base class; embedded + external (mpv) modes |
| Shortcuts | `lib/global_shortcuts.dart` | Currently `FocusNode`-based (buggy); rewrite pending |
| SQLite | `lib/sqlite/` | `database_helper.dart` + models |
| Settings | `lib/settings_page.dart` | Large file (18kb); settings stored in SharedPreferences |

### Fragile Areas
- `DatabaseHelper` is a singleton constructed with `DatabaseHelper()` — **never parallelize writes** — always use batch methods
- `_allVideos` in `MediaManager` must only be mutated from the main thread (after `Future.wait`)
- `VideoPlayer` has two implementations (embedded / external mpv); shortcuts must work for both

---

## Pending / Backlog

- [ ] Implement `HardwareKeyboard`-based global shortcuts (shortcuts track)
- [ ] Configurable shortcuts UI (Phase 2 of shortcuts track)
- [ ] Add unit tests for `MediaManager` (needs DB mocking infrastructure)

---

## Conventions

- All DB operations must be `await`ed
- New methods that call DB should return `Future<void>` not `void`
- Null-guard before `!` force-unwraps on model IDs; log + return instead of crashing
- Batch operations preferred over per-item loops for DB writes
