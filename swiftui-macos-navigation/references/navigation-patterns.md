# Navigation Patterns — Reference

Complete working recipes for macOS SwiftUI navigation. Each section is a self-contained pattern with full code. Swift 6.3 / macOS 26+.

---

## 1. Three-Column NavigationSplitView with Programmatic Column Toggle

A full three-column layout (sidebar/content/detail) with a toolbar button to toggle the sidebar and programmatic visibility control.

```swift
import SwiftUI

struct MainView: View {
    @State private var selectedFolder: Folder?
    @State private var selectedItem: Item?
    @State private var columnVisibility: NavigationSplitViewVisibility = .all

    var body: some View {
        NavigationSplitView(columnVisibility: $columnVisibility) {
            // MARK: - Sidebar
            List(Folder.root, children: \.children, selection: $selectedFolder) { folder in
                Label(folder.name, systemImage: folder.icon)
            }
            .navigationSplitViewColumnWidth(min: 160, ideal: 200, max: 280)
            .listStyle(.sidebar)
            .searchable(text: .constant(""), prompt: "Search folders")

        } content: {
            // MARK: - Content (middle column)
            if let selectedFolder {
                List(selectedFolder.items, selection: $selectedItem) { item in
                    ItemRow(item: item)
                }
                .navigationSplitViewColumnWidth(min: 240, ideal: 300, max: 400)
                .searchable(text: .constant(""), prompt: "Search items")
            } else {
                ContentUnavailableView("Select a Folder",
                    systemImage: "folder",
                    description: Text("Choose a folder from the sidebar."))
            }

        } detail: {
            // MARK: - Detail (right column)
            if let selectedItem {
                ItemDetailView(item: selectedItem)
            } else {
                ContentUnavailableView("No Selection",
                    systemImage: "doc",
                    description: Text("Select an item to view its contents."))
            }
        }
        .toolbar {
            ToolbarItem(placement: .navigation) {
                Button {
                    withAnimation {
                        if columnVisibility == .detailOnly {
                            columnVisibility = .all
                        } else {
                            columnVisibility = .detailOnly
                        }
                    }
                } label: {
                    Label("Toggle Sidebar",
                           systemImage: "sidebar.leading")
                }
                .keyboardShortcut("s", modifiers: [.command, .control])
                .help("Toggle sidebar (⌃⌘S)")
            }

            ToolbarItem(placement: .principal) {
                Text(selectedItem?.name ?? selectedFolder?.name ?? "Browser")
                    .font(.headline)
                    .lineLimit(1)
            }
        }
    }
}

// MARK: - Supporting Types

struct Folder: Identifiable, Hashable {
    let id: UUID = UUID()
    let name: String
    let icon: String
    let items: [Item]
    let children: [Folder]

    static let root: Folder = Folder(
        name: "Root",
        icon: "folder.fill",
        items: [],
        children: []
    )
}

struct Item: Identifiable, Hashable {
    let id: UUID = UUID()
    let name: String
    let content: String
}

struct ItemRow: View {
    let item: Item

    var body: some View {
        Label(item.name, systemImage: "doc.text")
            .help(item.name)
    }
}

struct ItemDetailView: View {
    let item: Item

    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 12) {
                Text(item.name)
                    .font(.title)
                Text(item.content)
                    .foregroundStyle(.secondary)
                Spacer()
            }
            .frame(maxWidth: .infinity, alignment: .leading)
            .padding()
        }
    }
}
```

**Key points**:
- The `columnVisibility` binding drives which columns are visible. `.all` shows all three; `.detailOnly` hides sidebar and content.
- The toolbar toggle button animates the transition for a smooth user experience.
- Each column has independent width constraints via `.navigationSplitViewColumnWidth`.
- `ContentUnavailableView` fills empty columns — this is the macOS-standard pattern for empty states.

---

## 2. Deep Link Routing with onOpenURL and WindowGroup Value Types

A complete deep link system: register a URL scheme, parse incoming URLs, and route to the correct window with data.

