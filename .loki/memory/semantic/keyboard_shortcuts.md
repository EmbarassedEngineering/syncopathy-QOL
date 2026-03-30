# Semantic Memory: Flutter Global Keyboard Shortcuts

> Pattern distilled from research + syncopathy-QOL shortcuts investigation (2026-03-26)

---

## Problem: Focus-Stealing & Text Field Bleed

The `Shortcuts`/`Actions`/`Focus`/`FocusNode` widget approach for global shortcuts has two failure modes in Flutter desktop apps:

1. **Focus stealing**: Any widget that gains focus (buttons, lists, dropdowns) captures key events before the root `FocusNode`, making shortcuts unreliable.
2. **Text field bleed**: Shortcuts fire while the user types if a raw listener doesn't guard against it.

---

## Solution: `HardwareKeyboard.instance.addHandler`

Register a raw hardware key handler at the `StatefulWidget` lifecycle level. This runs **before** the focus tree, doesn't steal focus, and lets text fields work normally.

```dart
class _MyWidgetState extends State<MyWidget> {
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
      if (_isEditingText()) return false; // ← critical guard
      // handle shortcuts here
      return false; // return true to consume, false to propagate
    };
    HardwareKeyboard.instance.addHandler(_keyHandler);
  }

  @override
  void dispose() {
    HardwareKeyboard.instance.removeHandler(_keyHandler);
    super.dispose();
  }
}
```

### Rules
- **Always** remove the handler in `dispose()` to prevent memory leaks
- Return `true` to consume/block the event from propagating; `false` to pass through
- The `isEditingText()` guard is **mandatory** — without it, shortcuts fire while typing
- This is desktop-only; not intended for mobile

### When NOT to use this
- If you need *system-wide* hotkeys (app unfocused) → use `hotkey_manager` package instead
- For simple widgets that *already have focus* → `Focus.onKeyEvent` is fine

---

## Phase 2: Configurable Shortcuts

1. Define a `ShortcutBinding` model: `{String action, LogicalKeyboardKey key, bool ctrl, bool shift}`
2. Store as JSON in `SharedPreferences`
3. Load into a singleton `ShortcutConfigService` at startup
4. UI: capture next key press for rebinding; validate no conflicts
5. `HardwareKeyboard` handler reads from service map instead of hardcoded keys

---

## References
- Flutter docs: `HardwareKeyboard` class
- Stack Overflow: ServicesBinding.instance.keyboard.addHandler reliable desktop shortcuts
- `hotkey_manager` (pub.dev) — for system-wide shortcuts
