# macOS State Patterns — Deep Dive

Complete Swift 6.3 code recipes for SwiftUI state management on macOS. Each pattern includes setup, wiring, and the common pitfall to avoid.

## @Observable View Model with @FocusedValue Menu Validation

The standard pattern for validating menu commands against the state of the currently focused view. This replaces the older `NSMenuItemValidation` + `validateMenuItem:` approach.

```swift
import SwiftUI
import Observation

// MARK: - View Model

@Observable
final class DocumentViewModel {
    var text: String = ""
    var selection: Range<String.Index>?
    var isModified: Bool = false

    @ObservationIgnored
    private let undoManager = UndoManager()

    func save() { /* ... */ }
    func export() { /* ... */ }

    var canSave: Bool { isModified }
    var canExport: Bool { !text.isEmpty }
}

// MARK: - Focused Value Keys

private struct DocumentViewModelFocusedKey: FocusedValueKey {
    typealias Value = DocumentViewModel
}

extension FocusedValues {
    var documentViewModel: DocumentViewModelFocusedKey.Type {
        DocumentViewModelFocusedKey.self
    }
}

// MARK: - Focused View

struct DocumentEditorView: View {
    @Bindable var viewModel: DocumentViewModel

    var body: some View {
        TextEditor(text: $viewModel.text)
            .focusedValue(\.documentViewModel, viewModel)
    }
}

// MARK: - Menu Commands

struct DocumentCommands: Commands {
    @FocusedValue(\.documentViewModel) private var viewModel

    var body: some Commands {
        CommandMenu("Document") {
            Button("Save") {
                viewModel?.save()
            }
            .keyboardShortcut("s", modifiers: .command)
            .disabled(viewModel == nil || !viewModel!.canSave)

            Button("Export…") {
                viewModel?.export()
            }
            .keyboardShortcut("e", modifiers: [.command, .shift])
            .disabled(viewModel == nil || !viewModel!.canExport)

            Divider()

            Button("Close") {
                viewModel?.save()
            }
            .keyboardShortcut("w", modifiers: .command)
            .disabled(viewModel == nil)
        }
    }
}

// MARK: - App Assembly

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(WindowController())
        }
        .commands {
            DocumentCommands()
        }
    }
}
```

**Gotcha:** `viewModel!` in the menu commands — force-unwrap is safe here because `.disabled(viewModel == nil)` prevents the button from being tappable when `nil`. But a cleaner approach uses a helper:

```swift
private var isDocumentActive: Bool {
    viewModel != nil
}
```

## Custom EnvironmentKey for NSApp Services

Inject a service object (like a `WindowManager`) into the SwiftUI environment so any view or command can access it without singleton references.

```swift
import SwiftUI
import AppKit

// MARK: - Service

@Observable
@MainActor
final class WindowManager {
    var openWindows: [NSWindow] = []

    func openInspector() {
        guard let window = NSWindow(contentRect: NSRect(x: 0, y: 0, width: 400, height: 600),
                                     styleMask: [.titled, .closable, .resizable],
                                     backing: .buffered,
                                     defer: false) else { return }
        window.title = "Inspector"
        window.makeKeyAndOrderFront(nil)
        openWindows.append(window)
    }

    func closeAllInspectors() {
        openWindows.filter { $0.title == "Inspector" }.close()
        openWindows.removeAll { $0.title == "Inspector" }
    }
}

// MARK: - Environment Key

private struct WindowManagerKey: EnvironmentKey {
    static let defaultValue = WindowManager()
}

extension EnvironmentValues {
    var windowManager: WindowManager {
        get { self[WindowManagerKey.self] }
        set { self[WindowManagerKey.self] = newValue }
    }
}

// MARK: - Usage in Views

struct ContentView: View {
    @Environment(\.windowManager) private var windowManager

    var body: some View {
        VStack {
            Button("Show Inspector") {
                windowManager.openInspector()
            }
            Text("\(windowManager.openWindows.count) windows open")
        }
        .padding()
    }
}

// MARK: - Usage in Commands

struct WindowCommands: Commands {
    @Environment(\.windowManager) private var windowManager

    var body: some Commands {
        CommandGroup(after: .windowArrangement) {
            Button("Inspector") {
                windowManager.openInspector()
            }
            .keyboardShortcut("i", modifiers: [.command, .option])

            Button("Close All Inspectors") {
                windowManager.closeAllInspectors()
            }
            .disabled(windowManager.openWindows.isEmpty)
        }
    }
}

// MARK: - App Assembly

@main
struct MyApp: App {
    @State private var windowManager = WindowManager()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.windowManager, windowManager)
        }
        .commands {
            WindowCommands()
        }
    }
}
```

