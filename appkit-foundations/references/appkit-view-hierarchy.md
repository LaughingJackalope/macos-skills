# AppKit View Hierarchy Reference

Deep reference for NSView and NSViewController patterns, lifecycle timing, containment, and Auto Layout on macOS 26+.

## NSView vs NSViewController

| Concern | NSView | NSViewController |
|---------|--------|------------------|
| Drawing | `draw(_:)` | Coordinates, doesn't draw |
| Event handling | `mouseDown:`, `keyDown:` | Routes to child controllers |
| Layout | `layout()`, constraints | `viewDidLoad()` setup |
| Lifecycle | `viewDidMoveToWindow` | `viewDidLoad` → `viewDidAppear` |
| Use when | Custom drawing, leaf UI | Coordinating a subtree |

**Rule:** If you're managing data, coordinating child views, or responding to lifecycle events, use `NSViewController`. If you're drawing custom content or handling raw events, subclass `NSView`.

## View Controller Lifecycle Timing

The full lifecycle in order:

```
init(nibName:bundle:)
  → loadView()              // Create self.view here
  → viewDidLoad()           // View exists, hierarchy not yet on screen
  → viewWillLayout()        // Before Auto Layout runs
  → layout()                // (NSView) Auto Layout resolves
  → viewDidLayout()         // After Auto Layout resolves
  → viewWillAppear()        // About to be visible
  → viewDidAppear()         // Visible on screen
  → ... time passes ...
  → viewWillDisappear()     // About to be removed
  → viewDidDisappear()      // Removed from screen
  → deinit
```

**Where to put setup code:**

- `loadView()`: Assign `self.view`. Create the root view. Nothing else.
- `viewDidLoad()`: Create subviews, configure constraints, set data sources, register for notifications. The view is loaded but not yet laid out.
- `viewWillLayout()`: Adjust constraints that depend on the final size.
- `viewDidLayout()`: Read final frames after layout. Adjust non-constraint layout.
- `viewWillAppear()`: Start timers, begin animations, refresh data that may have changed while offscreen.
- `viewDidAppear()`: Begin expensive work (network calls, file I/O) that should only run when visible.

```swift
final class DetailViewController: NSViewController {
    var item: Item!  // Set before view loads

    override func loadView() {
        let view = NSView(frame: NSRect(x: 0, y: 0, width: 400, height: 300))
        view.wantsLayer = true
        self.view = view
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        // Safe to access self.view
        // Create subviews, set up constraints
        // Do NOT depend on final frame here
    }

    override func viewWillLayout() {
        super.viewWillLayout()
        // Adjust constraints based on available width
    }

    override func viewDidLayout() {
        super.viewDidLayout()
        // Read final frames, adjust non-constraint layout
    }

    override func viewWillAppear() {
        super.viewWillAppear()
        // Refresh data, start animations
    }
}
```

## addSubview vs addChild

These are fundamentally different operations:

- **`addSubview(_:)`**: Adds a view to the visual hierarchy. No responder chain or lifecycle integration.
- **`addChild(_:)`**: Establishes a view controller containment relationship. Required for the child controller to receive lifecycle events.

**Always use both for view controller containment:**

```swift
// CORRECT: Full containment
func addChildController(_ child: NSViewController, in container: NSView) {
    addChild(child)                              // 1. Establish containment
    child.view.translatesAutoresizingMaskIntoConstraints = false
    container.addSubview(child.view)             // 2. Add to view hierarchy
    // Add constraints...
    child.didMove(toParent: self)                // 3. Notify child
}

// WRONG: Just adding the view
func addChildControllerWrong(_ child: NSViewController, in container: NSView) {
    container.addSubview(child.view)  // Child never gets viewDidLoad, viewWillAppear, etc.
}

// WRONG: Just adding the child
func addChildControllerWrong2(_ child: NSViewController, in container: NSView) {
    addChild(child)  // Child gets lifecycle events but its view is not in the hierarchy
}
```

**Removing a child controller:**

