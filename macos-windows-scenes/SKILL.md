---
name: macos-windows-scenes
description: "Design and implement macOS multi-window architectures using WindowGroup, Window, Settings, MenuBarExtra, openWindow/dismissWindow, scene styles, and DocumentGroup. Use when building apps with multiple windows, custom window chrome, menu bar utilities, parameterized window opening, state restoration, or document-based workflows on macOS 26+."
sources: https://developer.apple.com/videos/play/wwdc2022/10061/, https://developer.apple.com/videos/play/wwdc2024/10148/, https://developer.apple.com/documentation/swiftui/customizing-window-styles-and-state-restoration-behavior-in-macos, https://nilcoalescing.com/blog/ScenesTypesInASwiftUIMacApp, https://developer.apple.com/tutorials/app-dev-training/supporting-multiple-windows
researched: 2026-06-05
platform: macOS 26+
---

# macOS Windows & Scenes

Scene types and window management for SwiftUI apps on macOS 26+ with Swift 6.3. Covers `WindowGroup`, `Window`, `Settings`, `MenuBarExtra`, `DocumentGroup`, parameterized window opening, window restoration, and toolbar chrome customization. Patterns assume SwiftUI's `App` protocol entry point.

For NavigationSplitView and sidebar detail layouts, see the `swiftui-navigation` skill. For MV* state patterns, see `swiftui-patterns`.

## Contents

