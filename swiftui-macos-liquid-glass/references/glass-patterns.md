# Glass Patterns

Complete working patterns for Liquid Glass on macOS 26. Each pattern includes the full SwiftUI implementation, explanation of the glass behavior, and gotchas specific to that arrangement.

---

## Contents

- [1. GlassEffectContainer with Toolbar Buttons](#1-glasseffectcontainer-with-toolbar-buttons)
- [2. Translucent Sidebar with Glass and Vibrancy](#2-translucent-sidebar-with-glass-and-vibrancy)
- [3. NSVisualEffectView Wrapper for Custom Vibrancy](#3-nsvisualeffectview-wrapper-for-custom-vibrancy)
- [4. Inspector Panel with Glass Background](#4-inspector-panel-with-glass-background)

---

## 1. GlassEffectContainer with Toolbar Buttons

Group related toolbar controls into unified glass surfaces using `GlassEffectContainer` and `glassEffectUnion`. This creates the segmented control look familiar from macOS 26 system apps.

```swift
import SwiftUI

struct ToolbarGlassView: View {
    @State private var viewMode: ViewMode = .grid
    @State private var sortOrder: SortOrder = .name

    var body: some View {
        VStack(spacing: 0) {
            // Toolbar area
            GlassEffectContainer {
                HStack(spacing: 0) {
                    // View mode segment — unified glass
                    HStack(spacing: 2) {
                        ForEach(ViewMode.allCases) { mode in
                            Button {
                                withAnimation(.spring(duration: 0.3)) {
                                    viewMode = mode
                                }
                            } label: {
                                Image(systemName: mode.icon)
                                    .frame(width: 28, height: 24)
                            }
                            .buttonStyle(.plain)
                            .foregroundStyle(viewMode == mode ? .primary : .secondary)
                            .glassEffect(displayMode: .whenMatched)
                            .glassEffectUnion(id: "viewMode", union: true)
                        }
                    }
                    .padding(4)
                    .clipShape(RoundedRectangle(cornerRadius: 8, style: .continuous))

                    Divider()
                        .frame(height: 20)

                    // Sort segment — separate unified glass
                    Menu {
                        ForEach(SortOrder.allCases) { order in
                            Button(order.label) { sortOrder = order }
                        }
                    } label: {
                        Label(sortOrder.label, systemImage: "arrow.up.arrow.down")
                            .labelStyle(.iconOnly)
                            .frame(width: 28, height: 24)
                    }
                    .glassEffect(displayMode: .whenMatched)
                    .glassEffectUnion(id: "sort", union: true)
                    .padding(4)

                    Spacer()

                    // Actions — independent glass surface
                    Button(action: addItem) {
                        Image(systemName: "plus")
                    }
                    .glassEffect()
                    .padding(4)

                    Button(action: shareItems) {
                        Image(systemName: "square.and.arrow.up")
                    }
                    .glassEffect()
                    .padding(4)
                }
                .padding(.horizontal, 12)
                .padding(.vertical, 6)
            }
            .windowBackgroundDragBehavior(.enable)

            // Content area
            ContentGrid(viewMode: viewMode, sortOrder: sortOrder)
        }
    }

    private func addItem() { /* ... */ }
    private func shareItems() { /* ... */ }
}

enum ViewMode: String, CaseIterable, Identifiable {
    case grid, list, columns
    var id: String { rawValue }
    var icon: String {
        switch self {
        case .grid: return "square.grid.2x2"
        case .list: return "list.bullet"
        case .columns: return "rectangle.split.3x1"
        }
    }
}

enum SortOrder: String, CaseIterable, Identifiable {
    case name, date, size
    var id: String { rawValue }
    var label: String {
        switch self {
        case .name: return "Name"
        case .date: return "Date Modified"
        case .size: return "Size"
        }
    }
}
```

### How It Works

- **`GlassEffectContainer`** wraps all toolbar items. Without it, each `.glassEffect()` button renders as an independent glass tile.
- **`glassEffectUnion(id: "viewMode", union: true)`** merges the grid/list/column buttons into a single glass surface. The system renders them as one continuous panel with rounded corners.
- **`glassEffectUnion(id: "sort", union: true)`** makes the sort menu its own glass surface, visually separated from the view mode segment.
- **The plus and export buttons** don't use a union — they render as individual glass buttons.
- **`displayMode: .whenMatched`** ensures glass only activates when the view is inside a `GlassEffectContainer`. If the same view is reused outside a container, it renders as a normal button.
- **`windowBackgroundDragBehavior(.enable)`** lets users drag the window by clicking empty areas of the toolbar glass.

### Gotchas

- The union ID must be unique within a container. Reusing the same ID across different visual groups merges them into one surface.
- `.padding()` inside the union group affects spacing between elements; `.padding()` outside affects the glass surface margin.
- Animated state changes (switching `viewMode`) animate smoothly within the glass union. The glass boundary adjusts to fit the selected state.

---

## 2. Translucent Sidebar with Glass and Vibrancy

A sidebar that uses `.glassBackgroundDisplayMode(.vibrant)` to show the window background through the list, with automatic text contrast adaptation.

```swift
import SwiftUI

struct SidebarGlassView: View {
    @State private var selectedFolder: Folder?
    @Bindable var library: Library

    var body: some View {
        NavigationSplitView {
            List(library.folders, selection: $selectedFolder) { folder in
                Label {
                    VStack(alignment: .leading, spacing: 2) {
                        Text(folder.name)
                            .foregroundStyle(.primary)
                        Text("\(folder.itemCount) items")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                } icon: {
                    Image(systemName: folder.icon)
                        .foregroundStyle(.secondary)
                        .frame(width: 20)
                }
                .padding(.vertical, 2)
                .tag(folder.id)
            }
            .listStyle(.sidebar)
            .navigationTitle("Library")
            .glassBackgroundDisplayMode(.vibrant)
            .navigationSplitViewColumnWidth(min: 180, ideal: 220, max: 300)
            .toolbar {
                ToolbarItem(placement: .primaryAction) {
                    Menu {
                        Button("New Folder", systemImage: "folder.badge.plus") {
                            library.addFolder()
                        }
                        Button("Import…", systemImage: "square.and.arrow.down") {
                            library.showImport = true
                        }
                    } label: {
                        Image(systemName: "ellipsis.circle")
                    }
                    .glassEffect()
                }
            }
        } detail: {
            if let folder = selectedFolder {
                FolderDetailView(folder: folder)
            } else {
                ContentUnavailableView(
                    "Select a Folder",
                    systemImage: "folder",
                    description: Text("Choose a folder from the sidebar to view its contents.")
                )
            }
        }
    }
}
```

### How It Works

- **`.glassBackgroundDisplayMode(.vibrant)`** is the key modifier. It makes the list background translucent with vibrancy — text and icons pick up contrast from whatever is behind the sidebar.
- **`.foregroundStyle(.primary)`** on text ensures it adapts to both light/dark mode and the vibrancy effect. The system renders primary text with higher contrast against the vibrant background.
- **`.foregroundStyle(.secondary)`** on the caption text gives appropriate hierarchy while still adapting to vibrancy.
- The **toolbar button** uses `.glassEffect()` for its own glass treatment, separate from the sidebar background.

### Custom Sidebar Row Background

To customize the selected row appearance over vibrant glass:

```swift
List(library.folders, selection: $selectedFolder) { folder in
    SidebarRow(folder: folder)
        .listRowBackground(
            selectedFolder?.id == folder.id
                ? Color.accentColor.opacity(0.3)
                : Color.clear
        )
}
.glassBackgroundDisplayMode(.vibrant)
```

### Gotchas

- **`.vibrant` vs `.transparent`:** `.vibrant` applies strong tinting that adapts to the background. `.transparent` shows more of the raw background with minimal processing. For sidebars, `.vibrant` is almost always the right choice — `.transparent` can look washed out.
- **Sidebar width:** Too narrow (< 160pt) and the glass effect has no room to show refraction. Keep sidebar columns ≥ 180pt for the glass to be visually effective.
- **`List` vs `ScrollView` + `LazyVStack`:** `.glassBackgroundDisplayMode` works with `List` and `ScrollView`. It does not work with raw `LazyVStack` — the modifier needs a scrollable container.

---

## 3. NSVisualEffectView Wrapper for Custom Vibrancy

When you need vibrancy materials not exposed through SwiftUI's `.glassBackgroundDisplayMode`, bridge to `NSVisualEffectView` via `NSViewRepresentable`.

```swift
import SwiftUI
import AppKit

struct VisualEffectBackground: NSViewRepresentable {
    var material: NSVisualEffectView.Material
    var blendingMode: NSVisualEffectView.BlendingMode = .withinWindow
    var state: NSVisualEffectView.State = .active

    func makeNSView(context: Context) -> NSVisualEffectView {
        let view = NSVisualEffectView()
        view.material = material
        view.blendingMode = blendingMode
        view.state = state
        view.wantsLayer = true
        return view
    }

    func updateNSView(_ nsView: NSVisualEffectView, context: Context) {
        nsView.material = material
        nsView.blendingMode = blendingMode
        nsView.state = state
    }
}

// MARK: - Convenience materials

extension VisualEffectBackground {
    static var sidebar: VisualEffectBackground {
        VisualEffectBackground(material: .sidebar)
    }

    static var titlebar: VisualEffectBackground {
        VisualEffectBackground(material: .titlebar)
    }

    static var popover: VisualEffectBackground {
        VisualEffectBackground(material: .popover)
    }

    static var hud: VisualEffectBackground {
        VisualEffectBackground(material: .hudWindow)
    }

    static var menu: VisualEffectBackground {
        VisualEffectBackground(material: .menu, blendingMode: .behindWindow)
    }
}
```

### Usage: Full-Window Vibrancy Behind Custom Content

```swift
struct CustomPanelView: View {
    var body: some View {
        ZStack {
            // AppKit vibrancy background
            VisualEffectBackground(material: .underWindowBackground)
                .ignoresSafeArea()

            // SwiftUI content on top
            VStack(spacing: 16) {
                Text("Custom Panel")
                    .font(.title2)
                    .fontWeight(.semibold)

                Text("This panel uses NSVisualEffectView for .underWindowBackground vibrancy — a material not available through SwiftUI's glassBackgroundDisplayMode.")
                    .foregroundStyle(.secondary)
                    .multilineTextAlignment(.center)

                Button("Close") { /* ... */ }
                    .buttonStyle(.borderedProminent)
            }
            .padding(32)
            .frame(width: 360, height: 240)
        }
    }
}
```

### Usage: Layered Vibrancy with Embedded Content

```swift
struct LayeredVibrancyView: View {
    var body: some View {
        ZStack {
            // Base vibrancy layer
            VisualEffectBackground(material: .windowBackground)
                .ignoresSafeArea()

            HStack(spacing: 0) {
                // Sidebar with stronger vibrancy
                ZStack {
                    VisualEffectBackground(material: .sidebar)

                    VStack(alignment: .leading) {
                        Text("Sidebar")
                            .font(.headline)
                            .foregroundStyle(.primary)
                    }
                    .padding()
                }
                .frame(width: 200)

                // Content area with lighter vibrancy
                ZStack {
                    VisualEffectBackground(material: .underPageBackground)

                    Text("Document content area")
                        .foregroundStyle(.primary)
                }
            }
        }
    }
}
```

### Dynamic Material Switching

```swift
struct AdaptiveVibrancyView: View {
    @Environment(\.colorScheme) private var colorScheme

    var body: some View {
        ZStack {
            VisualEffectBackground(
                material: colorScheme == .dark
                    ? .underWindowBackground
                    : .windowBackground
            )
            .ignoresSafeArea()
            .animation(.easeInOut(duration: 0.2), value: colorScheme)

            ContentView()
        }
    }
}
```

### Gotchas

- **No Liquid Glass refraction.** `NSVisualEffectView` renders a traditional blur. It does not participate in the glass refraction system. Content behind the vibrancy view appears blurred but not distorted.
- **`blendingMode` matters.** `.withinWindow` blends with content in the same window. `.behindWindow` blends with content behind the window (desktop, other apps). Use `.behindWindow` for menu and HUD materials.
- **Layer-backed views.** `NSVisualEffectView` requires `wantsLayer = true`. The wrapper sets this in `makeNSView`. If you subclass or modify, ensure it stays set.
- **Performance.** Each `NSVisualEffectView` is a separate render pass. Multiple layered vibrancy views cost more than a single SwiftUI glass container. Prefer SwiftUI glass for new code.
- **Coordinate system.** `NSVisualEffectView` uses AppKit's bottom-left origin by default. When mixing with SwiftUI (top-left), be aware of flipped coordinates in layout calculations.

---

## 4. Inspector Panel with Glass Background

A standard macOS 26 inspector panel using `.inspector(isPresented:)` with custom glass treatment and vibrancy-aware content.

```swift
import SwiftUI

struct DocumentInspector: View {
    @Binding var document: Document
    @State private var showInspector = true

    var body: some View {
        DocumentCanvas(document: $document)
            .inspector(isPresented: $showInspector) {
                InspectorSidebar(document: $document)
            }
    }
}

struct InspectorSidebar: View {
    @Binding var document: Document
    @State private var selectedTab: InspectorTab = .info

    var body: some View {
        TabView(selection: $selectedTab) {
            InfoTab(document: $document)
                .tabItem {
                    Label("Info", systemImage: "info.circle")
                }
                .tag(InspectorTab.info)

            StyleTab(document: $document)
                .tabItem {
                    Label("Style", systemImage: "paintbrush")
                }
                .tag(InspectorTab.style)

            ExportTab(document: $document)
                .tabItem {
                    Label("Export", systemImage: "square.and.arrow.up")
                }
                .tag(InspectorTab.export)
        }
        .glassBackgroundDisplayMode(.vibrant)
        .inspectorColumnWidth(min: 260, ideal: 300, max: 380)
    }
}

enum InspectorTab: String, CaseIterable {
    case info, style, export
}
```

### Custom Inspector Without `.inspector()`

For a standalone inspector-style panel that isn't attached to the system inspector:

```swift
struct CustomInspectorPanel: View {
    @Binding var selection: CanvasElement?
    @State private var isVisible = true

    var body: some View {
        HStack(spacing: 0) {
            CanvasView(selection: $selection)
                .frame(maxWidth: .infinity)

            if isVisible {
                GlassEffectContainer {
                    VStack(alignment: .leading, spacing: 0) {
                        // Inspector header
                        HStack {
                            Text("Inspector")
                                .font(.headline)
                                .foregroundStyle(.primary)

                            Spacer()

                            Button {
                                withAnimation(.spring(duration: 0.35)) {
                                    isVisible = false
                                }
                            } label: {
                                Image(systemName: "sidebar.right")
                            }
                            .glassEffect()
                        }
                        .padding(.horizontal, 16)
                        .padding(.vertical, 12)

                        Divider()

                        // Inspector content
                        ScrollView {
                            VStack(alignment: .leading, spacing: 16) {
                                if let element = selection {
                                    ElementInspector(element: element)
                                } else {
                                    Text("Select an element to inspect")
                                        .foregroundStyle(.secondary)
                                        .frame(maxWidth: .infinity, alignment: .center)
                                        .padding(.top, 40)
                                }
                            }
                            .padding(16)
                        }
                    }
                }
                .frame(width: 300)
                .glassBackgroundDisplayMode(.vibrant)
            } else {
                // Collapsed state — show a toggle button
                VStack {
                    Button {
                        withAnimation(.spring(duration: 0.35)) {
                            isVisible = true
                        }
                    } label: {
                        Image(systemName: "sidebar.right")
                    }
                    .glassEffect()
                    .padding(8)

                    Spacer()
                }
                .frame(width: 44)
                .glassBackgroundDisplayMode(.vibrant)
            }
        }
        .windowBackgroundDragBehavior(.enabled)
    }
}

struct ElementInspector: View {
    @Bindable var element: CanvasElement

    var body: some View {
        VStack(alignment: .leading, spacing: 12) {
            GroupBox("Position") {
                Grid(alignment: .leading) {
                    GridRow {
                        Text("X:").foregroundStyle(.secondary)
                        TextField("X", value: $element.x, format: .number)
                            .textFieldStyle(.roundedBorder)
                    }
                    GridRow {
                        Text("Y:").foregroundStyle(.secondary)
                        TextField("Y", value: $element.y, format: .number)
                            .textFieldStyle(.roundedBorder)
                    }
                }
            }

            GroupBox("Appearance") {
                VStack(alignment: .leading, spacing: 8) {
                    ColorPicker("Fill", selection: $element.fillColor)
                    Slider(value: $element.opacity, in: 0...1) {
                        Text("Opacity").foregroundStyle(.secondary)
                    }
                }
            }
        }
    }
}
```

### How It Works

- **`.inspector(isPresented:)`** on the parent view tells SwiftUI to place the attached view in the system inspector position (right side of the window on LTR).
- **`.glassBackgroundDisplayMode(.vibrant)`** on the inspector content makes it translucent. The canvas shows through the inspector edges.
- **`.inspectorColumnWidth(min:ideal:max:)`** sets the inspector width. Users can drag to resize within the specified range.
- **The custom panel variant** doesn't use `.inspector()`. Instead, it's an `HStack` with a glass sidebar that can collapse to a slim toggle strip. The `GlassEffectContainer` groups the header and content into one glass surface.
- **Collapse animation** uses `.spring(duration: 0.35)` for a natural-feeling transition. The glass surface reflows during the animation.

### Gotchas

- **System inspector vs custom panel:** `.inspector()` gives you the standard macOS 26 inspector behavior (system-managed presentation, keyboard shortcut ⌘⌥I). The custom panel gives you full control but must manage its own visibility state.
- **Inspector content scrolling:** Always wrap inspector content in a `ScrollView`. The inspector has fixed width but variable height — content that overflows needs to scroll.
- **TabView in inspector:** `TabView` inside an inspector works, but each tab's content should be independently scrollable. Don't put a `ScrollView` inside a tab that's already inside a `ScrollView`.
- **Glass on GroupBox:** `GroupBox` renders its own background. Applying `.glassEffect()` to a `GroupBox` creates a glass-on-glass effect that looks muddy. Instead, apply glass to the container and let `GroupBox` render transparently.
