# Semantic Memory: MediaManager Patterns

> Patterns distilled from media_manager.dart refactoring (2026-03-26)

---

## DB Operations Must Always Be Awaited

`DatabaseHelper` is a singleton. Fire-and-forget calls (`DatabaseHelper().someMethod()` without `await`) cause silent failures and desync between DB and in-memory state.

**Anti-pattern:**
```dart
DatabaseHelper().deleteVideo(video.id!); // ← WRONG
```
**Pattern:**
```dart
await DatabaseHelper().deleteVideo(video.id!); // ← correct
```

Methods that call DB should always return `Future<void>`, never `void`.

---

## Never Mutate Shared Lists Inside `Future.wait`

Concurrent closures inside `Future.wait` calling `.add()` on the same `List` are not thread-safe in Dart.

**Anti-pattern:**
```dart
processFutures.add(() async {
  _allVideos.add(video); // ← concurrent mutation, data loss risk
}());
await Future.wait(processFutures);
```

**Pattern:**
```dart
final List<Future<Video?>> futures = [...];
final results = await Future.wait(futures);
_allVideos.addAll(results.nonNulls); // ← safe, single-threaded
```

---

## Preserve User Data on Re-Process

When a file changes and needs re-processing, always copy over user-owned fields from the existing DB record:
- `isFavorite`, `isDislike`, `playCount`, `dateFirstFound`, `id`

Forgetting this resets user preferences silently on every library scan.

---

## Null-Guard Before Force-Unwrap on Model IDs

Model IDs (`video.id`, `category.id`) can be null for new-but-unsaved records. Always guard:

```dart
if (video.id == null) {
  Logger.error('Cannot delete video with null id: ${video.title}');
  return;
}
await DatabaseHelper().deleteVideo(video.id!);
```

---

## Batch DB Operations Over Per-Item Loops

Use `batchInsertVideos`, `batchUpdateVideos`, `batchUpdateFunscriptMetadata` rather than per-video individual inserts inside loops. This significantly reduces DB round-trips on large libraries.