- [WindowGroup](#windowgroup)
- [Window (Single-Instance)](#window-single-instance)
- [Settings (Preferences Window)](#settings-preferences-window)
- [MenuBarExtra](#menurextra)
- [Parameterized Window Opening (openWindow)](#parameterized-window-opening-openwindow)
- [Window Restoration](#window-restoration)
- [WindowStyle (Chrome Customization)](#windowstyle-chrome-customization)
- [Toolbar and Title Bar Removal](#toolbar-and-title-bar-removal)
- [DocumentGroup](#documentgroup)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## WindowGroup

`WindowGroup` is the default multi-window scene. Each new window creates a fresh instance of the root view. macOS automatically supports **File → New Window** (⌘N) and window restoration across launches.

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
    }
}
```

**Passing data to every window:** Inject dependencies through the environment or an `@Observable` object. Each window gets its own view instance, so avoid relying on shared mutable state without proper synchronization.

```swift
WindowGroup {
    ContentView()
        .environment(AppStore.shared)
}
```

**Restricting window count:** Use `handlesExternalEvents(matching:)` to control which URLs or intents can open new windows. By default, `WindowGroup` allows unlimited windows.

```swift
WindowGroup {
    ContentView()
}
.handlesExternalEvents(matching: ["main"])  // Only open windows for "main" events
```

**WindowGroup with an identifier:** Assign an `id` to reference the group from `openWindow(id:)`:

```swift
WindowGroup(id: "inspector") {
    InspectorView()
}
```

## Window (Single-Instance)

Use `Window` (macOS 14+) for a single-instance window that is opened and closed programmatically. Unlike `WindowGroup`, only one instance exists at a time.

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }

        Window("Quick Note", id: "quick-note") {
            QuickNoteView()
        }
        .defaultSize(width: 400, height: 300)
        .keyboardShortcut("n", modifiers: [.command, .shift])
    }
}
```

Open and close it from any view:

```swift
struct ContentView: View {
    @Environment(\.openWindow) private var openWindow
    @Environment(\.dismissWindow) private var dismissWindow

    var body: some View {
        VStack {
            Button("Open Quick Note") {
                openWindow(id: "quick-note")
            }
            Button("Close Quick Note") {
                dismissWindow(id: "quick-note")
            }
        }
    }
}
```

**Key difference from WindowGroup:** `Window` is a singleton scene. Calling `openWindow` when the window is already visible brings it to front instead of creating a duplicate. `WindowGroup` always creates a new instance.

## Settings (Preferences Window)

The `Settings` scene type produces the standard **Preferences** window (⌘,). It is a singleton — only one instance exists, and it is automatically managed by the system.

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }

        Settings {
            SettingsView()
        }
    }
}
```

**Layout:** Use `Form` or `TabView` inside `Settings` for standard preferences layout. Each `Tab` in a `TabView` maps to a preferences pane in the toolbar.

```swift
struct SettingsView: View {
    @AppStorage("showNotifications") private var showNotifications = true
    @AppStorage("refreshInterval") private var refreshInterval = 30.0

    var body: some View {
        TabView {
            Form {
                Toggle("Show Notifications", isOn: $showNotifications)
                Slider(value: $refreshInterval, in: 5...300, step: 5) {
                    Text("Refresh Interval: \(Int(refreshInterval))s")
                }
            }
            .tabItem { Label("General", systemImage: "gear") }

            Form {
                // Account settings
            }
            .tabItem { Label("Account", systemImage: "person") }
        }
        .frame(width: 400, height: 300)
    }
}
```

The system adds the standard preferences toolbar with icons for each tab. Do not add your own toolbar — the system manages it.

## MenuBarExtra

`MenuBarExtra` creates a menu bar utility app. Two presentation styles are available:

### .menu Style (Default)

A standard menu that opens on click and closes when the user clicks away or selects an item.

```swift
@main
struct MenuBarApp: App {
    var body: some Scene {
        MenuBarExtra("MyApp", systemImage: "star.fill") {
            MenuBarView()
        }
        .menuBarExtraStyle(.menu)
    }
}
```

### .window Style

A popover-style window that behaves like a mini-app. It stays open until explicitly dismissed and can contain any SwiftUI view.

```swift
MenuBarExtra("MyApp", systemImage: "star.fill") {
    PopoverContentView()
        .frame(width: 320, height: 400)
}
.menuBarExtraStyle(.window)
```

**Lifecycle and keep-alive:** Menu bar extras are terminated by the system when not visible (`.menu` style) to save resources. For `.window` style, the view stays alive while the popover is open. To keep a `.menu` extra alive between opens (e.g., for timers or network monitoring), use a background `AppDelegate` or `NSApplicationDelegate` to hold a reference:

```swift
class AppDelegate: NSObject, NSApplicationDelegate {
    func applicationDidFinishLaunching(_ notification: Notification) {
        // Keep alive: prevents the menu bar extra from being deallocated
    }
}

@main
struct MenuBarApp: App {
    @NSApplicationDelegateAdaptor(AppDelegate.self) var appDelegate

    var body: some Scene {
        MenuBarExtra("MyApp", systemImage: "star.fill") {
            MenuBarView()
        }
    }
}
```

**Important:** Menu bar extra apps should disable the dock icon by setting `LSUIElement` to `YES` in `Info.plist`, or by using:

```swift
// In AppDelegate
func applicationDidFinishLaunching(_ notification: Notification) {
    NSApp.setActivationPolicy(.accessory)
}
```

## Parameterized Window Opening (openWindow)

Use `openWindow(id:value:)` to pass data to a window at creation time. The target scene must declare `acceptsData` in its `WindowGroup` or `Window` configuration.

**Step 1 — Define a codable value type:**

```swift
struct WindowPayload: Codable, Hashable {
    let documentID: UUID
    let title: String
}
```

**Step 2 — Declare the window with `default:` value handling:**

```swift
WindowGroup("Document", id: "document", for: WindowPayload.self) { $payload in
    DocumentDetailView(documentID: payload.documentID, title: payload.title)
} defaultValue: {
    WindowPayload(documentID: UUID(), title: "Untitled")
}
```

**Step 3 — Open from any view:**

```swift
struct ContentView: View {
    @Environment(\.openWindow) private var openWindow

    var body: some View {
        Button("Open Document") {
            openWindow(
                id: "document",
                value: WindowPayload(documentID: doc.id, title: doc.title)
            )
        }
    }
}
```

**Multiple parameterized windows:** Each `WindowGroup` with a unique `id` can accept its own payload type. Use separate `openWindow(id:value:)` calls to target the correct window.

**Dismissal:** Use `dismissWindow(id:)` to close a specific window, or `@Environment(\.dismiss)` inside the window's own view to self-close.

## Window Restoration

SwiftUI automatically restores window positions and sizes across app relaunch on macOS. Control defaults with modifiers:

```swift
WindowGroup {
    ContentView()
}
.defaultSize(width: 800, height: 600)
.defaultPosition(.center)
```

**Persistent state:** Use `@AppStorage` for user preferences that should survive relaunch. For complex state, use `Codable` models saved to `Application Support`:

```swift
struct DocumentView: View {
    @AppStorage("sidebarVisible") private var sidebarVisible = true
    @State private var document: Document

    var body: some View {
        NavigationSplitView(sidebar: {
            if sidebarVisible { SidebarView() }
        }, detail: {
            DetailView(document: document)
        })
    }
}
```

**Restoration scope:** Window restoration is per-scene. Each `WindowGroup` with an `id` restores independently. Unnamed `WindowGroup` scenes restore as a group.

## WindowStyle (Chrome Customization)

Apply window styles to control the title bar and chrome appearance:

```swift
WindowGroup {
    ContentView()
}
.windowStyle(.titleBar)          // Standard title bar (default)
.windowStyle(.hiddenTitleBar)    // No title bar, content extends to edges
.windowStyle(.unified)           // Unified toolbar + title bar (macOS 26+)
.windowStyle(.unifiedCompact)    // Compact unified style (macOS 26+)
```

**`.hiddenTitleBar`:** Removes the title bar entirely. You must provide your own window drag region and close/minimize/zoom buttons. Use `.onWindowDrag()` (macOS 26+) to make the entire window draggable:

```swift
WindowGroup {
    VStack {
        CustomTitleBarView()
        ContentView()
    }
    .onWindowDrag()
}
.windowStyle(.hiddenTitleBar)
```

**`.unified` and `.unifiedCompact`:** Merge the title bar and toolbar into a single continuous surface. The title appears inline with toolbar items. Use `toolbar(removing:)` and `toolbarBackground(visibility:)` to fine-tune the appearance.

## Toolbar and Title Bar Removal

Remove default toolbar elements and customize chrome:

```swift
WindowGroup {
    ContentView()
        .toolbar {
            ToolbarItem(placement: .navigation) {
                Button("Back") { /* ... */ }
            }
        }
        .toolbar(removing: .title)           // Remove the title
        .toolbar(removing: .sidebarToggle)   // Remove sidebar toggle
        .toolbarBackground(.hidden, for: .windowToolbar) // Transparent toolbar bg
}
```

**Available removables (macOS 26+):**
- `.title` — removes the window title from the toolbar
- `.sidebarToggle` — removes the sidebar toggle button
- `.auto` — restores system defaults

**Custom title bar with unified style:**

```swift
.toolbar {
    ToolbarItem(placement: .principal) {
        Text("My Document")
            .font(.headline)
    }
}
.toolbarBackground(.hidden, for: .windowToolbar)
```

## DocumentGroup

`DocumentGroup` provides a full document-based app with system-managed open/save panels, versions browser, and iCloud integration. Define a `Document` conforming to `ReferenceFileDocument` (value type) or `FileDocument` (class).

```swift
@main
struct DocumentApp: App {
    var body: some Scene {
        DocumentGroup(newDocument: MyDocument()) { file in
            DocumentEditorView(document: file.$document)
        }
    }
}

struct MyDocument: Codable, ReferenceFileDocument {
    var text: String
    var createdAt: Date

    init(text: String = "") {
        self.text = text
        self.createdAt = Date()
    }

    init(configuration: ReadConfiguration) throws {
        guard let data = configuration.file.regularFileContents,
              let doc = try? JSONDecoder().decode(Self.self, from: data)
        else { throw CocoaError(.fileReadCorruptFile) }
        self = doc
    }

    func fileWrapper(configuration: WriteConfiguration) throws -> FileWrapper {
        let data = try JSONEncoder().encode(self)
        return FileWrapper(regularFileWithContents: data)
    }
}
```

**Document-based window styles:** `DocumentGroup` supports the same window style modifiers as `WindowGroup`:

```swift
DocumentGroup(newDocument: MyDocument()) { file in
    DocumentEditorView(document: file.$document)
}
.windowStyle(.unified)
```

**UTI declaration:** Declare your document type in `Info.plist` under `UTExportedTypeDeclarations` with a unique identifier, conformance to `public.data` or `public.content`, and a file extension.

## Common Mistakes

1. Using `WindowGroup` when you need a singleton — use `Window` for single-instance windows like inspectors or quick-entry panels.
2. Forgetting `@Environment(\.openWindow)` — it must be captured from the environment, not stored in a global.
3. Passing non-`Codable` types through `openWindow(id:value:)` — the value must conform to `Codable` and `Hashable`.
4. Adding a custom toolbar inside `Settings` — the system manages the preferences toolbar; use `TabView` for panes.
5. Using `.menu` style for a menu bar extra that needs to stay alive — use `.window` style or an `AppDelegate` keep-alive.
6. Not setting `LSUIElement` for menu bar-only apps — the dock icon will appear without it.
7. Applying `.hiddenTitleBar` without providing a drag region — the window becomes immovable.
8. Using `@State` for data that should persist across window creation — use `@AppStorage` or a persisted model.
9. Nesting `DocumentGroup` inside `WindowGroup` — they are separate scene types at the same level.
10. Forgetting to declare UTIs in `Info.plist` for document-based apps — the system won't know how to open your files.

## Review Checklist

- [ ] `WindowGroup` used for multi-window scenes, `Window` for single-instance
- [ ] `openWindow(id:value:)` used for parameterized window opening with `Codable` payload
- [ ] `dismissWindow(id:)` or `@Environment(\.dismiss)` used for window closure
- [ ] `Settings` scene used for preferences (not a custom window)
- [ ] `MenuBarExtra` style (`.menu` vs `.window`) matches the interaction model
- [ ] Menu bar-only apps set `LSUIElement` or `.accessory` activation policy
- [ ] Window restoration tested with `.defaultSize` and `.defaultPosition`
- [ ] `.hiddenTitleBar` paired with `.onWindowDrag()` for custom chrome
- [ ] `DocumentGroup` UTIs declared in `Info.plist`
- [ ] `@Environment(\.openWindow)` captured locally, not stored globally

## References

- Multi-window patterns, window restoration, and MenuBarExtra lifecycle: [references/scene-patterns.md](references/scene-patterns.md)
- WindowGroup and Window: [https://sosumi.ai/documentation/swiftui/windowgroup](https://sosumi.ai/documentation/swiftui/windowgroup)
- Window styles and state restoration: [https://sosumi.ai/documentation/swiftui/customizing-window-styles-and-state-restoration-behavior-in-macos](https://sosumi.ai/documentation/swiftui/customizing-window-styles-and-state-restoration-behavior-in-macos)
- DocumentGroup: [https://sosumi.ai/documentation/swiftui/documentgroup](https://sosumi.ai/documentation/swiftui/documentgroup)
- MenuBarExtra: [https://sosumi.ai/documentation/swiftui/menurextra](https://sosumi.ai/documentation/swiftui/menurextra)
- State management patterns: see `swiftui-patterns` skill
- Navigation and sidebar layouts: see `swiftui-navigation` skill