**Gotcha:** The `WindowManager` is `@MainActor` because it touches `NSWindow`. SwiftUI environment injection respects the actor context — reading `@Environment(\.windowManager)` from a background thread will crash. Ensure commands and views accessing the service run on the main actor.

## @AppStorage with App Groups Suite

Share `UserDefaults` between a main app, XPC helper, or app extension using an App Groups suite. This requires the App Groups entitlement on both targets.

```swift
import SwiftUI

// MARK: - Shared Defaults

extension UserDefaults {
    static let appGroup: UserDefaults = {
        guard let suite = UserDefaults(suiteName: "group.com.example.myapp") else {
            fatalError("App group suite not available — check entitlements")
        }
        return suite
    }()
}

// MARK: - Preference Keys

enum PreferenceKey: String, CaseIterable {
    case showLineNumbers = "showLineNumbers"
    case fontSize = "fontSize"
    case lastOpenedFile = "lastOpenedFile"
    case syncEnabled = "syncEnabled"

    var defaultValue: Any {
        switch self {
        case .showLineNumbers: true
        case .fontSize: 14.0
        case .lastOpenedFile: ""
        case .syncEnabled: false
        }
    }

    func registerDefault() {
        UserDefaults.appGroup.register(defaults: [rawValue: defaultValue])
    }
}

// MARK: - Type-Safe Wrapper

@propertyWrapper
struct AppGroupStorage<T> {
    let key: String

    var wrappedValue: T {
        get { UserDefaults.appGroup.object(forKey: key) as? T ?? PreferenceKey(rawValue: key)!.defaultValue as! T }
        set { UserDefaults.appGroup.set(newValue, forKey: key) }
    }

    init(_ key: PreferenceKey) {
        self.key = key.rawValue
    }
}

// MARK: - View Model

@Observable
final class PreferencesViewModel {
    @AppGroupStorage(.showLineNumbers) var showLineNumbers: Bool
    @AppGroupStorage(.fontSize) var fontSize: Double
    @AppGroupStorage(.syncEnabled) var syncEnabled: Bool

    init() {
        // Register defaults on first use
        PreferenceKey.allCases.forEach { $0.registerDefault() }
    }
}

// MARK: - SwiftUI View (uses standard @AppStorage with custom store)

struct PreferencesView: View {
    // Standard @AppStorage with custom store for views that need Binding
    @AppStorage("showLineNumbers", store: .appGroup) private var showLineNumbers = true
    @AppStorage("fontSize", store: .appGroup) private var fontSize: Double = 14.0
    @AppStorage("syncEnabled", store: .appGroup) private var syncEnabled = false

    var body: some View {
        Form {
            Toggle("Show line numbers", isOn: $showLineNumbers)

            HStack {
                Text("Font size: \(Int(fontSize))")
                Slider(value: $fontSize, in: 8...32, step: 1)
            }

            Toggle("Enable sync", isOn: $syncEnabled)
        }
        .frame(width: 350)
        .fixedSize()
    }
}

// MARK: - XPC Helper Writing to Same Suite

// In a helper/XPC service:
class SyncHelper: NSObject, NSXPCListenerDelegate {
    func enableSync() {
        UserDefaults.appGroup.set(true, forKey: "syncEnabled")
        UserDefaults.appGroup.synchronize() // Force immediate write
    }

    func readPreferences() -> (showLineNumbers: Bool, fontSize: Double) {
        let defaults = UserDefaults(suiteName: "group.com.example.myapp")!
        return (
            defaults.bool(forKey: "showLineNumbers"),
            defaults.double(forKey: "fontSize")
        )
    }
}
```

**Gotcha:** `@AppStorage` with a custom store does not auto-sync across process boundaries. If the XPC helper writes a value, the SwiftUI view won't see it until the next read cycle. For cross-process reactivity, use `DistributedNotificationCenter` or `CFNotificationCenter` to broadcast changes:

