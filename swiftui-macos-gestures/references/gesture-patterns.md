# Gesture Patterns — Complete Recipes

Reference patterns for macOS gesture handling in SwiftUI. Each section is a self-contained, working recipe with code, explanation, and gotchas.

---

## Contents

- [1. Custom Hover with Cursor Change via NSViewRepresentable](#1-custom-hover-with-cursor-change-via-nsviewrepresentable)
- [2. DragGesture with Coordinate Space Conversion](#2-draggesture-with-coordinate-space-conversion)
- [3. onScrollWheel for Custom Scroll Behavior](#3-onscrollwheel-for-custom-scroll-behavior)
- [4. NSEvent Local Monitor for Global Hotkey Detection](#4-nsevent-local-monitor-for-global-hotkey-detection)
- [5. Right-Click Context Menu with Primary Action](#5-right-click-context-menu-with-primary-action)

---

## 1. Custom Hover with Cursor Change via NSViewRepresentable

Pure SwiftUI `.cursor()` is unreliable outside `NSViewRepresentable`. For guaranteed cursor control, wrap an `NSView` subclass.

```swift
import SwiftUI
import AppKit

struct CursorControllingView: NSViewRepresentable {
    var cursorType: CursorType
    var onHoverChange: ((Bool) -> Void)? = nil

    enum CursorType {
        case crosshair, arrow, pointingHand, iBeam, openHand, closedHand
        case resizeLeftRight, resizeUpDown

        var nsCursor: NSCursor {
            switch self {
            case .crosshair:      return .crosshair
            case .arrow:          return .arrow
            case .pointingHand:   return .pointingHand
            case .iBeam:          return .IBeam
            case .openHand:       return .openHand
            case .closedHand:     return .closedHand
            case .resizeLeftRight: return .resizeLeftRight
            case .resizeUpDown:    return .resizeUpDown
            }
        }
    }

    func makeNSView(context: Context) -> CursorHostingNSView {
        let view = CursorHostingNSView()
        view.cursorType = cursorType
        view.hoverCallback = onHoverChange
        return view
    }

    func updateNSView(_ nsView: CursorHostingNSView, context: Context) {
        nsView.cursorType = cursorType
        nsView.hoverCallback = onHoverChange
    }
}

final class CursorHostingNSView: NSView {
    var cursorType: CursorControllingView.CursorType = .arrow
    var hoverCallback: ((Bool) -> Void)?
    var trackingArea: NSTrackingArea?

    override func updateTrackingAreas() {
        super.updateTrackingAreas()
        if let existing = trackingArea {
            removeTrackingArea(existing)
        }
        let area = NSTrackingArea(
            rect: bounds,
            options: [.mouseEnteredAndExited, .activeAlways, .inVisibleRect],
            owner: self,
            userInfo: nil
        )
        addTrackingArea(area)
        trackingArea = area
    }

    override func mouseEntered(with event: NSEvent) {
        cursorType.nsCursor.set()
        hoverCallback?(true)
    }

    override func mouseExited(with event: NSEvent) {
        NSCursor.arrow.set()
        hoverCallback?(false)
    }

    override func resetCursorRects() {
        super.resetCursorRects()
        addCursorRect(bounds, cursor: cursorType.nsCursor)
    }
}
```

### Usage in SwiftUI

```swift
struct CanvasView: View {
    @State private var isHovering = false
    @State private var activeTool: Tool = .select

    var body: some View {
        ZStack {
            // The cursor-controlling background
            CursorControllingView(
                cursorType: activeTool.cursorType,
                onHoverChange: { hovering in isHovering = hovering }
            )
            .background(Color.black.opacity(0.05))

            // Drawing content overlaid
            DrawingContent()
        }
    }
}

enum Tool {
    case select, draw, erase, text

    var cursorType: CursorControllingView.CursorType {
        switch self {
        case .select: return .arrow
        case .draw:   return .crosshair
        case .erase:  return .crosshair
        case .text:   return .iBeam
        }
    }
}
```

### Gotchas

- **`resetCursorRects` is the reliable cursor mechanism.** The cursor rect system automatically restores the arrow when the pointer leaves, without explicit `mouseExited` handling. Both `mouseEntered`/`mouseExited` AND `resetCursorRects` together provide the most robust behavior.
- **Tracking areas must be recreated** when the view frame changes. `updateTrackingAreas()` handles this — do not create tracking areas in `init`.
- **Do not forget `options: .inVisibleRect`** in the tracking area — without it, events may not fire when the view is offscreen and scrolls back.

---

## 2. DragGesture with Coordinate Space Conversion

When dragging across different coordinate spaces (e.g., an item inside a `Canvas` inside a `ScrollView`), you must convert coordinates correctly.

```swift
struct DraggableItem: View {
    let item: CanvasItem
    @State private var localOffset = CGSize.zero
    @GestureState private var dragState = DragState.inactive

    var body: some View {
        Text(item.title)
            .padding(8)
            .background(Color.blue.opacity(0.3))
            .cornerRadius(6)
            .offset(
                x: localOffset.width + dragState.translation.width,
                y: localOffset.height + dragState.translation.height
            )
            .gesture(
                DragGesture(coordinateSpace: .named("canvas"))
                    .updating($dragState) { value, state, _ in
                        state = .dragging(translation: value.translation)
                    }
                    .onChanged { value in
                        // Apply constraint: hold Shift to lock to one axis
                        let mods = NSApp.currentEvent?.modifierFlags ?? []
                        if mods.contains(.shift) {
                            let absX = abs(value.translation.width)
                            let absY = abs(value.translation.height)
                            if absX > absY {
                                localOffset = CGSize(
                                    width: value.translation.width,
                                    height: localOffset.height
                                )
                            } else {
                                localOffset = CGSize(
                                    width: localOffset.width,
                                    height: value.translation.height
                                )
                            }
                        }
                    }
                    .onEnded { value in
                        // Commit the drag
                        localOffset.width += value.translation.width
                        localOffset.height += value.translation.height

                        // Snap to grid (every 20pt)
                        localOffset = CGSize(
                            width: round(localOffset.width / 20) * 20,
                            height: round(localOffset.height / 20) * 20
                        )
                    }
            )
    }
}

enum DragState {
    case inactive
    case dragging(translation: CGSize)

    var translation: CGSize {
        switch self {
        case .inactive: return .zero
        case .dragging(let t): return t
        }
    }
}
```

### Named Coordinate Space Setup

```swift
struct CanvasWorkspace: View {
    @State private var items: [CanvasItem] = []

    var body: Some View {
        Canvas { context, size in
            // Draw background grid
        } context: { graphics, size in
            for item in items {
                DraggableItem(item: item)
                    .position(item.position)
            }
        }
        .coordinateSpace(.named("canvas"))
    }
}
```

### Gotchas

- **`.global` coordinate space gives window-relative positions**, not screen-relative. For multi-window apps, convert via `window?.convertPoint(...)`.
- **Named coordinate spaces are scoped** to the view that declares `.coordinateSpace(.named("x"))`. The gesture must be attached to a descendant of that view.
- **`value.startLocation` includes accumulated offset.** Use `value.translation` for delta calculations, not `value.location - value.startLocation` (they can differ when coordinate spaces are transformed).

---

## 3. onScrollWheel for Custom Scroll Behavior

Implement non-linear scroll mapping, velocity-based zoom, or custom axis handling.

```swift
struct ZoomableCanvas: View {
    @State private var scale: CGFloat = 1.0
    @State private var offset = CGSize.zero
    @GestureState private var scrollState = CGSize.zero

    var body: some View {
        Canvas { g, size in
            g.by(scale: scale, around: CGPoint(x: size.width/2, y: size.height/2))
            g.translateBy(x: offset.width + scrollState.width, y: offset.height + scrollState.height)
            // draw content
        }
        .onScrollWheel { event in
            let mods = event.modifierFlags

            if mods.contains(.command) {
                // Cmd+scroll: zoom
                let zoomDelta = 1.0 + (event.scrollingDeltaY / 100.0)
                let newScale = (scale * zoomDelta).clamped(to: 0.1...10.0)

                // Zoom toward pointer location
                let pointerLocation = CGPoint(
                    x: event.locationInWindow.x,
                    y: event.locationInWindow.y
                )
                let scaleRatio = newScale / scale
                offset = CGSize(
                    width: pointerLocation.x - (pointerLocation.x - offset.width) * scaleRatio,
                    height: pointerLocation.y - (pointerLocation.y - offset.height) * scaleRatio
                )
                scale = newScale
            } else if mods.contains(.shift) {
                // Shift+scroll: horizontal scroll
                offset.width += event.scrollingDeltaY * 2
            } else {
                // Normal scroll: vertical
                offset.height += event.scrollingDeltaY * 2
                offset.width += event.scrollingDeltaX * 2
            }

            // Handle momentum phase for smooth deceleration
            if event.momentumPhase == .began {
                isMomentumScrolling = true
            } else if event.momentumPhase == .ended || event.momentumPhase == .cancelled {
                isMomentumScrolling = false
            }
        }
    }
}
```

### Disable Default Scroll, Apply Custom Only

```swift
ScrollView([]) {
    content
}
.onScrollWheel { event in
    // Consume all scroll events
    myCustomScrollHandler(event)
    // No return = event is consumed
}
.simultaneousGesture(
    DragGesture() // still allow other gestures
)
```

### Gotchas

- **`onScrollWheel` is macOS 14.4+.** Wrap in `#available(macOS 14.4, *)` for compatibility.
- **`scrollingDeltaY` is positive for scrolling content up / natural scrolling** (content moves in the direction of finger movement). This is the opposite of what you might expect for "scroll position" adjustments.
- **`momentumPhase` fires after the gesture phase.** A single scroll interaction produces: `.began → .changed* → .ended` followed by `.momentumPhase(.began) → .momentumPhase(.changed*) → .momentumPhase(.ended)`.
- **Trackpad provides `hasPreciseScrollingDeltas = true`.** Mouse wheels provide line-based deltas (~10-40px per tick). Trackpad deltas are sub-pixel smooth. Check this flag to scale appropriately.

---

## 4. NSEvent Local Monitor for Global Hotkey Detection

Intercept key events app-wide, even when FocusState doesn't cover the active element.

```swift
import SwiftUI
import AppKit
import Combine

@MainActor
final class GlobalHotkeyController: ObservableObject {
    @Published var searchVisible = false
    @Published var lastKeyCombo: String = ""

    private var localMonitor: Any?
    private var flagsMonitor: Any?

    func install() {
        // Monitor keyDown events
        localMonitor = NSEvent.addLocalMonitorForEvents(matching: .keyDown) { [weak self] event in
            guard let self else { return event }
            return self.handleKeyDown(event)
        }

        // Monitor modifier-only changes (e.g., double-tap Option)
        flagsMonitor = NSEvent.addLocalMonitorForEvents(matching: .flagsChanged) { [weak self] event in
            guard let self else { return event }
            return self.handleFlagsChanged(event)
        }
    }

    func uninstall() {
        if let localMonitor {
            NSEvent.removeMonitor(localMonitor)
            self.localMonitor = nil
        }
        if let flagsMonitor {
            NSEvent.removeMonitor(flagsMonitor)
            self.flagsMonitor = nil
        }
    }

    private func handleKeyDown(_ event: NSEvent) -> NSEvent? {
        let mods = event.modifierFlags.intersection(.deviceIndependentFlagsMask)

        // Cmd+K: Toggle search
        if mods == .command && event.keyCode == 40 { // 40 = K
            searchVisible.toggle()
            lastKeyCombo = "⌘K"
            return nil // Consume
        }

        // Cmd+Shift+/: Show help
        if mods == [.command, .shift] && event.keyCode == 44 { // 44 = /
            showHelp()
            lastKeyCombo = "⌘⇧/"
            return nil
        }

        // Escape: dismiss if search is visible
        if event.keyCode == 53 && searchVisible {
            searchVisible = false
            lastKeyCombo = "Esc"
            return nil
        }

        return event // Pass through
    }

    private func handleFlagsChanged(_ event: NSEvent?) -> NSEvent? {
        // Detect double-tap of Option key for quick-access features
        let mods = event.modifierFlags.intersection(.deviceIndependentFlagsMask)
        if mods == .option {
            handleOptionKeyDoubleTap()
        }
        return event
    }

    private var lastOptionTap: Date?
    private func handleOptionKeyDoubleTap() {
        let now = Date()
        if let last = lastOptionTap, now.timeIntervalSince(last) < 0.3 {
            // Double-tap detected
            quickAction()
            lastOptionTap = nil
        } else {
            lastOptionTap = now
        }
    }

    deinit {
        if let localMonitor { NSEvent.removeMonitor(localMonitor) }
        if let flagsMonitor { NSEvent.removeMonitor(flagsMonitor) }
    }
}
```

### SwiftUI Integration

```swift
struct RootView: View {
    @StateObject private var hotkeys = GlobalHotkeyController()

    var body: some View {
        NavigationSplitView {
            Sidebar()
        } detail: {
            DetailView()
        }
        .onAppear { hotkeys.install() }
        .onDisappear { hotkeys.uninstall() }
        .sheet(isPresented: $hotkeys.searchVisible) {
            QuickSearchView()
        }
    }
}
```

### Common Key Codes Reference

| Key | Code |
|-----|------|
| A-Z | 0-25 |
| 0-9 | 29-39 (non-linear) |
| Return | 36 |
| Tab | 48 |
| Space | 49 |
| Delete | 51 |
| Escape | 53 |
| Left Arrow | 123 |
| Right Arrow | 124 |
| Down Arrow | 125 |
| Up Arrow | 126 |
| F1-F12 | 122-111 (descending) |

*(Full key code reference: [NSEvent macOS virtual key codes](https://developer.apple.com/documentation/appkit/nsevent/virtual_key_codes))*

### Gotchas

- **Returning `nil` from the monitor consumes the event.** Returning the event passes it through. Only consume events you intentionally handle.
- **Do not block events your app needs to function.** Avoid consuming single-letter keys without modifiers — text fields, search bars, and other inputs need them.
- **`addLocalMonitorForEvents` only works for the current process.** For system-wide hotkeys outside your app, use `RegisterEventHotKey` (Carbon) or the Media Player framework, which requires additional entitlements.
- **Always remove monitors in `deinit`.** Monitors survive view disappearance if not cleaned up. Use `.onDisappear` as additional safety.

---

## 5. Right-Click Context Menu with Primary Action

The `primaryAction` parameter on `.contextMenu` enables left-click / right-click duality. This is distinct from `.contextMenu(menuItems:)` alone, which only provides right-click behavior.

```swift
struct DraggableItemCard: View {
    let item: ProjectItem
    let onOpen: () -> Void
    let onDuplicate: () -> Void
    let onMoveToTrash: () -> Void
    @State private var dragOffset = CGSize.zero
    @State private var isDragging = false

    var body: some View {
        VStack(alignment: .leading, spacing: 4) {
            Image(systemName: item.icon)
                .font(.title2)
                .foregroundStyle(item.tintColor)

            Text(item.name)
                .font(.headline)
                .lineLimit(2)

            Text(item.detail)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .frame(width: 140, height: 140)
        .background {
            RoundedRectangle(cornerRadius: 10)
                .fill(isDragging ? Color.accentColor.opacity(0.15) : Color(nsColor: .controlBackgroundColor))
                .stroke(Color.secondary.opacity(0.2), lineWidth: 1)
        }
        .scaleEffect(isDragging ? 1.05 : 1.0)
        .animation(.easeInOut(duration: 0.15), value: isDragging)
        .offset(dragOffset)
        // Primary action = fires on left-click
        .contextMenu {
            Button("Open ") { onOpen() }
            Button("Duplicate") { onDuplicate() }
            Button("Get Info") { showInfo() }
            Divider()
            Menu("Move To") {
                Button("Inbox") { move(to: .inbox) }
                Button("Archive") { move(to: .archive) }
            }
            Divider()
            Button("Move to Trash", role: .destructive) { onMoveToTrash() }
        } primaryAction: {
            onOpen()
        }
        .gesture(
            DragGesture()
                .onChanged { value in
                    isDragging = true
                    dragOffset = value.translation
                }
                .onEnded { _ in
                    isDragging = false
                    dragOffset = .zero
                }
        )
    }
}
```

### Secondary Click Without ContextMenu (Event-based)

For rare cases where you need to detect right-click directly:

```swift
struct RawSecondaryClickView: View {
    var body: some View {
        Rectangle()
            .fill(Color.gray.opacity(0.1))
            .frame(width: 200, height: 200)
            .onTapGesture { location in
                guard let event = NSApp.currentEvent else { return }
                switch event.type {
                case .leftMouseUp:
                    handlePrimaryTap(at: location)
                case .rightMouseUp:
                    handleSecondaryTap(at: location)
                default:
                    break
                }
            }
    }
}
```

### Gotchas

- **`primaryAction` changes tap behavior.** Without it, tapping a view with only `.contextMenu` does nothing and shows the context menu on right-click. With `primaryAction`, left-click fires the primary action, right-click shows the menu.
- **Don't combine `primaryAction` with `onTapGesture`** — they conflict. The primary action replaces the default left-click behavior.
- **Context menus are not keyboard-accessible by default.** Add `.keyboardShortcut` to menu items that should have shortcuts (though `.contextMenu` shortcuts don't show in the menu on macOS — they only fire via the keyboard).
- **On iOS, `primaryAction` replaces the tap gesture entirely.** On macOS, it adds a left-click action alongside right-click — be aware of cross-platform differences.
