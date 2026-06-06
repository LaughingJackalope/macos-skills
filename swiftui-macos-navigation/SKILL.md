---
name: swiftui-macos-navigation
description: "SwiftUI navigation on macOS — NavigationSplitView (three-column, column visibility, programmatic control), NavigationStack, TabView styles, sheet/popover/alert/confirmationDialog presentation, openURL/openWindow/dismissWindow environment actions, deep linking. Use when implementing navigation structure in a macOS SwiftUI app."
:sources: https://developer.apple.com/documentation/swiftui/navigationsplitview, https://developer.apple.com/documentation/scenekit/scene, https://developer.apple.com/documentation/swiftui/tabview, https://developer.apple.com/documentation/swiftui/tabviewstyle, https://developer.apple.com/documentation/swiftui/environmentvalues/openwindow, https://developer.apple.com/documentation/swiftui/environmentvalues/dismisswindow
researched: 2026-06-06
platform: macOS 26+
---

# SwiftUI Navigation on macOS

Navigation architecture for macOS SwiftUI apps: `NavigationSplitView` (sidebar/content/detail), `NavigationStack`, modal and popover presentation, window management via `openWindow`/`dismissWindow`, and deep linking. Targets macOS 26+ with Swift 6.3.

For toolbar and sidebar styling, see `macos-toolbar-sidebar`. For scene types (`WindowGroup`, `Window`, `Settings`, `MenuBarExtra`), see `macos-windows-scenes`. For menu commands and keyboard shortcuts, see `macos-menus-commands`.

## Contents

