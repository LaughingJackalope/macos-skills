---
name: macos-menus-commands
description: "Build and customize macOS menu bar menus, keyboard shortcuts, context menus, dock menus, and services integration using SwiftUI commands. Covers .commands placement, CommandMenu, CommandGroup, Button.keyboardShortcut, .contextMenu, NSMenu escape hatch, menu validation, and first-responder routing. Use when adding or modifying menu bar items, keyboard shortcuts, contextual menus, or dock/services menu behavior in a macOS SwiftUI app."
sources: https://apple-docs.everest.mt/docs/swiftui/building-and-customizing-the-menu-bar-with-swiftui/, https://developer.apple.com/documentation/swiftui/commandgroup, https://danielsaidi.com/blog/2023/11/22/customizing-the-macos-menu-bar-in-swiftui, https://swiftwithmajid.com/2020/11/24/commands-in-swiftui/
researched: 2026-06-05
platform: macOS 26+
---

# macOS Menus & Commands

Build and customize menu bar menus, keyboard shortcuts, context menus, dock menus, and services integration in macOS SwiftUI apps. Targets macOS 26+ with Swift 6.3. Patterns are backward-compatible to macOS 13+ unless noted.

See [references/menu-patterns.md](references/menu-patterns.md) for complete code patterns including custom menu recipes, keyboard shortcut conflict resolution, NSMenu integration, dock menu, and services menu.

## Contents

