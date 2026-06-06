---
name: swiftui-macos-patterns
description: "SwiftUI state and architecture patterns as they work on macOS — @Observable, MV pattern, @State/@Binding/@Environment/@AppStorage, .task/.onChange lifecycle, @FocusedValue/@FocusedBinding/@FocusedObject for menu validation, defaultFocus/.focusedSceneValue, scene-scoped environment. Use when building macOS SwiftUI views with state management or menu command validation."
:sources: https://developer.apple.com/documentation/swiftui, https://developer.apple.com/documentation/swiftui/focused-values, https://developer.apple.com/documentation/swiftui/environmentvalues, https://developer.apple.com/documentation/SwiftUI/Migrating-from-the-observable-object-protocol-to-the-observable-macro, https://developer.apple.com/documentation/swiftui/appstorage, https://developer.apple.com/documentation/swiftui/entry()
researched: 2026-06-06
platform: macOS 26+
---

# SwiftUI State & Architecture Patterns — macOS

SwiftUI state management patterns as they work on macOS, with coverage of `@Observable`, MV architecture, all property wrappers for state ownership, lifecycle modifiers, the focused-values system for menu validation, the focus system, and scene-scoped environment injection. Targets macOS 26+ with Swift 6.3.

See [references/macos-state-patterns.md](references/macos-state-patterns.md) for deep-dive patterns: `@Observable` with `@FocusedValue` menu validation, custom `EnvironmentKey` for `NSApp` services, `@AppStorage` with App Groups, `@SceneStorage` for window restoration, and `@FocusedObject` responder-chain data flow.

## Contents

