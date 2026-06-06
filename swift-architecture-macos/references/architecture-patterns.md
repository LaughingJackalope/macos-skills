# Architecture Patterns — macOS

Complete code recipes for the patterns described in the `swift-architecture-macos` skill. Each section is a self-contained, compilable pattern.

## Complete NSDocument Subclass with Window Controllers

A full `NSDocument` subclass that bridges to SwiftUI, manages multiple window controllers, and supports autosave:

```swift
import Cocoa
import SwiftUI

// MARK: - Document Model

struct DocumentContent: Codable {
    var text: String = ""
    var tags: [String] = []
    var createdAt: Date = .init()
}

// MARK: - NSDocument

final class TextDocument: NSDocument {
    var content = DocumentContent()

    // Enable autosave and file coordination
    override class var autosavesInPlace: Bool { true }
    override class var isNativeType: Bool { true }

    // MARK: - Window Controller Management

    override func makeWindowControllers() {
        // Remove any stale window controllers (e.g., during document duplication)
        removeWindowControllers()

        // Primary editor window
        let editorWC = makeEditorWindowController()
        editorWC.document = self
        addWindowController(editorWC)

        // Document properties panel (secondary window)
        let propertiesWC = makePropertiesWindowController()
        propertiesWC.document = self
        addWindowController(propertiesWC)
    }

    private func makeEditorWindowController() -> NSWindowController {
        let rootView = DocumentEditorView(document: self)
        let hosting = NSHostingController(rootView: rootView)

        let window = NSWindow(contentViewController: hosting)
        window.title = "Untitled"
        window.setContentSize(NSSize(width: 800, height: 600))
        window.isRestorable = true

        let controller = NSWindowController(window: window)
        controller.windowFrameAutosaveName = "TextEditorWindow"
        return controller
    }

    private func makePropertiesWindowController() -> NSWindowController {
        let rootView = PropertiesPanelView(document: self)
        let hosting = NSHostingController(rootView: rootView)

        let window = NSWindow(contentViewController: hosting)
        window.title = "Properties"
        window.setContentSize(NSSize(width: 300, height: 400))
        window.isRestorable = true

        let controller = NSWindowController(window: window)
        controller.windowFrameAutosaveName = "PropertiesWindow"
        return controller
    }

    private func removeWindowControllers() {
        windowControllers.forEach { removeWindowController($0) }
    }

    // MARK: - Data Serialization

    override func data(ofType typeName: String) throws -> Data {
        content.createdAt = .init()
        return try JSONEncoder().encode(content)
    }

    override func read(from data: Data, ofType typeName: String) throws {
        content = try JSONDecoder().decode(DocumentContent.self, from: data)
        // Notify window controllers that the content changed
        windowControllers.forEach { wc in
            (wc.window?.contentViewController as? NSHostingController<DocumentEditorView>)?
                .rootView = DocumentEditorView(document: self)
        }
    }

    // MARK: - Autosave

    override func autosave(withImplicitCancellability autosavingIsImplicitlyCancellable: Bool, completionHandler: @escaping (Error?) -> Void) {
        do {
            let data = try data(ofType: fileType ?? "public.json")
            try writeToAutosavedFile(data: data)
            completionHandler(nil)
        } catch {
            completionHandler(error)
        }
    }

    // MARK: - Termination

    override func canClose(withDelegate delegate: Any, shouldClose shouldCloseSelector: Selector?, contextInfo: UnsafeMutableRawPointer?) {
        if isDraft || !isDocumentEdited {
            super.canClose(withDelegate: delegate, shouldClose: shouldCloseSelector, contextInfo: contextInfo)
            return
        }
        // Return control to the system to present the save/don't save/cancel alert
        super.canClose(withDelegate: delegate, shouldClose: shouldCloseSelector, contextInfo: contextInfo)
    }

    // MARK: - Menu Validation

    override func validateUserInterfaceItem(_ item: NSValidatedUserInterfaceItem) -> Bool {
        switch item.action {
        case #selector(duplicate(_:)):
            return true
        case #selector(moveToTrash):
            return !isDraft
        default:
            return super.validateUserInterfaceItem(item)
        }
    }
}

// MARK: - SwiftUI Views (document parameter)

struct DocumentEditorView: View {
    weak var document: TextDocument?

    private var content: Binding<DocumentContent> {
        Binding(
            get: { document?.content ?? DocumentContent() },
            set: { newValue in
                document?.content = newValue
                document?.updateChangeCount(.changeDone)
            }
        )
    }

    var body: some View {
        VStack {
            TextEditor(text: content.text)
                .font(.body)

            HStack {
                Text("Tags:")
                ForEach(content.tags.wrappedValue, id: \.self) { tag in
                    Text(tag)
                        .padding(4)
                        .background(.quaternary, in: RoundedRectangle(cornerRadius: 4))
                }
            }
        }
        .padding()
    }
}

struct PropertiesPanelView: View {
    weak var document: TextDocument?

    var body: some View {
        if let document {
            Form {
                Section("Metadata") {
                    Text("Created: \(document.content.createdAt, style: .date)")
                    Text("Tags: \(document.content.tags.count)")
                }
            }
            .padding()
        } else {
            Text("No document")
        }
    }
}
```

