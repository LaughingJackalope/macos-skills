# Scene Patterns — Multi-Window Apps, Restoration, MenuBarExtra, DocumentGroup

Complete patterns for macOS multi-window architectures. Each section is a self-contained recipe.

---

## 1. Multi-Window App with Inspector

A main content window and a single inspector window. The inspector shows details for the currently selected item.

```swift
@main
struct MyApp: App {
    @State private var selectedItem: Item?

    var body: some Scene {
        WindowGroup {
            ContentView(selectedItem: $selectedItem)
        }

        Window("Inspector", id: "inspector") {
            InspectorView(item: selectedItem)
        }
        .defaultSize(width: 300, height: 500)
        .keyboardShortcut("i", modifiers: [.command, .shift])
        .windowResizability(.contentSize)
    }
}

struct ContentView: View {
    @Environment(\.openWindow) private var openWindow
    @Binding var selectedItem: Item?

    var body: some View {
        List(items, selection: $selectedItem) { item in
            Text(item.name)
                .tag(item)
        }
        .toolbar {
            ToolbarItem {
                Button("Toggle Inspector") {
                    openWindow(id: "inspector")
                }
                .keyboardShortcut("i", modifiers: [.command, .shift])
            }
        }
    }
}
```

**Key points:**
- `Window` (not `WindowGroup`) ensures only one inspector exists.
- `selectedItem` is passed as a binding; the inspector updates reactively.
- `.windowResizability(.contentSize)` prevents the user from resizing beyond the content's ideal size.

---

## 2. Parameterized Window with Codable Payload

Open a detail window for a specific document, passing the document ID as a payload.

```swift
struct DocumentWindowPayload: Codable, Hashable {
    let documentID: UUID
}

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            DocumentListView()
        }

        WindowGroup("Document", id: "document", for: DocumentWindowPayload.self) { $payload in
            DocumentDetailView(documentID: payload.documentID)
        } defaultValue: {
            DocumentWindowPayload(documentID: UUID())
        }
        .defaultSize(width: 700, height: 500)
        .commands {
            // Remove the default "File → New Window" for this group
        }
    }
}

struct DocumentListView: View {
    @Environment(\.openWindow) private var openWindow

    var body: some View {
        List(documents) { doc in
            Button(doc.title) {
                openWindow(id: "document", value: DocumentWindowPayload(documentID: doc.id))
            }
        }
    }
}

struct DocumentDetailView: View {
    let documentID: UUID
    @Environment(\.dismissWindow) private var dismissWindow

    var body: some View {
        VStack {
            Text("Document: \(documentID.uuidString)")
            Button("Close") {
                dismissWindow(id: "document")
            }
        }
    }
}
```

**Key points:**
- `for: DocumentWindowPayload.self` declares the payload type.
- `defaultValue:` provides a fallback when the window is opened without a payload (e.g., from File → New Window).
- Each call to `openWindow(id:value:)` with a different payload creates a new window instance.

---

## 3. Window Restoration with @AppStorage

Persist window-level preferences across launches.

```swift
struct ContentView: View {
    @AppStorage("sidebarVisible") private var sidebarVisible = true
    @AppStorage("selectedTab") private var selectedTab = "all"
    @State private var searchText = ""

    var body: some View {
        NavigationSplitView {
            List {
                // Sidebar content
            }
            .searchable(text: $searchText)
            .opacity(sidebarVisible ? 1 : 0)
        } detail: {
            DetailView()
        }
        .toolbar {
            ToolbarItem {
                Toggle("Sidebar", isOn: $sidebarVisible)
                    .toggleStyle(.switch)
            }
        }
    }
}
```

**Key points:**
- `@AppStorage` automatically persists to `UserDefaults` and restores on relaunch.
- Window position and size are restored automatically by the system.
- Use `@AppStorage` for user preferences, not for large data sets.

---

## 4. MenuBarExtra — .menu Style with Keep-Alive

A menu bar utility that monitors system state even when the menu is closed.

```swift
import SwiftUI

class AppDelegate: NSObject, NSApplicationDelegate {
    private var monitor: Timer?

    func applicationDidFinishLaunching(_ notification: Notification) {
        NSApp.setActivationPolicy(.accessory)

        // Keep-alive: timer fires even when menu is closed
        monitor = Timer.scheduledTimer(withTimeInterval: 30, repeats: true) { _ in
            StatusBarModel.shared.refresh()
        }
    }

    func applicationWillTerminate(_ notification: Notification) {
        monitor?.invalidate()
    }
}

@main
struct MenuBarApp: App {
    @NSApplicationDelegateAdaptor(AppDelegate.self) var appDelegate
    @StateObject private var model = StatusBarModel.shared

    var body: some Scene {
        MenuBarExtra {
            VStack(alignment: .leading, spacing: 8) {
                HStack {
                    Image(systemName: model.statusIcon)
                    Text(model.statusText)
                }
                .font(.headline)

                Divider()

                ForEach(model.items) { item in
                    Label(item.title, systemImage: item.icon)
                }

                Divider()

                Button("Quit") {
                    NSApp.terminate(nil)
                }
                .keyboardShortcut("q")
            }
            .padding()
        } label: {
            Image(systemName: model.statusIcon)
        }
        .menuBarExtraStyle(.menu)
    }
}

@MainActor
class StatusBarModel: ObservableObject {
    static let shared = StatusBarModel()

    @Published var statusIcon = "circle"
    @Published var statusText = "Checking..."
    @Published var items: [StatusItem] = []

    func refresh() {
        // Perform background work
        Task {
            // Update published properties
        }
    }
}
```