```swift
func removeChildController(_ child: NSViewController) {
    child.willMove(toParent: nil)
    child.view.removeFromSuperview()
    child.removeFromParent()
}
```

## Auto Layout with NSLayoutConstraint

AppKit Auto Layout uses the same engine as UIKit but with some API differences.

**Always set `translatesAutoresizingMaskIntoConstraints = false`** on views you constrain programmatically:

```swift
let label = NSTextField(labelWithString: "Hello")
label.translatesAutoresizingMaskIntoConstraints = false
view.addSubview(label)

NSLayoutConstraint.activate([
    label.centerXAnchor.constraint(equalTo: view.centerXAnchor),
    label.centerYAnchor.constraint(equalTo: view.centerYAnchor),
])
```

**NSScrollView + document view pattern:**

```swift
let scrollView = NSScrollView()
scrollView.hasVerticalScroller = true
scrollView.autohidesScrollers = true
scrollView.translatesAutoresizingMaskIntoConstraints = false

let documentView = NSView()  // Your content view
documentView.translatesAutoresizingMaskIntoConstraints = false

scrollView.documentView = documentView
view.addSubview(scrollView)

NSLayoutConstraint.activate([
    scrollView.topAnchor.constraint(equalTo: view.topAnchor),
    scrollView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
    scrollView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
    scrollView.bottomAnchor.constraint(equalTo: view.bottomAnchor),

    // Pin document view to scrollView's content area
    documentView.leadingAnchor.constraint(equalTo: scrollView.contentView.leadingAnchor),
    documentView.trailingAnchor.constraint(equalTo: scrollView.contentView.trailingAnchor),
    documentView.topAnchor.constraint(equalTo: scrollView.contentView.topAnchor),
])
```

**NSTableView in NSScrollView (the standard pattern):**

```swift
let tableView = NSTableView()
let column = NSTableColumn(identifier: NSUserInterfaceItemIdentifier("main"))
column.title = "Items"
tableView.addTableColumn(column)
tableView.headerView = nil  // optional: hide header

let scrollView = NSScrollView()
scrollView.documentView = tableView
scrollView.hasVerticalScroller = true
scrollView.translatesAutoresizingMaskIntoConstraints = false
view.addSubview(scrollView)
```

**Content hugging and compression resistance:**

```swift
// A label that resists shrinking below its intrinsic width
label.setContentHuggingPriority(.defaultHigh, for: .horizontal)
label.setContentCompressionResistancePriority(.defaultHigh, for: .horizontal)

// A text field that stretches to fill available space
textField.setContentHuggingPriority(.defaultLow, for: .horizontal)
```

## NSSplitViewController Patterns

`NSSplitViewController` is the standard way to create resizable panes:

```swift
let splitController = NSSplitViewController()

let sidebarItem = NSSplitViewItem(sidebarWithViewController: sidebarVC)
sidebarItem.minimumThickness = 150
sidebarItem.maximumThickness = 300
sidebarItem.canCollapse = true
sidebarItem.holdingPriority = .defaultLow  // Sidebar shrinks first

let contentItem = NSSplitViewItem(viewController: contentVC)
contentItem.minimumThickness = 300

splitController.addSplitViewItem(sidebarItem)
splitController.addSplitViewItem(contentItem)
```

**Split view item behavior:**

- `canCollapse`: Whether the user can double-click the divider to collapse the pane.
- `minimumThickness`: The pane won't shrink below this.
- `holdingPriority`: Lower priority panes shrink first when space is constrained.
- `automaticMaximumThickness`: Constrains the pane to a percentage of the split view.

## View Debugging

Use `NSView._printHierarchy()` in the debugger to inspect the view tree:

```swift
// In lldb:
// po view._printHierarchy()
```

Or programmatically:

```swift
func printHierarchy(_ view: NSView, indent: Int = 0) {
    let prefix = String(repeating: "  ", count: indent)
    print("\(prefix)\(type(of: view)) frame=\(view.frame)")
    for subview in view.subviews {
        printHierarchy(subview, indent: indent + 1)
    }
}
```