The `weak var document` in the SwiftUI views prevents a retain cycle: the document holds the window controller, the window controller holds the hosting controller, the hosting controller holds the root view. A strong reference back from the view to the document would prevent deallocation. The `content` binding uses `get`/`set` with `updateChangeCount(.changeDone)` so the document system tracks whether there are unsaved changes.

## @Observable View Model with macOS-Specific Pattern

A view model that integrates with `NSMenuItemValidation`, handles async operations, and uses the Observation framework:

```swift
import Observation
import AppKit

@Observable
@MainActor
final class LibraryViewModel: NSMenuItemValidation {
    // Tracked properties (observation is automatic)
    var items: [LibraryItem] = []
    var selectedItemID: UUID?
    var isLoading = false
    var errorMessage: String?

    // Not observed — stable identity
    @ObservationIgnored
    private let repository: ItemRepository

    @ObservationIgnored
    private var loadTask: Task<Void, Never>?

    init(repository: ItemRepository = .live) {
        self.repository = repository
    }

    // MARK: - Actions

    func load() {
        loadTask?.cancel()
        isLoading = true
        errorMessage = nil

        loadTask = Task { @MainActor [weak self] in
            guard let self else { return }
            do {
                let fetched = try await repository.fetchAll()
                guard !Task.isCancelled else { return }
                self.items = fetched
            } catch {
                guard !Task.isCancelled else { return }
                self.errorMessage = error.localizedDescription
            }
            self.isLoading = false
        }
    }

    func deleteSelected() {
        guard let id = selectedItemID else { return }
        items.removeAll { $0.id == id }
        selectedItemID = nil

            Task {
                try? await repository.delete(id: id)
        }
    }

    func renameSelected(to newName: String) {
        guard let index = items.firstIndex(where: { $0.id == selectedItemID }) else { return }
        items[index].name = newName

            Task {
                try? await repository.update(items[index])
        }
    }

    // MARK: - NSMenuItemValidation

    func validateMenuItem(_ menuItem: NSMenuItem) -> Bool {
        switch menuItem.action {
        case #selector(delete(_:)),
             #selector(rename(_:)),
             #selector(exportSelected(_:)):
            return selectedItemID != nil && !isLoading
        case #selector(refresh(_:)):
            return !isLoading
        case #selector(duplicateSelected(_:)):
            guard let id = selectedItemID,
                  let item = items.first(where: { $0.id == id }) else { return false }
            return item.isDuplicatable
        default:
            return true
        }
    }

    // MARK: - Menu Actions

    @objc func delete(_ sender: Any?) { deleteSelected() }
    @objc func rename(_ sender: Any?) { /* Show rename dialog */ }
    @objc func exportSelected(_ sender: Any?) { /* Export logic */ }
    @objc func refresh(_ sender: Any?) { load() }
    @objc func duplicateSelected(_ sender: Any?) { /* Duplicate logic */ }
}

// Usage in an NSWindowController that sets itself as the menu target:
extension LibraryWindowController: NSMenuItemValidation {
    func validateMenuItem(_ menuItem: NSMenuItem) -> Bool {
        // Delegate to the view model
        viewModel.validateMenuItem(menuItem)
    }
}
```