- [.commands Placement and Scoping](#commands-placement-and-scoping)
- [CommandMenu: Custom Top-Level Menus](#commandmenu-custom-top-level-menus)
- [CommandGroup: Reordering Built-in Groups](#commandgroup-reordering-built-in-groups)
- [Button.keyboardShortcut](#buttonkeyboardshortcut)
- [Context Menus](#context-menus)
- [NSMenu Escape Hatch](#nsmenu-escape-hatch)
- [Menu Validation and First-Responder Routing](#menu-validation-and-first-responder-routing)
- [Dock Menu](#dock-menu)
- [Services Menu Integration](#services-menu-integration)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

**Scope boundary:** This skill covers menu bar structure, keyboard shortcuts, context menus, dock menu, and services integration. SwiftUI state management patterns (MV, `@Observable`, environment wiring) are covered in the `swiftui-patterns` skill. AppKit fundamentals (NSWindow, NSViewController, responder chain basics) are covered in the `appkit-foundations` skill.

## .commands Placement and Scoping

The `.commands {}` modifier is placed on the root content inside a `WindowGroup` or `DocumentGroup`. It defines the app's menu bar structure. Only one `.commands {}` block is active per window — the one closest to the focused window's content.

```swift
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
                .commands {
                    // App-wide menu customizations go here
                    CommandMenu("Custom") {
                        Button("Do Something") { /* action */ }
                            .keyboardShortcut("s", modifiers: [.command, .shift])
                    }

                    CommandGroup(after: .newItem) {
                        Button("Import…") { /* action */ }
                            .keyboardShortcut("i", modifiers: .command)
                    }
                }
        }
    }
}
```

**Scoping rules:**
- `.commands {}` on the root view of a `WindowGroup` applies to that window's menu bar.
- If a secondary window has its own `.commands {}`, it takes over the menu bar when that window is focused. This is how inspector panels or tool palettes customize the menu bar contextually.
- Commands are evaluated top-to-bottom. The first matching shortcut wins — order matters when multiple commands declare the same key equivalent.
- `.commands {}` does nothing on iOS/iPadOS. It is compiled out on those platforms.

## CommandMenu: Custom Top-Level Menus

`CommandMenu` creates a new top-level menu in the menu bar. The string parameter becomes the menu title. Place custom menus where they make the most sense — typically between the View and Window menus, or after standard menus.

```swift
.commands {
    CommandMenu("Shapes") {
        Button("Circle") { addShape(.circle) }
            .keyboardShortcut("c", modifiers: [.command, .option])
        Button("Rectangle") { addShape(.rectangle) }
            .keyboardShortcut("r", modifiers: [.command, .option])
        Button("Triangle") { addShape(.triangle) }
            .keyboardShortcut("t", modifiers: [.command, .option])

        Divider()

        Button("Clear All") { clearShapes() }
            .keyboardShortcut(.delete, modifiers: [.command, .shift])
    }
}
```

**Ordering:** Menus appear in the order they are declared inside `.commands {}`. Use `CommandGroup` placement relative to built-in groups to control where your custom menus land:

```swift
.commands {
    // Appears after the File menu
    CommandGroup(after: .newItem) { /* ... */ }

    // Custom menu placed after the View menu
    CommandGroup(after: .windowArrangement) {
        CommandMenu("Tools") { /* ... */ }
    }
}
```

**Menu content:** Inside `CommandMenu`, you can use `Button`, `Divider`, `Group`, and other SwiftUI views. Only `Button` with `.keyboardShortcut` produces a real menu item with a key equivalent. Other views render as custom menu item views.

## CommandGroup: Reordering Built-in Groups

`CommandGroup` modifies existing built-in menu groups (File, Edit, View, etc.) by inserting, replacing, or appending items. The `placement` parameter controls where the group's content goes relative to the built-in items.

```swift
.commands {
    // Append after the built-in "New" items (New Window, etc.)
    CommandGroup(after: .newItem) {
        Button("New from Template…") { showTemplatePicker() }
            .keyboardShortcut("n", modifiers: [.command, .option])
    }

    // Prepend before the built-in text formatting items
    CommandGroup(before: .textFormatting) {
        Button("Heading 1") { applyStyle(.heading1) }
            .keyboardShortcut("1", modifiers: [.command, .control])
    }

    // Replace the entire Save group
    CommandGroup(replacing: .saveItem) {
        Button("Save") { save() }
            .keyboardShortcut("s", modifiers: .command)
        Button("Save As…") { saveAs() }
            .keyboardShortcut("s", modifiers: [.command, .shift])
    }
}
```

**Built-in group identifiers (macOS):**

| Group | Menu | Typical Contents |
|-------|------|-----------------|
| `.newItem` | File | New Window, New Tab |
| `.saveItem` | File | Save, Save As, Revert |
| `.printItem` | File | Print, Page Setup |
| `.undoRedo` | Edit | Undo, Redo |
| `.pasteboard` | Edit | Cut, Copy, Paste, Delete |
| `.textFormatting` | Format | Bold, Italic, Underline |
| `.textStyle` | Format | Bigger, Smaller, Fonts |
| `.windowArrangement` | Window | Minimize, Zoom, Tile |
| `.windowList` | Window | List of open windows |
| `.help` | Help | App-specific help |
| `.systemServices` | App | Services submenu |

**Important:** When using `CommandGroup(replacing:)`, you must re-declare every item you want in that section. The built-in items are removed entirely. If you only want to add items, use `after:` or `before:` instead.

## Button.keyboardShortcut

`Button.keyboardShortcut(_:modifiers:)` attaches a keyboard shortcut to any SwiftUI button. On macOS, this creates a menu item key equivalent when the button is inside a `.commands {}` block, and a local shortcut when the button is in a view.

```swift
// Global menu bar shortcut — works regardless of focus
.commands {
    CommandMenu("Actions") {
        Button("Export") { export() }
            .keyboardShortcut("e", modifiers: [.command, .shift])
    }
}

// Local shortcut — only works when the view's window is key
struct EditorView: View {
    var body: some View {
        Button("Bold") { toggleBold() }
            .keyboardShortcut("b", modifiers: .command)
    }
}
```

**Scope rules:**
- Shortcuts inside `.commands {}` are global — they fire when any window of the app is active.
- Shortcuts on buttons in views are local — they fire when that view's window is key and no text field or other first responder consumes the event.
- `keyboardShortcut` on a view-level button does NOT create a menu item. It only registers a key equivalent for the responder chain.

**Conflict resolution:**
- When two commands share the same key equivalent, the one declared first in `.commands {}` wins.
- App-level shortcuts (in `.commands {}`) take priority over view-level shortcuts.
- If a text field or text editor is focused, it consumes text-editing shortcuts (Cmd+C, Cmd+V, etc.) before menu commands fire.
- To avoid conflicts, check existing built-in shortcuts before assigning custom ones. Use `.control` or `.option` modifiers to reduce collision risk.

**Modifier options:** `.command`, `.shift`, `.option`, `.control`, `.function`. Combine as needed: `[.command, .shift]`.

## Context Menus

`.contextMenu {}` attaches a contextual menu to a view, triggered by right-click (or two-finger tap on trackpad).

```swift
struct ItemRow: View {
    let item: Item
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        HStack {
            Image(systemName: item.icon)
            Text(item.name)
            Spacer()
        }
        .contextMenu {
            Button("Open") { openItem(item) }
            Button("Duplicate") { duplicateItem(item) }

            Divider()

            Menu("Move To") {
                Button("Inbox") { moveTo(item, .inbox) }
                Button("Archive") { moveTo(item, .archive) }
                Button("Trash") { moveTo(item, .trash) }
            }

            Divider()

            Button("Delete", role: .destructive) { deleteItem(item) }
                .keyboardShortcut(.delete, modifiers: [])
        }
    }
}
```

**Primary action:** On macOS, context menus can define a primary action that fires on a simple click (without right-clicking). Use `.contextMenu(menuItems:primaryAction:)`:

```swift
.contextMenu {
    Button("Edit") { editItem(item) }
    Button("Delete", role: .destructive) { deleteItem(item) }
} primaryAction: {
    openItem(item)
}
```

**macOS vs iOS behavior delta:**
- On macOS, context menus appear on right-click or two-finger tap. On iOS, they appear on long press.
- On macOS, the primary action fires on left-click. On iOS, the primary action fires on tap (replacing the default tap behavior).
- On macOS, context menus include a disclosure indicator for submenus. On iOS, submenus are presented as nested navigation.
- On macOS, keyboard shortcuts declared in `.contextMenu` do NOT appear as menu item equivalents (context menus are not menu bar menus). On iOS, shortcuts are shown in the context menu.
- On macOS, `.contextMenu` works with `List` row selection — right-clicking a row shows the context menu without changing selection. On iOS, long press may trigger selection changes depending on the list style.

## NSMenu Escape Hatch

SwiftUI's commands API covers most menu customization, but some AppKit-only features require dropping to `NSMenu` via `NSApp.mainMenu`. Use this escape hatch when you need:

- Dynamic menu items that change at runtime based on state not expressible in SwiftUI.
- Menu items with custom `NSMenuItem` properties (e.g., `isHidden`, `indentationLevel`, `userKeyEquivalent`).
- Validation beyond what SwiftUI's `CommandGroup(replacing:)` provides.
- Toggling menu item state (checked/unchecked) based on app state.

```swift
import SwiftUI
import AppKit

final class MenuController: ObservableObject {
    @Published var isDarkMode = false

    func setupMenu() {
        guard let mainMenu = NSApp.mainMenu else { return }

        // Find the View menu
        guard let viewMenu = mainMenu.item(withTitle: "View")?.submenu else { return }

        // Add a custom toggle item
        let toggleItem = NSMenuItem(
            title: "Dark Canvas",
            action: #selector(toggleDarkCanvas),
            keyEquivalent: "d"
        )
        toggleItem.keyEquivalentModifierMask = [.command, .option]
        toggleItem.target = self
        viewMenu.addItem(toggleItem)
    }

    @objc func toggleDarkCanvas(_ sender: NSMenuItem) {
        isDarkMode.toggle()
        sender.state = isDarkMode ? .on : .off
    }
}

// In SwiftUI, trigger setup on app launch
@main
struct MyApp: App {
    @StateObject private var menuController = MenuController()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .onAppear {
                    menuController.setupMenu()
                }
        }
    }
}
```

**Important:** Modifying `NSApp.mainMenu` directly is fragile. Menu item positions can change across macOS versions. Always find items by title or tag, never by index. Prefer SwiftUI commands whenever possible and only drop to `NSMenu` for features that SwiftUI does not expose.

## Menu Validation and First-Responder Routing

Menu commands in SwiftUI are automatically enabled or disabled based on context. For custom validation, use the `NSMenuItemValidation` protocol via AppKit interop.

**First-responder routing:** When a menu item is triggered, the action travels down the responder chain: first responder → first responder's next responder → ... → window → window controller → app delegate. If no object in the chain responds to the action, the menu item is disabled.

```swift
// In an NSViewController subclass:
override func validateMenuItem(_ menuItem: NSMenuItem) -> Bool {
    switch menuItem.action {
    #selector(deleteSelection):
        return selectedItems.count > 0
    #selector(duplicateSelection):
        return selectedItems.count > 0 && selectedItems.allSatisfy({ $0.isDuplicable })
    default:
        return super.validateMenuItem(menuItem)
    }
}
```

**SwiftUI validation pattern:** In pure SwiftUI, control menu item availability through the button's `disabled` state. The commands system evaluates this automatically:

```swift
.commands {
    CommandGroup(after: .saveItem) {
        Button("Export as PDF") { exportPDF() }
            .keyboardShortcut("p", modifiers: [.command, .shift])
            .disabled(!canExport)  // Menu item is grayed out when true
    }
}
```

**NSMenuItemValidation protocol:** For AppKit-backed menu items, implement `validateMenuItem(_:)` to dynamically enable/disable items. This is called every time the menu is about to display — keep it fast.

```swift
extension MyWindowController: NSMenuItemValidation {
    func validateMenuItem(_ menuItem: NSMenuItem) -> Bool {
        guard let action = menuItem.action else { return false }

        switch action {
        case #selector(copy(_:)):
            return selection != nil
        case #selector(paste(_:)):
            return NSPasteboard.general.canReadObject(forClasses: [NSString.self], options: nil)
        default:
            return true
        }
    }
}
```

## Dock Menu

The dock menu appears when the user right-clicks the app's dock icon. In SwiftUI, implement the `applicationDockMenu(_:)` delegate method via an `NSApplicationDelegate` adapter.

```swift
import SwiftUI
import AppKit

final class AppDelegate: NSObject, NSApplicationDelegate {
    var dockMenuItems: [DockMenuItem] = []

    func applicationDockMenu(_ sender: NSApplication) -> NSMenu? {
        let menu = NSMenu()
        menu.autoenablesItems = false

        for item in dockMenuItems {
            let menuItem = NSMenuItem(
                title: item.title,
                action: #selector(handleDockMenuItem(_:)),
                keyEquivalent: ""
            )
            menuItem.target = self
            menuItem.representedObject = item.id
            menuItem.isEnabled = item.isEnabled
            menu.addItem(menuItem)
        }

        return menu
    }

    @objc private func handleDockMenuItem(_ sender: NSMenuItem) {
        guard let id = sender.representedObject as? String else { return }
        NotificationCenter.default.post(
            name: .dockMenuItemSelected,
            object: nil,
            userInfo: ["id": id]
        )
    }
}

struct DockMenuItem: Identifiable {
    let id: String
    let title: String
    var isEnabled: Bool = true
}

extension Notification.Name {
    static let dockMenuItemSelected = Notification.Name("dockMenuItemSelected")
}

// Wire up in the SwiftUI App:
@main
struct MyApp: App {
    @NSApplicationDelegateAdaptor(AppDelegate.self) var appDelegate
    @StateObject private var store = AppStore()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(store)
                .onChange(of: store.dockItems) { _, newItems in
                    appDelegate.dockMenuItems = newItems
                }
        }
    }
}
```

**Key points:**
- `applicationDockMenu(_:)` is called every time the user right-clicks the dock icon. Build the menu dynamically.
- Set `autoenablesItems = false` to prevent the system from overriding your enabled/disabled state.
- Use `representedObject` to attach context to menu items without subclassing `NSMenuItem`.
- The dock menu is separate from the menu bar — it does not use SwiftUI's `.commands {}`.

## Services Menu Integration

macOS Services menu lets your app expose functionality to other apps and receive services from them. Services appear in the Application → Services submenu.

**Providing a service:** Register services in `Info.plist` under `NSServices`:

```xml
<key>NSServices</key>
<array>
    <dict>
        <key>NSMessage</key>
        <string>doCustomAction</string>
        <key>NSPortName</key>
        <string>MyApp</string>
        <key>NSRequiredContext</key>
        <dict>
            <key>NSTextContext</key>
            <array>
                <string>Word</string>
                <string>FilePath</string>
            </array>
        </dict>
        <key>NSSendTypes</key>
        <array>
            <string>NSStringPboardType</string>
            <string>NSRTFPboardType</string>
        </array>
        <key>NSReturnTypes</key>
        <array>
            <string>NSStringPboardType</string>
        </array>
        <key>NSMenuItem</key>
        <dict>
            <key>default</key>
            <string>Process with MyApp</string>
        </dict>
    </dict>
</array>
```

**Handling the service:** Implement the service handler in your app delegate:

```swift
final class AppDelegate: NSObject, NSApplicationDelegate {
    func application(_ application: NSApplication, pasteboard: NSPasteboard, userData: String, error: AutoreleasingUnsafeMutablePointer<NSString?>) -> Bool {
        guard let string = pasteboard.string(forType: .string) else {
            error.pointee = "No text on pasteboard" as NSString
            return false
        }
        // Process the string
        processServiceInput(string)
        return true
    }
}
```

**Receiving services:** To use a service from another app, your view must be a first responder that supports the service's send type. For text-based services, `NSTextView` and `NSTextField` automatically participate. For custom views, override `validRequestor(forSendType:ReturnType:)`:

```swift
final class CustomTextView: NSTextView {
    override func validRequestor(forSendType sendType: NSPasteboard.PasteboardType?, returnType: NSPasteboard.PasteboardType?) -> Any? {
        if sendType == .string {
            return self
        }
        return super.validRequestor(forSendType: sendType, returnType: returnType)
    }
}
```

**Services and SwiftUI:** SwiftUI's `TextField` and `TextEditor` do not automatically participate in the Services menu. To enable services for text input, wrap an `NSTextView` using `NSViewRepresentable` (see `swiftui-appkit-interop` skill).

## Common Mistakes

1. **Placing `.commands {}` on a child view instead of the root.** Commands only affect the menu bar when placed on the root view of a `WindowGroup` or `DocumentGroup`. Placing it deeper in the hierarchy silently does nothing.

2. **Using `CommandGroup(replacing:)` and wondering why built-in items disappeared.** `replacing:` removes all default items in that group. Re-declare every item you need, or use `after:`/`before:` to add without removing.

3. **Declaring conflicting keyboard shortcuts without understanding priority.** The first declared shortcut wins. App-level `.commands {}` shortcuts override view-level shortcuts. Text fields consume editing shortcuts before menu commands see them.

4. **Assuming `.contextMenu` shortcuts appear in the menu bar.** Context menu shortcuts are visual hints on macOS — they do not register as global shortcuts. Use `.commands {}` for global keyboard shortcuts.

5. **Modifying `NSApp.mainMenu` by index.** Menu item positions change across macOS versions. Always find items by title or tag.

6. **Forgetting `autoenablesItems = false` on custom `NSMenu`.** The system will override your enabled/disabled state unless you disable auto-enable.

7. **Not implementing `validateMenuItem(_:)` for AppKit menu items.** Without validation, menu items stay enabled even when the action is not applicable, leading to crashes or no-op behavior.

8. **Expecting SwiftUI `TextField` to participate in Services menu.** It does not. Use `NSTextView` via `NSViewRepresentable` for Services integration.

## Review Checklist

- [ ] `.commands {}` is placed on the root view of `WindowGroup` or `DocumentGroup`
- [ ] Custom menus use `CommandMenu` with clear, localized titles
- [ ] `CommandGroup` placement (`before`/`after`/`replacing`) matches intent
- [ ] `CommandGroup(replacing:)` re-declares all needed items
- [ ] Keyboard shortcuts avoid conflicts with built-in shortcuts
- [ ] Shortcuts use appropriate modifiers (`.command` for primary actions, `.option`/`.control` for secondary)
- [ ] Context menus use `.contextMenu` with `primaryAction` for click vs right-click behavior
- [ ] NSMenu escape hatch uses title/tag-based lookup, never index-based
- [ ] Custom `NSMenu` sets `autoenablesItems = false`
- [ ] `validateMenuItem(_:)` is implemented for AppKit-backed menu items
- [ ] Dock menu is built dynamically in `applicationDockMenu(_:)`
- [ ] Services are registered in `Info.plist` with correct `NSSendTypes`/`NSReturnTypes`
- [ ] Service handler returns `false` and sets `error` on failure
- [ ] Tested on macOS 26+ with multiple windows to verify command scoping

## References

- Complete patterns and recipes: [references/menu-patterns.md](references/menu-patterns.md)
- SwiftUI state management: see `swiftui-patterns` skill
- AppKit fundamentals: see `appkit-foundations` skill
- SwiftUI-AppKit interop: see `swiftui-appkit-interop` skill
