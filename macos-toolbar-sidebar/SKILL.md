---
name: macos-toolbar-sidebar
description: "Implement macOS toolbars, sidebars, and inspectors using SwiftUI. Covers NavigationSplitView three-column layouts, toolbar item placements (.principal, .navigation, .primaryAction, .secondaryAction, .status), searchable integration, Inspector panels, List sidebar styling, OutlineGroup hierarchies, and NSSplitViewController interop. Use when building Mac toolbar-based UI, sidebar navigation, inspector panels, or custom split-view layouts."
sources: https://developer.apple.com/documentation/swiftui/navigationsplitview, https://developer.apple.com/videos/play/wwdc2023/10161/, https://www.createwithswift.com/presenting-an-inspector-with-swiftui/, https://stackoverflow.com/questions/79914546/custom-sidebar-toggle-with-integrated-chevron-menu
researched: 2026-06-05
platform: macOS 26+
---

# macOS Toolbar & Sidebar

Toolbar, sidebar, and inspector patterns for Mac apps using SwiftUI on macOS 26+. All patterns assume Swift 6.3 and target macOS 26 or later.

For NavigationSplitView basics (two-column setup, selection binding, destination mapping), see the `swiftui-navigation` skill.
For SwiftUI List basics, see the `swiftui-layout-components` skill.
For AppKit fundamentals, see the `appkit-foundations` skill.

## Contents