**Key points:**
- `NSApp.setActivationPolicy(.accessory)` hides the dock icon.
- The `AppDelegate` timer keeps the model alive between menu opens.
- The `label` closure (macOS 14+) provides the menu bar icon; the main closure provides the menu content.

---

## 5. MenuBarExtra — .window Style Popover

A richer popover that stays open and contains interactive content.

```swift
@main
struct MenuBarApp: App {
    var body: some Scene {
        MenuBarExtra {
            PopoverView()
                .frame(width: 360, height: 480)
        } label: {
            Label("MyApp", systemImage: "star.fill")
        }
        .menuBarExtraStyle(.window)
    }
}

struct PopoverView: View {
    @State private var isRunning = false

    var body: some View {
        NavigationStack {
            VStack(spacing: 16) {
                Image(systemName: "star.fill")
                    .font(.system(size: 48))
                    .foregroundStyle(.blue)

                Text("MyApp Dashboard")
                    .font(.title2)

                Button(isRunning ? "Stop" : "Start") {
                    isRunning.toggle()
                }
                .buttonStyle(.borderedProminent)
            }
            .padding()
            .navigationTitle("Dashboard")
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    Button("Settings") {
                        NSApp.sendAction(Selector(("showSettingsWindow:")), to: nil, from: nil)
                    }
                }
            }
        }
    }
}
```

**Key points:**
- `.window` style keeps the popover open until the user clicks away or presses Escape.
- The view stays alive and maintains state while open.
- Use `NavigationStack` inside the popover for multi-screen flows.

---

## 6. DocumentGroup — Full Document-Based App

A complete document-based app with custom document type, editor, and file format.

```swift
import SwiftUI
import UniformTypeIdentifiers

// MARK: - Document Model

struct NoteDocument: Codable, ReferenceFileDocument {
    var title: String
    var body: String
    var tags: [String]
    var createdAt: Date
    var modifiedAt: Date

    init(title: String = "Untitled", body: String = "", tags: [String] = []) {
        self.title = title
        self.body = body
        self.tags = tags
        self.createdAt = Date()
        self.modifiedAt = Date()
    }

    init(configuration: ReadConfiguration) throws {
        guard let data = configuration.file.regularFileContents,
              let doc = try? JSONDecoder().decode(Self.self, from: data)
        else { throw CocoaError(.fileReadCorruptFile) }
        self = doc
    }

    func fileWrapper(configuration: WriteConfiguration) throws -> FileWrapper {
        var copy = self
        copy.modifiedAt = Date()
        let data = try JSONEncoder().encode(copy)
        return FileWrapper(regularFileWithContents: data)
    }
}

// MARK: - App Entry

@main
struct NotesApp: App {
    var body: some Scene {
        DocumentGroup(newDocument: NoteDocument()) { file in
            NoteEditorView(document: file.$document)
        }
        .windowStyle(.unified)
        .commands {
            CommandGroup(after: .newItem) {
                // Custom new document command
            }
        }
    }
}

// MARK: - Editor View

struct NoteEditorView: View {
    @Binding var document: NoteDocument
    @State private var isEditingTitle = false

    var body: some View {
        VStack(alignment: .leading, spacing: 0) {
            // Title bar
            HStack {
                TextField("Title", text: $document.title)
                    .font(.title)
                    .textFieldStyle(.plain)
                Spacer()
                Text("Modified: \(document.modifiedAt, style: .relative)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }
            .padding()

            Divider()

            // Tags
            ScrollView(.horizontal, showsIndicators: false) {
                HStack {
                    ForEach(document.tags, id: \.self) { tag in
                        Text(tag)
                            .padding(.horizontal, 8)
                            .padding(.vertical, 4)
                            .background(Color.blue.opacity(0.1))
                            .clipShape(Capsule())
                    }
                }
                .padding(.horizontal)
            }
            .padding(.vertical, 8)

            // Body editor
            TextEditor(text: $document.body)
                .font(.body)
                .scrollContentBackground(.hidden)
                .padding()
        }
        .toolbar {
            ToolbarItem(placement: .primaryAction) {
                Button("Add Tag") {
                    document.tags.append("new-tag")
                }
            }
        }
    }
}
```

**Info.plist additions required:**