```swift
// In the XPC helper after writing:
DistributedNotificationCenter.default().postNotificationName(
    .init("com.example.myapp.preferencesChanged"),
    object: nil,
    deliverImmediately: true
)

// In the main app view model:
.onReceive(NotificationCenter.default.publisher(for: .init("com.example.myapp.preferencesChanged"))) { _ in
    // Force re-read from UserDefaults
    self.showLineNumbers = UserDefaults.appGroup.bool(forKey: "showLineNumbers")
}
```

## @SceneStorage for Window Restoration

Restore window-level UI state (selection, scroll position, column visibility) across app launches using `@SceneStorage`. Each window gets independent state.

```swift
import SwiftUI

// MARK: - Document Model

struct Document: Identifiable, Codable {
    let id: UUID
    var name: String
    var content: String
}

// MARK: - View Model

@Observable
final class LibraryViewModel {
    var documents: [Document] = []
    var filter: String = ""

    var filteredDocuments: [Document] {
        guard !filter.isEmpty else { return documents }
        return documents.filter { $0.name.localizedCaseInsensitiveContains(filter) }
    }

    func addDocument() {
        let doc = Document(id: UUID(), name: "Untitled", content: "")
        documents.append(doc)
    }
}

// MARK: - Content View with SceneStorage

struct ContentView: View {
    @State private var library = LibraryViewModel()

    // Per-window persisted state
    @SceneStorage("selectedDocumentID") private var selectedDocumentID: String?
    @SceneStorage("sidebarVisible") private var sidebarVisible = true
    @SceneStorage("searchText") private var searchText = ""

    var body: some View {
        NavigationSplitView(columnVisibility: $sidebarVisible) {
            DocumentList(
                documents: library.filteredDocuments,
                selection: Binding(
                    get: { selectedDocumentID.flatMap(UUID.init) },
                    set: { selectedDocumentID = $0?.uuidString }
                )
            )
            .searchable(text: $searchText)
            .navigationSplitViewColumnWidth(min: 200, ideal: 250)
        } detail: {
            if let id = selectedDocumentID.flatMap(UUID.init),
               let document = library.documents.first(where: { $0.id == id }) {
                DocumentEditor(document: document)
            } else {
                ContentUnavailableView("No Document Selected",
                    systemImage: "doc.text",
                    description: Text("Select a document from the sidebar"))
            }
        }
        .onAppear {
            if library.documents.isEmpty {
                library.addDocument()
                library.addDocument()
            }
        }
    }
}

struct DocumentList: View {
    let documents: [Document]
    @Binding var selection: UUID?

    var body: some View {
        List(documents, selection: $selection) { doc in
            Label(doc.name, systemImage: "doc.text")
                .tag(doc.id)
        }
    }
}

struct DocumentEditor: View {
    let document: Document

    var body: some View {
        TextEditor(text: .constant(document.content))
            .font(.system(.body, design: .monospaced))
    }
}

// MARK: - App

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .windowStyle(.titleBar)
        .defaultSize(width: 900, height: 600)
    }
}
```

**Gotcha:** `@SceneStorage` stores values as strings (for `String`-backed keys). For `UUID`, store the `uuidString` and convert back. For complex types, use `Codable` + `Data`:

```swift
@SceneStorage("scrollPosition") private var scrollPositionData: Data?

var scrollPosition: CGPoint? {
    get {
        guard let data = scrollPositionData else { return nil }
        return try? JSONDecoder().decode(CGPoint.self, from: data)
    }
    set {
        scrollPositionData = try? JSONEncoder().encode(newValue)
    }
}
```

## @FocusedObject for Responder Chain Data Flow

Use `@FocusedObject` to connect an `@Observable` object between a focused view and a command or toolbar. This is the preferred pattern over `@FocusedValue` when you need observation (not just a snapshot).

