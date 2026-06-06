---
name: swift-architecture-macos
description: "Architecture patterns for macOS apps — MV with @Observable, MVVM, NSDocument-based architecture, multi-window state, NSApplicationDelegate lifecycle, scene-based vs delegate-based app entry points, dependency injection. Use when structuring the architecture of a macOS app."
sources: https://developer.apple.com/documentation/swiftui/app, https://developer.apple.com/documentation/appkit/nsapplicationdelegate, https://developer.apple.com/documentation/swiftui/windowgroup, https://developer.apple.com/documentation/observation, https://developer.apple.com/documentation/appkit/nsdocument, https://developer.apple.com/documentation/Updates/AppKit, https://developer.apple.com/library/archive/documentation/DataManagement/Conceptual/DocBasedAppProgrammingGuideForOSX/KeyObjects/KeyObjects.html, https://developer.apple.com/tutorials/app-dev-training/supporting-multiple-windows, https://developer.apple.com/documentation/swiftui/customizing-window-styles-and-state-restoration-behavior-in-macos, https://soarias.com/swiftui/how-to-build-dependency-injection/, https://swiftcrafted.dev/article/swiftui-observable-macro-complete-guide-observation-framework, https://dev.to/programmingcentral/mastering-dependency-injection-in-swiftui-sharing-ai-clients-with-environment-24kk, https://fatbobman.com/en/posts/the-state-specter-analyzing-a-bug-in-multi-window-swiftui-applications/, https://chaosinmotion.com/2026/02/13/swiftui-style-recomposition-in-appkit-uikit/, https://github.com/pointfreeco/swift-dependencies, https://github.com/kylebrowning/swift-dependency-injection, https://developer.apple.com/videos/play/wwdc2022/10075/
researched: 2026-06-06
platform: macOS 26+
---

# Swift Architecture — macOS

Architecture patterns for macOS apps built with SwiftUI and AppKit. Covers app lifecycle with `NSApplicationDelegateAdaptor`, the `@Observable` macro for MV/MVVM, `NSDocument`-based document architecture, multi-window state management, dependency injection, and choosing between SwiftUI `App` protocol and direct `NSApplicationDelegate`. Targets macOS 26+ with Swift 6.2+ patterns. See [references/architecture-patterns.md](references/architecture-patterns.md) for complete code recipes (NSDocument subclass, `@Observable` view model, DI container, window restoration).

## Contents

