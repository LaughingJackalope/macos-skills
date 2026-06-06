# Layout Patterns — Deep-Dive Reference

Complete patterns for macOS SwiftUI layout. Each section is a self-contained recipe with code, key points, and pitfalls. Targets macOS 26+ / Swift 6.3.

---

## 1. Table with Sort Descriptors, Multi-Selection, Context Menu

A production file manager table with multi-column sorting via `SortDescriptor` (macOS 14+), full row selection, and per-row context menus.

```swift
import SwiftUI

struct File: Identifiable, Hashable {
    let id = UUID()
    let name: String
    let size: Int64
    let modified: Date
    let kind: String
}

class FileStore: ObservableObject {
    @Published var files: [File] = []
}

struct FileTableView: View {
    @StateObject private var store = FileStore()
    @State private var sortOrder: [SortDescriptor<File>] = [
        SortDescriptor(\.name, order: .forward),
    ]
    @State private var selection = Set<File.ID>()
    @State private var searchText = ""

    var sortedFiles: [File] {
        let filtered = searchText.isEmpty
            ? store.files
            : store.files.filter { $0.name.localizedStandardContains(searchText) }

        return filtered.sorted(using: sortOrder)
    }

    var body: some View {
        Table(sortedFiles, selection: $selection, sortOrder: $sortOrder) {
            TableColumn("Name", value: \.name) { file in
                HStack(spacing: 6) {
                    Image(systemName: iconFor(file.kind))
                        .foregroundStyle(.tint)
                    Text(file.name)
                }
            }
            .width(min: 220, ideal: 300)

            TableColumn("Size", value: \.size) { file in
                Text(file.size, format: .fileSize)
                    .frame(maxWidth: .infinity, alignment: .trailing)
                    .foregroundStyle(.secondary)
            }
            .width(90)

            TableColumn("Modified", value: \.modified) { file in
                Text(file.modified, format: .dateTime.month().day().year())
                    .foregroundStyle(.secondary)
            }
            .width(120)

            TableColumn("Kind", value: \.kind)
                .width(70)
        } rows: {
            ForEach(sortedFiles) { file in
                TableRow(file)
                    .contextMenu {
                        Button("Open") { openFile(file) }
                        Button("Reveal in Finder") { revealInFinder(file) }
                        Divider()
                        Button("Duplicate") { duplicateFile(file) }
                        Button("Move to Trash", role: .destructive) { trashFile(file) }
                            .keyboardShortcut(.delete, modifiers: [])
                    }
            }
        }
        .searchable(text: $searchText, prompt: "Search files")
        .toolbar {
            ToolbarItem(placement: .primaryAction) {
                Button {
                    // Add file action
                } label: {
                    Label("Add", systemImage: "plus")
                }
                .disabled(selection.isEmpty) // enable based on context
            }
            ToolbarItem(placement: .secondaryAction) {
                Menu {
                    Button("Select All") { selection = Set(sortedFiles.map(\.id)) }
                    Button("Deselect All") { selection = [] }
                } label: {
                    Label("Selection", systemImage: "checkmark.circle")
                }
            }
        }
        .onChange(of: selection) { _, newValue in
            // React to selection change (e.g., update inspector)
        }
    }

    private func iconFor(_ kind: String) -> String {
        switch kind.lowercased() {
        case "image": return "photo"
        case "document": return "doc.text"
        case "folder": return "folder"
        default: return "doc"
        }
    }
    private func openFile(_ file: File) { /* ... */ }
    private func revealInFinder(_ file: File) { /* ... */ }
    private func duplicateFile(_ file: File) { /* ... */ }
    private func trashFile(_ file: File) { /* ... */ }
}
```

**Key points:**

- `SortDescriptor` (macOS 14+) replaces the deprecated `KeyPathComparator`. It uses `\.keyPath` syntax with an optional `order:` parameter.
- `sorted(using: sortOrder)` applies all comparators in order — first by name, then by size as tiebreaker, etc.
- The `selection: Set<T.ID>` binding gives you multi-selection (⌘-click, Shift-click). Remove the selection parameter for a read-only table.
- `.contextMenu` on `TableRow` adds per-row right-click menus. `.keyboardShortcut(.delete, modifiers: [])` on a button inside the context menu automatically triggers on Delete key press (no modifiers needed).
- **Performance**: the `sorted()` call runs on every state change. For 10k+ rows, use a `Task`-based debounced sort or a cached `sortedFiles` `@Published` property on the store that invalidates only when `sortOrder` actually changes.
- Avoid putting expensive work (e.g., image decoding) in the cell closure. The cell closure runs for every visible row on every state change.