- [NavigationSplitView Three-Column Layout](#navigationsplitview-three-column-layout)
- [Toolbar Placements](#toolbar-placements)
- [ToolbarTitleMenu](#toolbartitlemenu)
- [Search in Toolbar](#search-in-toolbar)
- [Inspector Panel](#inspector-panel)
- [List Sidebar Style](#list-sidebar-style)
- [OutlineGroup](#outlinegroup)
- [HSplitView / NSSplitViewController Interop](#hsplitview--nssplitviewcontroller-interop)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## NavigationSplitView Three-Column Layout

On macOS, `NavigationSplitView` supports three columns: sidebar, content, and detail. Use the three-closure initializer and control visibility with `@State` bindings and `NavigationSplitViewVisibility`.

```swift
struct MainView: View {
    @State private var selectedItem: Item?
    @State private var columnVisibility: NavigationSplitViewVisibility = .all

    var body: some View {
        NavigationSplitView(columnVisibility: $columnVisibility) {
            // Sidebar
            List(items, id: \.self, selection: $selectedItem) { item in
                Text(item.name)
                    .tag(item)
            }
            .navigationSplitViewColumnWidth(min: 180, ideal: 220, max: 300)
        } content: {
            // Content column (middle)
            if let selectedItem {
                ItemContentView(item: selectedItem)
            } else {
                ContentUnavailableView("Select an Item",
                    systemImage: "sidebar.leading")
            }
        } detail: {
            // Detail column (right)
            if let selectedItem {
                ItemDetailView(item: selectedItem)
            } else {
                Text("No selection")
                    .foregroundStyle(.secondary)
            }
        }
    }
}
```

**Column visibility**: Pass a `NavigationSplitViewVisibility` binding to control which columns are shown. Values: `.all`, `.doubleColumn`, `.detailOnly`. On macOS the user can toggle columns via a toolbar button automatically when you provide a binding. The system persists the user's preferred column widths and restores them on relaunch.

**Column width constraints**: Use `.navigationSplitViewColumnWidth(min:ideal:max:)` on each column. The `ideal` width is the initial size; users can resize within `min...max` bounds. Omit `max` for unconstrained growth.

**Visibility with two columns**: If you only need sidebar + detail (no content column), use the two-closure initializer and set visibility to `.doubleColumn` or `.all` — the content column collapses automatically when unused.

## Toolbar Placements

SwiftUI on macOS supports placement-specific semantics for `ToolbarItem`. Not all placements are available on iOS — the following are macOS-only behaviors.

| Placement | macOS Behavior |
|---|---|
| `.principal` | Centered in the toolbar; used for the document title or primary control. Replaces the window title area. |
| `.navigation` | Leading edge, used for back/forward or navigation controls. |
| `.primaryAction` | Leading side after navigation items; primary actions for the current context. |
| `.secondaryAction` | Trailing side; secondary actions (share, export, etc.). |
| `.status` | Status bar area; progress indicators, status text. |
| `.confirmationAction` | Confirmation/accept buttons (e.g., Save, Done). macOS-only grouping behavior. |

```swift
.toolbar {
    // Leading: navigation + primary actions
    ToolbarItem(placement: .navigation) {
        Button(action: goBack) {
            Label("Back", systemImage: "chevron.left")
        }
    }

    ToolbarItem(placement: .primaryAction) {
        Button(action: addItem) {
            Label("Add", systemImage: "plus")
        }
    }

    // Principal: centered title area
    ToolbarItem(placement: .principal) {
        Text("Documents")
            .font(.headline)
    }

    // Trailing: secondary actions
    ToolbarItem(placement: .secondaryAction) {
        ShareLink(item: exportURL) {
            Label("Share", systemImage: "square.and.arrow.up")
        }
    }

    // Confirmation / status
    ToolbarItem(placement: .confirmationAction) {
        Button("Save") { saveDocument() }
            .keyboardShortcut("s")
    }

    ToolbarItem(placement: .status) {
        if isProcessing {
            ProgressView()
                .controlSize(.small)
        }
    }
}
```

**Combined placements**: When multiple items share a placement, they appear in the order declared, grouped together in the toolbar region. Use `ToolbarSpacer()` to create visual separation within a region.

```swift
ToolbarItem(placement: .primaryAction) { editButton }
ToolbarSpacer()
ToolbarItem(placement: .primaryAction) { shareButton }
```

The spacer distributes `.primaryAction` items apart from each other automatically on macOS 26+.

## ToolbarTitleMenu

`ToolbarTitleMenu` renders the document title as a clickable menu in the `.principal` toolbar position. This is the standard macOS pattern for document-path menus (showing parent folders, favorites, and recent locations).

```swift
.toolbar {
    ToolbarItem(placement: .principal) {
        ToolbarTitleMenu {
            Button("Duplicate") { duplicateDocument() }
            Button("Move To…") { moveDocument() }
            Divider()

            Menu("Share") {
                ShareLink("Send Copy", item: documentURL)
                ExportButton(document: document)
            }

            Divider()
            Button("Rename") { renameDocument() }
        }
    }
}
```

The menu trigger shows the current document name and a chevron indicator, matching the standard macOS document window appearance. You can combine this with `.navigationTitle()` — `ToolbarTitleMenu` overrides the navigation title display in the `.principal` position while the plain text title is used for the window title bar text when the toolbar is hidden.

## Search in Toolbar

Place a `SearchField` in the toolbar with a specific `ToolbarItemPlacement.search` placement on macOS. Combine with `.searchable()` on the content view for the full search experience including suggestions and tokens.

```swift
struct ContentView: View {
    @State private var searchText = ""
    @State private var selectedTokens: [TagToken] = []

    var body: some View {
        NavigationSplitView {
            List(filteredItems) { item in
                Text(item.name)
            }
            .searchable(
                $searchText,
                tokens: $selectedTokens,
                suggestedTokens: .constant(TagToken.allCases),
                placement: .toolbar,
                prompt: "Search items"
            ) {
                // Suggestion UI
                ForEach(suggestions) { suggestion in
                    Text(suggestion.name)
                        .searchCompletion(suggestion.name)
                        .tag(suggestion.tag)
                }
            }
        } detail: {
            DetailView()
        }
    }

    private var filteredItems: [Item] {
        items.filter { item in
            let matchesSearch = searchText.isEmpty ||
                item.name.localizedStandardContains(searchText)
            let matchesTokens = selectedTokens.isEmpty ||
                selectedTokens.allSatisfy { item.tags.contains($0) }
            return matchesSearch && matchesTokens
        }
    }
}
```

**Token fields**: Use `tokens:` binding with `suggestedTokens` and `.searchCompletion()` to enable tokenized search. On macOS, tokens appear as pill-shaped chips in the search field, matching the system Spotlight/search experience.

**Search placement options**: `.toolbar` renders in the toolbar area; `.sidebar` places it in the sidebar header; `.automatic` chooses based on context.

**Keyboard shortcut**: When `.searchable(placement: .toolbar)` is active, the system provides `⌘F` shortcut automatically.

## Inspector Panel

The Inspector is a slide-in panel anchored to the trailing edge, standard for document properties, formatting, and metadata. SwiftUI provides native support via `inspector(isPresented:content:)` and `inspectorColumnWidth`.

```swift
struct DocumentView: View {
    @State private var showingInspector = true

    var body: some View {
        DocumentEditor(document: document)
            .inspector(isPresented: $showingInspector) {
                InspectorContent(document: document)
                    .inspectorColumnWidth(min: 250, ideal: 300, max: 400)
            }
            .toolbar {
                ToolbarItem {
                    Toggle("Inspector", systemImage: "sidebar.trailing",
                           isOn: $showingInspector)
                        .keyboardShortcut("i", modifiers: [.command, .option])
                }
            }
    }
}
```

**Key points**:
- The inspector overlays the detail column and slides in from the trailing edge.
- It works within `NavigationSplitView` — you do not manage a separate column.
- Width is controlled with `.inspectorColumnWidth(min:ideal:max:)` applied inside the inspector content.
- Toggle visibility with a toolbar button (standard pattern) using `.keyboardShortcut("i", modifiers: [.command, .option])`.
- The system persists the inspector's presented state and width across launches.

**Programmatic control**: Bind `showingInspector` to state to show/hide from code (e.g., when entering a formatting mode that requires the inspector).

```swift
.onReceive(notificationPublisher) { _ in
    withAnimation { showingInspector = true }
}
```

## List Sidebar Style

Apply `.listStyle(.sidebar)` to Lists used as navigation sidebars. On macOS, this gives the standard translucent sidebar appearance with proper padding and selection highlighting.

```swift
List(folders, id: \.self, selection: $selectedFolder) { folder in
    Label(folder.name, systemImage: folder.icon)
}
.listStyle(.sidebar)
```

**Custom sidebar row background**: Use `.listRowBackground()` with `sidebarRowBackground(isSelected:)` to apply the platform-appropriate selection highlight.

```swift
List(items, selection: $selected) { item in
    ItemRow(item: item)
        .listRowBackground(
            sidebarRowBackground(isSelected: selected?.id == item.id)
        )
}
```

**Disclosure groups in sidebars**: Use `DisclosureGroup` for expandable sections within a sidebar List.

```swift
List {
    DisclosureGroup("Favorites") {
        ForEach(favorites) { item in
            Label(item.name, systemImage: item.icon)
                .tag(item)
        }
    }

    DisclosureGroup("Folders") {
        ForEach(folders) { folder in
            Label(folder.name, systemImage: "folder")
                .tag(folder)
        }
    }
}
.listStyle(.sidebar)
```

On macOS, disclosure groups in the `.sidebar` style use the standard chevron disclosure with proper indentation.

## OutlineGroup

`OutlineGroup` displays hierarchical data in a sidebar with automatic disclosure triangles. Use this for file trees, nested categories, or any tree structure.

```swift
struct FileTreeView: View {
    let files: [FileNode]
    @State private var selected: FileNode.ID?
    @State private var expanded: Set<FileNode.ID> = []

    var body: some View {
        List(files, id: \.id, children: \.children,
             selection: $selected, expansion: $expanded) { node in
            Label(node.name, systemImage: node.icon)
                .help(node.path)
        }
        .listStyle(.sidebar)
    }
}

@Observable
class FileNode {
    let id: UUID
    let name: String
    let icon: String
    var children: [FileNode]?

    init(id: UUID = UUID(), name: String, icon: String,
         children: [FileNode]? = nil) {
        self.id = id
        self.name = name
        self.icon = icon
        self.children = children
    }
}
```

**Using `OutlineGroup` with `ForEach`**: For more control over the row content:

```swift
List {
    OutlineGroup(files, id: \.id, children: \.children) { node in
        FileRow(node: node)
            .contextMenu {
                Button("Reveal in Finder") { revealInFinder(node) }
                Button("Get Info") { showInfo(node) }
            }
    }
}
.listStyle(.sidebar)
```

**Expansion state**: Bind `expansion:` to track which nodes are expanded. The binding uses the node's `ID` type, so ensure your model is `Identifiable` with a stable ID.

## HSplitView / NSSplitViewController Interop

When SwiftUI's `NavigationSplitView` is insufficient — for example, you need multiple split dividers with independent pane behavior, or you need AppKit split view features not exposed in SwiftUI — wrap `NSSplitViewController` in an `NSViewControllerRepresentable`.

```swift
struct SplitViewContainer: NSViewControllerRepresentable {
    @Binding var leftContent: AnyView
    @Binding var rightContent: AnyView
    @Binding var dividerPosition: CGFloat

    func makeNSViewController(context: Context) -> NSSplitViewController {
        let controller = NSSplitViewController()

        let leftItem = NSSplitViewItem(sidebarWithViewController: NSHostingController(
            rootView: leftContent
        ))
        leftItem.minimumThickness = 200
        leftItem.maximumThickness = 400
        leftItem.canCollapse = true

        let rightItem = NSSplitViewItem(viewController: NSHostingController(
            rootView: rightContent
        ))

        controller.addSplitViewItem(leftItem)
        controller.addSplitViewItem(rightItem)

        // Divider style
        controller.splitView.dividerStyle = .thin

        return controller
    }

    func updateNSViewController(
        _ controller: NSSplitViewController,
        context: Context
    ) {
        // Update child content when SwiftUI state changes
        if let left = controller.splitViewItems.first,
           let hosting = left.viewController as? NSHostingController<AnyView> {
            hosting.rootView = leftContent
        }
        if let right = controller.splitViewItems.dropFirst().first,
           let hosting = right.viewController as? NSHostingController<AnyView> {
            hosting.rootView = rightContent
        }
    }
}
```

**When to use NSSplitViewController**:
- More than 3 columns with independent dividers
- Panes that need `canCollapse` behavior with auto-layout
- Custom divider drawing or behavior
- Integrating with AppKit-based panes (e.g., an `NSTextView` rich text editor in one pane)

**When to stay with NavigationSplitView**:
- Standard 2-3 column layouts
- You want system-managed column visibility toggling
- You need the `.sidebar` list style integration
- You want automatic state restoration for column widths

**Multiple split panes with NSSplitViewController**: Add more `NSSplitViewItem`s to the controller for 4+ panes. Each item gets its own `minimumThickness`, `maximumThickness`, and `canCollapse` properties.

```swift
let middleItem = NSSplitViewItem(contentListWithViewController: NSHostingController(
    rootView: middleContent
))
middleItem.minimumThickness = 300
controller.addSplitViewItem(middleItem)
```

## Common Mistakes

1. **Using two-closure `NavigationSplitView` for three-column layouts** — the two-closure form only supports sidebar + detail. Use the three-closure form with `columnVisibility:` for content column support.

2. **Forgetting `columnVisibility` binding** — without it, the user cannot toggle columns from the toolbar. Always pass a `@State` binding.

3. **Applying `.listStyle(.sidebar)` to non-sidebar Lists** — the `.sidebar` style is optimized for navigation. Using it on a settings list or form produces incorrect padding and selection behavior.

4. **Using `.principal` placement for non-title items** — `.principal` is reserved for the document title or primary control. Putting action buttons there breaks the standard macOS toolbar layout.

5. **Not setting `inspectorColumnWidth`** — without explicit width constraints, the inspector may appear too narrow or too wide. Always set `ideal` width.

6. **Placing `.searchable()` on the wrong view** — for toolbar search, apply `.searchable(placement: .toolbar)` to the content column view, not the sidebar or detail.

7. **Using `HSplitView` directly instead of `NSSplitViewController`** — `HSplitView` is a lower-level AppKit class. Prefer `NSSplitViewController` for standard split layouts; it handles divider dragging, collapse, and auto-layout automatically.

8. **Not providing `children` key path in `OutlineGroup`** — the `children` parameter must return an optional array (`[Node]?`). Returning an empty array instead of `nil` for leaf nodes causes them to show disclosure triangles with no children.

9. **Mixing `ToolbarItem` placement semantics with iOS patterns** — `.confirmationAction` and `.status` have macOS-specific grouping. Test on Mac; iOS placement behavior differs.

10. **Forgetting `@MainActor` on `NSViewControllerRepresentable`** — AppKit view controller creation must happen on the main actor. Annotate the struct or the `makeNSViewController` method.

## Review Checklist

- [ ] Three-column `NavigationSplitView` uses the three-closure initializer with `columnVisibility:` binding
- [ ] Column widths set with `.navigationSplitViewColumnWidth(min:ideal:max:)`
- [ ] Toolbar items use macOS-appropriate placements (`.principal`, `.navigation`, `.primaryAction`, `.secondaryAction`, `.status`, `.confirmationAction`)
- [ ] `ToolbarTitleMenu` used for document title menus in `.principal` position
- [ ] `.searchable(placement: .toolbar)` applied to the content column view
- [ ] Inspector uses `inspector(isPresented:content:)` with `inspectorColumnWidth`
- [ ] Inspector toggle button in toolbar with keyboard shortcut
- [ ] Sidebar Lists use `.listStyle(.sidebar)`
- [ ] `OutlineGroup` provides correct `children` key path returning optional array
- [ ] `NSSplitViewController` interop used only when `NavigationSplitView` is insufficient
- [ ] `NSViewControllerRepresentable` is `@MainActor`-safe

## References

- Toolbar customization, inspector patterns, sidebar styling, and NSSplitViewController interop: [references/toolbar-sidebar-patterns.md](references/toolbar-sidebar-patterns.md)
- NavigationSplitView basics: see `swiftui-navigation` skill
- SwiftUI List basics: see `swiftui-layout-components` skill
- AppKit fundamentals: see `appkit-foundations` skill
