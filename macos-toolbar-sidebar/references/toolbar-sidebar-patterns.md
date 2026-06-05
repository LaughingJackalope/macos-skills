# Toolbar & Sidebar Patterns — Reference

Complete patterns for macOS toolbar customization, inspector panels, sidebar styling, and NSSplitViewController interop. Swift 6.3 / macOS 26+.

---

## 1. Custom Toolbar with Grouped Actions

Group related toolbar items with `ToolbarGroup` (macOS 26+) for visual separation and overflow behavior.

```swift
.toolbar {
    ToolbarGroup(placement: .primaryAction) {
        Button(action: newDocument) {
            Label("New", systemImage: "doc.badge.plus")
        }
        .help("Create a new document")

        Button(action: openDocument) {
            Label("Open", systemImage: "folder")
        }
        .help("Open an existing document")
    }

    ToolbarGroup(placement: .secondaryAction) {
        ShareLink(item: documentURL) {
            Label("Share", systemImage: "square.and.arrow.up")
        }

        Button(action: exportDocument) {
            Label("Export", systemImage: "square.and.arrow.down")
        }
    }
}
```

`ToolbarGroup` collects items into a visual cluster. When the window is too narrow to show all items, grouped items collapse into a single overflow menu entry.

---

## 2. Toolbar with Search and Filter

Combine a search field with filter controls in the toolbar for a Finder-like experience.

```swift
struct BrowserView: View {
    @State private var searchText = ""
    @State private var selectedCategory: Category = .all
    @State private var sortOrder: SortOrder = .nameAscending

    var body: some View {
        NavigationSplitView {
            List(filteredItems) { item in
                ItemRow(item: item)
            }
            .searchable($searchText, prompt: "Search files")
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    Picker("Category", selection: $selectedCategory) {
                        ForEach(Category.allCases) { category in
                            Text(category.name).tag(category)
                        }
                    }
                    .pickerStyle(.menu)
                }

                ToolbarItem(placement: .secondaryAction) {
                    Menu {
                        Picker("Sort By", selection: $sortOrder) {
                            ForEach(SortOrder.allCases) { order in
                                Label(order.title, systemImage: order.icon)
                                    .tag(order)
                            }
                        }
                    } label: {
                        Label("Sort", systemImage: "arrow.up.arrow.down")
                    }
                }
            }
        } detail: {
            DetailView()
        }
    }

    private var filteredItems: [Item] {
        items
            .filter { selectedCategory == .all || $0.category == selectedCategory }
            .filter { searchText.isEmpty || $0.name.localizedStandardContains(searchText) }
            .sorted(by: sortOrder.comparator)
    }
}
```

---

## 3. Document-Based Toolbar with Title Menu

Standard document window toolbar with path menu, document status, and save indicator.

```swift
struct DocumentWindow: View {
    @Bindable var document: MyDocument
    @State private var showingInspector = true
    @State private var isSaving = false

    var body: some View {
        DocumentEditor(document: document)
            .inspector(isPresented: $showingInspector) {
                DocumentInspector(document: document)
                    .inspectorColumnWidth(min: 260, ideal: 300, max: 400)
            }
            .navigationTitle(document.displayName)
            .toolbar {
                ToolbarItem(placement: .principal) {
                    ToolbarTitleMenu {
                        Button("Duplicate") { document.duplicate() }
                        Button("Lock") { document.lock() }
                        Divider()

                        Menu("Move To") {
                            ForEach(Folder.favorites) { folder in
                                Button(folder.name) { document.move(to: folder) }
                            }
                        }

                        Divider()
                        ShareLink("Send Copy", item: document.fileURL)
                    }
                }

                ToolbarItem(placement: .status) {
                    if isSaving {
                        Label("Saving…", systemImage: "arrow.triangle.2.circlepath")
                            .labelStyle(.titleAndIcon)
                    } else if let lastSaved = document.lastSaved {
                        Text("Saved \(lastSaved, style: .relative) ago")
                            .foregroundStyle(.secondary)
                    }
                }

                ToolbarItem(placement: .confirmationAction) {
                    Button("Done") { saveAndClose() }
                        .keyboardShortcut(.return, modifiers: .command)
                }

                ToolbarItem(placement: .secondaryAction) {
                    Toggle("Inspector", systemImage: "sidebar.trailing",
                           isOn: $showingInspector)
                        .keyboardShortcut("i", modifiers: [.command, .option])
                }
            }
    }

    private func saveAndClose() {
        isSaving = true
        Task {
            try await document.save()
            isSaving = false
        }
    }
}
```

