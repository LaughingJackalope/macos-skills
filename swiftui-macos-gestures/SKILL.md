---
name: swiftui-macos-gestures
description: "Gesture handling on macOS — TapGesture, LongPressGesture, DragGesture, MagnifyGesture, RotateGesture, onHover, cursor control (.cursor), trackpad gestures, onScrollWheel, right-click/secondary click, EventModifiers, NSEvent bridge for raw mouse/keyboard. Use when handling pointer, trackpad, scroll, or keyboard input in macOS SwiftUI views."
sources: https://developer.apple.com/documentation/swiftui/gestures, https://developer.apple.com/documentation/appkit/nsevent, https://developer.apple.com/documentation/swiftui/view/oncontinuoushover(coordinatespace:perform:)-4ehfq, https://developer.apple.com/documentation/swiftui/composing-swiftui-gestures, https://developer.apple.com/documentation/swiftui/magnifygesture, https://developer.apple.com/documentation/appkit/nsresponder/scrollwheel(with:), https://developer.apple.com/documentation/swiftui/view/onscrollphasechange(_:), https://developer.apple.com/documentation/swiftui/view/accessibilityrotor(_:entries:entrylabel:)-2jbai, https://developer.apple.com/videos/play/wwdc2025/229/, https://developer.apple.com/videos/play/wwdc2025/306/, https://developer.apple.com/videos/play/wwdc2025/266/, https://developer.apple.com/videos/play/wwdc2023/10162/, https://developer.apple.com/videos/play/wwdc2023/10248/, https://developer.apple.com/videos/play/wwdc2023/10036/, https://developer.apple.com/videos/play/wwdc2021/10023/, https://developer.apple.com/videos/play/wwdc2020/10094/, https://developer.apple.com/documentation/accessibility/creating_accessible_views, https://developer.apple.com/documentation/swiftui/view-accessibility, https://developer.apple.com/documentation/xcode/understanding-and-improving-swiftui-performance, https://developer.apple.com/documentation/xcode/identifying-responsiveness-issues-in-your-app, https://developer.apple.com/documentation/swiftui/view-equatableview, https://developer.apple.com/documentation/swiftui/timelineview, https://developer.apple.com/documentation/swiftui/lazyvstack, https://developer.apple.com/documentation/swiftui/lazyvgrid, https://help.apple.com/instruments/mac/current/, https://developer.apple.com/documentation/swiftui/view/onscrollwheel(_:), https://developer.apple.com/documentation/swiftui/view/onkeypress(_:action:), https://developer.apple.com/documentation/appkit/nsevent/1534971-addlocalmonitorforevents, https://developer.apple.com/documentation/swiftui/eventmodifiers, https://developer.apple.com/documentation/appkit/nscursor, https://developer.apple.com/documentation/swiftui/view/ontapgesture(count:perform:), https://developer.apple.com/documentation/swiftui/view/gesture(_:including:), https://developer.apple.com/documentation/swiftui/rotategesture, https://nilcoalescing.com/blog/TrackingHoverLocationInSwiftUI, https://levelup.gitconnected.com/swiftui-macos-map-finger-position-on-trackpad-to-view-6e5c5b872f43, https://levelup.gitconnected.com/swiftui-macos-handle-mouse-trackpad-events-mouse-up-mouse-down-pressure-change-f3ff9755be0e, https://github.com/stephancasas/CGEventSupervisor, https://github.com/0Itsuki0/MouseTrackpadEventsDemo-SwiftUI-MacOS, https://github.com/lloydsargent/SwipeModifier, https://developer.apple.com/forums/thread/742868, https://developer.apple.com/forums/thread/702137, https://developer.apple.com/forums/thread/133398, https://developer.apple.com/forums/thread/739849, https://www.hackingwithswift.com/quick-start/swiftui/how-to-use-instruments-to-profile-your-swiftui-code-and-identify-slow-layouts, https://www.donnywals.com/using-instruments-to-profile-a-swiftui-app/, https://useyourloaf.com/blog/swiftui-tasks-blocking-the-mainactor/, https://useyourloaf.com/blog/context-menus-for-tables/, https://wadetregaskis.com/including-services-in-contextual-menus-in-swiftui/, https://digitalblake.com/2026/04/28/swiftui-vs-appkit-macos-ui-performance/, https://coldfusion-example.blogspot.com/2026/02/optimizing-swiftui-table-performance.html, https://swiftwithmajid.com/2024/07/02/mastering-scrollview-in-swiftui-scroll-phases/, https://wesleydegroot.nl/blog/keyboard-navigation, https://wesleydegroot.nl/blog/voiceover, https://developer.apple.com/tutorials/instruments/identifying-a-hang, https://sosumi.ai/documentation/xcode/understanding-hangs-in-your-app, https://dkvekariya.medium.com/swiftui-performance-tuning-guide-2026-part-2-fd1ef785fbf7
researched: 2026-06-06
platform: macOS 26+
---

