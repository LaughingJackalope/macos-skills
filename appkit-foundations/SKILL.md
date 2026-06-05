---
name: appkit-foundations
description: "Core AppKit patterns for macOS apps: NSWindow/NSWindowController lifecycle, NSView/NSViewController hierarchy, responder chain event flow, NSDocument architecture, custom view drawing with layer-backing, drag & drop, and printing. Use when building or refactoring any AppKit-based macOS app, managing windows and view controllers, implementing custom drawing, or wiring the responder chain."
sources: https://developer.apple.com/videos/play/wwdc2024/10124/, https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/WinPanel/Concepts/UsingWindowController.html, https://apple-docs.everest.mt/docs/appkit/nsviewcontroller/, https://sosumi.ai/documentation/AppKit/NSWindow, https://developer.apple.com/documentation/appkit/nsdocument
researched: 2026-06-05
platform: macOS 26+
---

# AppKit Foundations

Core AppKit building blocks for macOS 26+ with Swift 6.3. Covers window management, view hierarchy, responder chain, document architecture, custom drawing, drag & drop, and printing. This skill focuses on the AppKit-specific patterns that have no SwiftUI equivalent. For SwiftUI interop (NSViewRepresentable, NSHostingView), see the `swiftui-appkit-interop` skill. For window scene management and tabbing, see `macos-windows-scenes`.

See [references/appkit-view-hierarchy.md](references/appkit-view-hierarchy.md) for detailed NSView/NSViewController lifecycle and auto layout patterns, [references/responder-chain.md](references/responder-chain.md) for first responder mechanics and event propagation, and [references/nsdocument-patterns.md](references/nsdocument-patterns.md) for NSDocument subclassing and autosave.

## Contents