---

## 4. Multi-Token Search with Scope

Advanced search with token fields and scope filtering, matching Spotlight-style behavior.

```swift
struct SearchView: View {
    @State private var searchText = ""
    @State private var tokens: [SearchToken] = []
    @State private var scope: SearchScope = .all

    var body: some View {
        NavigationSplitView {
            List(filteredResults) { result in
                SearchResultRow(result: result)
            }
            .searchable(
                $searchText,
                tokens: $tokens,
                suggestedTokens: .constant(SearchToken.allCases),
                placement: .toolbar,
                prompt: "Search"
            ) {
                // Suggested completions
                ForEach(completions) { completion in
                    Label(completion.title, systemImage: completion.icon)
                        .searchCompletion(completion.text)
                        .tag(completion.token)
                }
            }
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    Picker("Scope", selection: $scope) {
                        Text("All").tag(SearchScope.all)
                        Text("Name").tag(SearchScope.name)
                        Text("Content").tag(SearchScope.content)
                        Text("Tags").tag(SearchScope.tags)
                    }
                    .pickerStyle(.segmented)
                    .frame(maxWidth: 300)
                }
            }
        } detail: {
            SearchDetailView()
        }
    }

    private var filteredResults: [SearchResult] {
        SearchEngine.search(
            query: searchText,
            tokens: tokens,
            scope: scope
        )
    }
}

enum SearchToken: Hashable, CaseIterable {
    case pdf, image, document, folder

    var title: String {
        switch self {
        case .pdf: return "PDF"
        case .image: return "Image"
        case .document: return "Document"
        case .folder: return "Folder"
        }
    }
}
```

---

## 5. Inspector with Multiple Tabs

Inspector panel with tabbed content for different property categories.

```swift
struct TabbedInspector: View {
    @Bindable var document: Document
    @State private var selectedTab: InspectorTab = .format

    var body: some View {
        TabView(selection: $selectedTab) {
            FormatInspector(document: document)
                .tabItem {
                    Label("Format", systemImage: "textformat")
                }
                .tag(InspectorTab.format)

            MetadataInspector(document: document)
                .tabItem {
                    Label("Info", systemImage: "info.circle")
                }
                .tag(InspectorTab.metadata)

            HistoryInspector(document: document)
                .tabItem {
                    Label("History", systemImage: "clock")
                }
                .tag(InspectorTab.history)
        }
        .inspectorColumnWidth(min: 260, ideal: 300, max: 400)
        .padding()
    }
}

enum InspectorTab: String, CaseIterable {
    case format, metadata, history
}
```

---

## 6. Sidebar with Custom Sections and Badges

Sidebar with section headers, badge counts, and context menus.

```swift
struct SidebarView: View {
    @State private var selectedItem: SidebarItem?
    @State private var expandedSections: Set<SectionID> = [.favorites, .folders]

    var body: some View {
        List(selection: $selectedItem) {
            Section(isExpanded: Binding(
                get: { expandedSections.contains(.favorites) },
                set: { isExpanded in
                    if isExpanded { expandedSections.insert(.favorites) }
                    else { expandedSections.remove(.favorites) }
                }
            )) {
                ForEach(favorites) { item in
                    Label {
                        Text(item.name)
                    } icon: {
                        Image(systemName: item.icon)
                            .foregroundStyle(.yellow)
                    }
                    .badge(item.unreadCount > 0 ? item.unreadCount : nil)
                    .tag(SidebarItem.favorite(item))
                    .contextMenu {
                        Button("Remove from Favorites") {
                            removeFavorite(item)
                        }
                        Divider()
                        Button("Get Info") { showInfo(item) }
                    }
                }
            } header: {
                HStack {
                    Text("Favorites")
                    Spacer()
                    if favorites.contains(where: \.hasUnread) {
                        Circle()
                            .fill(.blue)
                            .frame(width: 8, height: 8)
                    }
                }
            }

            Section("Folders") {
                OutlineGroup(folderTree, id: \.id, children: \.children) { node in
                    FolderRow(node: node)
                        .tag(SidebarItem.folder(node))
                }
            }

            Section("Tags") {
                ForEach(tags) { tag in
                    Label(tag.name, systemImage: "tag")
                        .badge(tag.count)
                        .foregroundStyle(tag.color)
                        .tag(SidebarItem.tag(tag))
                }
            }
        }
        .listStyle(.sidebar)
        .searchable(text: $searchText, prompt: "Filter sidebar")
    }
}
```