```swift
import SwiftUI

// MARK: - Deep Link Payload

struct DeepLink: Hashable {
    let path: String
    let parameters: [String: String]

    static func parse(url: URL) -> DeepLink? {
        guard url.scheme == "myapp" else { return nil }
        var params: [String: String] = [:]
        if let components = URLComponents(url: url, resolvingAgainstBaseURL: false),
           let queryItems = components.queryItems {
            for item in queryItems {
                params[item.name] = item.value
            }
        }
        return DeepLink(path: url.host ?? "", parameters: params)
    }
}

// MARK: - Window Payloads

struct DocumentWindowPayload: Codable, Hashable {
    let documentID: UUID
}

struct ProfileWindowPayload: Codable, Hashable {
    let userID: String
}

// MARK: - App

@main
struct MyApp: App {
    var body: some Scene {
        // Main window — handles deep links via onOpenURL
        WindowGroup {
            MainBrowserView()
                .onOpenURL { url in
                    handleDeepLink(url)
                }
        }

        // Document window group — opened by deep link or user action
        WindowGroup("Document", id: "document", for: DocumentWindowPayload.self) { $payload in
            DocumentDetailView(documentID: payload.documentID)
        } defaultValue: {
            DocumentWindowPayload(documentID: UUID())
        }
        .defaultSize(width: 700, height: 500)

        // Profile inspector — singleton window
        Window("User Profile", id: "profile") {
            Text("Select a user")
                .foregroundStyle(.secondary)
        }
        .defaultSize(width: 360, height: 480)
    }

    private func handleDeepLink(_ url: URL) {
        guard let link = DeepLink.parse(url: url) else { return }

        switch link.path {
        case "document":
            if let idString = link.parameters["id"],
               let uuid = UUID(uuidString: idString) {
                openWindow(id: "document", value: DocumentWindowPayload(documentID: uuid))
            }
        case "profile":
            if let userID = link.parameters["user"] {
                openWindow(id: "profile")
                // To pass data to a singleton Window, use the environment
                // or a shared observable object instead of WindowGroup value types.
            }
        default:
            break
        }
    }
}

// MARK: - Main Browser (in the main WindowGroup)

struct MainBrowserView: View {
    @Environment(\.openWindow) private var openWindow

    var body: some View {
        NavigationSplitView {
            List {
                Label("All Documents", systemImage: "doc.on.doc")
                Label("Recent", systemImage: "clock")
                Label("Shared", systemImage: "person.2")
            }
            .listStyle(.sidebar)
        } detail: {
            ContentUnavailableView("Select a category",
                systemImage: "sidebar.leading")
        }
        .toolbar {
            ToolbarItem(placement: .primaryAction) {
                Button("Open Document") {
                    // Constructs: myapp://document/550e8400-e29b-41d4-a716-446655440000
                    let payload = DocumentWindowPayload(documentID: UUID())
                    openWindow(id: "document", value: payload)
                }
            }
        }
    }
}

struct DocumentDetailView: View {
    let documentID: UUID
    @Environment(\.dismissWindow) private var dismissWindow

    var body: some View {
        VStack(spacing: 16) {
            Text("Document")
                .font(.title)
            Text(documentID.uuidString)
                .font(.monospaced(.body)())
                .foregroundStyle(.secondary)
            Button("Close") {
                dismissWindow(id: "document")
            }
            .keyboardShortcut(.return, modifiers: .command)
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
    }
}
```

**Info.plist registration**:

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>com.example.myapp</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>myapp</string>
        </array>
    </dict>
</array>
```

**Key points**:
- `DeepLink.parse()` extracts the host as the path and query items as parameters.
- `WindowGroup(for:)` requires the payload to be `Codable` & `Hashable`.
- `defaultValue:` prevents crashes when the window is opened without a URL payload.
- Singleton `Window` scenes cannot receive payloads via `openWindow(value:)` — use `@Observable` shared state or `@Environment` instead.
- Test with: `xcrun simctl openurl booted "myapp://document?id=550e8400-e29b-41d4-a716-446655440000"`

---

## 3. Programmatic NavigationStack with NavigationPath

A drill-down browser with back/forward/root toolbar controls and codable path for state restoration.

```swift
import SwiftUI

// All destinations must be Hashable (and Codable for restoration).
// Enums work well as a closed set of destinations.