---

## 2. List with Sidebar Style, Custom Row Background, Section Headers

A sidebar navigation list that looks like Finder's sidebar, with custom row backgrounds and smart section headers.

```swift
import SwiftUI

struct NavigationItem: Identifiable, Hashable {
    let id: String
    let title: String
    let icon: String
    let badgeCount: Int?
}

struct SidebarView: View {
    @State private var selection: NavigationItem.ID?
    @State private var searchText = ""

    let favorites: [NavigationItem] = [
        NavigationItem(id: "all", title: "All Files", icon: "doc.on.doc", badgeCount: nil),
        NavigationItem(id: "recent", title: "Recent", icon: "clock", badgeCount: 3),
        NavigationItem(id: "shared", title: "Shared", icon: "person.2", badgeCount: nil),
    ]

    let tags: [NavigationItem] = [
        NavigationItem(id: "work", title: "Work", icon: "folder", badgeCount: 12),
        NavigationItem(id: "personal", title: "Personal", icon: "folder", badgeCount: 5),
        NavigationItem(id: "archived", title: "Archived", icon: "archivebox", badgeCount: nil),
    ]

    var body: some View {
        List(selection: $selection) {
            Section {
                ForEach(favorites) { item in
                    sidebarRow(item)
                }
            } header: {
                Text("Favorites")
                    .font(.caption.weight(.medium))
                    .foregroundStyle(.secondary)
                    .textCase(nil) // prevent all-caps default
            }

            Section {
                ForEach(tags) { item in
                    sidebarRow(item)
                }
            } header: {
                HStack {
                    Text("Tags")
                        .font(.caption.weight(.medium))
                        .foregroundStyle(.secondary)
                    Spacer()
                    Button {
                        // Create new tag
                    } label: {
                        Image(systemName: "plus.circle.fill")
                    }
                    .buttonStyle(.plain)
                    .foregroundStyle(.secondary)
                }
                .textCase(nil)
            }
        }
        .listStyle(.sidebar)
        .searchable(text: $searchText, placement: .sidebar) // iOS 16+ / macOS 13+
        .navigationTitle("MyApp")
        .onChange(of: selection) { _, newValue in
            guard let id = newValue else { return }
            handleNavigation(to: id)
        }
    }

    private func sidebarRow(_ item: NavigationItem) -> some View {
        Label {
            HStack {
                Text(item.title)
                Spacer()
                if let count = item.badgeCount {
                    Text("\(count)")
                        .font(.caption.weight(.medium))
                        .foregroundStyle(.secondary)
                        .padding(.horizontal, 6)
                        .padding(.vertical, 2)
                        .background {
                            Capsule()
                                .fill(.secondary.opacity(0.15))
                        }
                }
            }
        } icon: {
            Image(systemName: item.icon)
                .symbolVariant(.fill)
                .foregroundStyle(.tint)
        }
        .tag(item.id)
        .listRowInsets(EdgeInsets(top: 4, leading: 12, bottom: 4, trailing: 12))
    }

    private func handleNavigation(to id: NavigationItem.ID) {
        // Trigger navigation change
    }
}
```

**Key points:**

- `.listStyle(.sidebar)` gives the Finder-style translucent sidebar. Pair with `.searchable(placement: .sidebar)` to embed the search field in the sidebar's title bar area.
- **`.textCase(nil)`** on section headers prevents macOS's default all-caps rendering. Without it, "Favorites" becomes "FAVORITES".
- **Badge counts**: render as capsules with `.secondary.opacity(0.15)` background. Avoid colors — sidebar badges should be subtle.
- `.listRowInsets` tightens the row padding from the default sidebar spacing (which can be too generous on macOS 15+).
- **Custom row background**: to override the default sidebar highlight, use `.listRowBackground(Color.accentColor.opacity(isSelected ? 0.5 : 0))`. However, this requires tracking the selection state externally because the `List` doesn't pass selection status into the row content closure. An alternative is using `Label` with a custom `icon:` that changes symbol variant on hover.

---

## 3. OutlineGroup with Expansion State Persistence

A hierarchical file browser that remembers which nodes are expanded across launches using `@AppStorage`.