---

## 7. NSSplitViewController with Three Panes

Full NSSplitViewController interop for complex multi-pane layouts.

```swift
struct ThreePaneSplitView: NSViewControllerRepresentable {
    @Binding var sidebarContent: AnyView
    @Binding var mainContent: AnyView
    @Binding var utilityContent: AnyView
    @Binding var sidebarVisible: Bool

    func makeNSViewController(context: Context) -> NSSplitViewController {
        let controller = NSSplitViewController()
        controller.splitView.dividerStyle = .thin

        // Sidebar pane
        let sidebarItem = NSSplitViewItem(sidebarWithViewController: NSHostingController(
            rootView: sidebarContent
        ))
        sidebarItem.minimumThickness = 180
        sidebarItem.maximumThickness = 350
        sidebarItem.canCollapse = true
        sidebarItem.isCollapsed = !sidebarVisible
        sidebarItem.automaticMaximumThickness = 250

        // Main content pane
        let mainItem = NSSplitViewItem(viewController: NSHostingController(
            rootView: mainContent
        ))
        mainItem.minimumThickness = 400

        // Utility pane
        let utilityItem = NSSplitViewItem(utilityWithViewController: NSHostingController(
            rootView: utilityContent
        ))
        utilityItem.minimumThickness = 200
        utilityItem.maximumThickness = 400
        utilityItem.canCollapse = true

        controller.addSplitViewItem(sidebarItem)
        controller.addSplitViewItem(mainItem)
        controller.addSplitViewItem(utilityItem)

        return controller
    }

    func updateNSViewController(
        _ controller: NSSplitViewController,
        context: Context
    ) {
        guard controller.splitViewItems.count == 3 else { return }

        // Update sidebar
        if let sidebar = controller.splitViewItems[0].viewController
            as? NSHostingController<AnyView> {
            sidebar.rootView = sidebarContent
            controller.splitViewItems[0].isCollapsed = !sidebarVisible
        }

        // Update main
        if let main = controller.splitViewItems[1].viewController
            as? NSHostingController<AnyView> {
            main.rootView = mainContent
        }

        // Update utility
        if let utility = controller.splitViewItems[2].viewController
            as? NSHostingController<AnyView> {
            utility.rootView = utilityContent
        }
    }
}
```

**Usage**:

```swift
struct AppView: View {
    @State private var sidebarVisible = true

    var body: some View {
        ThreePaneSplitView(
            sidebarContent: AnyView(SidebarContent()),
            mainContent: AnyView(MainContent()),
            utilityContent: AnyView(UtilityContent()),
            sidebarVisible: $sidebarVisible
        )
        .toolbar {
            ToolbarItem {
                Toggle("Sidebar", systemImage: "sidebar.leading",
                       isOn: $sidebarVisible)
                    .keyboardShortcut("s", modifiers: [.command, .option])
            }
        }
    }
}
```

---

## 8. OutlineGroup with Drag and Drop

Hierarchical sidebar with drag-and-drop reordering using `NSItemProvider`.

```swift
struct DraggableFileTree: View {
    @Bindable var model: FileTreeModel
    @State private var selected: Set<FileNode.ID> = []
    @State private var expanded: Set<FileNode.ID> = []

    var body: some View {
        List(model.rootNodes, id: \.id, children: \.children,
             selection: $selected, expansion: $expanded) { node in
            FileRow(node: node)
                .draggable(node.itemProvider) {
                    FileRow(node: node)
                        .opacity(0.7)
                }
                .dropDestination(for: FileNode.self) { items, location in
                    handleDrop(items: items, target: node)
                } isTargeted: { isTargeted in
                    model.highlightTarget(node.id, isTargeted: isTargeted)
                }
        }
        .listStyle(.sidebar)
    }

    private func handleDrop(items: [FileNode], target: FileNode) -> Bool {
        for item in items {
            model.move(item, into: target)
        }
        return true
    }
}

extension FileNode {
    var itemProvider: NSItemProvider {
        let provider = NSItemProvider()
        provider.registerObject(self, visibility: .ownProcess)
        return provider
    }
}
```

