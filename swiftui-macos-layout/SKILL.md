---
name: swiftui-macos-layout
description: "SwiftUI layout on macOS — VStack/HStack/ZStack, LazyVGrid/LazyHGrid/Grid, ViewThatFits, GeometryReader, Form styles (grouped vs automatic), Table (sortable multi-column), List with source list style, OutlineGroup, DisclosureGroup, window sizing/fixedSize/frame interactions. Use when building macOS SwiftUI layouts, tables, lists, or forms."
:sources: https://developer.apple.com/documentation/swiftui/list, https://developer.apple.com/documentation/swiftui/table, https://developer.apple.com/documentation/swiftui/form, https://developer.apple.com/documentation/swiftui/outlinegroup, https://developer.apple.com/documentation/swiftui/groupedformstyle
researched: 2026-06-06
platform: macOS 26+
---

# SwiftUI Layout on macOS

Build native macOS layouts with SwiftUI's stacking, grid, table, list, and window-sizing primitives. Patterns target macOS 26+ with Swift 6.3; notes indicate backward-compatible APIs where applicable. See [references/layout-patterns.md](references/layout-patterns.md) for deep-dive recipes on Table, List sidebar style, OutlineGroup with persistence, Form, and ViewThatFits.

## Contents

- [Layout Primitives](#layout-primitives)
- [Lazy Containers](#lazy-containers)
- [GeometryReader](#geometryreader)
- [Form Styles](#form-styles)
- [Table](#table)
- [List Styles](#list-styles)
- [OutlineGroup](#outlinegroup)
- [DisclosureGroup](#disclosuregroup)
- [Window Sizing](#window-sizing)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Layout Primitives

SwiftUI's core layout trio: `VStack`, `HStack`, and `ZStack`. On macOS, these behave the same as iOS but interact with AppKit-originated sizing differently — AppKit views auto-sized by `intrinsicContentSize` drive the initial measurement, then SwiftUI resolves the remainder.

### VStack and HStack

```swift
VStack(alignment: .leading, spacing: 8) {
    Text("Title")
        .font(.headline)
    Text("Subtitle")
        .foregroundStyle(.secondary)
}
.frame(maxWidth: .infinity, alignment: .leading)
```

- **`spacing: nil`** (the default when omitted) lets SwiftUI use platform-appropriate spacing. On macOS, this is typically 8–10pt between views; on iOS, it varies by context. Explicit `spacing: 0` removes all gaps.
- **`alignment`** stretches views to the stack's full cross-axis width (`HStack` aligns to full height; `VStack` to full width). Use `.frame(maxWidth:)` to override.
- Stacks are **single-axis**: children are laid out sequentially along the axis. For 2D layout, use `Grid` or `LazyVGrid`.

### ZStack

```swift
ZStack(alignment: .topTrailing) {
    Image("background")
        .resizable()
        .aspectRatio(contentMode: .fill)

    Badge(count: unreadCount)
        .padding(12)
}
.clipsToBounds(true)
```

- Children are layered back-to-front (first child is furthest back).
- All children share the same bounds; use `alignment` to position each child independently.
- Wrap with `.clipsToBounds(true)` when the ZStack's content can exceed its frame (e.g., images).

### Alignment Guides

Custom alignment guides let you align views across different stacks — something `.frame alignment:` can't do.

```swift
HStack(alignment: .cardTop) {
    VStack {
        Text("Label")
            .alignmentGuide(.cardTop) { $0[.top] + 8 }
        Spacer()
    }

    VStack {
        Image(systemName: "star")
        Text("Info")
            .alignmentGuide(.cardTop) { $0[.firstTextBaseline] }
    }
}

extension VerticalAlignment {
    private enum CardTopID: AlignmentID {
        static func defaultValue(in d: ViewDimensions) -> CGFloat {
            d[.top]
        }
    }
    static let cardTop = VerticalAlignment(CardTopID.self)
}
```

- Alignment guides are the only way to align subviews across different stacks or across `NSViewRepresentable` boundaries.
- Define a custom `AlignmentID` enum with a `defaultValue(in:)` static method.

## Lazy Containers

### LazyVGrid

```swift
let columns = [
    GridItem(.adaptive(minimum: 180, maximum: 280), spacing: 12),
    GridItem(.adaptive(minimum: 180, maximum: 280), spacing: 12),
]

ScrollView {
    LazyVGrid(columns: columns, spacing: 12) {
        ForEach(items) { item in
            ItemCard(item: item)
                .frame(height: 120)
        }
    }
    .padding(16)
}
```

- **`.adaptive(minimum:)`**: creates as many columns as fit given the minimum width. On a wide macOS window, you may get 4–5 columns; narrow windows get fewer.
- **`.flexible(minimum:maximum:)`**: one column that stretches between bounds. Use when you want exactly one column that fills available width.
- **`.fixed(200)`**: exact width. Useful for magazine layouts or when the grid must match a specific pixel count.

On macOS, `LazyVGrid` inside a `ScrollView` uses AppKit's `NSScrollView` under the hood. Momentum scrolling and scroll bar behavior are inherited from the system.

### Grid (macOS 16+)

```swift
Grid(horizontalSpacing: 16, verticalSpacing: 12) {
    GridRow {
        Text("Name:")
            .gridColumnAlignment(.trailing)
        TextField("Enter name", text: $name)
            .gridCellAnchor(.leading)
    }

    GridRow {
        Text("Email:")
        TextField("Enter email", text: $email)
    }

    Divider()
        .gridCellUnsizedAxes(.horizontal)

    GridRow {
        Button("Cancel") { dismiss() }
        Button("Save") { save() }
            .buttonStyle(.borderedProminent)
    }
}
```

- `Grid` is ideal for form-like row/column alignment without the overhead of `LazyVGrid` cells.
- `GridRow` aligns columns automatically (like a two-column `Form` but with manual control).
- `gridCellUnsizedAxes(.horizontal)` lets a cell span all columns.
- `gridColumnAlignment(.trailing)` right-aligns the first column (useful for label-value pairs).

### ViewThatFits

`ViewThatFits` picks the first child that fits in the proposed size. On macOS, this is essential for responsive sidebar panes, inspector columns, and toolbar content that must adapt to window resize.

```swift
ViewThatFits(in: .horizontal) {
    // Wide: show full label + value on one row
    HStack {
        Label("Status", systemImage: "circle.fill")
        Text(statusText)
    }

    // Narrow: stack vertically
    VStack(alignment: .leading) {
        Label("Status", systemImage: "circle.fill")
        Text(statusText)
    }
}
```

- `ViewThatFits` evaluates each child in order against the **proposed size** from the parent. The first child whose ideal size fits is used.
- Only the **horizontal** axis is commonly used on macOS; `.both` works but can cause visual jumps if the vertical space also changes.
- **`ViewThatFits` is not lazy** — all children are created, but only one is inserted into the view tree. Avoid expensive init side effects in non-selected variants.

Typical use: responsive inspector sidebar, adaptive toolbar text, narrow/wide content layouts.

## GeometryReader

`GeometryReader` provides the parent's proposed size and coordinate space. On macOS, behavior differs from iOS:

### Caveats

1. **Coordinate space**: macOS views typically have a **flipped** coordinate system (origin at bottom-left in AppKit). SwiftUI normalizes this to top-left, but if you mix `GeometryReader` with `NSViewRepresentable`, the coordinate space of geometry values depends on whether the nearest `NSView` returns `isFlipped`. When in doubt, use `.named("local")` coordinate spaces explicitly.

2. **Zero proposed size**: A `GeometryReader` inside a `ScrollView` may receive a `.zero` proposed size on the scroll axis. Always provide defaults:
   ```swift
   GeometryReader { geometry in
       let width = max(geometry.size.width, 200)
       ContentView(width: width)
   }
   ```

3. **Layout passes**: `GeometryReader` always greedily consumes the full proposed size. It can cause sibling views in a `VStack` to receive `0` height if not explicitly sized. Prefer `.frame` constraints when you only need to react to a size change without altering layout.

### onGeometryChange (macOS 15+)

```swift
struct ResponsiveView: View {
    @State private var containerWidth: CGFloat = 0

    var body: some View {
        VStack {
            if containerWidth > 600 {
                WideLayout()
            } else {
                NarrowLayout()
            }
        }
        .onGeometryChange(for: CGFloat.self) { proxy in
            proxy.size.width
        } action: { newWidth in
            containerWidth = newWidth
        }
    }
}
```

- `onGeometryChange` is the **preferred** alternative to `GeometryReader` when you only need to observe a value (width, height, bounds) and don't need to change layout behavior.
- It does not affect layout — the view retains its normal sizing behavior.

### Window Resize Observation

```swift
struct MainWindow: View {
    @State private var windowSize: CGSize = .zero

    var body: some View {
        MainContent()
            .onGeometryChange(for: CGSize.self) { proxy in
                proxy.size
            } action: { size in
                if size.width > 0 && size.height > 0 {
                    windowSize = size
                }
            }
            .onChange(of: windowSize) { _, newSize in
                guard newSize != .zero else { return }
                UserDefaults.standard.set(newSize.width, forKey: "windowWidth")
            }
    }
}
```

- Combine with `@AppStorage` or `UserDefaults` to persist window size across launches (for windows not using `defaultSize(scene:)`).

## Form Styles

### .grouped vs .automatic on macOS

```swift
// Grouped: native macOS settings-panel appearance
Form {
    Section("Account") {
        TextField("Email", text: $email)
        SecureField("Password", text: $password)
    }
    Section("Preferences") {
        Toggle("Enable notifications", isOn: $notifications)
        Toggle("Start at login", isOn: $startAtLogin)
    }
}
.formStyle(.grouped)

// Automatic: SwiftUI chooses the platform-default style
// On macOS 15+, .automatic is equivalent to .grouped in most
// contexts, but may differ in sheet presentations
Form {
    TextField("Search", text: $searchField)
}
.formStyle(.automatic)
```

- **`.grouped`**: mimics AppKit's grouped `NSTableView` / `NSForm` style. Section headers get a slightly smaller font, rows have subtle rounded background grouping. This is the **settings UI** look.
- **`.automatic`**: the default. macOS resolves this to a grouped-like appearance in standard windows but may use a `FormStyle` closer to `.insetGrouped` in popovers or sheets.
- On macOS, `Form` automatically adds `TextField` labels as row prefixes (no need for manual `Label` wrapping in most cases). Each `TextField`, `Toggle`, `Picker` gets a labeled row.

### Form with Section

```swift
Form {
    Section {
        TextField("Name", text: $name)
        ColorPicker("Accent Color", selection: $accentColor)
    } header: {
        Label("Appearance", systemImage: "paintbrush")
            .font(.headline)
    } footer: {
        Text("Changes take effect immediately.")
            .foregroundStyle(.secondary)
    }

    Section {
        Toggle("Dark mode", isOn: $darkMode)
        Picker("Font size", selection: $fontSize) {
            ForEach([11, 12, 13, 14, 16], id: \.self) {
                Text("\($0)pt").tag(CGFloat($0))
            }
        }
    }
}
```

- On macOS, section headers and footers are left-aligned, not inset like iOS `insetGrouped`. The `header:` content replaces the automatic `<sectionName>` heading.
- Use `Label` in headers for icon+text style matching macOS Settings panes.

### .insetGrouped Form Style (macOS 15+)

The `.insetGrouped` form style is available on macOS 15+ and renders with a rounded-rectangle background and more padding than `.grouped`. It's suitable for settings panes and inspector panels:

```swift
Form {
    Section("Account") {
        TextField("Email", text: $email)
        SecureField("Password", text: $password)
    }
    Section("Preferences") {
        Toggle("Enable notifications", isOn: $notifications)
    }
}
.formStyle(.insetGrouped)
```

**Key difference from `.grouped`:** `.insetGrouped` renders controls with lighter visuals by default (borderless fields, compact switches) and only shows heavier borders/backgrounds on hover. Pickers render as borderless popup buttons.

### GroupedFormStyle Width Limit (macOS 15+)

`GroupedFormStyle` imposes a **maximum content width of ~600pt** on macOS 15+. Tables and wide content inside `.grouped` forms may not stretch to full width. Use `.plain` or `.insetGrouped` if you need full-width content, or place the `Form` inside a `ScrollView` with explicit frame.

## Table

`Table` is macOS-only (macOS 12+). It provides a multi-column, sortable, native-feeling table with `NSTableView` under the hood.

### Multi-Column Sortable Table

```swift
struct FileTable: View {
    @State private var sortOrder = [TableColumnSortComparator<FileItem>(\.name)]
    @State private var selection = Set<FileItem.ID>()

    var body: some View {
        Table(of: FileItem.self, selection: $selection, sortOrder: $sortOrder) {
            TableColumn("Name", value: \.name) { item in
                HStack {
                    Image(systemName: item.icon)
                        .foregroundStyle(.secondary)
                    Text(item.name)
                }
            }
            .width(min: 200, ideal: 280, max: 400)

            TableColumn("Size", value: \.size) { item in
                Text(item.size, format: .fileSize)
            }
            .width(80)

            TableColumn("Modified", value: \.modified) { item in
                Text(item.modified, format: .dateTime.day().month().year())
            }
            .width(120)

            TableColumn("Type", value: \.kind) { item in
                Text(item.kind)
                    .foregroundStyle(.secondary)
            }
            .width(60)
        } rows: {
            ForEach(filteredItems) { item in
                TableRow(item)
            }
        }
        .tableStyle(.bordered) // .inset also available on macOS
    }
}
```

### TableColumn Configuration

- **`.width(min:ideal:max:)`**: Controls column sizing. `ideal` is the starting width; user can resize between `min` and `max`.
- **`.width(100)`**: Fixed width. Prevents user resizing.
- **Value-based columns** (e.g., `TableColumn("Name", value: \.name)`) are automatically sortable. Custom sort comparators use key paths:
  ```swift
  @State private var sortOrder: [KeyPathComparator<FileItem>] = [
      .init(\.name, order: .forward),
      .init(\.size, order: .reverse),
  ]
  ```

### Row Selection

- **Single selection**: Use `Table(selection: $selection)` with `selection: Binding<FileItem.ID?>`.
  On macOS 13+ the simpler overload is `Binding<Set<FileItem.ID>>` for multi-selection.
- **Multi-selection**: `Binding<Set<T.ID>>` — `Table` supports ⌘-click and Shift-click automatically.

### Performance at Scale

- `Table` uses **cell reuse** (like `NSTableView`). 10k+ rows perform well because only visible rows are materialized.
- Avoid complex views in cell closures. Use `Text`, `Label`, or lightweight `HStack` — no `GeometryReader` or deeply nested stacks inside cells.
- Sort at the data layer using `SortDescriptor` (macOS 14+) instead of sorting in SwiftUI to avoid re-sorting on every state change.

### TableColumnCustomization (macOS 14+)

Enable drag-to-reorder and show/hide columns with built-in customization. Each `TableColumn` gets a `customizationID:` and users can configure visibility and order:

```swift
@State private var columnCustomization = TableColumnCustomization()

Table(of: FileItem.self, selection: $selection, sortOrder: $sortOrder) {
    TableColumn("Name", value: \.name) { item in Text(item.name) }
        .width(min: 200, ideal: 280)
        .customizationID("name")

    TableColumn("Size", value: \.size) { item in Text(item.size.formatted()) }
        .width(80)
        .customizationID("size")
        .customizationHidden(true) // hidden by default; user can show

    TableColumn("Modified", value: \.modified) { item in Text(item.modified, format: .dateTime) }
        .width(120)
        .customizationID("modified")
} rows: {
    ForEach(items) { TableRow($0) }
}
.tableStyle(.bordered)
.tableColumnCustomization($columnCustomization)
```

Persist customization with `@AppStorage`:

```swift
@AppStorage("fileTableColumns") private var columnCustomization: TableColumnCustomization = .init()
```

### SortDescriptor (macOS 14+)

For type-safe sorting, use `SortDescriptor` instead of `KeyPathComparator`:

```swift
@State private var sortDescriptors: [SortDescriptor<FileItem>] = [
    .init(\.name, order: .forward),
    .init(\.size, order: .reverse)
]

// In your data layer:
items.sort(using: sortDescriptors)
```

`SortDescriptor` supports dynamic multi-column sorting and integrates with `@FetchRequest` for Core Data.

### DisclosureTableRow (macOS 13+)

Create expandable rows in a `Table`:

```swift
Table(of: FileItem.self) {
    TableColumn("Name", value: \.name) { item in
        DisclosureTableRow(item.isDirectory ? "Folder" : "File") {
            ForEach(item.children ?? []) { child in
                TableRow(child)
            }
        }
    }
    .width(min: 200)
} rows: {

### List Background Customization

To customize the background of a `List` on macOS (iOS 16+/macOS 13+), hide the system background first:

```swift
List {
    ForEach(items) { item in
        ItemRow(item: item)
    }
}
.scrollContentBackground(.hidden) // removes system background
.background(Color(.windowBackgroundColor)) // apply custom background
```

This is essential for sidebar lists where you want a custom translucent color or gradient. Without `.scrollContentBackground(.hidden)`, the system draws its background on top of yours.
    ForEach(rootItems) { TableRow($0) }
}
```

## List Styles

### .sidebar (Source List)

```swift
NavigationSplitView {
    List(selection: $selectedItem) {
        Section("Library") {
            Label("All Files", systemImage: "doc.on.doc")
                .tag(SidebarItem.allFiles)
            Label("Recent", systemImage: "clock")
                .tag(SidebarItem.recent)
        }

        Section("Collections") {
            ForEach(collections) { collection in
                Label(collection.name, systemImage: "folder")
                    .tag(SidebarItem.collection(collection.id))
            }
        }
    }
    .listStyle(.sidebar)
    .navigationTitle("Browser")
    .toolbar {
        ToolbarItem {
            Button(action: newCollection) {
                Image(systemName: "plus")
            }
        }
    }
} detail: {
    DetailView(item: selectedItem)
}
```

- `.sidebar` applies the translucent source-list styling with rounded row highlight. Always pair with `List(selection:)` for a properly functioning source list.
- On macOS, `.sidebar` automatically adds keyboard navigation (arrow keys) and selection highlight styling.
- Section headers in `.sidebar` style use a smaller, all-caps font like Finder's sidebar.

### Selection Management

```swift
struct Sidebar: View {
    @State private var selection: SidebarItem?
    @State private var searchText = ""

    var body: some View {
        List(selection: $selection) {
            ForEach(filteredItems) { item in
                NavigationLink(value: item.id) {
                    Label(item.name, systemImage: item.icon)
                }
                .contextMenu {
                    Button("Rename") { rename(item) }
                    Button("Delete", role: .destructive) { delete(item) }
                }
                .tag(item.id)
            }
        }
        .searchable(text: $searchText)
        .onChange(of: selection) { _, newValue in
            guard let id = newValue else { return }
            navigateTo(id)
        }
    }
}
```

- Use `NavigationLink(value:)` inside `List` for `NavigationSplitView`/`NavigationStack` integration. The `.tag()` on each row type must be hashable.
- **Never** use `onTapGesture` for list row selection — it bypasses keyboard navigation and accessibility. Use the `selection:` parameter and respond to changes via `.onChange`.

### .insetGrouped on macOS

```swift
List {
    Section {
        ForEach(items) { item in
            ItemRow(item: item)
        }
    } header: {
        Text("Items")
    }
}
.listStyle(.insetGrouped) // macOS: insetGrouped renders similar to .sidebar with more padding
```

- On macOS 15+, `.insetGrouped` renders with **more padding** than `.sidebar` and uses a rounded-rectangle background. It's suitable for secondary lists (e.g., inspector panels) but not for primary navigation sidebars.

## OutlineGroup

`OutlineGroup` renders hierarchical data with disclosure triangles for expansion. macOS provides native `NSOutlineView`-style behavior.

```swift
struct FileSystemOutline: View {
    @State private var expansion: TreeNode.ID??
    @State private var selection = Set<TreeNode.ID>()

    var body: some View {
        List(selection: $selection) {
            OutlineGroup(rootNode.children, id: \.id, children: \.children) { node in
                Label(node.name, systemImage: node.isDirectory ? "folder" : "doc")
                    .tag(node.id)
            }
        }
        .listStyle(.sidebar)
        .outlineGroupExpansion($expansion)
    }
}
```

- `OutlineGroup(data:children:)` takes the root array, an `id` key path, and a `children` key path for the recursive structure.
- The content closure is called for every node at every depth.
- **`.outlineGroupExpansion($binding)`** (macOS 26+): bind an `Item.ID??` to track and programmatically control which item is expanded. Pass `nil` to collapse all.
- For expansion state persistence, store a `Set<Item.ID>` of expanded IDs in `@AppStorage` and map to/from the binding.

### Custom Expansion State

```swift
class ExpansionModel: ObservableObject {
    @Published var expandedIDs: Set<UUID> = []

    func isExpanded(_ id: UUID) -> Bool { expandedIDs.contains(id) }
    func toggle(_ id: UUID) {
        if expandedIDs.contains(id) { expandedIDs.remove(id) }
        else { expandedIDs.insert(id) }
    }
}

struct CustomOutline: View {
    @StateObject private var expansion = ExpansionModel()

    var body: some View {
        List {
            ForEach(rootNodes) { node in
                DisclosureGroup(
                    isExpanded: Binding(
                        get: { expansion.isExpanded(node.id) },
                        set: { _ in expansion.toggle(node.id) }
                    )
                ) {
                    Label(node.name, systemImage: node.icon)
                } content: {
                    ForEach(node.children ?? []) { child in
                        DisclosureGroup(
                            isExpanded: Binding(
                                get: { expansion.isExpanded(child.id) },
                                set: { _ in expansion.toggle(child.id) }
                            )
                        ) {
                            Label(child.name, systemImage: child.icon)
                        } content: {
                            ForEach(child.children ?? []) { leaf in
                                Label(leaf.name, systemImage: leaf.icon)
                                    .padding(.leading, 24)
                            }
                        }
                    }
                }
            }
        }
    }
}
```

## DisclosureGroup

Standalone `DisclosureGroup` for settings-like expandable sections (not hierarchical):

```swift
struct SettingsView: View {
    @State private var showingAdvanced = false
    @State private var showingDebug = false

    var body: some View {
        Form {
            Section {
                Toggle("Enable feature", isOn: $featureEnabled)
                Picker("Quality", selection: $quality) {
                    Text("Low").tag(Quality.low)
                    Text("High").tag(Quality.high)
                }
            }

            DisclosureGroup("Advanced", isExpanded: $showingAdvanced) {
                VStack(alignment: .leading, spacing: 8) {
                    Toggle("Experimental mode", isOn: $experimental)
                    Toggle("Debug logging", isOn: $debugLogging)
                    HStack {
                        Text("Cache size:")
                        TextField("MB", value: $cacheSize, format: .number)
                            .frame(width: 60)
                    }
                }
                .padding(.vertical, 4)
            }

            DisclosureGroup("Debug", isExpanded: $showingDebug) {
                VStack(alignment: .leading) {
                    Button("Reset cache") { resetCache() }
                    Button("Export logs") { exportLogs() }
                    Button("Simulate crash", role: .destructive) { simulateCrash() }
                }
            }
        }
        .formStyle(.grouped)
        .frame(width: 400)
    }
}
```

- Disclosure groups inside `Form` auto-indent content with a subtle animation.
- Use one `@State` var per disclosure group, or a single `Set<AnyHashable>` with `\.self` tags if the groups are dynamic.

## Window Sizing

### Scene-Level Sizing

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .defaultSize(width: 900, height: 600)
        .windowResizability(.contentSize) // prevents resizing beyond content's ideal size
    }
}
```

- `defaultSize(width:height:)` sets the initial window size. macOS 14+.
- `.windowResizability(.contentSize)`: the user can't resize below the window's `idealSize`. Other values: `.contentMinSize` (min only), `.automatic` (default — full user control).

### View-Level Frame Constraints

```swift
struct ContentView: View {
    var body: some View {
        VStack {
            HeaderView()

            MainContent()
                .frame(minWidth: 300, maxWidth: .infinity, minHeight: 200)

            FooterView()
                .frame(height: 40)
        }
        .frame(minWidth: 500, maxWidth: .infinity, minHeight: 400)
    }
}
```

- **`frame(min:max:)`**: clamps the view's layout between soft bounds. SwiftUI suggests these sizes but may override if space is constrained.
- **`fixedSize()`**: prevents the view from being compressed or stretched. Useful for buttons that shouldn't resize and for views that break when compressed below their natural size.
- **`fixedSize(horizontal:vertical:)`**: fix one axis only, e.g., `.fixedSize(horizontal: false, vertical: true)` lets width flex but locks height.

### Resistance and Compression Priorities

```swift
HStack {
    Text("Label")
        .layoutPriority(1) // resists compression

    Spacer()

    Text("Value")
        .layoutPriority(0) // compresses first
}
```
- `layoutPriority(_:)` controls which views get space first when competing. Higher priority = more space allocated.
- Default priority is `0`. Any value > 0 gives a view precedence.
- Use sparingly — most layout needs are met with `frame(min:max:)` and `Spacer`.

## Constraints & Gotchas

1. **`Table` requires macOS 12+.** There is no `Table` on iOS. For iOS/macOS cross-platform code, use `#available(macOS 12, *)` or `NavigationSplitView` with conditional compilation.

2. **`GeometryReader` greedy sizing.** In a `VStack`, a `GeometryReader` without explicit sizing expands to all available space, potentially leaving `0` height for siblings. Use `.frame(height:)` on the `GeometryReader` or prefer `onGeometryChange`.

3. **Sidebar style differences.** `.listStyle(.sidebar)` on macOS renders with a translucent background and rounded highlight. On iPadOS, it renders as a plain list with no special styling. Don't assume consistent visual appearance.

4. **`Form` on macOS is not a container.** Unlike UIKit's `NSForm`, SwiftUI `Form` is a layout container that styles its children. It wraps content in a vertical table-like layout. `Form` inside `Form` creates nested forms — avoid this.

5. **`DisclosureGroup` animation inside `Form`**: Disclosure group expansion inside a `Form` uses a vertical slide animation by default. There's no built-in way to customize this on macOS. Use custom `DisclosureGroupStyle` if a different animation is needed.

6. **`LazyVGrid` inside `ScrollView`**: On macOS, the grid inherits scroll acceleration from AppKit. There is no direct API to disable momentum scrolling.

7. **`ViewThatFits` evaluation order**: Children are evaluated top-to-bottom. Put the most common layout first to minimize unused view creation.

8. **`OutlineGroup` with large trees**: Each expanded node instantiates all its children's views. For trees with 10k+ items, consider lazy loading children via `onAppear`.

9. **Coordinate space in SwiftUI 26+**: `.global`, `.local`, `.named()` coordinate spaces are the same concept as iOS. AppKit-originated views may report geometry in flipped coordinates. Test geometry-dependent code with `NSViewRepresentable` wrappers.

## Common Mistakes

1. **Using `GeometryReader` only to read width/height.** Prefer `onGeometryChange(for:action:)` — it doesn't alter layout and communicates intent.

2. **Applying `.frame(maxWidth: .infinity)` to a `List` or `Table`.** These are scrollable containers; `.frame` controls their outer size, not cell width. Column sizing goes on `TableColumn(.width:)`.

3. **Forgetting `selection:` parameter on sidebar `List`.** Without it, `NavigationLink(value:)` inside the list won't highlight the selected row, and `NavigationSplitView` won't connect sidebar selection to detail content.

4. **Using `List` with `ForEach` + `onTapGesture` instead of `selection:`.** This breaks keyboard navigation, accessibility VoiceOver, and multi-selection. Always use the `selection:` parameter for interactive lists.

5. **Placing a `Form` inside a `ScrollView`.** `Form` is already scrollable on macOS when content overflows. Wrapping it in `ScrollView` causes double-scroll and layout conflicts.

6. **Assuming `.grouped` and `.automatic` are identical.** On macOS 15+ they're close, but in popovers and sheets, `.automatic` may drop the grouped background. Explicit `.grouped` is the safer choice for settings windows.

7. **Sorting `Table` data in the `ForEach`/`Table` closure.** Sort at the data model layer (`SortDescriptor` or `.sorted()`) and feed the sorted array into the table. Sorting in the cell closure causes meaningless computation and prevents stable column sort indicators.

## Review Checklist

- [ ] `Table` usage is behind `#available(macOS 12, *)` if targeting macOS 11 or below
- [ ] `GeometryReader` isn't used where `onGeometryChange` would suffice
- [ ] Sidebar `List` has `selection:` parameter and `.listStyle(.sidebar)`
- [ ] `ViewThatFits` children are ordered most-common-first
- [ ] `Form` is not nested inside another `Form`
- [ ] `Form` is not wrapped in a `ScrollView`
- [ ] Window `defaultSize` is set for non-resizable windows
- [ ] `fixedSize()` is used only for views that shouldn't stretch/compress
- [ ] Sort descriptors are applied at the data layer, not in cell closures
- [ ] `OutlineGroup` expansion state is persisted via `@AppStorage` or equivalent
- [ ] Layout priorities are used sparingly and documented with inline comments

## References

- [SwiftUI Table documentation](https://developer.apple.com/documentation/swiftui/table)
- [SwiftUI List documentation](https://developer.apple.com/documentation/swiftui/list)
- [SwiftUI Grid documentation](https://developer.apple.com/documentation/swiftui/grid)
- [SwiftUI Form documentation](https://developer.apple.com/documentation/swiftui/form)
- [SwiftUI OutlineGroup documentation](https://developer.apple.com/documentation/swiftui/outlinegroup)
- [SwiftUI ViewThatFits documentation](https://developer.apple.com/documentation/swiftui/viewthatfits)
- [SwiftUI GeometryReader documentation](https://developer.apple.com/documentation/swiftui/geometryreader)
- [SwiftUI onGeometryChange documentation](https://developer.apple.com/documentation/swiftui/view/ongeometrychange(for:of:action:))
- [WWDC 22: SwiftUI on macOS](https://developer.apple.com/videos/play/wwdc2022/10050/)
- [SwiftUI GroupedFormStyle documentation](https://developer.apple.com/documentation/swiftui/groupedformstyle)
- [SwiftUI TableColumnCustomization documentation](https://developer.apple.com/documentation/swiftui/tablecolumncustomization)
- References: [references/layout-patterns.md](references/layout-patterns.md)