- [Window Management](#window-management)
- [View Hierarchy](#view-hierarchy)
- [Responder Chain](#responder-chain)
- [NSDocument Architecture](#nsdocument-architecture)
- [Custom View Drawing](#custom-view-drawing)
- [Layer-Backing](#layer-backing)
- [Drag & Drop](#drag--drop)
- [Printing](#printing)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Window Management

Every AppKit window is backed by an `NSWindow` instance. For anything beyond a single-window utility, use `NSWindowController` to own the window lifecycle. Window controllers load from XIBs or storyboards, or programmatically from a `NSWindow` initializer.

**Prefer `NSWindowController` over bare `NSWindow`.** It provides lifecycle hooks (`windowDidLoad`, `windowWillClose`), first responder management, and document association. A window controller should own exactly one window and the root view controller for that window's content.

```swift
final class DocumentWindowController: NSWindowController {
    override func windowDidLoad() {
        super.windowDidLoad()
        guard let window else { return }

        window.titleVisibility = .visible
        window.styleMask.insert(.fullSizeContentView)
        window.titlebarAppearsTransparent = true

        let sidebarController = SidebarViewController()
        let contentController = ContentViewController()

        let splitController = NSSplitViewController()
        splitController.addSplitViewItem(NSSplitViewItem(sidebarWithViewController: sidebarController))
        splitController.addSplitViewItem(NSSplitViewItem(viewController: contentController))

        contentViewController = splitController
    }
}
```

**Multiple windows:** Each window gets its own controller. For document-based apps, instantiate a new `NSWindowController` subclass per document in `makeWindowControllers()`. For non-document apps, use a factory or coordinator to create window controllers on demand.

```swift
final class AppCoordinator {
    var inspectorWindowController: NSWindowController?

    func showInspector() {
        guard inspectorWindowController == nil else {
            inspectorWindowController?.window?.makeKeyAndOrderFront(nil)
            return
        }
        let wc = InspectorWindowController()
        wc.showWindow(nil)
        inspectorWindowController = wc
    }
}
```

**Window lifecycle order:** `init` → `loadWindow()` (if not from nib/storyboard) → `windowDidLoad()` → `windowWillClose:` → `windowShouldClose:` → `deinit`. Always do setup in `windowDidLoad()`, not `init` or `loadWindow()`.

## View Hierarchy

`NSView` is the fundamental drawing and event-handling unit. `NSViewController` manages a view and its subtree. The key distinction: views draw and handle events; controllers coordinate, load data, and respond to lifecycle events.

**View controller lifecycle order:** `init` → `loadView()` → `viewDidLoad()` → `viewWillAppear()` → `viewDidAppear()` → `viewWillDisappear()` → `viewDidDisappear()` → `deinit`. Setup that depends on the view hierarchy being fully loaded belongs in `viewDidLoad()`. Layout adjustments that depend on the view's final frame belong in `viewWillAppear()` or later.

```swift
final class ContentViewController: NSViewController {
    private var tableView: NSTableView!

    override func loadView() {
        // Create the root view. Do NOT call super.loadView() if you assign self.view directly.
        view = NSView(frame: NSRect(x: 0, y: 0, width: 600, height: 400))
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        setupTableView()
        setupConstraints()
    }

    private func setupTableView() {
        tableView = NSTableView()
        let column = NSTableColumn(identifier: NSUserInterfaceItemIdentifier("name"))
        column.title = "Name"
        tableView.addTableColumn(column)

        let scrollView = NSScrollView()
        scrollView.documentView = tableView
        scrollView.hasVerticalScroller = true
        scrollView.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(scrollView)

        NSLayoutConstraint.activate([
            scrollView.topAnchor.constraint(equalTo: view.topAnchor),
            scrollView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            scrollView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            scrollView.bottomAnchor.constraint(equalTo: view.bottomAnchor),
        ])
    }
}
```

**Adding child view controllers:** Use `addChild(_:)` + `addSubview(_:)` for containment. Never just `addSubview` a child controller's view without also calling `addChild` — the responder chain breaks.

```swift
func embed(_ child: NSViewController, in container: NSView) {
    addChild(child)
    child.view.translatesAutoresizingMaskIntoConstraints = false
    container.addSubview(child.view)
    NSLayoutConstraint.activate([
        child.view.topAnchor.constraint(equalTo: container.topAnchor),
        child.view.leadingAnchor.constraint(equalTo: container.leadingAnchor),
        child.view.trailingAnchor.constraint(equalTo: container.trailingAnchor),
        child.view.bottomAnchor.constraint(equalTo: container.bottomAnchor),
    ])
    child.didMove(toParent: self)
}
```

## Responder Chain

The responder chain is AppKit's event-routing mechanism. Events that the first responder doesn't handle propagate up through the chain: `NSResponder` → superview → `NSWindow` → `NSWindowController` → `NSDocument` → `NSApplication` → `NSApp.delegate`.

**First responder** is the currently focused control (text field, table view, etc.). `NSWindow.firstResponder` returns it. The key view loop determines tab order — set `nextKeyView` on each view to define the cycle, or use Auto Layout-based key view loops on macOS 14+.

```swift
// Programmatic first responder management
window.makeFirstResponder(textField)

// Key view loop (explicit)
textField.nextKeyView = secureField
secureField.nextKeyView = button
button.nextKeyView = textField  // cycle back
```

**`sendAction(_:to:from:)`** sends a selector up the responder chain. This is how menu items and toolbar buttons work — the action is sent from the menu item through the responder chain until a responder handles it.

```swift
// In a menu item or toolbar button:
NSApp.sendAction(#selector(MyResponder.delete(_:)), to: nil, from: nil)

// In a responder that handles the action:
@objc func delete(_ sender: Any?) {
    // handle delete
}
```

**Validating menu items:** Implement `validateUserInterfaceItem(_:)` to enable/disable menu items dynamically. The responder chain walks until it finds a responder that returns `true`.

```swift
override func validateUserInterfaceItem(_ item: NSValidatedUserInterfaceItem) -> Bool {
    switch item.action {
    case #selector(delete(_:)):
        return selectedItems.isEmpty == false
    default:
        return super.validateUserInterfaceItem(item)
    }
}
```

## NSDocument Architecture

Use `NSDocument` for file-based apps. It provides document lifecycle, autosave, window management, and undo/redo integration. Don't use it for non-file data (preferences, transient state) — use a plain controller instead.

**Subclassing NSDocument:** Override `makeWindowControllers()` to create your window controller(s), `data(ofType:)` for saving, and `read(from:ofType:)` for loading.

```swift
final class TextDocument: NSDocument {
    var content: String = ""

    override func makeWindowControllers() {
        let wc = NSWindowController(window: NSWindow(contentRect: NSRect(x: 0, y: 0, width: 600, height: 400),
                                                       styleMask: [.titled, .closable, .resizable, .miniaturizable],
                                                       backing: .buffered, defer: false))
        wc.contentViewController = DocumentViewController(document: self)
        addWindowController(wc)
    }

    override func data(ofType typeName: String) throws -> Data {
        guard let data = content.data(using: .utf8) else {
            throw NSError(domain: NSOSStatusErrorDomain, code: unimpErr)
        }
        return data
    }

    override func read(from data: Data, ofType typeName: String) throws {
        guard let string = String(data: data, encoding: .utf8) else {
            throw NSError(domain: NSOSStatusErrorDomain, code: unimpErr)
        }
        content = string
    }
}
```

**Autosave:** Set `isDraft = false` (default) and implement `autosavesInPlace = true` (default on macOS 10.7+). The system handles periodic autosave. For custom autosave names, override `autosaveDocument(withDelegate:didAutosave:contextInfo:)`.

**Window controllers per document:** Each document owns its window controllers via `addWindowController()`. Access them through `windowControllers`. For multi-window documents (e.g., a main window + an inspector), create additional controllers in `makeWindowControllers()`.

## Custom View Drawing

Override `draw(_:)` for custom rendering. Always check `dirtyRect` and only draw what's needed for performance.

```swift
final class ProgressView: NSView {
    var progress: Double = 0.0 { didSet { needsDisplay = true } }

    override var isFlipped: Bool { true }  // top-left origin, like UIKit

    override func draw(_ dirtyRect: NSRect) {
        super.draw(dirtyRect)

        // Background
        NSColor.controlBackgroundColor.setFill()
        dirtyRect.fill()

        // Progress bar
        let barRect = NSRect(x: 0, y: 0, width: bounds.width * progress, height: bounds.height)
        NSColor.systemBlue.setFill()
        barRect.fill()
    }

    override var isOpaque: Bool { true }  // optimization: no alpha compositing needed
}
```

**`isFlipped`:** AppKit's default coordinate system has the origin at the bottom-left. Override `isFlipped` to return `true` for top-left origin (UIKit-style). Do this in your custom view subclass, not via a category.

**`isOpaque`:** Return `true` from `isOpaque` when your view fills its entire bounds with opaque content. This tells AppKit to skip compositing views behind it. If your view has any transparency, return `false`.

**Triggering redraw:** Set `needsDisplay = true` to mark the entire view dirty, or `setNeedsDisplay(_:)` for a sub-rect. Never call `draw(_:)` directly.

## Layer-Backing

AppKit views can use Core Animation layers for animation and compositing. Set `wantsLayer = true` to enable layer-backing. On macOS 14+, you can set `wantsLayer` before the view is added to a window without performance penalty.

**Layer-backed vs. layer-hosting:** Layer-backed (`wantsLayer = true`) means AppKit manages the layer and calls `draw(_:)` into it. Layer-hosting (`layer = CALayer()` and `wantsLayer = true`) means you manage the layer tree yourself — `draw(_:)` is not called.

```swift
// Layer-backed: AppKit draws into the layer
final class AnimatedView: NSView {
    override init(frame: NSRect) {
        super.init(frame: frame)
        wantsLayer = true
        layer?.backgroundColor = NSColor.controlBackgroundColor.cgColor
    }

    // Use updateLayer instead of draw(_:) for layer-backed content
    override func updateLayer() {
        super.updateLayer()
        layer?.backgroundColor = NSColor.controlBackgroundColor.cgColor
        layer?.cornerRadius = 8.0
    }

    override var wantsUpdateLayer: Bool { true }  // use updateLayer, not draw(_:)
}
```

**When to use layer-backing:** Animations (position, opacity, transform), corner radius, shadows, border, masks. Don't enable it globally via `NSAppearance` — it changes the drawing model and can break custom `draw(_:)` code.

## Drag & Drop

Implement drag and drop by registering for types and conforming to `NSDraggingDestination`.

```swift
final class DropZoneView: NSView {
    var onDrop: (([URL]) -> Void)?

    override init(frame: NSRect) {
        super.init(frame: frame)
        registerForDraggedTypes([.fileURL])
    }

    required init?(coder: NSCoder) { fatalError() }

    // NSDraggingDestination
    override func draggingEntered(_ sender: NSDraggingInfo) -> NSDragOperation {
        let pasteboard = sender.draggingPasteboard
        guard pasteboard.canReadObject(forClasses: [NSURL.self], options: [.urlReadingFileURLsOnly: true]) else {
            return []
        }
        layer?.borderColor = NSColor.systemBlue.cgColor
        layer?.borderWidth = 2.0
        return .copy
    }

    override func draggingExited(_ sender: NSDraggingInfo?) {
        layer?.borderWidth = 0
    }

    override func performDragOperation(_ sender: NSDraggingInfo) -> Bool {
        layer?.borderWidth = 0
        guard let urls = sender.draggingPasteboard.readObjects(forClasses: [NSURL.self], options: [.urlReadingFileURLsOnly: true]) as? [URL] else {
            return false
        }
        onDrop?(urls)
        return true
    }
}
```

**Pasteboard types:** Use `NSPasteboard.PasteboardType.fileURL` for files, `.string` for text, `.tiff` for images. For custom types, create a unique identifier: `NSPasteboard.PasteboardType("com.myapp.custom-type")`.

## Printing

Use `NSPrintOperation` to print a view. The simplest path is `printView(_:)` which creates a standard print operation.

```swift
// Print a view with default settings
let printInfo = NSPrintInfo.shared
printInfo.topMargin = 36
printInfo.bottomMargin = 36
printInfo.leftMargin = 36
printInfo.rightMargin = 36

let operation = NSPrintOperation(view: myView, printInfo: printInfo)
operation.run()

// For custom pagination, subclass NSView and override:
// - knowsPageRange(_:) -> NSRange
// - rectForPage(_:) -> NSRect
final class PrintableReportView: NSView {
    override func knowsPageRange(_ range: NSRangePointer) -> Bool {
        range.pointee = NSRange(location: 1, length: 3)  // 3 pages
        return true
    }

    override func rectForPage(_ page: Int) -> NSRect {
        // Return the rect for each page
        return bounds  // simplified
    }
}
```

## Constraints & Gotchas

- **Thread safety:** All AppKit UI operations must happen on the main thread. Use `@MainActor` on window and view controller classes. Swift 6.3 will warn about main-thread violations.
- **View controller containment:** Always use `addChild(_:)` + `addSubview(_:)` + `didMove(toParent:)`. Missing `addChild` breaks the responder chain and lifecycle events.
- **Window controller ownership:** A window controller must be retained (by a strong property or the document system). If nothing holds a reference, the window deallocates immediately.
- **`loadView()` vs `viewDidLoad()`:** Create the root view in `loadView()`. Do all other setup in `viewDidLoad()`. Never access `self.view` in `init` — it triggers premature view loading.
- **Layer-backing side effects:** Enabling `wantsLayer` changes the drawing model. `draw(_:)` may not be called if `wantsUpdateLayer` returns `true`. Test custom drawing after enabling layers.
- **Responder chain is not the view hierarchy:** The responder chain goes through view → superview → window → window controller → document → application. It does not follow `nextResponder` in the same way as the view tree.
- **NSDocument and Swift concurrency:** `data(ofType:)` and `read(from:ofType:)` are not async. Use `NSFileCoordinator` for coordinated file access if needed.

## Common Mistakes

- **Accessing `view` in `init`:** Triggers `loadView()` before the controller is fully initialized. Move setup to `viewDidLoad()`.
- **Forgetting `translatesAutoresizingMaskIntoConstraints = false`:** Views created programmatically default to `true`, which conflicts with Auto Layout constraints.
- **Using `NSViewController` without `loadView()` override:** If you don't use a storyboard, you must override `loadView()` to assign `self.view`.
- **Not calling `super` in lifecycle methods:** `viewDidLoad()`, `viewWillAppear()`, etc. — always call `super`.
- **Enabling `wantsLayer` globally:** Setting `wantsLayer` on a window's content view propagates to all subviews and can break custom `draw(_:)` implementations. Enable it per-view.
- **Ignoring `isFlipped`:** Assuming bottom-left origin when the rest of your code expects top-left. Be explicit.
- **Strong reference cycles in `onDrop` closures:** Capture `self` weakly in drag-and-drop callbacks to avoid retain cycles with the pasteboard owner.

## Review Checklist

- [ ] Window controllers own their windows and are retained by a strong reference
- [ ] View controller containment uses `addChild`/`didMove(toParent:)` pairs
- [ ] `loadView()` creates the root view; `viewDidLoad()` does all other setup
- [ ] `translatesAutoresizingMaskIntoConstraints = false` on programmatic views
- [ ] Responder chain actions use `@objc` selectors and `validateUserInterfaceItem(_:)`
- [ ] NSDocument subclasses implement both `data(ofType:)` and `read(from:ofType:)`
- [ ] Layer-backing is enabled per-view, not globally
- [ ] Custom `draw(_:)` checks `dirtyRect` and avoids unnecessary work
- [ ] Drag & drop implements at least `draggingEntered:`, `performDragOperation:`, and `draggingExited:`
- [ ] All UI code runs on the main thread (`@MainActor` on controller classes)

## References

- [references/appkit-view-hierarchy.md](references/appkit-view-hierarchy.md) — Detailed NSView/NSViewController patterns, view lifecycle timing, addSubview vs addChild, auto layout with NSLayoutConstraint
- [references/responder-chain.md](references/responder-chain.md) — First responder mechanics, sendAction(_:to:from:), key view loop, event propagation
- [references/nsdocument-patterns.md](references/nsdocument-patterns.md) — NSDocument subclassing, autosave, window controller management, read(from:ofType:), write(to:ofType:)