---

## 9. Sidebar Toggle with Integrated Chevron Menu

Custom sidebar toggle button that combines the collapse action with a context menu for sidebar options.

```swift
struct SidebarToggleButton: View {
    @Binding var isSidebarVisible: Bool
    @State private var selectedFilter: SidebarFilter = .all

    var body: some View {
        Menu {
            Toggle("Show Sidebar", isOn: $isSidebarVisible)

            Divider()

            Picker("Filter", selection: $selectedFilter) {
                ForEach(SidebarFilter.allCases) { filter in
                    Label(filter.title, systemImage: filter.icon)
                        .tag(filter)
                }
            }

            Divider()

            Button("Sidebar Options…") {
                showSidebarPreferences()
            }
        } label: {
            Image(systemName: "sidebar.leading")
        }
        .keyboardShortcut("s", modifiers: [.command, .option])
    }
}
```

---

## 10. Adaptive Column Width

NavigationSplitView with columns that adapt to size class changes.

```swift
struct AdaptiveColumnsView: View {
    @State private var selectedItem: Item?
    @State private var visibility: NavigationSplitViewVisibility = .all
    @Environment(\.horizontalSizeClass) private var sizeClass

    var body: some View {
        NavigationSplitView(columnVisibility: $visibility) {
            SidebarContent(selection: $selectedItem)
                .navigationSplitViewColumnWidth(
                    min: 160,
                    ideal: sizeClass == .compact ? 200 : 240,
                    max: 320
                )
        } content: {
            ContentColumn(item: selectedItem)
                .navigationSplitViewColumnWidth(
                    min: 280,
                    ideal: 360,
                    max: 500
                )
        } detail: {
            DetailColumn(item: selectedItem)
        }
        .onChange(of: sizeClass) { _, newSizeClass in
            if newSizeClass == .compact {
                visibility = .doubleColumn
            }
        }
    }
}
```

---

## 11. Toolbar with Validation

Toolbar buttons that validate state before enabling actions.

```swift
struct ValidatingToolbar: View {
    @Bindable var document: Document
    @State private var canUndo = false
    @State private var canRedo = false

    var body: some View {
        DocumentEditor(document: document)
            .inspector(isPresented: $showingInspector) {
                InspectorContent(document: document)
                    .inspectorColumnWidth(min: 260, ideal: 300, max: 400)
            }
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    Button(action: document.undo) {
                        Label("Undo", systemImage: "arrow.uturn.backward")
                    }
                    .disabled(!canUndo)
                    .help(canUndo ? "Undo last action" : "Nothing to undo")
                }

                ToolbarItem(placement: .primaryAction) {
                    Button(action: document.redo) {
                        Label("Redo", systemImage: "arrow.uturn.forward")
                    }
                    .disabled(!canRedo)
                }

                ToolbarItem(placement: .confirmationAction) {
                    Button("Save") {
                        Task { try await document.save() }
                    }
                    .disabled(!document.hasUnsavedChanges)
                    .keyboardShortcut("s")
                }
            }
            .onReceive(document.undoStackPublisher) { stack in
                canUndo = stack.canUndo
                canRedo = stack.canRedo
            }
    }
}
```

---

## 12. Minimal Sidebar (Icon-Only)

Compact sidebar that shows only icons, expanding on hover — useful for utility windows.

```swift
struct MinimalSidebar: View {
    @State private var selectedTool: Tool?
    @State private var isExpanded = false

    var body: some View {
        NavigationSplitView {
            List(Tool.allCases, id: \.self, selection: $selectedTool) { tool in
                Label {
                    if isExpanded { Text(tool.name) }
                } icon: {
                    Image(systemName: tool.icon)
                }
                .labelStyle(isExpanded ? .titleAndIcon : .iconOnly)
                .tag(tool)
            }
            .listStyle(.sidebar)
            .navigationSplitViewColumnWidth(
                min: isExpanded ? 160 : 44,
                ideal: isExpanded ? 200 : 44,
                max: 240
            )
            .onHover { hovering in
                withAnimation(.easeInOut(duration: 0.2)) {
                    isExpanded = hovering
                }
            }
        } detail: {
            if let tool = selectedTool {
                tool.view
            } else {
                ContentUnavailableView("Select a Tool",
                    systemImage: "sidebar.leading")
            }
        }
    }
}
```