enum Route: Hashable, Codable {
    case folder(UUID)
    case document(UUID)
    case settings
    case settingsGeneral
    case settingsAccount
}

@Observable
class NavigationRouter {
    var path = NavigationPath()

    func navigate(to route: Route) {
        path.append(route)
    }

    func goBack() {
        guard !path.isEmpty else { return }
        path.removeLast()
    }

    func goBack(levels: Int) {
        let count = min(levels, path.count)
        for _ in 0..<count {
            path.removeLast()
        }
    }

    func popToRoot() {
        path = NavigationPath()
    }

    var canGoBack: Bool { !path.isEmpty }
    var depth: Int { path.count }
}

struct ProgrammaticBrowser: View {
    @State private var router = NavigationRouter()
    @State private var searchText = ""

    var body: some View {
        NavigationStack(path: $router.path) {
            // Root view
            List {
                Label("Documents", systemImage: "doc.on.doc")
                    .onTapGesture { router.navigate(to: .folder(rootID)) }
                Label("Settings", systemImage: "gear")
                    .onTapGesture { router.navigate(to: .settings) }
            }
            .navigationTitle("Browser")
            .navigationDestination(for: Route.self) { route in
                destinationView(for: route, router: router)
            }
            .searchable(text: $searchText)
            .toolbar {
                ToolbarItemGroup(placement: .navigation) {
                    Button {
                        router.goBack()
                    } label: {
                        Image(systemName: "chevron.left")
                    }
                    .disabled(!router.canGoBack)

                    Button {
                        router.popToRoot()
                    } label: {
                        Image(systemName: "house")
                    }
                    .disabled(!router.canGoBack)
                }

                ToolbarItem(placement: .status) {
                    if router.depth > 0 {
                        Text("\(router.depth) level\(router.depth == 1 ? "" : "s") deep")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }
            }
        }
    }

    @ViewBuilder
    private func destinationView(for route: Route, router: NavigationRouter) -> some View {
        switch route {
        case .folder(let id):
            FolderView(folderID: id, router: router)
        case .document(let id):
            DocumentEditorView(documentID: id)
        case .settings:
            SettingsRouterView(router: router)
        case .settingsGeneral:
            GeneralSettingsView()
        case .settingsAccount:
            AccountSettingsView()
        }
    }
}

// Example leaf views

struct FolderView: View {
    let folderID: UUID
    let router: NavigationRouter

    var body: some View {
        List {
            ForEach(sampleItems) { item in
                Label(item.name, systemImage: "doc")
                    .onTapGesture {
                        router.navigate(to: .document(item.id))
                    }
            }
        }
        .navigationTitle("Folder")
    }

    private var sampleItems: [SampleItem] {
        (1...5).map { SampleItem(id: UUID(),name: "Document \($0)") }
    }
}

struct SampleItem: Identifiable {
    let id: UUID
    let name: String
}

private let rootID = UUID()

struct DocumentEditorView: View {
    let documentID: UUID

    var body: some View {
        Text("Editing: \(documentID.uuidString)")
            .font(.title)
            .frame(maxWidth: .infinity, maxHeight: .infinity)
    }
}

struct SettingsRouterView: View {
    let router: NavigationRouter

