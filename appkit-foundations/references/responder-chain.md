# Responder Chain Reference

Deep reference for first responder mechanics, event propagation, action dispatch, and key view loop on macOS 26+.

## How the Responder Chain Works

The responder chain is a linked list of `NSResponder` objects. When an event or action isn't handled by the first responder, it propagates up the chain:

```
First Responder (e.g., focused NSTextField)
  → Superview
  → ... up the view hierarchy ...
  → NSWindow (contentView → window)
  → NSWindowController
  → NSDocument (if document-based)
  → NSApplication
  → NSApp.delegate
```

Each `NSResponder` has a `nextResponder` property. You can override it to insert custom objects into the chain, but this is rarely needed.

```swift
// Inspect the responder chain
func printResponderChain(from responder: NSResponder?) {
    var current = responder
    var depth = 0
    while let r = current {
        print("\(String(repeating: "→ ", count: depth))\(type(of: r))")
        current = r.nextResponder
        depth += 1
    }
}

// Usage:
printResponderChain(from: window.firstResponder)
```

## First Responder

The first responder is the currently focused UI element. `NSWindow.firstResponder` returns it (or `nil` if nothing is focused).

**Making a view the first responder:**

```swift
// Request first responder status (returns whether the view accepted)
let accepted = window.makeFirstResponder(textField)

// A view can refuse first responder:
override func becomeFirstResponder() -> Bool {
    return isEditable  // refuse if not editable
}

override func resignFirstResponder() -> Bool {
    commitPendingChanges()  // save before losing focus
    return true
}
```

**Common first responder scenarios:**

- `NSTextField`: Becomes first responder when the user clicks it or tabs to it.
- `NSTableView`: Becomes first responder for keyboard navigation (arrow keys).
- `NSWindow`: If no subview accepts first responder, the window itself is first responder.

## sendAction(_:to:from:)

`NSApplication.sendAction(_:to:from:)` sends an action message through the responder chain. This is the primary mechanism for menu items and toolbar buttons.

**How it works:**

1. If `to` is non-nil, send the action directly to that target.
2. If `to` is `nil`, start at the first responder and walk the responder chain.
3. Each responder is asked if it responds to the selector (`responds(to:)`).
4. The first responder that handles the action wins.

```swift
// From a menu item or toolbar button — target is nil, walks the chain
NSApp.sendAction(#selector(delete(_:)), to: nil, from: nil)

// From code with a specific target
NSApp.sendAction(#selector(delete(_:)), to: myController, from: nil)

// The responder that handles it:
@objc func delete(_ sender: Any?) {
    // This method must be @objc and take a single parameter (or none)
}
```

**Action method signatures:**

```swift
// Standard: takes the sender
@objc func save(_ sender: Any?) { }

// No parameter — also valid
@objc func save() { }

// Multiple parameters — NOT valid for target-action
// @objc func save(_ sender: Any?, context: Context) { }  // Won't work
```

## Validating Menu Items and Toolbar Items

`validateUserInterfaceItem(_:)` is called on each responder in the chain to determine if a menu item or toolbar item should be enabled.

```swift
override func validateUserInterfaceItem(_ item: NSValidatedUserInterfaceItem) -> Bool {
    switch item.action {
    case #selector(delete(_:)):
        return !selectedItems.isEmpty
    case #selector(copy(_:)):
        return selectedItems.count > 0
    case #selector(paste(_:)):
        return NSPasteboard.general.canReadObject(forClasses: [NSString.self])
    default:
        // Pass to the next responder in the chain
        return super.validateUserInterfaceItem(item)
    }
}
```

**Key points:**

- Return `true` to enable the item, `false` to disable it.
- Call `super` for actions you don't handle — the next responder may handle them.
- This is called frequently (every time the menu is opened). Keep it fast.
- The system automatically grays out disabled items.

## Key View Loop

The key view loop determines the order in which views receive focus when the user presses Tab (or Shift+Tab for reverse).

**Explicit key view loop:**

```swift
// Set nextKeyView on each view
textField1.nextKeyView = textField2
textField2.nextKeyView = textField3
textField3.nextKeyView = button
button.nextKeyView = textField1  // cycle back to start
```

**Auto Layout-based key view loop (macOS 14+):**

On macOS 14 and later, AppKit can automatically determine the key view loop from the layout order. Set `NSWindow.allowsAutomaticKeyViewLoop = true` (the default) and ensure your constraints define a logical reading order.

```swift
// Disable automatic loop if you need manual control
window.allowsAutomaticKeyViewLoop = false
```

**Customizing loop behavior:**

```swift
// A view that should be skipped in the tab loop
override var canBecomeKeyView: Bool { false }

// A view that accepts click-to-focus but not tab-to-focus
override func acceptsFirstMouse(for event: NSEvent?) -> Bool { true }
```

## Event Propagation

Raw events (`NSEvent`) follow a different path than actions:

1. `NSApplication` receives the event from the window server.
2. `NSWindow` receives it via `sendEvent(_:)`.
3. The window dispatches mouse events to the view under the cursor (`hitTest(_:)`).
4. The event is handled by the target view's event methods.
5. If unhandled, it does NOT propagate up the responder chain (unlike actions).

**Mouse events:**

```swift
override func mouseDown(with event: NSEvent) {
    // Handle or call super to pass to next responder
    super.mouseDown(with: event)
}

override func rightMouseDown(with event: NSEvent) {
    // Right-click — show context menu
    let menu = NSMenu()
    menu.addItem(NSMenuItem(title: "Copy", action: #selector(copy(_:)), keyEquivalent: ""))
    NSMenu.popUpContextMenu(menu, with: event, for: self)
}
```

**Keyboard events:**

```swift
override func keyDown(with event: NSEvent) {
    // Handle special keys
    if event.keyCode == 53 {  // Escape
        cancelOperation(self)
        return
    }
    // Pass unhandled keys to the input context (for IME, etc.)
    interpretKeyEvents([event])
}

// For menu key equivalents, override performKeyEquivalent:
override func performKeyEquivalent(with event: NSEvent) -> Bool {
    if event.modifierMask == .command && event.characters == "s" {
        save(self)
        return true  // handled
    }
    return super.performKeyEquivalent(with: event)
}
```

## NSWindow's First Responder Management

`NSWindow` provides additional control over first responder behavior:

```swift
// Prevent a window from becoming key (e.g., for panels)
window.isFloatingPanel = true

// Force the window to resign first responder when it loses key status
window.resignsFirstResponderWhenKey = false  // default is true

// Check if the window can become key
override var canBecomeKeyWindow: Bool { true }

// Check if the window can become main
override var canBecomeMainWindow: Bool { true }
```

## Custom Responder Chain Objects

You can insert non-view objects into the responder chain by overriding `nextResponder`:

```swift
final class Coordinator {
    weak var viewController: MyViewController?

    var nextResponder: NSResponder? { viewController }
}

// In the view controller:
override var nextResponder: NSResponder? {
    get { coordinator }
    set { super.nextResponder = newValue }
}
```

This is useful for MV-style coordination where you want a model object to handle actions without the view controller acting as a middleman. Use sparingly — it makes the chain harder to debug.