The `@MainActor` annotation ensures all mutations happen on the main thread. `@ObservationIgnored` on `repository` and `loadTask` prevents the observation framework from traversing into the repository's properties (which may not be `@Observable` themselves). The `[weak self]` capture in async tasks prevents retain cycles when the view model outlives the task. Validation selectively enables/disables menu items based on the current application state — this is the primary pattern for menu item management in document-based macOS apps.

## Dependency Injection Container for macOS

A service locator and environment-based hybrid. Use a container for composition at the root, then inject into SwiftUI via `EnvironmentKey`:

```swift
import SwiftUI
import Foundation

// MARK: - Services

protocol ItemRepositoryProtocol: Sendable {
    func fetchAll() async throws -> [LibraryItem]
    func delete(id: UUID) async throws
    func update(_ item: LibraryItem) async throws
}

protocol LoggingServiceProtocol: Sendable {
    func log(_ message: String, level: LogLevel)
}

enum LogLevel: String { case debug, info, error }

// MARK: - Live Implementations

final class LiveItemRepository: ItemRepositoryProtocol {
    func fetchAll() async throws -> [LibraryItem] {
        // Core Data / network / file system
        []
    }
    func delete(id: UUID) async throws { /* ... */ }
    func update(_ item: LibraryItem) async throws { /* ... */ }
}

final class LiveLoggingService: LoggingServiceProtocol {
    func log(_ message: String, level: LogLevel) {
        print("[\(level.rawValue)] \(message)")
    }
}

// MARK: - Container

@MainActor
final class DependencyContainer {
    private var services: [String: Any] = [:]

    static let shared = DependencyContainer()

    func register<T>(_ type: T.Type, factory: @escaping () -> T) {
        let key = String(describing: type)
        services[key] = factory
    }

    func registerInstance<T>(_ instance: T) {
        let key = String(describing: type(of: instance))
        services[key] = instance
    }

    func resolve<T>(_ type: T.Type) -> T? {
        let key = String(describing: type)
        if let factory = services[key] as? () -> T {
            return factory()
        }
        return services[key] as? T
    }
}

extension DependencyContainer {
    static func live() -> DependencyContainer {
        let container = DependencyContainer()
        container.register(ItemRepositoryProtocol.self) { LiveItemRepository() }
        container.register(LoggingServiceProtocol.self) { LiveLoggingService() }
        return container
    }
}

// MARK: - Environment Keys

struct ItemRepositoryKey: EnvironmentKey {
    static let defaultValue: ItemRepositoryProtocol? = nil
}

struct LoggingServiceKey: EnvironmentKey {
    static let defaultValue: LoggingServiceProtocol? = nil
}

extension EnvironmentValues {
    var itemRepository: ItemRepositoryProtocol? {
        get { self[ItemRepositoryKey.self] }
        set { self[ItemRepositoryKey.self] = newValue }

    var loggingService: LoggingServiceProtocol? {
        get { self[LoggingServiceKey.self] }
        set { self[LoggingServiceKey.self] = newValue }
    }
}

// MARK: - Composition Root

@main
struct MyApp: App {
    // Create container at app level — lives as long as the app
    private let container = DependencyContainer.live()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environment(\.itemRepository, container.resolve(ItemRepositoryProtocol.self))
                .environment(\.loggingService, container.resolve(LoggingServiceProtocol.self))
        }
    }
}

// MARK: - Consumption in Views

struct ItemsView: View {
    @Environment(\.itemRepository) private var repository
    @Environment(\.loggingService) private var logger
    @State private var viewModel: LibraryViewModel?

    var body: some View {
        Group {
            if let viewModel {
                List(selection: $viewModel.selectedItemID) {
                    ForEach(viewModel.items) { item in
                        Text(item.name).tag(item.id)
                    }
                }
                .onAppear { viewModel.load() }
            } else {
                ProgressView()
            }
        }
        .task {
            // Create view model once services are available
            guard let repo = repository else {
                logger?.log("ItemRepository not injected", level: .error)
                return
            }
            viewModel = await LibraryViewModel(repository: repo)
        }
    }
}

// MARK: - Testing

#if DEBUG
final class StubItemRepository: ItemRepositoryProtocol {
    var stubbedItems: [LibraryItem] = [.mock(), .mock()]
    var fetchCallCount = 0

    func fetchAll() async throws -> [LibraryItem] {
        fetchCallCount += 1
        return stubbedItems
    }
    func delete(id: UUID) async throws {}
    func update(_ item: LibraryItem) async throws {}
}

final class SpyLoggingService: LoggingServiceProtocol {
    var messages: [(String, LogLevel)] = []
    func log(_ message: String, level: LogLevel) {
        messages.append((message, level))
    }
}

// In tests:
class LibraryViewModelTests: XCTestCase {
    @MainActor
    func testLoadFetchesItems() async throws {
        let stub = StubItemRepository()
        let spy = SpyLoggingService()
        let viewModel = LibraryViewModel(repository: stub)

        viewModel.load()
        try await Task.sleep(nanoseconds: 100_000_000) // Let async complete

        XCTAssertEqual(stub.fetchCallCount, 1)
        XCTAssertEqual(viewModel.items.count, 2)
        XCTAssertFalse(viewModel.isLoading)
    }
}
#endif
```