    var body: some View {
        List {
            Label("General", systemImage: "gearshape")
                .onTapGesture { router.navigate(to: .settingsGeneral) }
            Label("Account", systemImage: "person")
                .onTapGesture { router.navigate(to: .settingsAccount) }
        }
        .navigationTitle("Settings")
    }
}

struct GeneralSettingsView: View {
    var body: some View {
        Form {
            Toggle("Show notifications", isOn: .constant(true))
            Toggle("Dark mode", isOn: .constant(false))
        }
        .navigationTitle("General")
    }
}

struct AccountSettingsView: View {
    var body: some View {
        Form {
            TextField("Display name", text: .constant("User"))
        }
        .navigationTitle("Account")
    }
}
```

**State restoration with Codable path**:

```swift
// In your App struct or a controller:
func saveNavigationState() {
    do {
        let data = try router.path.codable
        // Save `data` to UserDefaults or a file
    } catch {
        print("Failed to encode navigation path: \(error)")
    }
}

func restoreNavigationState(from data: Data) {
    do {
        let codablePath = try JSONDecoder().decode(
            NavigationPath.CodableRepresentation.self,
            from: data
        )
        router.path = NavigationPath(codablePath)
    } catch {
        router.path = NavigationPath()
    }
}
```

**Key points**:
- The `NavigationRouter` `@Observable` class owns the path, enabling programmatic control from any view that holds a reference.
- All route types in the `Route` enum must be `Codable` for the path to support state restoration.
- `NavigationPath.codable` returns a `CodableRepresentation` that can be serialized to/from JSON.
- `goBack(levels:)` pops multiple levels — useful for "skip to section" navigation from a sitemap.

---

## 4. Custom Popover with Attachment Anchor

A popover anchored to a button, containing a complex SwiftUI view with its own interactions.

```swift
import SwiftUI

struct PopoverExample: View {
    @State private var showingPopover = false
    @State private var selectedColor: Color = .blue
    @State private var enabled = true

    var body: some View {
        HStack(spacing: 20) {
            // Button with popover
            ColorSwatch(color: selectedColor, isEnabled: enabled)
                .onTapGesture {
                    showingPopover = true
                }
                .popover(
                    isPresented: $showingPopover,
                    attachmentAnchor: .point(.topTrailing),
                    arrowEdge: .top
                ) {
                    PopoverContent(
                        selectedColor: $selectedColor,
                        enabled: $enabled
                    )
                    .padding()
                    .frame(width: 220)
                }

            Text("Tap the swatch to edit")
                .foregroundStyle(.secondary)
        }
        .padding()
    }
}

struct ColorSwatch: View {
    let color: Color
    let isEnabled: Bool

    var body: some View {
        RoundedRectangle(cornerRadius: 8)
            .fill(isEnabled ? color : Color.gray.opacity(0.3))
            .frame(width: 40, height: 40)
            .overlay {
                RoundedRectangle(cornerRadius: 8)
                    .stroke(.quaternary, lineWidth: 1)
            }
    }
}

struct PopoverContent: View {
    @Binding var selectedColor: Color
    @Binding var enabled: Bool
    @Environment(\.dismiss) private var dismiss

    private let palette: [Color] = [
        .red, .orange, .yellow, .green,
        .blue, .purple, .pink, .brown
    ]

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            Text("Color Settings")
                .font(.headline)

            Toggle("Enabled", isOn: $enabled)

            Text("Color")
                .font(.subheadline)
                .foregroundStyle(.secondary)

            LazyVGrid(columns: Array(repeating: GridItem(.flexible()), count: 4), spacing: 8) {
                ForEach(palette, id: \.self) { color in
                    Circle()
                        .fill(color)
                        .frame(width: 28, height: 28)
                        .overlay {
                            Circle()
                                .stroke(selectedColor == color ? Color.primary : Color.clear, lineWidth: 2)
                        }
                        .onTapGesture {
                            selectedColor = color
                        }
                }
            }

            Divider()

            Button("Done") {
                dismiss()
            }
            .keyboardShortcut(.return, modifiers: .command)
            .frame(maxWidth: .infinity)
        }
    }
}
```

**Key points**:
- `attachmentAnchor: .point(.topTrailing)` anchors the popover's arrow to the top-trailing point of the source view.
- `arrowEdge: .top` tells the popover which edge the arrow should appear on (opposite to the attachment point).
- The popover dismisses automatically when the user clicks outside it. Call `dismiss()` explicitly for a "Done" button.
- Popovers size to their content — set an explicit `.frame(width:)` to prevent layout ambiguity.
- The popover repositions automatically when the source view moves (e.g., during window resize).

---

## 5. Fullscreen Replacement for fullscreenCover on macOS

Since `fullscreenCover` does not exist on macOS, use a dedicated `Window` scene or a large sheet as a replacement. This pattern uses a separate window with a minimal chrome overlay.

```swift
import SwiftUI

// MARK: - Fullscreen Window Manager

@Observable
class FullscreenModel {
    static let shared = FullscreenModel()
    var isOpen = false
}

// MARK: - Main View with Fullsheet Trigger

