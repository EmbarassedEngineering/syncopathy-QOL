# Implementation Plan - Reliable Global Keyboard Shortcuts

Solve the focus-stealing / text field interference problem at its root, and lay groundwork for configurable shortcuts.

## Context & Problem

The existing `GlobalShortcuts` widget wraps a `FocusNode` around the entire widget tree and uses `onKeyEvent`. This has two failure modes:
1. **Focus stealing**: When any button, list item, or UI element receives focus, it consumes the key event before the root `FocusNode` can see it — e.g., space bar scrolls a list instead of toggling playback.
2. **Text field bleed**: Raw key interception triggers playback shortcuts while the user is typing.

The creator confirmed this was bad enough to remove shortcuts entirely.

## Chosen Solution: `HardwareKeyboard.addHandler` + `EditableText` Guard

Replace the `Focus`-based system with a **raw `HardwareKeyboard` handler** registered at the `StatefulWidget` lifecycle level, completely bypassing the focus tree. This is the correct approach for global desktop shortcuts in Flutter.

Guard against text field input by checking `FocusManager.instance.primaryFocus?.context?.widget is EditableText` before processing any shortcut.

### Why not `hotkey_manager` package?
The `hotkey_manager` package handles *system-wide* hotkeys (work even when app is not focused), which is overkill and adds a native dependency. `HardwareKeyboard.addHandler` is the right scope.

### Configurable Shortcuts (Phase 2)
Store bindings in `SharedPreferences` as a `Map<String, ShortcutBinding>` serialized to JSON. Build a `ShortcutConfigService` singleton injected via the existing IoC pattern. Expose a settings page section for rebinding.

---

## Proposed Changes

### [MODIFY] `lib/global_shortcuts.dart`

Replace `FocusNode` + `Focus.onKeyEvent` with `HardwareKeyboard.instance.addHandler`:

```dart
class _GlobalShortcutsState extends State<GlobalShortcuts> {
  late final bool Function(KeyEvent) _keyHandler;

  bool _isEditingText() {
    final focus = FocusManager.instance.primaryFocus;
    return focus?.context?.widget is EditableText;
  }

  @override
  void initState() {
    super.initState();
    _keyHandler = (KeyEvent event) {
      if (event is! KeyDownEvent) return false;
      if (_isEditingText()) return false; // never steal from text fields
      final videoPlayer = /* get from context or service locator */;
      if (event.logicalKey == LogicalKeyboardKey.space) {
        videoPlayer.togglePause();
        return true; // consumed
      }
      // ... etc
      return false;
    };
    HardwareKeyboard.instance.addHandler(_keyHandler);
  }

  @override
  void dispose() {
    HardwareKeyboard.instance.removeHandler(_keyHandler);
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => widget.child; // no Focus wrapper needed
}
```

> **Key insight:** `addHandler` returns `true` to consume, `false` to propagate. Native text fields still receive their events because the handler runs *before* the focus tree, not instead of it.

### [MODIFY] `conductor/tracks/shortcuts/plan.md`
Update this file to reflect the new approach.

---

## Verification Plan

### Automated Tests
- `flutter test` — confirm existing tests still pass
- `dart analyze lib/global_shortcuts.dart` — no issues

### Manual Test Checklist
- [ ] Space bar toggles playback (no UI element needs to be unfocused first)
- [ ] Left/Right arrows seek without first clicking the video area
- [ ] Ctrl+Left/Right jumps playlist entries
- [ ] Space bar works normally in search/filter text fields (does NOT trigger playback)
- [ ] After clicking a button (e.g., favorite), arrow keys still seek
- [ ] After typing in a text field, tabbing out restores playback shortcuts

---

## Phase 2: Configurable Shortcuts (Backlog)

1. `lib/shortcuts/shortcut_config_service.dart` — loads/saves `Map<String, LogicalKeyboardKey>` from `SharedPreferences`
2. Settings page section: key rebinding UI (capture next key press, validate no conflicts)
3. `GlobalShortcuts` reads from service instead of hardcoded keys