# SwiftUI macOS Gestures

Handle pointer, trackpad, scroll, and keyboard input in macOS SwiftUI views. Targets macOS 26+ with Swift 6.3 patterns; notes backward-compatible to macOS 13 unless stated. See [references/gesture-patterns.md](references/gesture-patterns.md) for complete recipes including custom hover with cursor change, drag coordinate conversion, scroll wheel handling, NSEvent local monitors, and right-click context menus with primary action.

## Contents

- [Pointer Gestures](#pointer-gestures)
  - [TapGesture](#tapgesture)
  - [LongPressGesture](#longpressgesture)
  - [DragGesture](#draggesture)
  - [Simultaneous Gestures](#simultaneous-gestures)
- [Trackpad Gestures](#trackpad-gestures)
  - [MagnifyGesture (Pinch)](#magnifygesture-pinch)
  - [RotateGesture](#rotategesture)
  - [Mojave Scroll Simulation](#mojave-scroll-simulation)
- [Hover & Cursor](#hover--cursor)
  - [onHover](#onhover)
  - [onContinuousHover](#oncontinuoushover)
  - [Cursor Types with `.cursor`](#cursor-types-with-cursor)
- [Scroll Gestures](#scroll-gestures)
  - [onScrollWheel](#onscrollwheel)
  - [Scroll Layout Modifiers](#scroll-layout-modifiers)
- [Secondary Click](#secondary-click)
- [EventModifiers](#eventmodifiers)
- [NSEvent Bridge](#nsevent-bridge)
- [Keyboard Events](#keyboard-events)
  - [onKeyPress](#onkeypress)
  - [keyboardShortcut](#keyboardshortcut)
- [Constraints & Gotchas](#constraints-and-gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Pointer Gestures

### TapGesture

`TapGesture` recognizes single and double taps. On macOS, it responds to left-click by default.

```swift
struct SelectableCard: View {
    @State private var selected = false

    var body: some View {
        RoundedRectangle(cornerRadius: 8)
            .fill(selected ? Color.accentColor : Color.gray.opacity(0.2))
            .frame(width: 120, height: 80)
            .onTapGesture { selected.toggle() }
    }
}
```

**Double-tap** with `count:`:

```swift
.onTapGesture(count: 2) { openItem() }
```

**Multi-count** taps (3+, if needed — rare on macOS):

```swift
.onTapGesture(count: 3) { resetView() }
```

**`EventModifiers` on tap** to require a modifier key:

```swift
.onTapGesture {
    // Access modifiers via the gesture closure when combined
    // See EventModifiers section
}
```

### LongPressGesture

`LongPressGesture` requires the user to press and hold for a minimum duration. On macOS, this triggers via sustained left-click (or trackpad press).

```swift
struct DraggableToken: View {
    @State private = false

    var body: some View {
        Circle()
            .fill(Color.blue)
            .frame(width: 40, height: 40)
            .scaleEffect(pressing ? 1.2 : 1.0)
            .animation(.easeInOut(duration: 0.15), value: pressing)
            .gesture(
                LongPressGesture(minimumDuration: 0.5)
                    .onChanged { pressing = true }
                    .onEnded { _ in
                        pressing = false
                        showDetail()
                    }
            )
    }
}
```

**Key parameters:**
- `minimumDuration` — seconds before the gesture is recognized (default: 0.5).
- `maximumDistance` — maximum movement in points before the gesture fails (default: 10). Increase this for trackpad use since fingers drift.

### DragGesture

`DragGesture` tracks continuous pointer movement from the initial press point. The most common gesture for repositioning views, scrubbing, and drag-and-drop.

```swift
struct PositionableAnnotation: View {
    @State private var position = CGPoint(x: 0, y: 0)
    @State private var dragOffset = CGSize.zero

    var body: some View {
        Circle()
            .fill(Color.red)
            .frame(width: 24, height: 24)
            .position(x: position.x + dragOffset.width, y: position.y + dragOffset.height)
            .gesture(
                DragGesture(minimumDistance: 2)
                    .onChanged { value in
                        dragOffset = value.translation
                    }
                    .onEnded { value in
                        position.x += value.translation.width
                        position.y += value.translation.height
                        dragOffset = .zero
                    }
            )
    }
}
```

**`minimumDistance`** — pixels of movement before the drag activates (default: 10). Set lower (2–5) for precise control, higher (15–20) to avoid accidental drags on click.

**Coordinate spaces** — `DragGesture` provides `.local`, `.global`, and named coordinate spaces:

```swift
DragGesture(coordinateSpace: .named("canvas"))
    .onChanged { value in
        // value.location — current point in the named coordinate space
        // value.startLocation — where the drag began
        // value.translation — cumulative offset from start
    }
```

### Simultaneous Gestures

SwiftUI gestures are exclusive by default — only one gesture per view "wins." Use `.simultaneousGesture()` or `.gesture(someGesture, including:)` to allow overlap.

```swift
.gesture(
    TapGesture()
        .onEnded { select() }
)
.simultaneousGesture(
    DragGesture()
        .onChanged { drag($0.translation) }
)
```

**`including:` for explicit mask control:**

```swift
.gesture(
    DragGesture(),
    including: .gesture // Allows other gestures alongside this one
)
```

## Trackpad Gestures

### MagnifyGesture (Pinch)

`MagnifyGesture` tracks pinch-to-zoom from the trackpad. The `magnification` value starts at 1.0 and scales proportionally.

```swift
struct ZoomableImageView: View {
    @State private var magnification: CGFloat = 1.0
    @GestureState private var gestureMagnification: CGFloat = 1.0

    var body: some View {
        Image("artwork")
            .resizable()
            .aspectRatio(contentMode: .fit)
            .scaleEffect(magnification * gestureMagnification)
            .frame(width: 400, height: 300)
            .gesture(
                MagnifyGesture()
                    .updating($gestureMagnification) { value, state, _ in
                        state = value.magnification
                    }
                    .onEnded { value in
                        magnification *= value.magnification
                        magnification = magnification.clamped(to: 0.5...4.0)
                    }
            )
    }
}
```

**`@GestureState`** automatically resets to its initial value when the gesture ends, making it ideal for transient tracking during the gesture lifetime.

**Alternative** — `magnitude` property tracking for cumulative update during gesture:

```swift
.onChanged { value in
    // value.magnification is relative to gesture start
    currentScale = baseScale * value.magnification
}
```

### RotateGesture

`RotateGesture` tracks two-finger rotation on the trackpad. `angle` is in `Angle` (`radians` by default).

```swift
struct RotatableShape: View {
    @State private var rotation: Angle = .zero
    @GestureState private var gestureRotation: Angle = .zero

    var body: some View {
        Rectangle()
            .fill(Color.orange)
            .frame(width: 120, height: 80)
            .rotationEffect(rotation + gestureRotation)
            .gesture(
                RotateGesture()
                    .updating($gestureRotation) { value, state, _ in
                        state = value.rotation
                    }
                    .onEnded { value in
                        rotation += value.rotation
                    }
            )
    }
}
```

### Mojave Scroll Simulation

**`.simultaneously(with:)` for multi-touch (2025):** The recommended pattern for simultaneous drag+magnify is `MagnificationGesture().simultaneously(with: DragGesture())`, which lets Apple's gesture recognition handle multi-touch coordination. Avoid `.simultaneousGesture()` chaining for this — it can cause unresponsiveness on macOS trackpads.
**Known issue:** `MagnificationGesture` on macOS may stop firing `onChanged` after repeated use. Workaround: bind scale to `@State` and apply via `scaleEffect`, or combine with a `DragGesture` and apply a tiny offset nudge at magnification end to reset gesture state.
Trackpad "scroll" in SwiftUI is handled via `onScrollWheel` (see [Scroll Gestures](#scroll-gestures)). For pre-macOS 14.4 compatibility, use `DragGesture` with the scroll axis locked:

```swift
.simultaneousGesture(
    DragGesture()
        .onChanged { value in
            // Negative because natural scrolling inverts the axis
            offset -= value.translation.y
        }
)
```

## Hover & Cursor

### onHover

`onHover(perform:)` triggers when the pointer enters or exits the view's bounds. Available macOS 11+.

```swift
struct HoverAwareButton: View {
    @State private var isHovered = false

    var body: some View {
        Text("Hover me")
            .padding()
            .background(isHovered ? Color.blue : Color.gray)
            .cornerRadius(6)
            .onHover { hovering in
                isHovered = hovering
            }
    }
}
```

### onContinuousHover

`onContinuousHover(perform:)` provides high-frequency hover position updates during movement. Useful for tooltips, magnifying glass effects, or drawing apps.

```swift
struct DrawingCanvas: View {
    @State private var lastPoint: CGPoint?

    var body: some View {
        Canvas { context, size in
            // render strokes
        }
        .onContinuousHover { phase in
            switch phase {
            case .active(let point):
                lastPoint = point
            case .ended:
                lastPoint = nil
            }
        }
    }
}
```

**Performance note:** `onContinuousHover` fires at display refresh rate. Keep the closure lightweight — update `@State` variables, not perform rendering.
**Reliability note (2025 research):** `onHover` can miss mouse-exit events at high cursor speeds, causing multiple targets to remain "active" in grids or fine-target layouts. For reliable hover tracking, prefer `onContinuousHover` which fires continuously and provides balanced enter/exit events. Alternatively, use a custom `NSTrackingArea` via `NSViewRepresentable` for pixel-perfect hover detection.
**NSCursor push/pop pattern (macOS 26):** A modern approach to cursor management uses `NSCursor.arrow.push()` on hover entry and `NSCursor.pop()` on exit, bypassing the system's cursor rect lookup. This is more reliable than `NSCursor.set()` which can be overridden by the system. Always balance push/pop calls and use `contentShape(Rectangle())` to ensure proper hit-testing bounds.

### Cursor Types with `.cursor`

`.cursor(_:)` changes the pointer appearance when hovering a view. Available macOS 11+ (unreliable in pure SwiftUI — works consistently inside `NSViewRepresentable` context; see [Constraints & Gotchas](#constraints-and-gotchas)).

```swift
// Pure SwiftUI — may not work outside NSViewRepresentable
.onHover { hovering in
    if hovering {
        NSCursor.crosshair.set()
    } else {
        NSCursor.arrow.set()
    }
}
```

**Available cursor types (AppKit `NSCursor`):**

| Cursor | Use Case |
|--------|----------|
| `.arrow` | Default pointer |
| `.crosshair` | Selection, drawing tools |
| `.iBeam` | Text insertion point |
| `.pointingHand` | Clickable links/buttons |
| `.openHand` | Draggable content (grab) |
| `.closedHand` | Actively dragging |
| `.resizeLeftRight` | Horizontal resize handle |
| `.resizeUpDown` | Vertical resize handle |
| `.resizeNorthWestSouthEast` | Corner resize |
| `.resizeNorthEastSouthWest` | Corner resize (other diagonal) |
| `.operationNotAllowed` | Drop target rejected |
| `.dragCopy` | Drag with copy modifier |
| `.dragLink` | Drag with link modifier |
| `.contextualMenu` | Context menu available |
| `.IBeamForVerticalLayout` | Vertical text |

## Scroll Gestures

### onScrollWheel

`onScrollWheel` intercepts scroll wheel and trackpad scroll events. Available macOS 14.4+.
**Horizontal scroll wheel (2025):** SwiftUI `ScrollView(.horizontal)` does not respond to the vertical mouse wheel by default (requires Shift). To enable direct horizontal scrolling, use `NSViewRepresentable` with a custom `NSView` subclass overriding `scrollWheel(with:)` to remap vertical delta to horizontal. Alternatively, use `NSEvent.addLocalMonitorForEvents(matching: .scrollWheel)` with `CGEvent` axis remapping. The `onScrollWheel` modifier only handles events within its own view, not redirection between axes.
**`onScrollPhaseChange` (macOS 14+):** Use this modifier to detect scroll phase transitions (`.panning`, `.decelerating`, `.idle`) — useful for showing/hiding scroll indicators or adjusting content based on scroll momentum.

```swift
struct CustomScrollView: View {
    @State private var offset: CGFloat = 0

    var body: some View {
        VStack {
            ForEach(0..<50) { i in
                Text("Row \(i)")
                    .frame(height: 30)
            }
        }
        .offset(y: offset)
        .onScrollWheel { event in
            // event.scrollingDeltaX — horizontal scroll pixels
            // event.scrollingDeltaY — vertical scroll pixels (natural)
            // event.phase — .began, .changed, .ended, .cancelled
            // event.momentumPhase — post-gesture inertial phase
            // event.hasPreciseScrollingDeltas — true for trackpad

            offset += event.scrollingDeltaY

            // Consume the event by not returning (void)
        }
    }
}
```

**Phase handling for momentum scrolling:**

```swift
.onScrollWheel { event in
    if event.phase == .began {
        isUserScrolling = true
    } else if event.phase == .ended || event.phase == .cancelled {
        isUserScrolling = false
    }

    if event.hasPreciseScrollingDelays {
        // Trackpad — use precise deltas
        offset += event.scrollingDeltaY
    } else {
        // Mouse wheel — apply multiplier for line-based steps
        offset += event.scrollingDeltaY * 2.0
    }
}
```

### Scroll Layout Modifiers

**`.scrollTargetLayout()`** — Marks a lazy layout as snap-to-item:

```swift
LazyHStack(spacing: 16) {
    ForEach(items) { item in
        ItemCard(item: item)
            .containerRelativeFrame(.horizontal, count: 3)
    }
}
.scrollTargetLayout()
```

**`.scrollPosition(id:)`** — Programmatically control scroll position:

```swift
@State private var scrollID: Item.ID?

ScrollView(.horizontal) { /* ... */ }
.scrollPosition(id: $scrollID)

// Programmatic scroll:
Button("Go to item 5") {
    scrollID = items[5].id
}
```

## Secondary Click

Right-click (secondary click) in SwiftUI is handled via context menus. Two approaches:

**Approach 1: `.contextMenu` (standard)**

```swift
.contextMenu {
    Button("Cut") { cut() }
    Button("Copy") { copy() }
    Button("Paste") { paste() }
}
```

**Approach 2: Primary action + context menu** — The primary action fires on left-click, context menu on right-click:

```swift
.contextMenu {
    Button("Duplicate") { duplicate() }
    Button("Delete", role: .destructive) { delete() }
} primaryAction: {
    open()
}
```

**Approach 3: Manual button mask via `NSEvent`** — For detecting right-click in a gesture:

```swift
.onTapGesture { location in
    let event = NSApp.currentEvent
    if event?.type == .rightMouseUp {
        showContextMenu(at: location)
    } else {
        handlePrimaryTap(at: location)
    }
}
```

**Context menu with macOS Services (2025):** SwiftUI's `.contextMenu` does not automatically include macOS Services (the system-wide "Services" submenu in contextual menus). To include Services, you must drop to AppKit: extend `NSView` and override `menu(for:)` to provide a custom `NSMenu` that includes the Services submenu. This is relevant for text editing and document apps where users expect Services like "Look Up" or "Share".
**Table row context menus (2025):** Use `.contextMenu(forSelectionType:)` on SwiftUI `Table` and `List` to attach per-row context menus. The `primaryAction` closure runs on left-click (double-click on macOS), while the context menu fires on right-click — giving you two distinct interaction modes from a single modifier.
This approach is fragile. Prefer Approach 1 or 2 for production code.

## EventModifiers

`EventModifiers` provide access to modifier key state during gesture handling. Available on macOS for drag and other continuous gestures.

```swift
DragGesture()
    .onChanged { value in
        let modifiers = NSApp.currentEvent?.modifierFlags ?? []

        if modifiers.contains(.shift) {
            // Constrain to horizontal axis
            constrainedOffset = CGSize(width: value.translation.width, height: 0)
        } else if modifiers.contains(.option) {
            // Duplicate while dragging
            isCopying = true
        } else {
            constrainedOffset = value.translation
        }
    }
```

**Common modifier flags:**

| Modifier | Flag |
|----------|------|
| Shift | `.shift` |
| Control | `.control` |
| Option (Alt) | `.option` |
| Command | `.command` |
| Function | `.function` |
| Caps Lock | `.capsLock` |

**Checking in a gesture closure:**

```swift
.onTapGesture {
    let mods = NSApp.currentEvent?.modifierFlags ?? []
    if mods.contains(.command) {
        // Cmd+click
        addToSelection()
    } else if mods.contains(.shift) {
        // Shift+click
        extendSelection()
    } else {
        singleSelect()
    }
}
```

**Caveat:** `NSApp.currentEvent` is only available during active event processing (inside gesture/tap callbacks). It is `nil` outside event handlers.

## NSEvent Bridge

When SwiftUI's gesture system is insufficient, drop to `NSEvent` via `NSView` subclasses or `NSViewRepresentable`.

### Local Event Monitor

`NSEvent.addLocalMonitorForEvents(matching:)` intercepts events in the current process before they reach the responder chain. Use for global hotkeys, system-wide shortcuts, or intercepting events that SwiftUI doesn't expose.

```swift
final class HotkeyMonitor: ObservableObject {
    private var monitor: Any?

    func startMonitoring() {
        monitor = NSEvent.addLocalMonitorForEvents(matching: .keyDown) { event in
            if event.modifierFlags.contains(.command) &&
               event.keyCode == 49 { // Space bar
                self.handleSearchHotkey()
                return nil // Consume the event
            }
            return event // Pass through
        }
    }

    func stopMonitoring() {
        if let monitor {
            NSEvent.removeMonitor(monitor)
            self.monitor = nil
        }
    }

    private func handleSearchHotkey() {
        NotificationCenter.default.post(name: .globalSearchRequested, object: nil)
    }
}
```

### addGlobalMonitorForEvents

For events outside your process (sandboxed apps: entitlements required):

```swift
NSEvent.addGlobalMonitorForEvents(matching: .leftMouseDown) { event in
    // Handle clicks anywhere on screen
    // Cannot modify/consume events — read-only
}
```

### NSViewRepresentable with Gesture Recognizer

For `NSGestureRecognizer` integration in SwiftUI:

```swift
struct PressureSensitiveView: NSViewRepresentable {
    var onChange: (NSEvent) -> Void

**`CGEventSupervisor` (open-source, 2025):** For apps needing global HID event interception (keyboard/mouse regardless of frontmost status), [CGEventSupervisor](https://github.com/stephancasas/CGEventSupervisor) provides a Swift-friendly wrapper around `CGEvent` tap APIs. It supports event cancellation, NSEvent bridging, and works alongside SwiftUI's gesture system via `NSViewRepresentable`. Useful for global keyboard shortcuts that must not conflict with the system.
**`SwipeModifier` (open-source, 2025):** The [SwipeModifier](https://github.com/lloydsargent/SwipeModifier) package provides a SwiftUI `.onSwipe` modifier for detecting directional swipe/scroll-wheel events on macOS — simpler than `NSViewRepresentable` wrapping for basic scroll direction detection.
**Pressure-sensitive trackpad (2025):** SwiftUI's `DragGesture` does not expose pressure data. To access per-point pressure from the trackpad, use a custom `NSView` subclass overriding `pressureChange(with:)` and `mouseDragged(with:)`. The pure SwiftUI `MagnificationGesture` handles pressure implicitly but does not expose it programmatically.
**Trackpad finger mapping (2025):** To map individual finger positions from the Mac trackpad into SwiftUI, use `NSEvent.addLocalMonitorForEvents(matching: .gesture)` with `touches(matching:in:)` — passing `nil` for the view gives global touch data. For view-scoped touches, wrap an `NSView` that overrides `touchesBegan`/`touchesMoved` and exposes positions via `@Binding`. This is critical for drawing/music apps requiring per-finger tracking.
    func makeNSView(context: Context) -> NSView {
        let view = NSView()
        // Custom gesture recognizer or event handling
        return view
    }

    func updateNSView(_ nsView: NSView, context: Context) {}
}
```

## Keyboard Events

### onKeyPress

`onKeyPress` is available macOS 15+ / iOS 17+. It provides typed character and modifier information in a SwiftUI-native way.

```swift
struct SearchField: View {
    @State private var text = ""
    @FocusState private var isFocused: Bool

    var body: some View {
        TextField("Search", text: $text)
            .focused($isFocused)
            .onKeyPress(.return) {
                performSearch()
                return .handled  // Consume the event
            }
            .onKeyPress(.escape) {
                isFocused = false
                text = ""
                return .handled
            }
            .onKeyPress(.delete, action: {
                // Custom delete handling
                return .ignored  // Let default behavior also fire
            })
    }
}
```

**Key equivalent options:**

```swift
.onKeyPress(.upArrow) { moveSelection(up: true); return .handled }
.onKeyPress(.downArrow) { moveSelection(up: false); return .handled }
.onKeyPress(.leftArrow) { /* ... */ }
.onKeyPress(.rightArrow) { /* ... */ }
.onKeyPress(.home) { scrollTo(.start); return .handled }
.onKeyPress(.end) { scrollTo(.end); return .handled }
.onKeyPress(.pageUp) { /* ... */ }
.onKeyPress(.pageDown) { /* ... */ }
.onKeyPress(.tab) { moveFocus(); return .handled }
```

**Characters with modifiers:**

```swift
.onKeyPress(characters: "abcdefghijklmnopqrstuvwxyz") { press in
    // Only fires for lowercase a-z without modifiers
    insertCharacter(press.characters)
    return .handled
}
```

**Important:** `onKeyPress` only fires when the view (or one of its children) is focused. It does not intercept system-level shortcuts handled by the menu bar.

### keyboardShortcut

For menu commands and view-level shortcuts, `keyboardShortcut` is the idiomatic approach:

```swift
// View-level shortcut (local to the view's window)
Button("Save") { save() }
    .keyboardShortcut("s", modifiers: .command)

// Menu-level shortcut (global, via .commands)
.commands {
    CommandMenu("File") {
        Button("Export…") { export() }
            .keyboardShortcut("e", modifiers: [.command, .shift])
    }
}
```

**Mac-specific shortcut keys:**

```swift
.keyboardShortcut(.delete, modifiers: [])         // Forward delete
.keyboardShortcut(.delete, modifiers: .command)   // Cmd+Delete
.keyboardShortcut(.return, modifiers: .command)    // Cmd+Return
.keyboardShortcut(.tab, modifiers: .control)       // Ctrl+Tab
```

## Constraints & Gotchas

1. **Cursor changes via `.cursor(HoverEffect)` only work reliably inside `NSViewRepresentable`.** Pure SwiftUI `.cursor(.crosshair)` is available macOS 11+ but inconsistent; the system may override it. For guaranteed cursor control, use `NSCursor.set()` inside `NSViewRepresentable` mouse event handlers.

2. **Gestures conflict with AppKit responder chain.** When an `NSViewRepresentable` sits inside an `NSScrollView`, the scroll view may consume drag events before SwiftUI gestures fire. Solution: set `scrollView.usesPredominantAxisOnly = false` or use `wantsLayer = true` on the hosting view.

3. **`onScrollWheel` requires macOS 14.4+.** For earlier versions, use `NSEvent.addLocalMonitorForEvents(matching: .scrollWheel)` from an `NSViewRepresentable`, or implement `scrollWheel(with:)` in an `NSView` subclass.

4. **`MagnifyGesture` and `RotateGesture` only work on trackpad.** They do not fire from the scroll wheel. If you need zoom from the wheel, use `onScrollWheel` with a `.command` modifier check.

5. **`@GestureState` resets automatically** when the gesture ends or is cancelled. Do not use it for persistent state — use a separate `@State` variable that you update in `.onEnded`.

6. **SwiftUI gestures vs. AppKit gesture recognizers.** You cannot combine SwiftUI `.gesture()` with `NSGestureRecognizer` on the same view. Use `NSViewRepresentable` to set up `NSGestureRecognizer` subclasses for AppKit-only gesture needs (e.g., pressure-sensitive drawing, force-touch).

7. **`onContinuousHover` is macOS-only.** No iOS equivalent. Keep it behind `#if os(macOS)` for multiplatform targets.

8. **`NSApp.currentEvent` is nil outside event-handling closures.** Do not capture it in a gesture closure and use it asynchronously.

## Common Mistakes

- **Forgetting `minimumDistance` on DragGesture.** Default (10 points) means the user must move 10px before the drag activates, making the view feel unresponsive to taps. Set to 2–4 for tap-and-drag scenarios.

- **Using `@State` for transient gesture tracking.** Use `@GestureState` instead — it auto-resets on gesture end/cancel. `@State` leaves stale values when the gesture is interrupted.

- **Checking `modifierFlags` in `.onEnded` instead of `.onChanged`.** Modifier key state changes *during* a gesture. If you only check at the end, you miss mid-gesture modifier changes that should alter behavior (e.g., constrain axis on Shift-drag).

- **Relying on `TapGesture` for buttons.** Use `Button` instead — it handles accessibility, keyboard activation, and hover states natively. Reserve `onTapGesture` for non-interactive views (images, canvases, containers).

- **Mixing `.gesture()` and `.simultaneousGesture()` incorrectly.** `.gesture(_:including:)` is more predictable than chaining `.gesture()` + `.simultaneousGesture()`. Explicitly include the gesture mask to control which concurrent gestures are allowed.

- **Using `onScrollWheel` instead of `ScrollView`.** Prefer SwiftUI's native `ScrollView` for standard scrolling. Only use `onScrollWheel` when you need custom scroll physics, non-linear scroll mapping, or have a `Canvas`-based layout.

- **Not calling `NSEvent.removeMonitor()` when done.** Leaked monitors accumulate and fire on every matching event. Always store the monitor token and clean up in `deinit` or `dismantleNSView`.

## Review Checklist

- [ ] `DragGesture` has an appropriate `minimumDistance` for the interaction type
- [ ] `@GestureState` is used for transient gesture values; `@State` for persistent results
- [ ] `MagnifyGesture`/`RotateGesture` track `baseValue * gestureValue` for correct accumulation
- [ ] Scroll wheel handling accounts for precise (trackpad) vs. line-based (mouse) deltas
- [ ] `NSEvent.addLocalMonitorForEvents` is paired with `removeMonitor` on cleanup
- [ ] `NSApp.currentEvent` is only accessed inside event-handling closures, never stored
- [ ] Cursor changes use `NSViewRepresentable` with `NSCursor.set()` for reliability
- [ ] `onKeyPress` checks `.handled`/`.ignored` return values appropriately
- [ ] Modifier key checks in drag gestures cover `.onChanged`, not just `.onEnded`
- [ ] Context menus use `primaryAction` when left-click and right-click should do different things
- [ ] `#if os(macOS)` guards on macOS-only APIs (`onContinuousHover`, `onScrollWheel`)
- [ ] No gesture recognizer conflicts between SwiftUI `.gesture()` and AppKit `NSGestureRecognizer`

## References

- [SwiftUI Gestures Documentation](https://developer.apple.com/documentation/swiftui/gestures)
- [NSEvent Documentation](https://developer.apple.com/documentation/appkit/nsevent)
- [NSCursor Documentation](https://developer.apple.com/documentation/appkit/nscursor)
- [View.onTapGesture](https://developer.apple.com/documentation/swiftui/view/ontapgesture(count:perform:))
- [View.gesture](https://developer.apple.com/documentation/swiftui/view/gesture(_:including:))
- [MagnifyGesture](https://developer.apple.com/documentation/swiftui/magnifygesture)
- [RotateGesture](https://developer.apple.com/documentation/swiftui/rotategesture)
- [View.onScrollWheel](https://developer.apple.com/documentation/swiftui/view/onscrollwheel(_:))
- [View.onKeyPress](https://developer.apple.com/documentation/swiftui/view/onkeypress(_:action:))
- [View.onContinuousHover](https://developer.apple.com/documentation/swiftui/view/oncontinuoushover(perform:))
- [NSEvent.addLocalMonitorForEvents](https://developer.apple.com/documentation/appkit/nsevent/1534971-addlocalmonitorforevents)
- [EventModifiers](https://developer.apple.com/documentation/swiftui/eventmodifiers)