```xml
<key>UTExportedTypeDeclarations</key>
<array>
    <dict>
        <key>UTTypeIdentifier</key>
        <string>com.example.note</string>
        <key>UTTypeDescription</key>
        <string>Note Document</string>
        <key>UTTypeConformsTo</key>
        <array>
            <string>public.data</string>
        </array>
        <key>UTTypeTagSpecification</key>
        <dict>
            <key>public.filename-extension</key>
            <array>
                <string>mynotes</string>
            </array>
        </dict>
    </dict>
</array>
<key>CFBundleDocumentTypes</key>
<array>
    <dict>
        <key>CFBundleTypeName</key>
        <string>Note Document</string>
        <key>CFBundleTypeRole</key>
        <string>Editor</string>
        <key>LSItemContentTypes</key>
        <array>
            <string>com.example.note</string>
        </array>
        <key>CFBundleTypeExtensions</key>
        <array>
            <string>mynotes</string>
        </array>
    </dict>
</array>
```

**Key points:**
- `ReferenceFileDocument` is a value type protocol — the document is copied on read and written on save.
- Use `FileDocument` (class-based) for large documents that need incremental loading.
- The system provides open/save panels, versions browser, and iCloud sync automatically.

---

## 7. Custom Window Chrome with Hidden Title Bar

A fully custom window with no system title bar, custom drag region, and traffic light buttons.

```swift
@main
struct CustomChromeApp: App {
    var body: some Scene {
        WindowGroup {
            CustomChromeView()
        }
        .windowStyle(.hiddenTitleBar)
        .defaultSize(width: 900, height: 600)
    }
}

struct CustomChromeView: View {
    var body: some View {
        VStack(spacing: 0) {
            // Custom title bar
            HStack {
                // Traffic light placeholder
                HStack(spacing: 8) {
                    Circle().fill(.red).frame(width: 12, height: 12)
                    Circle().fill(.yellow).frame(width: 12, height: 12)
                    Circle().fill(.green).frame(width: 12, height: 12)
                }
                .padding(.leading, 16)

                Spacer()

                Text("MyApp")
                    .font(.headline)

                Spacer()

                // Balance the traffic lights
                Color.clear.frame(width: 68)
            }
            .frame(height: 38)
            .background(.bar)
            .onWindowDrag()  // macOS 26+

            Divider()

            // Main content
            ContentView()
        }
    }
}
```

**Key points:**
- `.windowStyle(.hiddenTitleBar)` removes the system title bar.
- `.onWindowDrag()` (macOS 26+) makes the entire view draggable. For earlier macOS versions, use `NSWindow` via `NSViewRepresentable`.
- Traffic light buttons (close, minimize, zoom) still function — they are positioned by the system in the standard location. Leave space for them or use `NSWindow` APIs to reposition.

---

## 8. Unified Toolbar Style (macOS 26+)

The unified style merges the title bar and toolbar into a single surface.

```swift
struct ContentView: View {
    var body: some View {
        NavigationSplitView {
            SidebarView()
        } detail: {
            DetailView()
        }
        .toolbar {
            ToolbarItem(placement: .navigation) {
                Button("Back") { /* ... */ }
            }
            ToolbarItem(placement: .principal) {
                Text("My Document")
                    .font(.headline)
            }
            ToolbarItem(placement: .primaryAction) {
                Button("Share") { /* ... */ }
            }
        }
        .toolbar(removing: .sidebarToggle)
        .toolbarBackground(.hidden, for: .windowToolbar)
    }
}
```

Applied at the scene level:

```swift
WindowGroup {
    ContentView()
}
.windowStyle(.unified)
```

**Key points:**
- `.unified` merges title bar and toolbar; `.unifiedCompact` uses a shorter bar.
- `.toolbar(removing: .sidebarToggle)` removes the default sidebar toggle since the unified style handles it differently.
- `.toolbarBackground(.hidden, for: .windowToolbar)` makes the background transparent for a seamless look.
- The `.principal` placement positions items in the center of the unified bar.

---

## 9. Scene Phase and Window Activation

React to window activation state changes.

```swift
struct ContentView: View {
    @Environment(\.scenePhase) private var scenePhase
    @Environment(\.controlActiveState) private var controlActiveState

    var body: some View {
        VStack {
            Text("Content")
        }
        .onChange(of: scenePhase) { _, newPhase in
            switch newPhase {
            case .active:
                // Window is active — resume timers, refresh data
                break
            case .inactive:
                // Window is visible but not key — pause animations
                break
            case .background:
                // App is backgrounded — save state
                break
            @unknown default:
                break
            }
        }
        .onChange(of: controlActiveState) { _, newState in
            // Fires when this specific window gains/loses focus
            if newState == .key {
                // This window is the key window
            }
        }
    }
}
```

**Key points:**
- `scenePhase` tracks the app-wide lifecycle (active/inactive/background).
- `controlActiveState` (macOS 14+) tracks per-window focus state.
- Use `controlActiveState` for per-window reactions; use `scenePhase` for app-wide lifecycle events.