- [&Observable & MV Pattern](#observable--mv-pattern)
- [State Primitives](#state-primitives)
- [Lifecycle: .task & .onChange](#lifecycle-task--onchange)
- [Focused Values](#focused-values)
- [Focus System](#focus-system)
- [Scene-Scoped Environment](#scene-scoped-environment)
- [Settings Scene Wiring](#settings-scene-wiring)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-makes)
- [Review Checklist](#review-checklist)
- [References](#references)

**Scope boundary:** This skill covers SwiftUI state primitives, observation, lifecycle, focus, and scene-level data flow. Menu bar commands and keyboard shortcuts are in `swiftui-macos-menus-commands`. Navigation patterns (`NavigationSplitView`, `openWindow`) are in `swiftui-macos-navigation`. Window and scene configuration (`WindowGroup`, `DocumentGroup`) are in `swiftui-macos-windows-scenes`. App-level lifecycle (`NSApplicationDelegateAdaptor`, termination) is in `swift-architecture-macos`.

## @Observable & MV Pattern

The `@Observable` macro (macOS 14+) replaces `ObservableObject` + `@Published`. Every stored property is automatically tracked for observation. The compiler synthesizes the Observation framework conformance — no protocol conformance needed.

```swift
@Observable
final class DocumentStore {
    var documents: [Document] = []
    var selectedID: UUID?

    @ObservationIgnored
    private let fileCoordinator = NSFileCoordinator()

    func openDocument(at url: URL) async throws { /* ... */ }
}
```

**Ownership mapping (old → new):**

| ObservableObject era | @Observable era |
|---|---|
| `class VM: ObservableObject` | `class VM` with `@Observable` macro |
| `@Published var x` | `var x` (all stored props tracked) |
| `@StateObject private var vm` | `@State private var vm` |
| `@ObservedObject var vm` | `@Bindable var vm` |
| `@EnvironmentObject var vm` | `@Environment(VM.self) var vm` |

`@State` owns the object lifetime. `@Bindable` creates bindings for passing to child views. `@Environment` reads from the environment without owning.

```swift
struct ContentView: View {
    @State private var store = DocumentStore()

    var body: some View {
        NavigationSplitView {
            Sidebar(store: store)
        } detail: {
            DetailView(store: store)
        }
        .environment(store)
    }
}

struct Sidebar: View {
    @Bindable var store: DocumentStore

    var body: some View {
        List(store.documents, selection: $store.selectedID) { doc in
            Text(doc.name)
        }
    }
}

struct DetailView: View {
    @Environment(DocumentStore.self) private var store

    var body: some View {
        if let id = store.selectedID,
           let doc = store.documents.first(where: { $0.id == id }) {
            DocumentEditor(document: doc)
        } else {
            ContentUnavailableView("No Selection", systemImage: "doc")
        }
    }
}
```

`@ObservationIgnored` opts a property out of tracking. Use it for stable values (UUIDs, delegates, caches) to avoid unnecessary observation overhead and prevent the framework from traversing reference-type graphs.

### @Entry Macro (Custom Environment & Focused Values)

The `@Entry` macro (Xcode 16+, backward-compatible to macOS 13+) eliminates boilerplate for custom environment values. Instead of defining an `EnvironmentKey` protocol conformance, extend `EnvironmentValues` with `@Entry`:

```swift
extension EnvironmentValues {
    @Entry var windowContext: WindowContext = WindowContext(id: UUID())
}
```

This generates the private key type, getter, setter, and default value. The same macro works for four collections:

| Target | Extension | Value requirement |
|--------|------------|------------------|
| Environment values | `extension EnvironmentValues { @Entry var key: T = default }` | Non-optional with default |
| Focused values | `extension FocusedValues { @Entry var key: T? }` | Must be Optional (default nil) |
| Transaction values | `extension Transaction { @Entry var key: T = default }` | Non-optional with default |
| Container values | `extension ContainerValues { @Entry var key: T = default }` | Non-optional with default |

Read in views: `@Environment(\.windowContext)`. Set via `.environment(\.windowContext, value)`. For focused values, publish with `.focusedValue(\.key, value)` or `.focusedSceneValue(\.key, value)`.

**When to use:** Prefer `@Entry` over manual `EnvironmentKey` for all new code. The macro reduces 15+ lines to 3 and prevents copy-paste errors in key type definitions.

**Migration checklist:**
1. Remove `: ObservableObject` conformance.
2. Remove `@Published` from every property.
3. Add `@Observable` macro to the class.
4. Add `@ObservationIgnored` to non-observable stored properties.
5. Replace `@StateObject` → `@State`, `@ObservedObject` → `@Bindable`, `@EnvironmentObject` → `@Environment(Type.self)`.
6. Remove `import Combine` if no longer needed.

## State Primitives

### @State

View-local mutable state. Owned by the view; SwiftUI manages its lifetime across redraws. For value types and `@Observable` class instances.

```swift
struct SearchBar: View {
    @State private var query: String = ""
    @State private var isSearching: Bool = false

    var body: some View {
        TextField("Search", text: $query)
            .onSubmit { isSearching = true }
    }
}
```


### MVVM with @Observable (2026 Pattern)

The standard architecture for macOS SwiftUI apps in 2026 is MVVM with `@Observable` ViewModels. The ViewModel owns state and exposes intent methods; the view reads data and forwards user actions.

```swift
@Observable
final class TaskViewModel {
    var tasks: [Task] = []
    var isLoading = false
    var errorMessage: String?

    private let repository: TaskRepository

    init(repository: TaskRepository = .live) {
        self.repository = repository
    }

    func fetchTasks() async {
        isLoading = true
        defer { isLoading = false }
        do {
            tasks = try await repository.fetchAll()
        } catch {
            errorMessage = error.localizedDescription
        }
    }

    func addTask(_ task: Task) {
        tasks.append(task)
    }

    func toggleTask(_ id: UUID) {
        if let index = tasks.firstIndex(where: { $0.id == id }) {
            tasks[index].isCompleted.toggle()
        }
    }
}

struct TaskListView: View {
    @State private var viewModel = TaskViewModel()

    var body: some View {
        Group {
            if viewModel.isLoading {
                ProgressView()
            } else if let error = viewModel.errorMessage {
                Text(error).foregroundStyle(.red)
            } else {
                List(viewModel.tasks) { task in
                    TaskRow(task: task) {
                        viewModel.toggleTask(task.id)
                    }
                }
            }
        }
        .task { await viewModel.fetchTasks() }
    }
}
```

**Key rules:**
- Declare ViewModel as `@State` in the root view — it owns the lifetime.
- Pass to child views via `@Bindable` (for mutation) or `@Environment` (for shared access).
- Never mutate `viewModel.tasks` directly from a view — use intent methods.
- Inject protocols (e.g., `TaskRepository`) for testability with mock implementations.
- Computed properties (e.g., `var completedCount: Int { tasks.filter(\.isCompleted).count }`) are automatically tracked.
`@State` with `@Observable` classes: SwiftUI tracks observation of the object's properties, not the reference. Reassigning the state to a new instance triggers a view update.

### @Binding

Two-way reference to state owned elsewhere. Does not own the value — creates a `Binding<T>` to an `@State`, `@Bindable`, or other `Binding` source.

```swift
struct ToggleRow: View {
    let label: String
    @Binding var isOn: Bool

    var body: some View {
        Toggle(label, isOn: $isOn)
    }
}

// Usage:
@State private var enabled = false
ToggleRow(label: "Dark Mode", isOn: $enabled)
```

### @Environment

Reads values injected through the environment via `.environment()` or `.environment(\.key, value)`. Does not own the value.

```swift
struct ThemeAwareView: View {
    @Environment(\.colorScheme) private var colorScheme
    @Environment(DocumentStore.self) private var store

    var body: some View {
        Text(colorScheme == .dark ? "Dark" : "Light")
    }
}
```

### @AppStorage

Bridge to `UserDefaults.standard`. Automatically re-reads on changes and triggers view updates.

```swift
struct PreferencesView: View {
    @AppStorage("showLineNumbers") private var showLineNumbers = true
    @AppStorage("fontSize") private var fontSize: Double = 14.0
    @AppStorage("recentFiles") private var recentFilesData: Data = Data()

    var body: some View {
        Toggle("Show line numbers", isOn: $showLineNumbers)
        Slider(value: $fontSize, in: 10...24, step: 1) {
            Text("Font size: \(Int(fontSize))")
        }
    }
}
```

For App Groups (sharing between app and helper/XPC), use `@AppStorage("key", store: UserDefaults(suiteName: "group.com.example.app"))`.

### @SceneStorage

Per-window state that survives scene dismissal and restoration. Persisted by the scene identifier. Ideal for window-level UI state like selection, scroll position, or split-view column visibility.


### defaultAppStorage Modifier (Custom Suite)

To apply a custom `UserDefaults` suite app-wide (e.g., App Groups), use the `.defaultAppStorage(_:)` modifier on the root scene instead of passing `store:` to every `@AppStorage`:

```swift
extension UserDefaults {
    static var appGroup: UserDefaults {
        UserDefaults(suiteName: "group.com.example.app")!
    }
}

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .defaultAppStorage(.appGroup) // all @AppStorage reads/writes use this suite
    }
}
```

Individual `@AppStorage` declarations then work as usual without repeating the store:

```swift
struct PreferencesView: View {
    @AppStorage("fontSize") private var fontSize: Double = 14.0 // reads from appGroup suite
}
```

**When to use:** Prefer `.defaultAppStorage(_:)` over per-property `store:` when multiple views share the same suite. It centralizes the configuration and avoids repeating the suite name. For per-property overrides, use `@AppStorage("key", store: UserDefaults(...))`.
```swift
struct ContentView: View {
    @SceneStorage("selectedTab") private var selectedTab: String = "general"
    @SceneStorage("sidebarVisible") private var sidebarVisible: Bool = true

    var body: some View {
        NavigationSplitView(columnVisibility: $sidebarVisible) {
            Sidebar(selection: $selectedTab)
        } detail: {
            DetailPane(tab: selectedTab)
        }
    }
}
```

`@SceneStorage` is scoped to the scene that owns the view. Two windows from the same `WindowGroup` get independent `@SceneStorage` values. Data must be `RawRepresentable` and reasonably small — do not store large blobs.

## Lifecycle: .task & .onChange

### .task

Runs an async closure when the view appears, cancels it when the view disappears. Replaces `onAppear { Task { ... } }` with automatic cancellation.

```swift
struct FeedView: View {
    @State private var store = FeedStore()
    @State private var articles: [Article] = []

    var body: some View {
        List(articles) { article in
            ArticleRow(article: article)
        }
        .task {
            // Runs on appear, cancelled on disappear
            await store.refresh()
            for await update in store.articleStream {
                articles = update
            }
        }
        .task(id: store.filter) {
            // Re-runs when `store.filter` changes
            articles = await store.fetch(matching: store.filter)
        }
    }
}
```

`.task(id:)` re-runs the closure when the `id` value changes, cancelling the previous invocation. This is the primary pattern for reactive async work in SwiftUI.

### .onChange

Observes a value and runs a closure when it changes. The closure receives both old and new values (macOS 14+ / iOS 17+).

```swift
struct DocumentView: View {
    @Bindable var store: DocumentStore
    @State private var lastSaved: Date?

    var body: some View {
        TextEditor(text: $store.currentDocument.text)
            .onChange(of: store.currentDocument.text) { oldValue, newValue in
                // Debounced auto-save
                scheduleAutoSave()
            }
    }
}
```

### Scene Phase

On macOS, scene phase changes are critical for resource management. Use `@Environment(\.scenePhase)` to pause/resume work.

```swift
struct ContentView: View {
    @Environment(\.scenePhase) private var scenePhase
    @State private var monitor: FileMonitor?

    var body: some View {
        DocumentList()
            .onChange(of: scenePhase) { _, newPhase in
                switch newPhase {
                case .active:
                    monitor?.resume()
                case .inactive:
                    monitor?.pause()
                case .background:
                    monitor?.suspend()
                @unknown default:
                    break
                }
            }
    }
}
```

On macOS, `.background` means the window is not visible (e.g., hidden behind other windows or minimized into the Dock). This differs from iOS where background means the app is not in the foreground at all.

## Focused Values

The focused-values system is the SwiftUI-native way to implement menu validation on macOS. It flows data through the responder chain: a focused view declares a value, and menu commands anywhere in the scene can read it.

### @FocusedValue

Reads a value from the currently focused view. Returns `nil` if no view in the focus chain declares the value.

```swift
// 1. Define a focused value key
struct DocumentFocusedValueKey: FocusedValueKey {
    typealias Value = Document?
}

### @Entry for Focused Values

The `@Entry` macro also replaces the manual `FocusedValueKey` protocol pattern. Instead of defining a `FocusedValueKey` and extending `FocusedValues` with a computed property:

```swift
// Old pattern (still works):
struct DocumentFocusedValueKey: FocusedValueKey {
    typealias Value = Document?
}
extension FocusedValues {
    var document: DocumentFocusedValueKey.Type { DocumentFocusedValueKey.self }
}
```

Use `@Entry` directly:

```swift
extension FocusedValues {
    @Entry var document: Document?
}
```

Read with `@FocusedValue(\.document)`. Publish with `.focusedValue(\.document, value)`. The `@Entry` macro generates the `FocusedValueKey` conformance automatically.

extension FocusedValues {
    var document: DocumentFocusedValueKey.Type {
        DocumentFocusedValueKey.self
    }
}

// 2. Declare the value from a focused view
struct DocumentEditor: View {
    let document: Document
    @FocusedValue(\.document) private var focusedDocument

    var body: some View {
        TextEditor(text: .constant(document.content))
            .focusedValue(\.document, document)
    }
}

// 3. Read the value in a menu command
struct Commands: Commands {
    @FocusedValue(\.document) private var document

    var body: some Commands {
        CommandMenu("Document") {
            Button("Export…") { export(document!) }
                .disabled(document == nil)
        }
    }
}
```

### @FocusedBinding

A two-way binding to a focused value. Changes propagate back to the focused view.

```swift
struct BoldBindingKey: FocusedBindingKey {
    typealias Value = Bool
}

extension FocusedValues {
    var isBold: BoldBindingKey.Type { BoldBindingKey.self }
}

struct TextEditorView: View {
    @State private var isBold = false

    var body: some View {
        Toggle("Bold", isOn: $isBold)
            .focusedBinding(\.isBold, $isBold)
    }
}

struct Commands: Commands {
    @FocusedBinding(\.isBold) private var isBold: Bool?

    var body: some Commands {
        CommandMenu("Format") {
            Button("Toggle Bold") { isBold?.toggle() }
                .disabled(isBold == nil)
                .keyboardShortcut("b", modifiers: .command)
        }
    }
}
```

### @FocusedObject

Binds to an `@Observable` object from the focused view. Combines focused-value lookup with observation.

```swift
struct CanvasView: View {
    @State private var canvasModel = CanvasModel()

    var body: some View {
        Canvas { context, size in
            canvasModel.draw(using: context, size: size)
        }
        .focusedObject(canvasModel)
    }
}

struct Commands: Commands {
    @FocusedObject private var canvas: CanvasModel?

    var body: some Commands {
        CommandMenu("Canvas") {
            Button("Clear") { canvas?.clear() }
                .disabled(canvas == nil)
            Button("Export SVG…") { exportSVG(canvas!) }
                .disabled(canvas == nil)
        }
    }
}
```

**Key rule:** Focused values are `Optional`. Always handle `nil` — the focused view may not be in the current window, or no view may be focused at all. Menu items should be `.disabled(value == nil)` to prevent crashes.

## Focus System

### defaultFocus(_:)

Sets the initial focus target when a view appears. On macOS this determines which control receives keyboard input first.

```swift
struct LoginView: View {
    @State private var username = ""
    @State private var password = ""
    @FocusState private var focusedField: Field?

    enum Field { case username, password }

    var body: some View {
        Form {
            TextField("Username", text: $username)
                .focused($focusedField, equals: .username)
            SecureField("Password", text: $password)
                .focused($focusedField, equals: .password)
        }
        .defaultFocus($focusedField, .username)
    }
}
```

### @AccessibilityFocusState

Drives accessibility focus (VoiceOver). Separate from keyboard focus — use when you need to programmatically move VoiceOver cursor.

```swift
struct AlertBanner: View {
    @AccessibilityFocusState private var isAnnouncementFocused: Bool
    let message: String

    var body: some View {
        Text(message)
            .accessibilityFocused($isAnnouncementFocused)
            .onAppear {
                isAnnouncementFocused = true
            }
    }
}
```

### .focused(_:)

The simplest focus modifier — a boolean that reflects whether the view is focused. Use for visual styling.

```swift
struct FocusableCard: View {
    @FocusState private var isFocused: Bool

    var body: some View {
        VStack { /* ... */ }
            .padding()
            .background(isFocused ? Color.accentColor.opacity(0.2) : Color.clear)
            .focused($isFocused)
    }
}
```

### focusedSceneValue(_:_:)

Scene-scoped focused value. Unlike `@FocusedValue` which is window-scoped, `focusedSceneValue` propagates across all windows in the same scene. Use for app-wide state that should be available to menu commands regardless of which window is focused.

```swift
struct ContentView: View {
    @State private var currentProject: Project?

    var body: some View {
        ProjectBrowser(selection: $currentProject)
            .focusedSceneValue(\.project, currentProject)
    }
}

struct Commands: Commands {
    @FocusedValue(\.project) private var project

    var body: some Commands {
        CommandGroup(after: .saveItem) {
            Button("Close Project") { closeProject(project!) }
                .disabled(project == nil)
        }
    }
}
```

## Scene-Scoped Environment

Inject different environment values per window (scene) to support multi-window apps with independent services.

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(WindowContext(id: UUID()))
        }
        .windowStyle(.titleBar)

        WindowGroup(id: "inspector") {
            InspectorView()
                .environment(WindowContext(id: UUID()))
        }
    }
}

struct WindowContextKey: EnvironmentKey {
    static let defaultValue = WindowContext(id: UUID())
}

extension EnvironmentValues {
    var windowContext: WindowContext {
        get { self[WindowContextKey.self] }
        set { self[WindowContextKey.self] = newValue }
    }
}
```

Each window gets its own `WindowContext`. Child views read it via `@Environment(\.windowContext)`. This is the pattern for per-window dependency injection — different windows can have different `DocumentStore` instances, different themes, or different undo managers.

For injecting different services per window:

```swift
WindowGroup {
    ContentView()
        .environment(DocumentStore(for: .main))
}

WindowGroup(id: "auxiliary") {
    ContentView()
        .environment(DocumentStore(for: .auxiliary))
}
```

## Settings Scene Wiring

The `Settings` scene in SwiftUI creates a standard macOS preferences window. Data sharing between the main scene and Settings requires explicit wiring — they do not share environment.

```swift
@main
struct MyApp: App {
    @State private var preferences = AppPreferences()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(preferences)
        }

        Settings {
            SettingsView()
                .environment(preferences)
        }
    }
}
```

**Key point:** The `Settings` scene is a separate scene with its own environment. You must inject shared objects into both scenes independently. `@State` at the `App` level is the natural owner for shared preferences.

For `@AppStorage`-based preferences, no wiring is needed — both scenes read from the same `UserDefaults`:

```swift
struct SettingsView: View {
    @AppStorage("showStatusBar") private var showStatusBar = true
    @AppStorage("theme") private var theme: Theme = .system

    var body: some View {
        Form {
            Toggle("Show status bar", isOn: $showStatusBar)
            Picker("Theme", selection: $theme) {
                ForEach(Theme.allCases) { theme in
                    Text(theme.name).tag(theme)
                }
            }
        }
        .frame(width: 400)
    }
}
```

For `@Observable`-based preferences that need to be shared, inject the same instance into both scenes from the `App`-level `@State`.

## Constraints & Gotchas

- **`@State` with reference types:** `@State` holds a strong reference. For `@Observable` classes, the view owns the object's lifetime. If two views need the same instance, inject via `@Environment` or pass as `@Bindable` — do not create two `@State` instances of the same logical object.

- **`@SceneStorage` size limit:** Persisted per-scene. Data is archived via `NSCodable`. Keep values small — large data causes slow scene restoration. Store identifiers, not full models.

- **`@AppStorage` and `UserDefaults` synchronization:** `@AppStorage` reads from `UserDefaults.standard` by default. Changes from XPC helpers or app extensions using App Groups require explicit `UserDefaults(suiteName:)` and do not auto-sync with `@AppStorage` — you must pass the custom store.

- **Focused values are window-scoped by default:** `@FocusedValue` only sees values from the currently focused window. If the user clicks a menu while a different window is focused, the value is `nil`. Use `focusedSceneValue` for cross-window propagation within a scene.

- **`.task` cancellation is cooperative:** The closure is cancelled by throwing a cancellation error into the current `Task`. Long-running synchronous work inside `.task` will not be interrupted — use `Task.checkCancellation()` or `try await` points.

- **`.onChange` does not fire on initial value:** The first assignment does not trigger `.onChange`. If you need initial setup, do it in `.onAppear` or `.task`.

- **`@Environment` crashes on missing keys:** Unlike `@FocusedValue`, `@Environment(\.key)` with no matching `.environment()` call crashes at runtime. Always provide a default via `EnvironmentKey.defaultValue`.

- **Settings scene is a separate process space:** On macOS, `Settings` runs in the same process but as a separate scene. It does not inherit the main scene's environment. You must inject shared state explicitly.

- **`@Observable` and `Sendable`:** `@Observable` classes are not implicitly `Sendable`. If you pass them across actor boundaries, you must either mark them `@unchecked Sendable` (if thread-safe by design) or use `@MainActor`.

## Common Mistakes

- Using `@StateObject` instead of `@State` with `@Observable` classes. `@StateObject` is the old API — it still works but bypasses the Observation framework's tracking.
- Forgetting `@ObservationIgnored` on non-observable stored properties, causing the framework to traverse reference-type graphs unnecessarily.
- Storing large objects in `@SceneStorage`. Store identifiers and re-fetch from a cache.
- Not handling `nil` in `@FocusedValue` / `@FocusedBinding` / `@FocusedObject`. Menu commands crash when no view is focused.
- Assuming `@Environment` values propagate across scenes. They don't — each scene has its own environment tree.
- Using `.onChange` for initial setup instead of `.task` or `.onAppear`.
- Creating a new `@State` instance of a view model in every view that needs it, instead of sharing via `@Environment`.
- Forgetting that `.task` closures are async — trying to update UI state without `@MainActor` context.

## Review Checklist

- [ ] `@Observable` classes replace `ObservableObject` + `@Published` (no mixing)
- [ ] `@ObservationIgnored` applied to non-observable stored properties
- [ ] `@State` used for owned view models, `@Bindable` for passed-in, `@Environment` for inherited
- [ ] `@FocusedValue` / `@FocusedBinding` / `@FocusedObject` handle `nil` (menu items disabled when no focus)
- [ ] `.task` used for async work with automatic cancellation, not `onAppear { Task { } }`
- [ ] `.onChange` used for reacting to state changes, not initial setup
- [ ] `@SceneStorage` used only for small, `RawRepresentable` window state
- [ ] `@AppStorage` store parameter set for App Groups usage
- [ ] Shared state injected into both main scene and `Settings` scene independently
- [ ] `@Environment` keys have sensible `defaultValue` to prevent crashes
- [ ] `@Observable` classes that cross actor boundaries are marked `@MainActor` or `@unchecked Sendable`

## References

- [references/macos-state-patterns.md](references/macos-state-patterns.md) — Deep-dive patterns: `@Observable` with `@FocusedValue` menu validation, custom `EnvironmentKey` for `NSApp` services, `@AppStorage` with App Groups, `@SceneStorage` for window restoration, `@FocusedObject` responder-chain data flow
