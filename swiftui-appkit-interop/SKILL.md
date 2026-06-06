---
name: swiftui-appkit-interop
description: "Bridges AppKit and SwiftUI by wrapping NS views and view controllers in SwiftUI with NSViewRepresentable and NSViewControllerRepresentable, embedding SwiftUI in AppKit with NSHostingController and NSHostingView, and coordinating delegate callbacks, state synchronization, and focus across the boundary. Use when integrating text views, scroll views, visual effect views, progress indicators, other AppKit-only or third-party AppKit surfaces into a SwiftUI macOS app, or when migrating an AppKit app to SwiftUI incrementally."
sources: https://developer.apple.com/videos/play/wwdc2022/10075/, https://developer.apple.com/documentation/swiftui/nsviewrepresentable, https://swiftui-lab.com/a-powerful-combo/, https://stackoverflow.com/questions/79862332/nshostingview-with-swiftui-gestures-not-receiving-mouse-events-behind-another-ns, https://developer.apple.com/documentation/swiftui/nsviewrepresentable/sizethatfits(_:nsview:context:)-fuqx, https://developer.apple.com/documentation/swiftui/nshostingview, https://developer.apple.com/documentation/swiftui/nshostingcontroller, https://developer.apple.com/documentation/swiftui/nshostingcontroller/sizingoptions, https://developer.apple.com/documentation/swiftui/nshostingview/sizingoptions, https://developer.apple.com/documentation/swiftui/nsviewrepresentable/makecoordinator(), https://developer.apple.com/forums/thread/749620, https://www.massicotte.org/swiftui-coordinator-parent/, https://developer.apple.com/videos/play/wwdc2019/231/, https://mjtsai.com/blog/2023/08/03/how-nshostingview-determines-its-sizing/, https://developer.apple.com/forums/thread/123603
researched: 2026-06-06
platform: macOS 26+
---

# SwiftUI-AppKit Interop

Bridge AppKit and SwiftUI in both directions. Wrap NSView subclasses and NSViewController instances for use in SwiftUI, embed SwiftUI views inside AppKit screens, and synchronize state across the boundary. Targets macOS 26+ with Swift 6.3 patterns; notes backward-compatible to macOS 13 unless stated otherwise. See [references/nsviewrepresentable-recipes.md](references/nsviewrepresentable-recipes.md) for complete wrapping recipes for common AppKit views.

## Contents