- [NavigationSplitView — Three-Column Layout](#navigationsplitview--three-column-layout)
- [Column Visibility](#column-visibility)
- [Sheet & Popover](#sheet--popover)
- [openWindow & dismissWindow](#openwindow--dismisswindow)
- [Deep Linking](#deep-linking)
- [NavigationStack on macOS](#navigationstack-on-macos)
- [TabView](#tabview)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## NavigationSplitView — Three-Column Layout

`NavigationSplitView` on macOS supports three columns: sidebar, content, and detail. Use the three-closure initializer with a `NavigationSplitViewVisibility` binding for programmatic control.

```swift
struct MainView: View {
    @State private var selectedItem: Item?
    @State private var columnVisibility: NavigationSplitViewVisibility = .all

    var body: some View {
        NavigationSplitView(columnVisibility: $columnVisibility) {
            List(items, selection: $selectedItem) { item in
                Label(item.name, systemImage: item.icon)
            }
            .navigationSplitViewColumnWidth(min: 180, ideal: 220, max: 300)
        } content: {
            if let selectedItem {
                ItemContentView(item: selectedItem)
            } else {
                ContentUnavailableView("Select an Item",
                    systemImage: "sidebar.leading")
            }
        } detail: {
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

**Two-closure form**: Omit the `content` closure for a standard sidebar+detail layout. The content column collapses automatically.

```swift
NavigationSplitView {
    SidebarView()
} detail: {
    DetailView()
}
```

**Column width control**: Apply `.navigationSplitViewColumnWidth(min:ideal:max:)` to each column. The `ideal` value sets the initial width; users can drag within `min...max`. Omit `max` for unconstrained growth. The system persists user-adjusted widths across launches.

**Programmatic column toggling**: Change `columnVisibility` in code to show/hide columns. Combine with a toolbar button for user-facing control (see [Column Visibility](#column-visibility)).


### Preferred Compact Column (macOS 14+)

Control which column appears when `NavigationSplitView` collapses to a single column (e.g., narrow window). Use the `preferredCompactColumn:` initializer:

```swift
struct MainView: View {
    @State private var selectedItem: Item?
    @State private var columnVisibility: NavigationSplitViewVisibility = .all
    @State private var preferredCompactColumn: NavigationSplitViewColumn = .content

    var body: some View {
        NavigationSplitView(
            preferredCompactColumn: $preferredCompactColumn,
            columnVisibility: $columnVisibility
        ) {
            Sidebar(items: items, selection: $selectedItem)
        } content: {
            ItemContentView(item: selectedItem)
        } detail: {
            ItemDetailView(item: selectedItem)
        }
    }
}
```

Bind a `NavigationSplitViewColumn` (`.sidebar`, `.content`, `.detail`) to control which column is visible in compact mode. The system animates transitions between column configurations.

### NavigationSplitViewStyle

Apply `.navigationSplitViewStyle(_:)` to control how columns resize:

| Style | Behavior |
|-------|----------|
| `.balanced` (default) | Columns share space proportionally |
| `.prominentDetail` | Detail column gets priority; others collapse first |

```swift
NavigationSplitView { /* ... */ } detail: { /* ... */ }
    .navigationSplitViewStyle(.prominentDetail)
```
## Column Visibility

`NavigationSplitViewVisibility` has three values:

| Value | Behavior |
|-------|----------|
| `.all` | All three columns visible (sidebar + content + detail) |
| `.doubleColumn` | Sidebar + detail only; content column hidden |
| `.detailOnly` | Detail column only; sidebar and content hidden |

When a visibility binding is provided, macOS automatically adds a column-toggle toolbar button. The user's column selection is persisted across launches.

**Toolbar toggle button**:

```swift
.toolbar {
    ToolbarItem(placement: .navigation) {
        Toggle("Show Sidebar", systemImage: "sidebar.leading",
               isOn: Binding(
                   get: { columnVisibility != .detailOnly },
                   set: { columnVisibility = $0 ? .all : .detailOnly }
               ))
        .keyboardShortcut("s", modifiers: [.command, .control])
    }
}
```

**User-initiated toggling**: Users can double-click the column divider or use the View menu to toggle columns. The binding stays in sync — read it to adapt your layout (e.g., show a different view when the sidebar is hidden).

**Two-column apps**: If your app only uses sidebar+detail (no content column), set initial visibility to `.doubleColumn`. The content closure is never invoked.

## Sheet & Popover

### Sheet

`.sheet(isPresented:)` and `.sheet(item:)` work on macOS as on iOS. Sheets attach to the current window.

```swift
struct ContentView: View {
    @State private var showingSettings = false

    var body: some View {
        VStack {
            Button("Settings") { showingSettings = true }
        }
        .sheet(isPresented: $showingSettings) {
            SettingsView()
                .frame(width: 400, height: 300)
        }
    }
}
```

**Detent support**: On macOS 26+, sheets support `.presentationDetents([.medium, .large])` and `.presentationDragIndicator(.visible)` for resizable sheets. The system enforces minimum sheet sizes.

### Popover

`.popover(isPresented:attachmentAnchor:)` presents content as an anchored popover. On macOS, popovers are the standard pattern for contextual detail — prefer them over sheets for non-modal, transient UI.

```swift
struct ContentView: View {
    @State private var showingInfo = false

    var body: some View {
        Button("Info") { showingInfo = true }
            .popover(isPresented: $showingInfo, attachmentAnchor: .point(.top)) {
                VStack(alignment: .leading) {
                    Text("Details").font(.headline)
                    Text("Additional information here.")
                }
                .padding()
                .frame(width: 240)
            }
    }
}
```

**Attachment anchors**: Use `.point(_)` to anchor to a specific point in the source view's bounds, or `.rect(_)` to anchor to a rectangle. The popover automatically positions itself to avoid screen edges.

**Popover vs sheet on macOS**:
- **Popover**: Non-modal, dismisses when clicking away, anchored to a source element. Use for contextual info, quick actions, color pickers.
- **Sheet**: Modal, blocks the parent window, requires explicit dismissal. Use for settings, confirmations, forms that must be completed.

### Alert and Confirmation Dialog

`.alert(isPresented:)` and `.confirmationDialog(isPresented:)` work identically on macOS and iOS. `confirmationDialog` is the standard pattern for choosing between multiple actions (e.g., "Save changes?" with Save/Don't Save/Cancel).

```swift
.confirmationDialog("Save changes?", isPresented: $showingDialog) {
    Button("Save", role: .destructive) { save() }
    Button("Don't Save") { discard() }
    Button("Cancel", role: .cancel) { }
} message: {
    Text("Your changes will be lost if you don't save them.")
}
```

**Key difference from iOS**: `fullscreenCover` is **not available** on macOS. Use `.sheet` with a large detent or a separate `Window` scene for fullscreen-like presentation. See [Constraints & Gotchas](#constraints--gotchas).

## openWindow & dismissWindow

Use `@Environment(\.openWindow)` and `@Environment(\.dismissWindow)` to manage windows programmatically. These work with `Window` (singleton) and `WindowGroup` (multi-instance) scenes.

### Opening by Identifier

```swift
struct ContentView: View {
    @Environment(\.openWindow) private var openWindow

    var body: some View {
        Button("Open Inspector") {
            openWindow(id: "inspector")
        }
    }
}
```

The scene must declare a matching `id`:

```swift
Window("Inspector", id: "inspector") {
    InspectorView()
}
.defaultSize(width: 300, height: 500)
```

### Opening with a Value Type (WindowGroup)

For `WindowGroup` scenes that accept a payload, use `openWindow(id:value:)`:

```swift
struct DocumentPayload: Codable, Hashable {
    let documentID: UUID
}

@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup("Document", id: "document", for: DocumentPayload.self) { $payload in
            DocumentDetailView(documentID: payload.documentID)
        } defaultValue: {
            DocumentPayload(documentID: UUID())
        }
    }
}

// Open from any view:
openWindow(id: "document", value: DocumentPayload(documentID: doc.id))
```


### Window Placement and pushWindow

Use `defaultWindowPlacement(_:)` to control where new windows appear on macOS. Compute placement relative to the current display:

```swift
Window("Inspector", id: "inspector") {
    InspectorView()
}
.defaultSize(width: 300, height: 500)
.defaultWindowPlacement { content, context in
    let display = context.defaultDisplay.visibleRect
    let size = content.sizeThatFits(.unspecified)
    return WindowPlacement(.leading(display), width: size.width, height: size.height)
}
```

For pushing a window from another (e.g., inspector panels, modal flows), use `pushWindow(id:)` — dismiss re-opens the presenting window. This is the SwiftUI equivalent of AppKit's sheet-like window stacking.
Each unique payload value opens a new window instance. The payload type must be `Codable` and `Hashable`.

### Dismissing Windows

```swift
// Dismiss the current window:
@Environment(\.dismissWindow) private var dismissWindow

// Self-dismiss from inside the window:
@Environment(\.dismiss) private var dismiss

// Dismiss a specific window by ID:
dismissWindow(id: "inspector")

// Dismiss all windows of a group:
dismissWindow(id: "document")
```

**`Window` vs `WindowGroup`**: `Window` is a singleton — `openWindow` brings it to front if already open. `WindowGroup` creates a new instance each time. Use `Window` for inspectors, palettes, and other single-instance panels. Use `WindowGroup` for document windows.

## Deep Linking

Handle URLs in a SwiftUI app to support deep linking from external sources (other apps, Terminal, browser).

### Step 1 — Register a URL Scheme in Info.plist

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

This registers `myapp://` as a URL scheme your app handles.

### Step 2 — Handle URLs in Your App

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .onOpenURL { url in
                    handleDeepLink(url)
                }
        }
    }

    private func handleDeepLink(_ url: URL) {
        // myapp://document/123
        guard url.scheme == "myapp" else { return }
        let components = URLComponents(url: url, resolvingAgainstBaseURL: false)
        // Route based on host and path
    }
}
```

### Step 3 — Construct Deep Links

```swift
// Open from Terminal: open "myapp://document/123"
// Open from another app:
if let url = URL(string: "myapp://document/123") {
    NSWorkspace.shared.open(url)
}
```

**Universal Links**: For HTTPS-based deep links, register your domain with Apple and use `com.apple.developer.associated-domains` entitlement. Universal Links are more secure than custom URL schemes because only your app can handle them.

**Testing**: Use `xcrun simctl openurl booted "myapp://document/123"` in Simulator, or `open "myapp://document/123"` on a real device.

## NavigationStack on macOS

`NavigationStack` works on macOS but is less common than `NavigationSplitView` for document-based apps. Use it for drill-down hierarchies where a three-column layout is unnecessary.

### When to Use NavigationStack vs NavigationSplitView

| Use `NavigationStack` when | Use `NavigationStack` when |
|---|---|
| Single-column drill-down (e.g., settings, wizard flows) | Multi-column document browser |
| Modal or sheet-contained navigation | The primary app structure |
| Temporary navigation that pops to root | Persistent sidebar navigation |

### Programmatic Navigation with NavigationPath

```swift
struct BrowserView: View {
    @State private var path = NavigationPath()

    var body: some View {
        NavigationStack(path: $path) {
            List(categories) { category in
                NavigationLink(category.name, value: category)
            }
            .navigationDestination(for: Category.self) { category in
                CategoryView(category: category, path: $path)
            }
            .navigationDestination(for: Item.self) { item in
                ItemDetailView(item: item)
            }
        }
        .toolbar {
            ToolbarItem(placement: .navigation) {
                Button("Root") { path = NavigationPath() }
                    .disabled(path.isEmpty)
            }
        }
    }
}

struct CategoryView: View {
    let category: Category
    @Binding var path: NavigationPath

    var body: some View {
        List(category.items) { item in
            NavigationLink(item.name, value: item)
        }
        .toolbar {
            ToolbarItem(placement: .navigation) {
                Button("Back") { path.removeLast() }
                    .disabled(path.isEmpty)
            }
        }
    }
}
```

**NavigationPath operations**:
- `path.append(value)` — push a new destination
- `path.removeLast()` — pop one level
- `path.removeLast(n)` — pop n levels
- `path = NavigationPath()` — pop to root
- `path.isEmpty` — check if at root

**Codable NavigationPath**: `NavigationPath` is `Codable` when all pushed values are `Codable`. Use this for state restoration — save the path on termination and restore on launch.

## TabView

`TabView` on macOS uses the `.automatic` style, which renders as a native tab bar. Tab views are typically used inside `NavigationSplitView` columns or as the root of a window.

```swift
struct ContentView: View {
    var body: some View {
        TabView {
            DashboardView()
                .tabItem {
                    Label("Dashboard", systemImage: "chart.bar")
                }

            DocumentsView()
                .tabItem {
                    Label("Documents", systemImage: "doc")
                }

            SettingsView()
                .tabItem {
                    Label("Settings", systemImage: "gear")
                }
        }
        .frame(minWidth: 600, minHeight: 400)
    }
}
```

**Tab item configuration**: Use `Label` with both title and icon for standard macOS tab appearance. The system renders icons above text in the tab bar.

**TabView inside NavigationSplitView**: A common pattern is placing a `TabView` in the detail column of a `NavigationSplitView`, giving each sidebar selection a tabbed detail area.

```swift
NavigationSplitView {
    Sidebar(selection: $selectedProject)
} detail: {
    TabView(selection: $selectedTab) {

### TabView sidebarAdaptable Style

The `.sidebarAdaptable` style (macOS 14+) renders tabs in both the sidebar and tab bar, adapting to available space. Use `TabSection` to group tabs hierarchically:

```swift
TabView {
    Tab("Home", systemImage: "house") { HomeView() }
    Tab("Reports", systemImage: "chart.bar") { ReportsView() }

    TabSection("Categories") {
        Tab("Climate", systemImage: "cloud") { ClimateView() }
        Tab("Lights", systemImage: "lightbulb") { LightsView() }
    }
}
.tabViewStyle(.sidebarAdaptable)
.tabViewCustomization(.constant(.init()))
```

### Tab View Customization

Users can show/hide and reorder tabs when customization is enabled. Assign a `customizationID` to each customizable tab or section:

```swift
.tabViewCustomization(.constant(TabViewCustomization()))
```


**TabView with `TabViewStyle`**: On macOS, `.tabViewStyle(.automatic)` is the default and recommended style. `.page` style is available but non-standard for macOS.

## Constraints & Gotchas

- **`fullscreenCover` is absent on macOS.** There is no SwiftUI equivalent of `fullscreenCover` on macOS. To present a fullscreen-like experience, use a separate `Window` scene with `.windowStyle(.plain)` and no toolbar, or use `.sheet` with `.presentationDetents([.large])`. For true fullscreen, use AppKit's `NSWindow.toggleFullScreen(_:)` via `NSViewRepresentable` or `NSApp.keyWindow?.toggleFullScreen(nil)`.

- **Popover vs sheet sizing**: Popovers size to their content and auto-position to avoid screen edges. Sheets are constrained to the parent window and support detents. Don't use a popover for large forms — use a sheet or a separate window.

- **NavigationSplitView with TabView nesting**: Nesting a `TabView` inside a `NavigationSplitView` detail column works, but avoid nesting `NavigationSplitView` inside a `TabView` — the column visibility controls conflict with the tab bar, and the layout becomes unpredictable.

- **NavigationStack inside NavigationSplitView**: You can place a `NavigationStack` inside any `NavigationSplitView` column. This is the standard pattern for drill-down within a column (e.g., a folder hierarchy in the sidebar). Each column gets its own independent navigation state.

- **Sheet dismissal on macOS**: Sheets on macOS can be dismissed by clicking outside the sheet (unlike iOS). Use `.interactiveDismissDisabled()` to prevent this when the user must explicitly confirm or cancel.

- **WindowGroup value types must be Hashable**: When using `openWindow(id:value:)`, the value type must conform to both `Codable` and `Hashable`. Two windows with the same hash will not be duplicated — the existing window is brought to front.

- **Column visibility persistence**: The system persists the user's column visibility choice. If you programmatically set `columnVisibility`, it overrides the persisted value until the user toggles it again.

## Common Mistakes

- **Using `NavigationLink(destination:)` instead of `NavigationLink(value:)` with `NavigationStack`**. The value-based form is required for programmatic navigation with `NavigationPath`. The destination-based form does not populate the path.

- **Forgetting to make WindowGroup payload types `Codable` and `Hashable`**. The compiler will catch this, but the error messages are confusing. Always declare both conformances explicitly.

- **Using `fullscreenCover` on macOS**. It compiles but crashes at runtime. Use `.sheet` with a large detent or a separate `Window` scene instead.

- **Not providing `defaultValue:` for parameterized `WindowGroup` scenes**. Without it, opening the window from File → New Window (⌘N) crashes because there's no payload.

- **Placing `NavigationSplitView` inside `NavigationStack`**. This creates conflicting navigation patterns. Use `NavigationStack` inside a `NavigationSplitView` column, not the reverse.

- **Ignoring `NavigationPath` codability**. If you need state restoration, ensure all types pushed onto the path are `Codable`. Non-codable types prevent the path from being saved/restored.

- **Using `.fullScreenCover` modifier (iOS spelling)**. The correct iOS name is `fullscreenCover` (lowercase 's' in Screen). On macOS, neither spelling works — the modifier doesn't exist.

## Review Checklist

- [ ] `NavigationSplitView` uses three-closure form for sidebar/content/detail layouts
- [ ] `NavigationSplitViewVisibility` binding provided for programmatic column control
- [ ] Column width constraints set with `.navigationSplitViewColumnWidth(min:ideal:max:)`
- [ ] Sheets use `.frame()` or `.presentationDetents()` to set appropriate size on macOS
- [ ] Popovers use `.popover(isPresented:attachmentAnchor:)` with explicit anchor
- [ ] `openWindow(id:)` / `dismissWindow(id:)` used for window management
- [ ] Parameterized `WindowGroup` scenes include `defaultValue:`
- [ ] Deep link URL scheme registered in `Info.plist`
- [ ] `.onOpenURL` handler routes URLs to appropriate views
- [ ] `NavigationStack` uses `NavigationPath` for programmatic control
- [ ] `TabView` uses `Label` for tab items with both title and icon
- [ ] No `fullscreenCover` usage (not available on macOS)
- [ ] No `NavigationSplitView` nested inside `NavigationStack`

## References

- [references/navigation-patterns.md](references/navigation-patterns.md) — Complete code recipes: three-column NavigationSplitView with toggle, deep link routing, programmatic NavigationStack, custom popover, fullscreen replacement
- [WWDC 24: Work with windows in SwiftUI](https://developer.apple.com/videos/play/wwdc2024/10149/)