- [App Lifecycle](#app-lifecycle)
- [NSApplicationDelegate Integration](#nsapplicationdelegate-integration)
- [Termination & Reopen Callbacks](#termination--reopen-callbacks)
- [@Observable MV Pattern](#observable-mv-pattern)
- [NSDocument Architecture](#nsdocument-architecture)
- [Multi-Window State](#multi-window-state)
- [macOS 26 Window Customization](#macos-26-window-customization)
- [Dependency Injection](#dependency-injection)
- [Scene-based vs Delegate-based](#scene-based-vs-delegate-based)
- [SwiftUI AppKit Bridging](#swiftui-appkit-bridging)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## App Lifecycle

Every SwiftUI macOS app starts from an entry point conforming to the `App` protocol:

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

SwiftUI's `App` protocol is the default entry. Behind the scenes, macOS still runs an `NSApplication` with a shared `NSApplicationDelegate`. The SwiftUI `App` gives you `Scene` declarations, `WindowGroup`, `DocumentGroup`, `Settings`, and `Window` as first-class constructs. Each `Scene` maps to one or more platform objects (`NSWindow`, `NSDocumentController`).

The `App.body` property returns a `Scene` graph. SwiftUI manages scene instantiation, restoration, and teardown. For most apps this is sufficient. When you need `NSApplicationDelegate` hooks — dock menu, termination control, reopen events — use `NSApplicationDelegateAdaptor`.

## NSApplicationDelegate Integration

Inject an `NSApplicationDelegate` into a SwiftUI `App` with `@NSApplicationDelegateAdaptor`:

```swift
@main
struct MyApp: App {
    @NSApplicationDelegateAdaptor(AppDelegate.self)
    private var appDelegate

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

final class AppDelegate: NSObject, NSApplicationDelegate {
    func applicationWillFinishLaunching(_ notification: Notification) {
        // Called before the app is visible but after it launches.
        // Set up singletons, configure logging, register defaults.
    }

    func applicationDidFinishLaunching(_ notification: Notification) {
        // App is fully launched and visible.
    }

    func applicationWillTerminate(_ notification: Notification) {
        // App is about to quit. Flush in-memory state.
    }
}
```

The `AppDelegate` is an `NSObject` subclass that conforms to `NSApplicationDelegate`. SwiftUI holds a strong reference to it for the lifetime of the app. All `@NSApplicationDelegateAdaptor` instances are instantiated before the first scene appears.

### Multiple Scenes

Each scene declaration in `App.body` is independent. You can mix window types:

```swift
var body: some Scene {
    WindowGroup {
        ContentView()
    }

    Inspector("Details") {
        DetailInspector()
    }

    Settings {
        SettingsView()
    }
}
```

### When to Use NSApplicationDelegateAdaptor

Use it when you need:
- `applicationShouldTerminate(_:)` — custom quit confirmation.
- `applicationShouldHandleReopen(_:hasVisibleWindows:)` — dock click behavior.
- `applicationDockMenu(_:)` — custom dock tile menu.
- `application:openURLs:` — custom URL scheme handling beyond the `onOpenURL` scene modifier.
- `applicationWillFinishLaunching` — setup that must run before any scene, like crash reporters or logging.

For most URL handling, prefer `.onOpenURL` on the scene. For dock menu customization, `applicationDockMenu` is the only path — SwiftUI provides no scene modifier for it.

## Termination & Reopen Callbacks

### applicationShouldTerminate

Return an `NSApplication.TerminateReply` to control whether the app can quit:

```swift
func applicationShouldTerminate(_ sender: NSApplication) -> NSApplication.TerminateReply {
    if documentController.hasUnsavedChanges {
        return .terminateCancel
    }
    return .terminateNow
}
```

For document-based apps, `NSDocumentController` handles this automatically. For non-document apps, return `.terminateNow` or `.terminateCancel`. Use `.terminateLater` only for apps that need to schedule background work — call `NSApp.reply(toApplicationShouldTerminate: true)` to proceed or `false` to cancel.

### applicationShouldHandleReopen

Called when the user clicks the dock icon and no visible windows remain:

```swift
func applicationShouldHandleReopen(_ sender: NSApplication, hasVisibleWindows flag: Bool) -> Bool {
    if !flag {
        NSApp.windows.first?.makeKeyAndOrderFront(nil)
        return false
    }
    return true
}
```

Return `true` to let AppKit handle it (cycling through windows). Return `false` and manage the window yourself.

### applicationDockMenu

```swift
func applicationDockMenu(_ sender: NSApplication) -> NSMenu? {
    let menu = NSMenu()
    menu.addItem(NSMenuItem(title: "New Document", action: #selector(newDocument), keyEquivalent: "n"))
    menu.addItem(NSMenuItem(title: "Quick Search", action: #selector(showSearch), keyEquivalent: "s"))
    return menu
}
```

This is the only way to customize the dock tile context menu. There is no SwiftUI equivalent.

## @Observable MV Pattern

The `@Observable` macro (macOS 14+) replaces `ObservableObject`, `@Published`, and `ObservableObjectPublisher`. It uses the new `Observation` framework with compiler-synthesized tracking.

### Migration from ObservableObject

| ObservableObject (old) | @Observable (new) |
|---|---|
| `class ViewModel: ObservableObject` | `class ViewModel` |
| `@Published var items: [Item]` | `var items: [Item]` (auto-tracked) |
| `@StateObject private var vm` in SwiftUI | `@State private var vm` in SwiftUI |
| `@EnvironmentObject var vm: ViewModel` | `@Environment(ViewModel.self) var vm` |

Every stored property is implicitly tracked under `@Observable`. You opt out of tracking with `@ObservationIgnored`:

```swift
@Observable
final class DocumentViewModel {
    var text: String = ""
    var isModified: Bool = false

    @ObservationIgnored
    private let uuid = UUID() // Stable identity, no need to observe
}
```

### State Ownership on macOS

| Equivalent | @Observable Era |
|---|---|
| `@StateObject` → owned, created by view | `@State` — the SwiftUI view owns the lifetime |
| `@ObservedObject` → passed in by parent | `@Bindable` — when you need to pass the object to child views and create bindings |
| `@EnvironmentObject` → inherited through scene | `@Environment(Type.self)` — set via `.environment(viewModel)` |

```swift
struct ContentView: View {
    @State private var viewModel = DocumentViewModel()

    var body: some View {
        TextEditor(text: $viewModel.text)
            .environment(viewModel)
    }
}

struct StatusBarView: View {
    @Environment(DocumentViewModel.self)
    private var viewModel

    var body: some Text("\(viewModel.text.count) characters")
}
```

For two-way binding across views:

```swift
struct ToolbarView: View {
    @Bindable var viewModel: DocumentViewModel
    // Use $viewModel.text to create Binding<String>
}
```

### @Observable with AppKit Automatic Recomposition

macOS 15+ (and default on macOS 26+) supports SwiftUI-style automatic recomposition in AppKit. Reading `@Observable` properties inside specific AppKit lifecycle methods triggers automatic updates when those properties change — no KVO, Combine, or manual refresh needed.

Methods that participate in tracking: `viewWillLayout()`, `viewDidLayout()`, `updateViewConstraints()`, `layout()`, `draw(_:)`, `updateConfiguration(using:)`.

```swift
@Observable @MainActor
final class SharedModel: Sendable {
    var message: String = ""
    var counter: Int = 0
}

// In an NSView subclass:
final class ModelDrivenView: NSView {
    var model: SharedModel?

    override func viewWillLayout() {
        super.viewWillLayout()
        if let model {
            label.stringValue = model.message
            counterLabel.stringValue = "\(model.counter)"
        }
    }
}
```

### macOS-Specific: NSMenuValidation

In NSDocument or AppKit-integrated architectures, validate menu items against your `@Observable` view model:

```swift
extension DocumentViewModel: NSMenuItemValidation {
    func validateMenuItem(_ menuItem: NSMenuItem) -> Bool {
        switch menuItem.action {
        case #selector(save):
            return isModified
        case #selector(delete):
            return selection != nil
        default:
            return true
        }
    }
}
```

The `@ObservationIgnored` macro is required on properties that store reference types (caches, UUIDs, delegates) to prevent the observation framework from deeply traversing them.

## NSDocument Architecture

`NSDocument` remains the standard document model on macOS. It integrates with `NSDocumentController` for open/save/revert, autosave, versioning (NSFilePresenter/NSFileCoordinator), and window management. In the MVC pattern, `NSDocument` acts as the **model controller** — a bridge between the model data and views.

### Core Classes

| Class | Role | MVC Role |
|-------|------|----------|
| `NSDocumentController` | Singleton managing all app documents | Controller (document management) |
| `NSDocument` | Per-document data management & I/O | Model Controller (bridges model & view) |
| `NSWindowController` | Per-window UI management | View Controller |

### Key Mechanics

```swift
final class ProjectDocument: NSDocument {
    var projectData: ProjectData = ProjectData()

    override func makeWindowControllers() {
        let storyboard = NSStoryboard(name: "Main", bundle: nil)
        let controller = storyboard.instantiateController(withIdentifier: "DocumentWindowController") as! NSWindowController
        controller.document = self
        addWindowController(controller)
    }

    override func data(ofType typeName: String) throws -> Data {
        return try JSONEncoder().encode(projectData)
    }

    override func read(from data: Data, ofType typeName: String) throws {
        projectData = try JSONDecoder().decode(ProjectData.self, from: data)
    }
}
```

### makeWindowControllers

Called after the document is initialized or restored. Create your `NSWindowController` instances here — whether from storyboard, nib, or programmatic. Each window controller gets a reference to the document via `controller.document = self`. Call `addWindowController(_:)` for each. The `windowControllers` array holds all window controllers belonging to this document.

### Multi-Window Documents

One document can have multiple window controllers:

```swift
override func makeWindowControllers() {
    let editorWC = EditorWindowController()
    editorWC.document = self
    addWindowController(editorWC)

    let inspectorWC = InspectorWindowController()
    inspectorWC.document = self
    addWindowController(inspectorWC)
}
```

Closing the last window controller of a document triggers auto-close (and its associated undo manager).

### Autosave

Set `self.isAutosave = true` in `init()`. The system autosaves periodically and on application resign. `NSDocument` handles local conflicts with `NSFileVersion`.

### NSDocument + SwiftUI

Bridge `NSDocument` to SwiftUI using `NSWindowController` with `NSHostingController`:

```swift
final class DocumentWindowController: NSWindowController {
    convenience init() {
        let hostingController = NSHostingController(rootView: DocumentEditorView())
        let window = NSWindow(contentViewController: hostingController)
        self.init(window: window)
    }

    override var document: AnyObject? {
        didSet { propagateDocumentToChild() }
    }

    func propagateDocumentToChild() {
        guard let hostingController = window?.contentViewController as? NSHostingController<DocumentEditorView>,
              let doc = document as? ProjectDocument else { return }
        hostingController.rootView = DocumentEditorView(document: doc)
    }
}
```

### NSDocument Subclass as Model Controller

The `NSDocument` subclass manages document behavior and coordinates reading/writing its data, but the actual model data should be separated into its own type. The document is a **model controller**, not the model itself.

```swift
// The model
struct ProjectData: Codable {
    var title: String
    var blocks: [Block]
}

// The document (model controller)
final class ProjectDocument: NSDocument {
    var projectData: ProjectData = ProjectData()

    // I/O methods work with the model
}
```

### NSDocument Access from NSViewController

Do not couple views directly to a specific `NSDocument` instance. Instead, have view controllers observe the data model through the document:

```swift
// In NSViewController:
if let doc = view.window?.windowController?.document as? ProjectDocument {
    // Access doc.projectData
}
```

### NSDocument KVO Dependencies

Use `keyPathsForValuesAffecting<Key>` to declare derived property dependencies:

```swift
final class ProjectDocument: NSDocument {
    @objc dynamic var blocks: [Block] = []
    @objc dynamic var title: String = ""

    @objc class func keyPathsForValuesAffectingDisplayName() -> Set<String> {
        [#keyPathsForValuesAffectingTitle]
    }
}
```

## Multi-Window State

### WindowGroup with Value Types

```swift
struct MyApp: App {
    var body: some Scene {
        WindowGroup("Item") {
            ItemListView()
        }
        .windowStyle(.titleBar)
        .windowResizability(.contentSize)

        WindowGroup("Detail", for: Item.ID.self) { $itemID in
            DetailView(itemID: itemID ?? UUID())
        }
    }
}
```

Each `WindowGroup` creates independent window instances. Each window stores its own `@State` and `@StateObject` data separately.

### Opening Windows Programmatically

Use `openWindow` from the environment:

```swift
struct ToolbarView: View {
    @Environment(\.openWindow) private var openWindow

    var body: some View {
        Button("Open Detail") {
            openWindow(value: "detailWindow")
        }
    }
}
```

### Window Restoration

`WindowGroup` persists the binding value for state restoration. When restoring, SwiftUI re-applies the value to `openWindow`. For custom restoration, implement `NSWindowDelegate`.

### Per-Window State with SceneStorage

```swift
struct ItemListView: View {
    @SceneStorage("selectedItemID") private var selectedItemID: String?
    @SceneStorage("sidebarWidth") private var sidebarWidth: Double = 200
}
```

`SceneStorage` is scoped per-scene (per-window). Two windows in the same `WindowGroup` have independent `SceneStorage` data. It persists across app launches.

### Avoiding Shared State Across Windows

A known issue: all windows from a `WindowGroup` may share initial values of `@State` and `@SceneState` on first creation. Use `onAppear` or `Task` to reset state on window appearance:

```swift
struct ItemView: View {
    @State private var localState: LocalState?

    var body: some View {
        Group {
            if let localState { ContentView(state: localState) }
        }
        .onAppear {
            localState = LocalState() // Ensure per-window freshness
        }
    }
}
```

Note: Apple addressed this issue in macOS 14.5+/iOS 17.5+. Ensure deployment target includes these fixes.

### Deduplicating openWindow Calls

`WindowGroup(for:)` opens a new window for each unique value. To avoid duplicates, track open window IDs:

```swift
@Observable
final class AppState {
    var openWindowIDs: Set<Item.ID> = []

    func openDetail(for id: Item.ID, openWindow: OpenWindowAction) {
        if !openWindowIDs.contains(id) {
            openWindowIDs.insert(id)
            openWindow(value: id)
        }
    }
}
```

## macOS 26 Window Customization

macOS 26 (Tahoe) extends window customization APIs:

### Customizing Window Styles

```swift
var body: some Scene {
    WindowGroup {
        ContentView()
    }
    .windowStyle(.hiddenTitleBar) // or .titleBar, .plain
    .windowResizability(.contentSize)
    .windowBackgroundDragBehavior(.enabled)
}
```

### Extended Drag Regions

Enable extended drag regions for titlebar-free windows:

```swift
.windowStyle(.hiddenTitleBar)
// Users can drag from any edge of the content area
```

### Window Toolbar Customization

```swift
.windowToolbarStyle(.expanded) // or .automatic, .compact
```

### New Control Size

macOS 26 adds an `extraLarge` control size metric for consistency:

```swift
Button("Action") { }
    .controlSize(.extraLarge)
```

### Tint Toolbar Items

```swift
.toolbar {
    ToolbarItem {
        Button("Share") { }
            .tint(.accentColor)
    }
}
```

## Dependency Injection

### EnvironmentKey for macOS Services

```swift
struct PrintingServiceKey: EnvironmentKey {
    static let defaultValue = PrintingService()
}

struct LicenseServiceKey: EnvironmentKey {
    static let defaultValue = LicenseService()
}

extension EnvironmentValues {
    var printingService: PrintingService {
        get { self[PrintingServiceKey.self] }
        set { self[PrintingServiceKey.self] = newValue }
    }

    var licenseService: LicenseService {
        get { self[LicenseServiceKey.self] }
        set { self[LicenseServiceKey.self] = newValue }
    }
}

// Inject at the root
struct MyApp: App {
    @State private var printingService = PrintingService()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.printingService, printingService)
        }
    }
}

// Consume in a leaf view
struct PrintView: View {
    @Environment(\.printingService) private var printingService

    var body: some View {
        Button("Print") { printingService.printCurrentDocument() }
    }
}
```

### Protocol-Based DI with Sendable

For concurrency safety, define service protocols as `Sendable`:

```swift
protocol PrintingServiceProtocol: Sendable {
    func printDocument(_ doc: Document) async throws
}

final class LivePrintingService: PrintingServiceProtocol { }
final class MockPrintingService: PrintingServiceProtocol { }

// Inject via environment:
@main
struct MyApp: App {
    let printingService: any PrintingServiceProtocol

    init() {
        #if DEBUG
        if ProcessInfo.processInfo.environment["XCTestConfigurationFilePath"] != nil {
            printingService = MockPrintingService()
        } else {
            printingService = LivePrintingService()
        }
        #else
        printingService = LivePrintingService()
        #endif
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.printingService, printingService)
        }
    }
}
```

### Modern DI Libraries (2026)

The Swift ecosystem now offers several lightweight DI libraries designed for SwiftUI:

| Library | Key Feature |
|---------|-------------|
| [swift-dependencies](https://github.com/pointfreeco/swift-dependencies) | Point-Free's DI system. Inspired by SwiftUI's environment. Overrides via withDependencies for tests. |
| [TinyDependency](https://github.com/fatbobman/TinyDependency) | SwiftUI-like environment API. Thread-safe with `@TaskLocal`. Three scopes: default, test, preview. |
| [Stitch](https://github.com/entrhq/stitch) | Compile-time safe. Uses macros. No codegen. `@propertyWrapper` for dependencies. |
| [Service](https://forums.swift.org/t/introducing-service-a-modern-di-library-designed-for-swift-concurrency-and-swiftui/84114) | Concurrency-first. Actor isolation support. `registerMain` for MainActor-isolated deps. |
| [DependencyInjectable](https://github.com/JamesSedlacek/DependencyInjectable) | < 50 lines of code. Understandable, minimal. |
| [kylebrowning/swift-dependency-injection](https://github.com/kylebrowning/swift-dependency-injection) | Closure-based services. Observable-friendly. |

**Choosing:** For most macOS apps, custom `EnvironmentKey` is sufficient. Use a library when you need compile-time safety (`Stitch`), test overrides (`swift-dependencies`), or cross-platform support (`Service`).

### Modern EnvironmentKey Pattern (iOS 17+/macOS 14+)

Recommended modern pattern with protocol, live + NoOp implementations:

```swift
// 1. Define the protocol
protocol AnalyticsService: Sendable {
    func track(event: String)
}

// 2. Define live, NoOp, and mock implementations
struct LiveAnalytics: AnalyticsService { }
struct NoOpAnalytics: AnalyticsService { }

@Observable
final class MockAnalytics: AnalyticsService {
    var trackedEvents: [String] = []
}

// 3. Register EnvironmentKey with NoOp default
private struct AnalyticsKey: EnvironmentKey {
    static let defaultValue: any AnalyticsService = NoOpAnalytics()
}

extension EnvironmentValues {
    var analyticsService: any AnalyticsService {
        get { self[AnalyticsKey.self] }
        set { self[AnalyticsKey.self] = newValue }
    }
}

// 4. Inject at root with live implementation
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.analyticsService, LiveAnalytics())
        }
    }
}

// 5. Swap on subtrees for scoping (A/B testing, onboarding)
struct OnboardingView: View {
    var body: some View {
        TutorialFlow()
            .environment(\.analyticsService, NoOpAnalytics()) // Suppress tracking during onboarding
    }
}
```

### NSUserDefaults Suite Sharing (App Groups)

```swift
let sharedDefaults = UserDefaults(suiteName: "group.com.example.myapp")!
```

### Testing with Injected Mocks

```swift
struct MyApp: App {
    let printingService: any PrintingServiceProtocol

    init() {
        #if DEBUG
        if ProcessInfo.processInfo.environment["XCTestConfigurationFilePath"] != nil {
            printingService = MockPrintingService()
        } else {
            printingService = PrintingService()
        }
        #else
        printingService = PrintingService()
        #endif
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.printingService, printingService)
        }
    }
}
```

## Scene-based vs Delegate-based

### Use SwiftUI `App` protocol when:
- Your app is primarily SwiftUI.
- You need multiple scene types (WindowGroup, Settings, Inspector, DocumentGroup).
- Your app lifecycle needs are covered by SwiftUI event modifiers.
- You want automatic state restoration and scene-based lifecycle.

### Use `NSApplicationDelegate` directly when:
- Your app is entirely AppKit (storyboard-based macOS app with no SwiftUI views).
- You need low-level `NSApplication` control beyond what `NSApplicationDelegateAdaptor` exposes.
- You're building a menu bar-only app (`LSUIElement`) with no dock icon.
- You subclass `NSApplication` (custom event filtering, accessibility overrides).

### Hybrid approach (recommended)

Use SwiftUI `App` + `NSApplicationDelegateAdaptor`. You get scene management from SwiftUI and delegate hooks from AppKit:

```swift
@main
struct MyApp: App {
    @NSApplicationDelegateAdaptor(AppDelegate.self) private var appDelegate

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}

final class AppDelegate: NSObject, NSApplicationDelegate {
    func applicationDockMenu(_ sender: NSApplication) -> NSMenu? { /* ... */ }
    func applicationShouldTerminate(_ sender: NSApplication) -> .TerminateReply { /* ... */ }
}
```

## SwiftUI AppKit Bridging

### NSViewRepresentable with Coordinator

The standard pattern for integrating AppKit views in SwiftUI (WWDC22):

```swift
struct ScriptEditorRepresentable: NSViewRepresentable {
    @Binding var sourceCode: String

    func makeCoordinator() -> Coordinator {
        Coordinator(sourceCode: $sourceCode)
    }

    func makeNSView(context: Context) -> ScriptEditorView {
        let view = ScriptEditorView()
        view.delegate = context.coordinator
        return view
    }

    func updateNSView(_ nsView: ScriptEditorView, context: Context) {
        if nsView.text != sourceCode {
            nsView.text = sourceCode
        }
    }

    final class Coordinator: NSObject, ScriptEditorViewDelegate {
        @Binding var sourceCode: String

        init(sourceCode: Binding<String>) {
            _sourceCode = sourceCode
        }

        func scriptEditor(_ editor: ScriptEditorView, didChangeText text: String) {
            sourceCode = text
        }
    }
}
```

### NSHostingController for Embedding SwiftUI in AppKit

```swift
let hostingController = NSHostingController(rootView: SwiftUIView(document: document))
let window = NSWindow(contentViewController: hostingController)
window.title = "Document"

// For document-based apps, propagate document changes:
controller.rootView = SwiftUIView(document: newDocument)
```

### NSHostingMenu (macOS 26+)

macOS Tahoe adds `NSHostingMenu` for using SwiftUI menus directly in AppKit:

```swift
let menu = NSHostingMenu {
    Menu("File") {
        Button("New") { newDocument() }
        Button("Open") { openDocument() }
    }
}
```

## Constraints & Gotchas

1. **`@NSApplicationDelegateAdaptor` must be declared at the `App` level**, not inside a `Scene` or `View`.

2. **`@State` with `@Observable` classes**: When passing an `@Observable` object into a view, use `@State private var vm: ViewModel` (type annotation required).

3. **`NSDocumentController` is a singleton** — `NSDocumentController.shared` manages all documents. Don't create additional instances unless in an XPC service.

4. **`WindowGroup` with a value type** opens a new window for each unique value passed to `openWindow(value:)`. Track open window IDs to avoid duplicates.

5. **`SceneStorage` has size limits** — it's not a database. Keep stored values small (< 1 KB per key).

6. **`@Environment` on macOS** only propagates within the same SwiftUI scene chain. It does NOT cross between an `NSViewController` and an `NSHostingController` unless explicitly set.

7. **`NSDocument` undo per-document**: Each `NSDocument` has its own `UndoManager`. Use `document.undoManager`, not a separate one.

8. **Order of callbacks**: `applicationWillFinishLaunching` → scenes created → `applicationDidFinishLaunching` → `applicationDidBecomeActive`.

9. **`@Observable` tracking scope**: Only properties explicitly read during a view's body are tracked. Reading a property in a button's action closure does NOT create a subscription.

10. **`DocumentGroup` vs custom `NSDocument`**: SwiftUI's `DocumentGroup` wraps `NSDocument` with a simpler protocol (`FileDocument`/`ReferenceFileDocument`). Use `DocumentGroup` for new SwiftUI-first document apps. Use custom `NSDocument` subclass for `NSDocumentController` features like duplicate, move, or locked documents.

11. **AppKit automatic recomposition** (macOS 26+): Only specific lifecycle methods participate in `@Observable` tracking. Reading properties in action methods, mouse event handlers, or `init` does NOT establish observation.

12. **Window initial state sharing**: On macOS versions before 14.5, `@State` values may be shared across newly created windows. Always use `onAppear` to ensure per-window state freshness.

## Common Mistakes

- **Using `@StateObject` with `@Observable` classes.** `@StateObject` is for `ObservableObject`. With `@Observable`, use `@State`.
- **Forgetting to call `addWindowController(_:)` in `makeWindowControllers`.** Without this, the document doesn't retain the window controller.
- **Implementing `NSMenuValidation` on a struct.** Validation requires object identity. Implement on `NSWindowController` or `NSDocument`.
- **Using `@EnvironmentObject` with `@Observable` classes.** Use `@Environment(MyClass.self)`.
- **Trying to use `@SceneStorage` for complex objects.** It supports `Bool`, `Int`, `Double`, `String`, `URL`, `Data`, and `Codable` types.
- **Setting `NSWindowController.document` after the window loads.** Always set `controller.document = self` in `makeWindowControllers`.
- **Calling `openWindow` for the same value repeatedly.** Creates duplicate windows. Guard the call.
- **Using `@Binding` directly on an `@Observable` object.** Use `@Bindable`.
- **Not implementing `applicationShouldTerminate` for unsaved-changes checks.** Users lose data on quit.
- **Treating `NSDocument` as the model.** The document is a model controller. Keep actual model data in a separate type.
- **Reading `@Observable` properties outside AppKit lifecycle methods for automatic updates.** Only designated methods participate in tracking.

## Review Checklist

- [ ] `@NSApplicationDelegateAdaptor` declared at `App` level only
- [ ] `@Observable` classes use `@State` (not `@StateObject`)
- [ ] Dependencies injected via `EnvironmentKey`/`EnvironmentValues` or a DI library
- [ ] `NSDocument.makeWindowControllers` calls `addWindowController` for each controller
- [ ] `NSDocument.document` is set on window controllers before window is shown
- [ ] `applicationShouldTerminate` implemented if there's unsaved state
- [ ] `openWindow(value:)` calls are guarded against duplicates
- [ ] `@SceneStorage` used for small per-window state; not large objects
- [ ] `@EnvironmentObject` not mixed with `@Observable` (use `@Environment`)
- [ ] `@Bindable` used instead of `@Binding` for `@Observable` instances
- [ ] Menu validation on reference type, not a struct
- [ ] Mock services injectable via environment or container for tests
- [ ] App Groups suite name consistently used across targets
- [ ] `NSDocument` subclass is a model controller with separate model type
- [ ] `@Observable` properties read in designated AppKit lifecycle methods (macOS 26+)

---

## References

- [App | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/app)
- [WindowGroup | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/windowgroup)
- [NSApplicationDelegate | Apple Developer Documentation](https://developer.apple.com/documentation/appkit/nsapplicationdelegate)
- [Adopting the Observation framework | Apple Developer Documentation](https://developer.apple.com/documentation/observation)
- [NSDocument | Apple Developer Documentation](https://developer.apple.com/documentation/appkit/nsdocument)
- [@Observable macro migration guide](https://developer.apple.com/documentation/observation/migrating-from-combine)
- [Customizing window styles in macOS](https://developer.apple.com/documentation/swiftui/customizing-window-styles-and-state-restoration-behavior-in-macos)
- [Supporting multiple windows](https://developer.apple.com/tutorials/app-dev-training/supporting-multiple-windows)
- [The Classes That Support Document-Based Apps](https://developer.apple.com/library/archive/documentation/DataManagement/Conceptual/DocBasedAppProgrammingGuideForOSX/KeyObjects/KeyObjects.html)
- [About the Cocoa Document Architecture](https://developer.apple.com/library/archive/documentation/DataManagement/Conceptual/DocBasedAppProgrammingGuideForOSX/Introduction/Introduction.html)
- [A Cleaner Architecture for Document-Based Cocoa Applications](https://medium.com/macoclock/a-cleaner-architecture-for-document-based-cocoa-applications-863b742e32f)
- [AppKit 2026 Updates](https://developer.apple.com/documentation/Updates/AppKit)
- [@State Specter: Multi-Window SwiftUI Bug](https://fatbobman.com/en/posts/the-state-specter-analyzing-a-bug-in-multi-window-swiftui-applications/)
- [SwiftUI-Style Recomposition in AppKit](https://chaosinmotion.com/2026/02/13/swiftui-style-recomposition-in-appkit-uikit/)
- [SwiftUI: How to Build Dependency Injection](https://soarias.com/swiftui/how-to-build-dependency-injection/)
- [Mastering DI in SwiftUI — DEV Community](https://dev.to/programmingcentral/mastering-dependency-injection-in-swiftui-sharing-ai-clients-with-environment-24kk)
- [pointfreeco/swift-dependencies](https://github.com/pointfreeco/swift-dependencies)
- [kylebrowning/swift-dependency-injection](https://github.com/kylebrowning/swift-dependency-injection)
- [TinyDependency](https://github.com/fatbobman/TinyDependency)
- [entrhq/stitch](https://github.com/entrhq/stitch)
- [Service: Modern DI for Swift Concurrency](https://forums.swift.org/t/introducing-service-a-modern-di-library-designed-for-swift-concurrency-and-swiftui/84114)
- [@Observable Macro Guide 2026](https://swiftcrafted.dev/article/swiftui-observable-macro-complete-guide-observation-framework)
- [WWDC22: Use SwiftUI with AppKit](https://developer.apple.com/videos/play/wwdc2022/10075/)
- [architecture-patterns.md](references/architecture-patterns.md) — Complete code recipes