struct ContentWithFullscreen: View {
    @Environment(\.openWindow) private var openWindow
    @Environment(\.dismissWindow) private var dismissWindow
    @StateObject private var model = FullscreenModel.shared

    var body: some View {
        VStack(spacing: 20) {
            Text("Fullscreen Demo")
                .font(.title)

            Button("Enter Fullscreen") {
                openWindow(id: "fullscreen")
            }
            .buttonStyle(.borderedProminent)

            Button("Enter as Sheet (alternative)") {
                model.isOpen = true
            }
        }
        .frame(minWidth: 400, minHeight: 300)
        // Sheet-based alternative — uses .large detent
        .sheet(isPresented: $model.isOpen) {
            FullscreenContentView()
                .presentationDetents([.large])
                .presentationDragIndicator(.hidden)
                .interactiveDismissDisabled()
        }
    }
}

// MARK: - Fullscreen Window Content

struct FullscreenContentView: View {
    @Environment(\.openWindow) private var openWindow
    @Environment(\.dismissWindow) private var dismissWindow

    var body: some View {
        ZStack {
            // Background
            LinearGradient(
                colors: [.blue.opacity(0.3), .purple.opacity(0.3)],
                startPoint: .topLeading,
                endPoint: .bottomTrailing
            )
            .ignoresSafeArea()

            VStack(spacing: 24) {
                Text("Fullscreen Mode")
                    .font(.largeTitle)

                Text("This window replaces fullscreenCover on macOS. Use a separate Window scene for a native fullscreen experience.")
                    .multilineTextAlignment(.center)
                    .foregroundStyle(.secondary)
                    .frame(maxWidth: 400)

                HStack {
                    Button("Exit") {
                        dismissWindow(id: "fullscreen")
                    }
                    .buttonStyle(.bordered)

                    Button("Exit (⌘W)") {
                        dismissWindow(id: "fullscreen")
                    }
                    .keyboardShortcut("w")
                }
            }
        }
        .toolbar {
            ToolbarItem(placement: .principal) {
                Text("Fullscreen")
                    .font(.headline)
            }
            ToolbarItem(placement: .confirmationAction) {
                Button("Done") {
                    dismissWindow(id: "fullscreen")
                }
                .keyboardShortcut(.return, modifiers: .command)
            }
        }
    }
}

// MARK: - App Declaration

/*
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentWithFullscreen()
        }

        // Fullscreen replacement window
        Window("Fullscreen", id: "fullscreen") {
            FullscreenContentView()
        }
        .defaultSize(width: 1024, height: 768)
        .windowResizability(.contentSize)
        // .windowStyle(.hiddenTitleBar) // Use for true fullscreen chrome
    }
}
*/

// MARK: - Alternative: Native Fullscreen via AppKit Bridge

/// For true native fullscreen (with the green traffic light button behavior),
/// bridge to AppKit. Use this inside a view that needs to trigger fullscreen.

struct NativeFullscreenBridge: View {
    var body: some View {
        Button("Toggle Native Fullscreen") {
            toggleNativeFullscreen()
        }
    }

    private func toggleNativeFullscreen() {
        guard let window = NSApp.keyWindow else { return }
        window.toggleFullScreen(nil)
    }
}
```

**Three approaches compared**:

| Approach | Chrome | Dismissal | Use Case |
|----------|--------|-----------|----------|
| Separate `Window` scene | Standard toolbar | `dismissWindow(id:)` | Presentations that need their own window |
| Sheet with `.large` detent | Sheet frame | Swipe down or button | Modal content that should feel attached |
| `NSWindow.toggleFullScreen` | Native fullscreen | Escape or green button | True fullscreen (media, presentations) |

**Key points**:
- `fullscreenCover` does not exist on macOS — using it causes a runtime crash.
- A separate `Window` scene is the most flexible replacement — it gets its own menu bar, toolbar, and window management.
- For true native fullscreen, bridge to `NSWindow.toggleFullScreen(_:)` via `NSApp.keyWindow`.
- `.windowStyle(.hiddenTitleBar)` removes the title bar for a cleaner fullscreen appearance.
- `.interactiveDismissDisabled()` on sheets prevents accidental dismissal of important content.