```swift
import SwiftUI
import Observation

@Observable
final class TreeNode {
    let id: UUID
    let name: String
    let icon: String
    let isDirectory: Bool
    var children: [TreeNode]?

    init(id: UUID = UUID(), name: String, icon: String, isDirectory: Bool, children: [TreeNode]? = nil) {
        self.id = id
        self.name = name
        self.icon = icon
        self.isDirectory = isDirectory
        self.children = children
    }
}

@Observable
class ExpansionStore {
    // Persist as comma-separated UUID strings
    @AppStorage("expandedNodeIDs") private var storedExpandedIDs: String = ""

    var expandedIDs: Set<UUID> {
        get {
            Set(
                storedExpandedIDs
                    .split(separator: ",")
                    .compactMap { UUID(uuidString: String($0)) }
            )
        }
        set {
            storedExpandedIDs = newValue.map(\.uuidString).joined(separator: ",")
        }
    }

    func isExpanded(_ id: UUID) -> Bool { expandedIDs.contains(id) }

    func toggle(_ id: UUID) {
        if expandedIDs.contains(id) {
            expandedIDs.remove(id)
        } else {
            expandedIDs.insert(id)
        }
    }

    func expand(_ ids: [UUID]) {
        expandedIDs.formUnion(ids)
    }

    func collapseAll() {
        expandedIDs = []
    }
}

struct PersistentOutlineView: View {
    @State private var expansion = ExpansionStore()
    @State private var selection: Set<UUID> = []

    let rootNodes: [TreeNode] = [
        TreeNode(name: "Documents", icon: "folder.fill", isDirectory: true, children: [
            TreeNode(name: "Work", icon: "folder", isDirectory: true, children: [
                TreeNode(name: "Report.pdf", icon: "doc", isDirectory: false),
                TreeNode(name: "Notes.txt", icon: "doc.text", isDirectory: false),
            ]),
            TreeNode(name: "Personal", icon: "folder", isDirectory: true, children: [
                TreeNode(name: "Budget.xlsx", icon: "tablecells", isDirectory: false),
            ]),
        ]),
        TreeNode(name: "Projects", icon: "folder.fill", isDirectory: true, children: [
            TreeNode(name: "App", icon: "hammer", isDirectory: true, children: nil),
        ]),
    ]

    var body: some View {
        List(selection: $selection) {
            ForEach(rootNodes) { node in
                outlineRow(node, depth: 0)
            }
        }
        .listStyle(.sidebar)
        .toolbar {
            ToolbarItem {
                Button("Collapse All") { expansion.collapseAll() }
            }
            ToolbarItem {
                Button("Expand All") { expansion.expand(allDirectoryIDs()) }
            }
        }
    }

    @ViewBuilder
    private func outlineRow(_ node: TreeNode, depth: Int) -> some View {
        if node.isDirectory, node.children != nil {
            DisclosureGroup(
                isExpanded: Binding(
                    get: { expansion.isExpanded(node.id) },
                    set: { _ in expansion.toggle(node.id) }
                )
            ) {
                Label(node.name, systemImage: node.icon)
                    .padding(.leading, CGFloat(depth) * 16)
            } content: {
                ForEach(node.children ?? []) { child in
                    outlineRow(child, depth: depth + 1)
                }
            }
        } else {
            Label(node.name, systemImage: node.icon)
                .padding(.leading, CGFloat(depth) * 16)
                .tag(node.id)
        }
    }

    private func allDirectoryIDs() -> [UUID] {
        var ids: [UUID] = []
        func collect(from nodes: [TreeNode]) {
            for node in nodes {
                if node.isDirectory { ids.append(node.id) }
                if let children = node.children { collect(from: children) }
            }
        }
        collect(from: rootNodes)
        return ids
    }
}
```

**Key points:**