- [NSViewRepresentable Protocol](#nsviewrepresentable-protocol)
- [NSViewControllerRepresentable Protocol](#nsviewcontrollerrepresentable-protocol)
- [The Coordinator Pattern](#the-coordinator-pattern)
- [NSHostingController](#nshostingcontroller)
- [NSHostingView](#nshostingview)
- [First Responder and Focus](#first-responder-and-focus)
- [Key Differences from UIViewRepresentable](#key-differences-from-uiviewrepresentable)
- [Sizing and Layout](#sizing-and-layout)
- [Common Pitfalls](#common-pitfalls)
- [Constraints & Gotchas](#constraints-and-gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## NSViewRepresentable Protocol

Use `NSViewRepresentable` to wrap any `NSView` subclass for use in SwiftUI. The protocol parallels `UIViewRepresentable` (iOS) but uses `NSView` types and macOS-specific lifecycle.

### Required Methods

```swift
struct WrappedTextView: NSViewRepresentable {
    @Binding var text: String

    func makeNSView(context: Context) -> NSTextView {
        // Called ONCE when SwiftUI inserts this view into the hierarchy.
        let textView = NSTextView()
        textView.delegate = context.coordinator
        textView.font = .preferredFont(forTextStyle: .body)
        textView.isRichText = false
        return textView
    }

    func updateNSView(_ nsView: NSTextView, context: Context) {
        // Called on EVERY SwiftUI state change that affects this view.
        if nsView.string != text {
            nsView.string = text
        }
    }
}
```

### Lifecycle Timing

| Method | When Called | Purpose |
|--------|------------|---------|
| `makeCoordinator()` | Before `makeNSView`. Once per representable lifetime. | Create the delegate reference type. |
| `makeNSView(context:)` | Once, when the representable enters the view tree. | Allocate and configure the AppKit view. |
| `updateNSView(_:context:)` | Immediately after `makeNSView`, then on every relevant state change. | Push SwiftUI state into the AppKit view. |
| `dismantleNSView(_:coordinator:)` | When the representable is removed from the view tree. | Clean up observers, timers, subscriptions. |
| `sizeThatFits(_:nsView:context:)` | During SwiftUI layout passes. | Return preferred size or nil for default behavior. |

### Optional: dismantleNSView

```swift
static func dismantleNSView(_ nsView: NSTextView, coordinator: Coordinator) {
    coordinator.cancellables.removeAll()
    nsView.delegate = nil
}
```

### sizeThatFits

Override `sizeThatFits(_:nsView:context:)` to return the preferred size for the representable given a proposed size. Return `nil` to let SwiftUI use its default sizing algorithm. This method may be called multiple times during a single layout pass with different proposals.

```swift
func sizeThatFits(_ proposal: ProposedViewSize, nsView: NSTextView, context: Context) -> CGSize? {
    // Let the NSTextView compute its ideal size based on text content
    let size = nsView.intrinsicContentSize
    if proposal.width != nil {
        return CGSize(width: proposal.width!, height: size.height)
    }
    return size
}
```

## NSViewControllerRepresentable Protocol

Use `NSViewControllerRepresentable` to wrap an `NSViewController` subclass — typically for panels, file browsers, or any controller that manages a complex view hierarchy.

```swift
struct FilePickerView: NSViewControllerRepresentable {
    @Binding var selectedURL: URL?
    @Environment(\.dismiss) private var dismiss

    func makeNSViewController(context: Context) -> NSOpenPanel {
        let panel = NSOpenPanel()
        panel.delegate = context.coordinator
        panel.allowedContentTypes = [.plainText]
        panel.canChooseDirectories = false
        return panel
    }

    func updateNSViewController(_ nsViewController: NSOpenPanel, context: Context) {
        // Usually empty for modal panels — nothing to push from SwiftUI.
    }

    func makeCoordinator() -> Coordinator { Coordinator(self) }
}
```

### Lifecycle Differences vs UIKit

`NSViewControllerRepresentable` mirrors `UIViewControllerRepresentable` in structure but differs in lifecycle:

- **No `viewDidLoad`/`viewWillAppear` equivalent in SwiftUI.** The controller's view is loaded lazily by AppKit. If you need to configure the controller after its view loads, do it in `makeNSViewController` after the controller is created, or override `loadView` in a custom subclass.
- **No `sizeThatFits` equivalent via controller.** AppKit uses `intrinsicContentSize` and Auto Layout constraints rather than UIKit's `sizeThatFits(_:)`. See [Sizing and Layout](#sizing-and-layout).
- **Coordinate system is flipped.** AppKit's default coordinate system has the origin at the bottom-left. `NSView` can return `isFlipped = true` to match UIKit/SwiftUI's top-left origin. This matters when mixing layout calculations.

## The Coordinator Pattern

### Why Coordinators Exist

AppKit delegates, data sources, and target-action patterns require a reference type (`class`). SwiftUI representable structs are value types and cannot serve as delegates. The Coordinator is a `class` instance that SwiftUI creates and manages for you — it lives as long as the representable view.

### Structure

```swift
struct SearchFieldWrapper: NSViewRepresentable {
    @Binding var text: String
    var onSearch: (String) -> Void

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    func makeNSView(context: Context) -> NSSearchField {
        let field = NSSearchField()
        field.delegate = context.coordinator  // Set delegate HERE, not in updateNSView
        field.target = context.coordinator
        field.action = #selector(Coordinator.searchFieldChanged)
        return field
    }

    func updateNSView(_ nsView: NSSearchField, context: Context) {
        if nsView.stringValue != text {
            nsView.stringValue = text
        }
    }

    @MainActor
    final class Coordinator: NSObject, NSSearchFieldDelegate {
        var parent: SearchFieldWrapper

        init(_ parent: SearchFieldWrapper) { self.parent = parent }

        @objc func searchFieldChanged(_ sender: NSSearchField) {
            parent.text = sender.stringValue
        }

        func searchFieldDidStartSearching(_ sender: NSSearchField) {
            parent.onSearch(parent.text)
        }
    }
}
```

### Key Rules

1. **Set the delegate in `makeNSView`/`makeNSViewController`, never in `updateNSView`.** The update method runs on every state change — setting the delegate there causes redundant assignment.

2. **The coordinator's `parent` property is a copy, updated automatically.** SwiftUI creates a new representable struct value for each render pass and updates the coordinator's `parent` reference before each call to `updateNSView`. The coordinator always sees current `@Binding` values through `parent`. However, since the coordinator holds a copy (not a reference to the original struct), mutations to `parent` only propagate to SwiftUI through `@Binding` writeback — not through direct struct mutation.

   **Important caveat from real-world use:** When wrapping `NSTextView` in `NavigationSplitView`, the representable may be recreated frequently by SwiftUI. In these cases, storing binding data *inside* the Coordinator (rather than relying solely on `parent`) can provide more reliable updates. See [The Curious Case of SwiftUI's Coordinator Parent](https://www.massicotte.org/swiftui-coordinator-parent/) for details.

3. **Use `[weak coordinator]` in closures** to avoid retain cycles between the coordinator and AppKit objects that capture it.

### Binding Bridging

For two-way sync, the coordinator writes to `parent.bindingProperty` in delegate callbacks; `updateNSView` reads the binding and pushes it into the AppKit view:

```swift
// SwiftUI -> AppKit: in updateNSView
if nsView.string != text { nsView.string = text }

// AppKit -> SwiftUI: in Coordinator delegate method
func controlTextDidChange(_ obj: Notification) {
    guard let field = obj.object as? NSTextField else { return }
    parent.text = field.stringValue
}
```

Guard against redundant updates to prevent infinite loops — the delegate fires on every change, which writes to the binding, which triggers `updateNSView` again.

## NSHostingController

Embed SwiftUI views inside AppKit view controllers using `NSHostingController`.

### Basic Embedding

```swift
final class SidebarViewController: NSViewController {
    private let hostingController: NSHostingController<SidebarContent>

    init() {
        hostingController = NSHostingController(rootView: SidebarContent())
        super.init(nibName: nil, bundle: nil)
    }

    required init?(coder: NSCoder) { fatalError() }

    override func loadView() {
        view = NSView()
    }

    override func viewDidLoad() {
        super.viewDidLoad()

        // 1. Add as child
        addChild(hostingController)

        // 2. Add and pin the view
        hostingController.view.translatesAutoresizingMaskIntoConstraints = false
        view.addSubview(hostingController.view)
        NSLayoutConstraint.activate([
            hostingController.view.topAnchor.constraint(equalTo: view.topAnchor),
            hostingController.view.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            hostingController.view.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            hostingController.view.bottomAnchor.constraint(equalTo: view.bottomAnchor),
        ])

        // 3. Notify the child
        hostingController.didMove(toParent: self)
    }
}
```

The three-step sequence (addChild, add view, didMove) is mandatory. Skipping any step causes containment callbacks to misfire.

### sizingOptions

`NSHostingController.sizingOptions` (macOS 13+) controls how Auto Layout constraints are derived from SwiftUI content:

```swift
// Default: derives min, ideal (intrinsic), and max constraints from SwiftUI content
hostingController.sizingOptions = [.minSize, .intrinsicContentSize, .maxSize]

// Performance optimization: skip constraint generation when size is known/fixed
hostingController.sizingOptions = .minSize

// No constraints (purely manual sizing)
hostingController.sizingOptions = []
```

When used as a window's `contentViewController`, the hosting controller can propagate SwiftUI content's `minSize` and `maxSize` to the window's `contentMinSize` and `contentMaxSize`. Reducing options can improve performance if you know the content size in advance or have a fixed frame. If the hosting controller's frame differs from what the SwiftUI content requires, the content is centered within the frame.

### Updating the Root View

```swift
func updateSidebar(items: [Item]) {
    hostingController.rootView = SidebarContent(items: items)
}
```

For observable models, pass an `@Observable` object and SwiftUI tracks changes automatically — no need to reassign `rootView`.

### Presentation Methods

```swift
// Present as popover
viewController.present(hostingController, asPopoverRelativeTo: rect, of: view, preferredEdge: .maxY, behavior: .transient)

// Present as sheet
viewController.presentAsSheet(hostingController)

// Present as modal window
viewController.presentAsModalWindow(hostingController)
```

## NSHostingView

Use `NSHostingView` when you need to embed SwiftUI in a context that doesn't involve a view controller — menus, popovers, toolbars, custom `NSView` subclasses, or layer-backed hierarchies.

```swift
final class CustomMenuItemView: NSView {
    private var hostingView: NSHostingView<MenuItemContent>?

    func configure(with item: MenuItem) {
        hostingView?.removeFromSuperview()

        let hv = NSHostingView(rootView: MenuItemContent(item: item))
        hv.translatesAutoresizingMaskIntoConstraints = false
        addSubview(hv)
        NSLayoutConstraint.activate([
            hv.topAnchor.constraint(equalTo: topAnchor),
            hv.leadingAnchor.constraint(equalTo: leadingAnchor),
            hv.trailingAnchor.constraint(equalTo: trailingAnchor),
            hv.bottomAnchor.constraint(equalTo: bottomAnchor),
        ])
        hostingView = hv
    }
}
```

### NSHostingView vs NSHostingController

| | `NSHostingController` | `NSHostingView` |
|--|----------------------|-----------------|
| **Use when** | You have an `NSViewController` managing the SwiftUI content | You need SwiftUI inside a plain `NSView`, menu, or panel |
| **Lifecycle** | Participates in `NSViewController` containment | Standalone; no controller lifecycle |
| **Trait propagation** | Automatic via `NSViewController` | Manually via `sizingOptions`; inherits from parent view's `NSWindow` |
| **Event handling** | Full responder chain via controller | Direct; may need manual mouse event forwarding |

### NSHostingView sizingOptions

`NSHostingView` also has a `sizingOptions` property (same `NSHostingSizingOptions` set as `NSHostingController`) controlling constraint generation:

```swift
let hv = NSHostingView(rootView: MyView())
hv.sizingOptions = [.minSize, .intrinsicContentSize, .maxSize]  // default
```

By default, `NSHostingView` uses all three sizing options, which can prevent a SwiftUI `ScrollView` or similar from shrinking in a stack. Reducing to `.minSize` or `[]` allows the view to be sized by its container without constraint resistance.

### Mouse Event Gotcha

`NSHostingView` does not automatically receive mouse events when another `NSView` sits on top of it in the same window. If a transparent overlay view covers the hosting view, SwiftUI gestures (buttons, sliders, hover) stop working. Solutions:

1. **Override `hitTest(_:)`** on the overlay to pass through events: return `nil` for points that should reach the hosting view.
2. **Override `mouseDown(with:)`** and related methods on the overlay to forward events to the next responder.
3. **Use `allowsHitTesting(false)`** on the SwiftUI side for views that should be transparent to events.

## First Responder and Focus

### Propagating Focus Across the Boundary

SwiftUI's focus system (`@FocusState`, `focusable()`) operates independently from AppKit's responder chain. When an AppKit view is wrapped in `NSViewRepresentable`, SwiftUI focus changes do not automatically make the AppKit view the first responder, and vice versa.

To bridge this, manage first responder status in `updateNSView`:

```swift
struct FocusableTextView: NSViewRepresentable {
    @Binding var text: String
    @Binding var isFocused: Bool

    func makeNSView(context: Context) -> NSTextView {
        let textView = NSTextView()
        textView.delegate = context.coordinator
        return textView
    }

    func updateNSView(_ nsView: NSTextView, context: Context) {
        if nsView.string != text {
            nsView.string = text
        }

        if isFocused {
            nsView.window?.makeFirstResponder(nsView)
        }
    }

    @MainActor
    final class Coordinator: NSObject, NSTextViewDelegate {
        var parent: FocusableTextView
        init(_ parent: FocusableTextView) { self.parent = parent }

        func textDidBeginEditing(_ notification: Notification) {
            parent.isFocused = true
        }

        func textDidEndEditing(_ notification: Notification) {
            parent.isFocused = false
        }
    }
}
```

### Responder Chain Considerations

- `NSHostingView` participates in the responder chain. SwiftUI views inside it can become first responder through the normal chain.
- `NSHostingController`'s view also participates, but the controller itself can intercept `acceptsFirstResponder` and `becomeFirstResponder()` if overridden.
- For custom `NSView` subclasses hosting SwiftUI, ensure `acceptsFirstResponder` returns `true` if the hosted SwiftUI content needs keyboard input.

## Key Differences from UIViewRepresentable

### Platform and Type

- **`NSViewRepresentable`**: wraps `NSView` subclasses for macOS (AppKit).
- **`UIViewRepresentable`**: wraps `UIView` subclasses for iOS/tvOS/watchOS (UIKit).

They are not interchangeable — `UIViewRepresentable` will not compile in a macOS-only target. For multiplatform targets, use conditional compilation or typealiases:

```swift
#if canImport(UIKit)
typealias ViewRepresentable = UIViewRepresentable
#elseif canImport(AppKit)
typealias ViewRepresentable = NSViewRepresentable
#endif
```

### No `sizeThatFits` on iOS Equivalent (Pre-iOS 16)

On iOS, `UIViewRepresentable` gained `sizeThatFits(_:uiView:context:)` in iOS 16+. `NSViewRepresentable` has the same method but AppKit primarily uses `intrinsicContentSize` and Auto Layout constraint-based sizing. To control sizing:

- Override `intrinsicContentSize` on a custom `NSView` subclass.
- Add explicit width/height constraints on the AppKit view.
- Use `.frame(width:height:)` in SwiftUI to impose a size.
- Override `sizeThatFits(_:nsView:context:)` to return a custom size.

### `intrinsicContentSize` Role

`NSView.intrinsicContentSize` is the primary mechanism for communicating natural size to SwiftUI. If the AppKit view returns `NSView.noIntrinsicMetric` for an axis, SwiftUI treats that axis as flexible. Ensure your wrapped view provides meaningful intrinsic content sizes, or supply explicit frame constraints.

### Flipped Coordinate System

AppKit's default coordinate system has the origin at the bottom-left, with Y increasing upward. SwiftUI and UIKit use top-left origin. This matters when:

- Performing manual layout calculations in the AppKit view.
- Converting points between coordinate systems.
- Drawing custom content in `draw(_:)`.

Override `var isFlipped: Bool { true }` on custom `NSView` subclasses to match SwiftUI's coordinate system and avoid confusion.

### Layer-Backing Differences

AppKit views require explicit `wantsLayer = true` for Core Animation layer backing. SwiftUI does not assume layer backing. If the wrapped AppKit view needs a layer (for animations, filters, or `NSVisualEffectView`), enable it in `makeNSView`:

```swift
func makeNSView(context: Context) -> NSVisualEffectView {
    let view = NSVisualEffectView()
    view.wantsLayer = true
    view.blendingMode = .behindWindow
    view.material = .sidebar
    return view
}
```

## Sizing and Layout

### intrinsicContentSize Bridging

AppKit views wrapped in `NSViewRepresentable` communicate their natural size to SwiftUI through `intrinsicContentSize`. SwiftUI respects this during layout unless overridden by `frame()` or `fixedSize()`.

For `NSTextView` specifically, override `intrinsicContentSize` in a subclass to return the layout manager's used rect — this ensures the SwiftUI layout allocates the correct height for the text content:

```swift
class WrappingTextView: NSTextView {
    override var intrinsicContentSize: NSSize {
        layoutManager?.usedRect(for: textContainer ?? NSTextContainer()) ?? super.intrinsicContentSize
    }

    override func didChangeText() {
        super.didChangeText()
        invalidateIntrinsicContentSize()
    }
}
```

### NSTextView Sizing Lag

`NSTextView` embedded in `NSViewRepresentable` may not render its full height on the first frame — only a single visible line appears before resizing. To mitigate:

1. Override `layout()` in the `NSTextView` subclass to force layoutManager layout and invalidate intrinsic content size.
2. Use `invalidateIntrinsicContentSize()` in `didChangeText()`.
3. Set `textContainer.widthTracksTextView = false` when using the text view inside a horizontal `ScrollView`, and give the container an unlimited size to avoid zero-width collapse.

### fixedSize() and frame() Interactions

| SwiftUI Modifier | Effect on Representable |
|-----------------|------------------------|
| No modifier | SwiftUI uses `intrinsicContentSize` as ideal size; the view is flexible. |
| `.fixedSize()` | Forces the representable to its ideal (intrinsic) size in both axes. |
| `.fixedSize(horizontal: true, vertical: false)` | Fixes width to intrinsic; height remains flexible. |
| `.frame(width:height:)` | Overrides the proposed size; AppKit view receives this size. |

### Layout Ambiguity

If the AppKit view does not provide `intrinsicContentSize` and no frame constraints are set, SwiftUI may collapse the view to zero size. Always ensure at least one sizing mechanism is active.

## Common Pitfalls

### Layout Ambiguity

When an `NSView` subclass returns `NSView.noIntrinsicMetric` for both axes and no external constraints are set, SwiftUI cannot determine a size. The view collapses to zero. Fix by overriding `intrinsicContentSize`, adding constraints inside the AppKit view, or applying `.frame()` in SwiftUI.

### `.fixedSize()` Traps

Applying `.fixedSize()` to a representable that contains a scroll view or text view prevents the content from growing. The view is locked to its initial intrinsic size. Use `.frame(minWidth:minHeight:maxWidth:maxHeight:)` instead to set bounds without collapsing flexibility.

### Layer Flash

When wrapping layer-backed AppKit views (especially `NSVisualEffectView`), a brief visual flash can occur on appearance. This happens because the layer is created after the view is already visible. Mitigate by:

1. Setting `wantsLayer = true` in `makeNSView`, not in `updateNSView`.
2. Pre-warming the layer by setting `layer?.backgroundColor` immediately.
3. Avoiding unnecessary `updateNSView` calls that reconfigure the layer.

### Update Loops

`updateNSView` is called whenever SwiftUI state changes — including changes triggered by the coordinator writing to a `@Binding`. Guard against redundant updates:

```swift
func updateNSView(_ nsView: NSTextView, context: Context) {
    if nsView.string != text {
        nsView.string = text
    }
}
```

Without the guard, setting `nsView.string` may trigger the delegate's `controlTextDidChange`, which writes to `parent.text`, which triggers `updateNSView` again.

### Scene Bridging Options

When using `NSHostingView`/`NSHostingController` to present SwiftUI content, `sceneBridgingOptions` controls which window aspects the hosting view manages (title, toolbar, tab). Set this to coordinate properly with your AppKit window controller's responsibilities.

## Constraints & Gotchas

- **Delegate assignment timing.** Set delegates in `makeNSView`, never in `updateNSView`. Redundant assignment on every state change can reset internal delegate state.
- **Coordinator parent is a copy.** The coordinator holds a copy of the representable struct. SwiftUI updates this copy before each `updateNSView` call, but mutations to `parent` in the coordinator only affect the copy — they propagate to SwiftUI through `@Binding` writeback, not through direct struct mutation.
- **NSHostingView in menus.** When using `NSHostingView` inside `NSMenuItem`, the hosting view is retained by the menu item. Ensure the hosting view is not shared across multiple menu items.
- **Responder chain breaks.** If a custom `NSView` sits between the `NSHostingView` and the window, it may intercept events intended for the SwiftUI content. Verify the responder chain with `print(nextResponder)`.
- **Text input with NSTextView.** `NSTextView` requires a proper text system setup (`NSTextContainer`, `NSLayoutManager`) when created programmatically outside of an `NSScrollView`. For simple text fields, prefer `NSTextField`.
- **Coordinator lifetime.** `makeCoordinator()` is called once per representable *lifetime*, but SwiftUI may recreate the representable struct many times during refreshes. The coordinator persists but receives a fresh `parent` copy on each update.

## Common Mistakes

**DON'T:** Create the AppKit view in `updateNSView`.
**DO:** Create the view once in `makeNSView`; only configure/update it in `updateNSView`.
*Why:* `updateNSView` runs on every state change. Creating a new view each time destroys all AppKit state (selection, scroll position, first responder).

**DON'T:** Set delegates in `updateNSView`.
**DO:** Set delegates in `makeNSView`/`makeNSViewController` only.
*Why:* Redundant delegate assignment on every update can reset internal state in views like `NSTableView` or `NSOutlineView`.

**DON'T:** Hold strong references to the Coordinator from closures.
**DO:** Use `[weak coordinator]` in closures.
*Why:* AppKit objects often store closures (completion handlers, action blocks). A strong reference cycle leaks memory.

**DON'T:** Forget to call `parent.dismiss()` or completion handlers.
**DO:** Use the coordinator to track dismissal and invoke `parent.dismiss()` in all delegate exit paths.
*Why:* Modal panels presented by SwiftUI need their dismiss binding toggled, or the sheet state becomes inconsistent.

**DON'T:** Ignore `dismantleNSView` for views that hold observers or timers.
**DO:** Clean up `NotificationCenter` observers, `Combine` subscriptions, and `Timer` instances in `dismantleNSView`.
*Why:* Without cleanup, observers and timers continue firing after the view is removed.

**DON'T:** Assume `intrinsicContentSize` is automatic.
**DO:** Override `intrinsicContentSize` on custom `NSView` subclasses or supply explicit sizing.
*Why:* Many AppKit views return `NSView.noIntrinsicMetric` for one or both axes, causing SwiftUI to collapse the view.

**DON'T:** Use `NSHostingView` when you need view controller lifecycle.
**DO:** Use `NSHostingController` when containment, trait propagation, or lifecycle callbacks matter.
*Why:* `NSHostingView` is a plain `NSView` with no controller; it cannot participate in `NSViewController` containment.

**DON'T:** Try to use `@State` in the Coordinator — it is not a `View`.
**DO:** Use regular stored properties on the Coordinator and communicate to SwiftUI via `parent`'s `@Binding` properties.
*Why:* `@State` only works inside `View` conformances.

**DON'T:** Ignore `sizeThatFits(_:nsView:context:)` when custom sizing is needed.
**DO:** Override `sizeThatFits` or ensure `intrinsicContentSize` is properly implemented on your AppKit view.
*Why:* Without explicit sizing, SwiftUI may use default layout that doesn't match the AppKit view's content.

## Review Checklist

- [ ] View/controller created in `make*`, not `update*`
- [ ] Coordinator set as delegate in `make*`, not `update*`
- [ ] `@Binding` used for two-way state sync
- [ ] `updateNSView` handles all SwiftUI state changes with redundancy guards
- [ ] `dismantleNSView` cleans up observers/timers if needed
- [ ] No retain cycles between coordinator and closures (`[weak coordinator]`)
- [ ] `NSHostingController` properly added as child (`addChild` + `didMove(toParent:)`)
- [ ] Sizing strategy chosen (`intrinsicContentSize` vs fixed `frame` vs constraints vs `sizeThatFits`)
- [ ] `intrinsicContentSize` overridden or explicit sizing provided for custom NSView subclasses
- [ ] Environment values read in `updateNSView` via `context.environment` where needed
- [ ] Coordinator marked `@MainActor` for strict concurrency
- [ ] Modal panels dismiss in all delegate exit paths (success, cancel, error)
- [ ] First responder bridging implemented if keyboard input crosses the boundary
- [ ] `NSHostingView` mouse event forwarding verified when overlays are present
- [ ] Flipped coordinate system handled if doing manual layout in AppKit views
- [ ] `NSHostingView`/`NSHostingController` `sizingOptions` configured appropriately
- [ ] `sizeThatFits(_:nsView:context:)` implemented for custom sizing behaviors
- [ ] Coordinator parent copy semantics understood — state flows through `@Binding` only

## References

- Wrapping recipes: [references/nsviewrepresentable-recipes.md](references/nsviewrepresentable-recipes.md)
- Apple docs: [NSViewRepresentable](https://developer.apple.com/documentation/swiftui/nsviewrepresentable)
- Apple docs: [NSViewControllerRepresentable](https://developer.apple.com/documentation/swiftui/nsviewcontrollerrepresentable)
- Apple docs: [NSHostingController](https://developer.apple.com/documentation/swiftui/nshostingcontroller)
- Apple docs: [NSHostingView](https://developer.apple.com/documentation/swiftui/nshostingview)
- Apple docs: [sizingOptions](https://developer.apple.com/documentation/swiftui/nshostingcontroller/sizingoptions)
- Apple docs: [sizeThatFits](https://developer.apple.com/documentation/swiftui/nsviewrepresentable/sizethatfits(_:nsview:context:)-fuqx)
- WWDC 2022: [Use SwiftUI with AppKit](https://developer.apple.com/videos/play/wwdc2022/10075/)
- WWDC 2019: [Integrating SwiftUI](https://developer.apple.com/videos/play/wwdc2019/231/)
- Michael Tsai: [How NSHostingView Determines Its Sizing](https://mjtsai.com/blog/2023/08/03/how-nshostingview-determines-its-sizing/)
- Massicotte: [The Curious Case of SwiftUI's Coordinator Parent](https://www.massicotte.org/swiftui-coordinator-parent/)
- Apple Developer Forums: [Coordinator pattern for NSViewRepresentable](https://developer.apple.com/forums/thread/749620)