```swift
import SwiftUI
import Observation

// MARK: - Canvas Model

@Observable
final class CanvasModel {
    var shapes: [CanvasShape] = []
    var selectedShapeID: UUID?
    var zoom: Double = 1.0
    var canvasSize: CGSize = .zero

    var canUndo: Bool { undoManager.canUndo }
    var canRedo: Bool { undoManager.canRedo }

    @ObservationIgnored
    let undoManager = UndoManager()

    func addShape(_ shape: CanvasShape) {
        undoManager.registerUndo(withTarget: self) { target in
            target.shapes.removeAll { $0.id == shape.id }
        }
        shapes.append(shape)
    }

    func deleteSelected() {
        guard let id = selectedShapeID else { return }
        shapes.removeAll { $0.id == id }
        selectedShapeID = nil
    }

    func clearAll() {
        shapes.removeAll()
        selectedShapeID = nil
        undoManager.removeAllActions()
    }

    func undo() { undoManager.undo() }
    func redo() { undoManager.redo() }

    func draw(using context: GraphicsContext, size: CGSize) {
        canvasSize = size
        for shape in shapes {
            shape.draw(in: context, isSelected: shape.id == selectedShapeID)
        }
    }
}

// MARK: - Canvas Shape Model

struct CanvasShape: Identifiable, Equatable {
    let id = UUID()
    var path: Path
    var fillColor: Color
    var strokeColor: Color
    var lineWidth: CGFloat

    func draw(in context: GraphicsContext, isSelected: Bool) {
        context.fill(path, with: .color(fillColor))
        context.stroke(path, with: .color(strokeColor), lineWidth: lineWidth)
        if isSelected {
            context.stroke(path, with: .color(.blue), style: StrokeStyle(lineWidth: 2, dash: [5]))
        }
    }
}

// MARK: - Canvas View

struct CanvasView: View {
    @Bindable var model: CanvasModel

    var body: some View {
        Canvas { context, size in
            model.draw(using: context, size: size)
        }
        .gesture(
            TapGesture()
                .onEnded { _ in
                    model.selectedShapeID = nil
                }
        )
        .focusedObject(model)
    }
}

// MARK: - Commands

struct CanvasCommands: Commands {
    @FocusedObject private var canvas: CanvasModel?

    var body: some Commands {
        CommandMenu("Canvas") {
            Button("Add Circle") {
                let shape = CanvasShape(
                    path: Path(ellipseIn: CGRect(x: 50, y: 50, width: 100, height: 100)),
                    fillColor: .blue.opacity(0.3),
                    strokeColor: .blue,
                    lineWidth: 2
                )
                canvas?.addShape(shape)
            }
            .keyboardShortcut("o", modifiers: [.command, .option])
            .disabled(canvas == nil)

            Divider()

            Button("Delete") {
                canvas?.deleteSelected()
            }
            .keyboardShortcut(.delete, modifiers: [])
            .disabled(canvas?.selectedShapeID == nil)

            Button("Clear All") {
                canvas?.clearAll()
            }
            .disabled(canvas?.shapes.isEmpty ?? true)

            Divider()

            Button("Undo") {
                canvas?.undo()
            }
            .keyboardShortcut("z", modifiers: .command)
            .disabled(!(canvas?.canUndo ?? false))

            Button("Redo") {
                canvas?.redo()
            }
            .keyboardShortcut("z", modifiers: [.command, .shift])
            .disabled(!(canvas?.canRedo ?? false))
        }
    }
}

// MARK: - Assembly

struct ContentView: View {
    @State private var canvasModel = CanvasModel()

    var body: some View {
        NavigationSplitView {
            ShapeListView(model: canvasModel)
        } detail: {
            CanvasView(model: canvasModel)
        }
    }
}

struct ShapeListView: View {
    @Bindable var model: CanvasModel

    var body: some View {
        List(model.shapes, selection: $model.selectedShapeID) { shape in
            Label("Shape", systemImage: "circle.fill")
                .foregroundStyle(shape.fillColor)
                .tag(shape.id)
        }
    }
}

@main
struct CanvasApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .commands {
            CanvasCommands()
        }
        .defaultSize(width: 1000, height: 700)
    }
}
```

**Pattern note:** `@FocusedObject` combines two things: (1) focused-value lookup via the responder chain, and (2) `@Observable` property tracking. When the `CanvasModel` changes (shapes added, undo stack changes), the `@FocusedObject` in `CanvasCommands` re-evaluates — menu items correctly enable/disable based on the latest state. This is an improvement over pure `@FocusedValue` which only provides a snapshot.

**Gotcha order:** The three focused variants serve different purposes:

| Pattern | Use when | Observation |
|---|---|---|
| `@FocusedValue` | You need read-only access to a value | No — snapshot |
| `@FocusedBinding` | You need two-way binding to a single value | No — you manage writes |
| `@FocusedObject` | You need to observe an entire view model | Yes — `@Observable` tracking |

Choose the simplest one that fits. Don't use `@FocusedObject` for a single `Bool` — use `@FocusedBinding` instead.