The container pattern separates construction from usage. Services are registered once at the compositional root (`myApp`) and flow through the environment. The `resolve` method is lightweight — it's a dictionary lookup and optional factory invocation. For testing, register `StubItemRepository` in the test container's `DependencyContainer()` initializer, or directly create a `LibraryViewModel(repository: StubItemRepository())` which is simpler for unit tests. The `@Environment` wrapper on `itemRepository` and `loggingService` is optional in tests: you can construct views with `.environment(\.itemRepository, stub)` without touching the container at all.

## Multi-Window State Restoration Pattern

A complete pattern for managing per-window state with `SceneStorage`, custom `NSWindowRestoration`, and coordinated state between the app-level view model and per-window view:

```swift
import SwiftUI

// MARK: - App-Level State (shared across all windows)

@Observable
@MainActor
final class AppState {
    var documents: [DocumentRef] = []
    var activeDocumentID: UUID? = nil

    struct DocumentRef: Identifiable, Codable {
        let id: UUID
        var title: String
        init(id: UUID = UUID(), title: String = "Untitled") {
            self.id = id
            self.title = title
        }
    }
}

// MARK: - Per-Window State

struct WindowState: Codable {
    var sidebarVisible = true
    var sidebarWidth: Double = 220
    var searchQuery = ""
    var selectedItemID: UUID? = nil
}

// MARK: - App Entry Point

@main
struct MyApp: App {
    @State private var appState = AppState()

    var body: some Scene {
        WindowGroup("Document", for: AppState.DocumentRef.ID.self) { $documentID in
            DocumentWindow(documentID: documentID ?? UUID())
                .environment(appState)
        }
        .commands {
            CommandGroup(after: .newItem) {
                Button("New Document") {
                    let newRef = AppState.DocumentRef(title: "Untitled \(appState.documents.count + 1)")
                    appState.documents.append(newRef)
                    NSApp.sendAction(Selector(("showWindow:")), to: nil, from: nil)
                }
                .keyboardShortcut("n", modifiers: [.command, .shift])
            }
        }

        // Main window (no argument — always exists)
        WindowGroup("Overview") {
            DocumentOverviewView()
                .environment(appState)
        }
    }
}

// MARK: - Per-Window View with SceneStorage

struct DocumentWindow: View {
    @Environment(AppState.self)
    private var appState

    @SceneStorage("windowState")
    private var windowState = WindowState()

    @SceneStorage("documentID")
    private var storedDocumentID: String?

    let documentID: UUID

    var body: some View {
        NavigationSplitView(columnVisibility: Binding(
            get: { windowState.sidebarVisible ? .all : .detailOnly },
            set: { windowState.sidebarVisible = ($0 == .all) }
        )) {
            SidebarView(
                searchQuery: $windowState.searchQuery,
                selection: $windowState.selectedItemID
            )
            .navigationSplitViewColumnWidth(
                ideal: windowState.sidebarWidth,
                min: 150,
                max: 400
            )
        } detail: {
            DetailView(itemID: windowState.selectedItemID)
        }
        .onAppear {
            // Sync stored ID for restoration debugging
            storedDocumentID = documentID.uuidString

            // Mark this document as active in shared state
            appState.activeDocumentID = documentID
        }
        .onChange(of: documentID) { _, newID in
            appState.activeDocumentID = newID
        }
    }
}

// MARK: - Window Restoration via NSDocument (for DocumentGroup)

// When using DocumentGroup, SwiftUI calls your Document's `makeWindowControllers` or `fileSnapshot` automatically.
// For WindowGroup-with-value, you handle restoration through these steps:
//
// 1. SceneStorage persists lightweight per-window state (selections, widths, search queries).
// 2. The WindowGroup `for: ID.self` clause means SwiftUI restores *one window per unique ID* that was
//    last open. You must track open-window IDs in your app state and re-invoke `openWindow(value:)`
//    in `AppDelegate.applicationDidFinishLaunching` for each ID that should be restored.
//
// Store open window IDs in UserDefaults or file:

extension AppDelegate: NSApplicationDelegate {
    func applicationDidFinishLaunching(_ notification: Notification) {
        // Restore open windows from saved state
        let savedWindowIDs = UserDefaults.standard.stringArray(forKey: "openWindowIDs") ?? []
        for idString in savedWindowIDs {
            if let uuid = UUID(uuidString: idString) {
                // Post notification so the SwiftUI app can call openWindow
                NotificationCenter.default.post(
                    name: .openWindowForID,
                    object: uuid
                )
            }
        }
    }
}

extension Notification.Name {
    static let openWindowForID = Notification.Name("openWindowForID")
}

// In your SwiftUI view or app struct:
struct WindowRestorationHandler: View {
    @Environment(\.openWindow) private var openWindow
    @Environment(AppState.self) private var appState

    var body: some View {
        EmptyView()
            .onReceive(NotificationCenter.default.publisher(for: .openWindowForID)) { notification in
                guard let uuid = notification.object as? UUID,
                      let doc = appState.documents.first(where: { $0.id == uuid }) else { return }
                openWindow(value: doc.id)
            }
    }
}
```

The key architectural decision is what gets stored where. `SceneStorage` holds per-window UI state (selections, layout, search queries) — small, codable, per-window. `UserDefaults` or a plist holds the list of open window identifiers. The `@Observable AppState` holds the shared model (document list, active document). When the app launches, `applicationDidFinishLaunching` reads the saved window IDs and triggers `openWindow(value:)` for each one. Each restored window then reads its own `SceneStorage` to reconstruct its UI. For `DocumentGroup`, this is largely automatic — SwiftUI calls your `Document`'s `fileSnapshot` and scene storage to reconstruct.