- Manual `DisclosureGroup` (instead of `OutlineGroup` view type) gives per-node expansion binding control, which is required for persistence.
- `@AppStorage` stores the expanded set as a comma-separated string of UUIDs. This is simple and avoids `NSCoding` requirements.
- **Performance**: for 500+ nodes, the string `@AppStorage` read on every view update is negligible (it's a short string). For 10k+ nodes, use a separate `Codable` struct stored via `NSUbiquitousKeyValueStore` or file-based storage.
- The `depth` parameter tracks nesting level for visual indentation. macOS sidebar style adds some indentation automatically, so adjust `depth * 16` to taste.
- `tag(node.id)` on leaf nodes enables selection. For directories in the sidebar, offering selection (highlight the currently browsed folder) vs. just expansion (disclosure triangle) is a design choice.

---

## 4. Form with Grouped Style for Settings-Like Views

A full settings window layout with multiple sections, header/footer annotations, and standard macOS controls.

```swift
import SwiftUI

struct SettingsView: View {
    @AppStorage("notificationsEnabled") private var notificationsEnabled = true
    @AppStorage("startAtLogin") private var startAtLogin = false
    @AppStorage("defaultLocation") private var defaultLocation = ""
    @AppStorage("theme") private var theme = "system"
    @AppStorage("fontSize") private var fontSize = 13.0
    @AppStorage("showHiddenFiles") private var showHiddenFiles = false
    @AppStorage("debugLoggingEnabled") private var debugLoggingEnabled = false
    @State private var showingLocationPicker = false

    var body: some View {
        TabView {
            generalTab
                .tabItem {
                    Label("General", systemImage: "gear")
                }

            appearanceTab
                .tabItem {
                    Label("Appearance", systemImage: "paintbrush")
                }

            advancedTab
                .tabItem {
                    Label("Advanced", systemImage: "gearshape.2")
                }
        }
        .formStyle(.grouped)
        .frame(width: 480, height: 420)
        .padding()
    }

    private var generalTab: some View {
        Form {
            Section {
                Toggle("Enable notifications", isOn: $notificationsEnabled)
                Toggle("Start at login", isOn: $startAtLogin)
            } header: {
                Label("Notifications", systemImage: "bell")
            } footer: {
                Text("Start at login adds the app to System Settings → General → Login Items.")
                    .foregroundStyle(.secondary)
            }

            Section {
                HStack {
                    Text("Default save location")
                    Spacer()
                    Text(defaultLocation.isEmpty ? "Not set" : defaultLocation)
                        .foregroundStyle(.secondary)
                        .lineLimit(1)
                    Button("Choose…") {
                        showingLocationPicker = true
                    }
                    .buttonStyle(.borderless)
                }
            } header: {
                Label("Storage", systemImage: "internaldrive")
            }
        }
        .fileImporter(
            isPresented: $showingLocationPicker,
            allowedContentTypes: [.directory],
            allowsMultipleSelection: false
        ) { result in
            if case .success(let urls) = result, let url = urls.first {
                defaultLocation = url.path
            }
        }
    }

    private var appearanceTab: some View {
        Form {
            Section {
                Picker("Theme", selection: $theme) {
                    Text("System").tag("system")
                    Text("Light").tag("light")
                    Text("Dark").tag("dark")
                }
                .pickerStyle(.segmented)

                HStack {
                    Text("Font size")
                    Slider(value: $fontSize, in: 10...24, step: 1) {
                        EmptyView()
                    } minimumValueLabel: {
                        Text("A").font(.caption)
                    } maximumValueLabel: {
                        Text("A").font(.title2)
                    }
                    Text("\(Int(fontSize)) pt")
                        .frame(width: 44, alignment: .trailing)
                        .foregroundStyle(.secondary)
                }
            } header: {
                Label("Display", systemImage: "display")
            } footer: {
                Text("Font size changes take effect on next launch.")
                    .foregroundStyle(.secondary)
            }
        }
    }

    private var advancedTab: some View {
        Form {
            Section {
                Toggle("Show hidden files", isOn: $showHiddenFiles)
                Toggle("Enable debug logging", isOn: $debugLoggingEnabled)
            } header: {
                Label("Developer", systemImage: "hammer")
            } footer: {
                if debugLoggingEnabled {
                    Button("Export Logs…") {
                        exportLogs()
                    }
                    .buttonStyle(.bordered)
                }
            }

            Section {
                Button("Reset All Settings", role: .destructive) {
                    resetAllSettings()
                }
            }
        }
    }

    private func exportLogs() { /* ... */ }

    private func resetAllSettings() {
        let domain = Bundle.main.bundleIdentifier!
        UserDefaults.standard.removePersistentDomain(forName: domain)
        UserDefaults.standard.synchronize()
    }
}
```

**Key points:**

- Use `TabView` inside a group-style `Form` for a System Settings–like multi-pane appearance. Each tab is a separate `Form` with its own sections.
- `@AppStorage` bridges directly to `UserDefaults`. Keys like `"notificationsEnabled"` are automatically persisted and restored. This is the idiomatic macOS preferences pattern.
- **Section footers** can contain interactive elements (buttons, toggles). Don't limit footers to text.
- `.fileImporter` is used from the `Form` level (not a section row) to present the native file picker. `allowedContentTypes: [.directory]` restricts to folder selection.
- For the reset pattern, `removePersistentDomain(forName:)` clears all `@AppStorage` keys for the app. This is straightforward but irreversible — add a confirmation dialog in production.
- **`.pickerStyle(.segmented)` inside `Form`**: segmented pickers render inline as a row, not as a menu dropdown. This is the expected macOS appearance for theme selection.

---

## 5. ViewThatFits for Responsive Sidebar Width

A responsive layout that switches between a wide detail view (image + metadata columns) and a compact stacked view depending on available space.

```swift
import SwiftUI

struct ResponsiveDetailView: View {
    let item: DetailItem

    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 16) {
                // Header: always shows, but layout adapts
                ViewThatFits(in: .horizontal) {
                    // Layout 1: wide — image left, metadata right
                    HStack(alignment: .top, spacing: 24) {
                        imageView
                        metadataGrid
                        Spacer(minLength: 0)
                    }

                    // Layout 2: medium — image stacked above metadata
                    VStack(alignment: .leading, spacing: 16) {
                        imageView
                        metadataGrid
                    }

                    // Layout 3: narrow — compressed single column
                    VStack(alignment: .leading, spacing: 8) {
                        HStack(alignment: .top, spacing: 12) {
                            Image(systemName: item.icon)
                                .font(.system(size: 32))
                            VStack(alignment: .leading, spacing: 4) {
                                Text(item.name)
                                    .font(.title2.weight(.semibold))
                                Text(item.kind)
                                    .foregroundStyle(.secondary)
                            }
                        }
                        Divider()
                        metadataList
                    }
                }

                Divider()

                // Description section is layout-independent
                VStack(alignment: .leading, spacing: 8) {
                    Text("Description")
                        .font(.headline)
                    Text(item.description)
                        .foregroundStyle(.secondary)
                }
            }
            .padding()
        }
    }

    private var imageView: some View {
        Image(systemName: item.icon)
            .font(.system(size: 64))
            .foregroundStyle(.tint)
            .frame(width: 180, height: 180)
            .background {
                RoundedRectangle(cornerRadius: 12)
                    .fill(.quaternary)
            }
    }

    private var metadataGrid: some View {
        Grid(alignment: .leading, horizontalSpacing: 24, verticalSpacing: 8) {
            GridRow {
                Text("Size")
                    .foregroundStyle(.secondary)
                Text(item.size, format: .fileSize)
            }
            GridRow {
                Text("Created")
                    .foregroundStyle(.secondary)
                Text(item.createdAt, format: .dateTime.day().month().year())
            }
            GridRow {
                Text("Modified")
                    .foregroundStyle(.secondary)
                Text(item.modifiedAt, format: .dateTime.day().month().year())
            }
            GridRow {
                Text("Kind")
                    .foregroundStyle(.secondary)
                Text(item.kind)
            }
        }
    }

    private var metadataList: some View {
        VStack(alignment: .leading, spacing: 6) {
            labelRow("Size:", item.size.formatted(.fileSize))
            labelRow("Created:", item.createdAt.formatted(.dateTime.month().day().year()))
            labelRow("Modified:", item.modifiedAt.formatted(.dateTime.month().day().year()))
            labelRow("Kind:", item.kind)
        }
    }

    private func labelRow(_ label: String, _ value: String) -> some View {
        HStack {
            Text(label)
                .foregroundStyle(.secondary)
                .frame(width: 80, alignment: .trailing)
            Text(value)
        }
    }
}

struct DetailItem {
    let name: String
    let icon: String
    let kind: String
    let size: Int64
    let createdAt: Date
    let modifiedAt: Date
    let description: String
}
```

**Key points:**

- `ViewThatFits(in: .horizontal)` evaluates children against the **available horizontal space**. The first child that fits without truncation is used.
- Order layouts from **most elaborate to simplest**. The wide layout goes first; it's the most desirable. The compact layout is the fallback.
- `ViewThatFits` creates all child views but only inserts the first-fitting one into the view tree. Keep all variants lightweight — avoid `Image(data:)` decoding or network requests in variants that may not be shown.
- For inspector/sidebar contexts, this pattern replaces the old approach of manually checking `GeometryReader` width and branching with `if/else`.
- **Testing**: resize the window slowly and verify each layout threshold. Ensure there's no flickering or sudden jumps — if there are, the fit boundary is too tight and the middle layout is too narrow.
- Combine with `.inspectorColumnWidth(min:ideal:max:)` to control the container's resize range:
  ```swift
  .inspector(isPresented: $showingInspector) {
      ResponsiveDetailView(item: selectedItem)
          .inspectorColumnWidth(min: 240, ideal: 400, max: 600)
  }
  ```
