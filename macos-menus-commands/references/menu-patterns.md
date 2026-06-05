# Menu Patterns — Complete Recipes

Reference patterns for macOS menu customization in SwiftUI. Each section is a self-contained recipe.

## Custom Menu with Dynamic Items

Build a menu whose items change based on app state:

```swift
struct ProjectCommands: View {
    @ObservedObject var project: ProjectManager

    var body: some View {
        CommandMenu("Project") {
            Button("New Project…") { project.createNew() }
                .keyboardShortcut("n", modifiers: [.command, .option])

            Button("Close Project") { project.close() }
                .keyboardShortcut("w", modifiers: [.command, .option])
                .disabled(!project.hasOpenProject)

            Divider()

            // Dynamic recent projects
            ForEach(project.recentProjects) { recent in
                Button(recent.name) { project.open(recent) }
            }

            Divider()

            Button("Clear Recents") { project.clearRecents() }
                .disabled(project.recentProjects.isEmpty)
        }
    }
}

// Usage in App:
WindowGroup {
    ContentView()
        .commands {
            ProjectCommands(project: projectManager)
        }
}
```

**Key insight:** Extract `CommandMenu` content into a separate `View` struct. SwiftUI re-evaluates the view body when `@ObservedObject` or `@StateObject` changes, so the menu items update dynamically.

## Keyboard Shortcut Conflict Resolution

When multiple commands declare the same key equivalent, follow this resolution order:

1. **First-declared wins** within a single `.commands {}` block.
2. **App-level beats view-level** — `.commands {}` shortcuts override `Button.keyboardShortcut` in views.
3. **First responder wins** — a focused text field consumes text-editing shortcuts before menu commands.

**Strategy to avoid conflicts:**

```swift
// Audit built-in shortcuts before assigning custom ones
// Common built-in shortcuts on macOS:
// Cmd+N  New
// Cmd+O  Open
// Cmd+S  Save
// Cmd+W  Close
// Cmd+Q  Quit
// Cmd+Z  Undo
// Cmd+Shift+Z  Redo
// Cmd+C  Copy
// Cmd+V  Paste
// Cmd+X  Cut
// Cmd+A  Select All
// Cmd+F  Find
// Cmd+P  Print
// Cmd+B  Bold (in text apps)
// Cmd+I  Italic (in text apps)

// Safe custom shortcuts use Option or Control modifiers:
.commands {
    CommandMenu("Tools") {
        Button("Inspect Selection") { inspect() }
            .keyboardShortcut("i", modifiers: [.command, .option])  // Avoids Cmd+I (italic)

        Button("Toggle Sidebar") { toggleSidebar() }
            .keyboardShortcut("s", modifiers: [.command, .control])  // Avoids Cmd+S (save)

        Button("Quick Export") { quickExport() }
            .keyboardShortcut("e", modifiers: [.command, .option])  // Avoids conflicts
    }
}
```

**Detecting conflicts at runtime:**

```swift
#if DEBUG
func auditMenuShortcuts() {
    guard let mainMenu = NSApp.mainMenu else { return }
    var seen: [String: String] = [:]

    func checkMenu(_ menu: NSMenu, path: String) {
        for item in menu.items {
            if !item.keyEquivalent.isEmpty {
                let mods = item.keyEquivalentModifierMask
                let key = "\(mods):\(item.keyEquivalent)"
                if let existing = seen[key] {
                    print("⚠️ Shortcut conflict: '\(key)' — '\(existing)' vs '\(path)/\(item.title)'")
                } else {
                    seen[key] = "\(path)/\(item.title)"
                }
            }
            if let submenu = item.submenu {
                checkMenu(submenu, path: "\(path)/\(item.title)")
            }
        }
    }

    checkMenu(mainMenu, path: "mainMenu")
}
#endif
```

## NSMenu Integration: Adding a Submenu at Runtime

When you need a submenu that SwiftUI's `CommandMenu` cannot express:

```swift
final class DynamicMenuController: ObservableObject {
    @Published var colorPresets: [ColorPreset] = []

    func buildColorMenu() -> NSMenu {
        let menu = NSMenu(title: "Colors")
        menu.autoenablesItems = false

        for preset in colorPresets {
            let item = NSMenuItem(
                title: preset.name,
                action: #selector(selectColor(_:)),
                keyEquivalent: ""
            )
            item.target = self
            item.representedObject = preset
            item.image = NSImage(systemSymbolName: "circle.fill", accessibilityDescription: nil)?
                .withTintColor(preset.nsColor)
            item.isEnabled = true
            menu.addItem(item)
        }

        if !colorPresets.isEmpty {
            menu.addItem(.separator())
        }

        let customItem = NSMenuItem(
            title: "Custom Color…",
            action: #selector(selectCustomColor),
            keyEquivalent: ""
        )
        customItem.target = self
        menu.addItem(customItem)

        return menu
    }

    func installColorMenu() {
        guard let mainMenu = NSApp.mainMenu,
              let viewMenu = mainMenu.item(withTitle: "View")?.submenu else { return }

        // Remove existing color submenu if present
        if let existing = viewMenu.item(withTitle: "Colors") {
            viewMenu.removeItem(existing)
        }

        let colorItem = NSMenuItem()
        colorItem.title = "Colors"
        colorItem.submenu = buildColorMenu()
        viewMenu.addItem(colorItem)
    }

    @objc private func selectColor(_ sender: NSMenuItem) {
        guard let preset = sender.representedObject as? ColorPreset else { return }
        NotificationCenter.default.post(name: .colorSelected, object: preset)
    }

    @objc private func selectCustomColor(_ sender: NSMenuItem) {
        NotificationCenter.default.post(name: .customColorRequested, object: nil)
    }
}

struct ColorPreset: Identifiable {
    let id = UUID()
    let name: String
    let nsColor: NSColor
}

extension Notification.Name {
    static let colorSelected = Notification.Name("colorSelected")
    static let customColorRequested = Notification.Name("customColorRequested")
}
```

## NSMenu Integration: Toggling Menu Item State

For checkmark-style menu items that reflect app state:

```swift
final class ViewOptionsController: NSObject {
    @objc var showLineNumbers = false {
        didSet { updateMenuItemState() }
    }

    @objc var showMinimap = false {
        didSet { updateMenuItemState() }
    }

    private weak var lineNumbersItem: NSMenuItem?
    private weak var minimapItem: NSMenuItem?

    func installItems() {
        guard let mainMenu = NSApp.mainMenu,
              let viewMenu = mainMenu.item(withTitle: "View")?.submenu else { return }

        let lineItem = NSMenuItem(
            title: "Show Line Numbers",
            action: #selector(toggleLineNumbers(_:)),
            keyEquivalent: ""
        )
        lineItem.target = self
        viewMenu.addItem(lineItem)
        self.lineNumbersItem = lineItem

        let miniItem = NSMenuItem(
            title: "Show Minimap",
            action: #selector(toggleMinimap(_:)),
            keyEquivalent: ""
        )
        miniItem.target = self
        viewMenu.addItem(miniItem)
        self.minimapItem = miniItem

        updateMenuItemState()
    }

    private func updateMenuItemState() {
        lineNumbersItem?.state = showLineNumbers ? .on : .off
        minimapItem?.state = showMinimap ? .on : .off
    }

    @objc func toggleLineNumbers(_ sender: NSMenuItem) {
        showLineNumbers.toggle()
    }

    @objc func toggleMinimap(_ sender: NSMenuItem) {
        showMinimap.toggle()
    }
}
```

## Dock Menu: Complete Pattern

Full dock menu with dynamic items and state observation:

```swift
import SwiftUI
import AppKit
import Combine

// Model for dock menu items
struct DockAction: Identifiable, Equatable {
    let id: String
    let title: String
    let icon: String?
    var isEnabled: Bool = true

    static func == (lhs: DockAction, rhs: DockAction) -> Bool {
        lhs.id == rhs.id
    }
}

// App delegate that provides the dock menu
final class DockMenuAppDelegate: NSObject, NSApplicationDelegate {
    @Published var actions: [DockAction] = []

    func applicationDockMenu(_ sender: NSApplication) -> NSMenu? {
        let menu = NSMenu()
        menu.autoenablesItems = false

        if actions.isEmpty {
            let item = NSMenuItem(title: "No Actions Available", action: nil, keyEquivalent: "")
            item.isEnabled = false
            menu.addItem(item)
            return menu
        }

        for action in actions {
            let item = NSMenuItem(
                title: action.title,
                action: #selector(dockActionSelected(_:)),
                keyEquivalent: ""
            )
            item.target = self
            item.representedObject = action.id
            item.isEnabled = action.isEnabled

            if let iconName = action.icon {
                item.image = NSImage(systemSymbolName: iconName, accessibilityDescription: action.title)
            }

            menu.addItem(item)
        }

        return menu
    }

    @objc private func dockActionSelected(_ sender: NSMenuItem) {
        guard let id = sender.representedObject as? String else { return }
        NotificationCenter.default.post(
            name: .dockActionTriggered,
            object: nil,
            userInfo: ["actionId": id]
        )
    }
}

extension Notification.Name {
    static let dockActionTriggered = Notification.Name("dockActionTriggered")
}

// SwiftUI integration
@main
struct MyApp: App {
    @NSApplicationDelegateAdaptor(DockMenuAppDelegate.self) var appDelegate
    @StateObject private var taskStore = TaskStore()

    var body: some Scene {
        WindowGroup {
            ContentView()
                .environmentObject(taskStore)
                .onChange(of: taskStore.activeTasks) { _, tasks in
                    appDelegate.actions = tasks.map { task in
                        DockAction(
                            id: task.id.uuidString,
                            title: "Complete \(task.name)",
                            icon: "checkmark.circle",
                            isEnabled: !task.isCompleted
                        )
                    }
                }
        }
    }
}

// Handling dock actions in SwiftUI
struct ContentView: View {
    @EnvironmentObject var taskStore: TaskStore

    var body: some View {
        List(taskStore.activeTasks) { task in
            TaskRow(task: task)
        }
        .onReceive(NotificationCenter.default.publisher(for: .dockActionTriggered)) { notification in
            guard let id = notification.userInfo?["actionId"] as? String,
                  let uuid = UUID(uuidString: id),
                  let task = taskStore.task(with: uuid) else { return }
            taskStore.complete(task)
        }
    }
}
```

## Services Menu: Complete Pattern

Providing and consuming services via `Info.plist` and code:

### Info.plist Service Registration

```xml
<key>NSServices</key>
<array>
    <dict>
        <!-- Service name shown in the Services menu -->
        <key>NSMenuItem</key>
        <dict>
            <key>default</key>
            <string>Send to MyApp</string>
        </dict>

        <!-- Method name the service calls on the app -->
        <key>NSMessage</key>
        <string>processSelectedText</string>

        <!-- App name for the service -->
        <key>NSPortName</key>
        <string>MyApp</string>

        <!-- What the service sends -->
        <key>NSSendTypes</key>
        <array>
            <string>NSStringPboardType</string>
        </array>

        <!-- What the service returns (optional) -->
        <key>NSReturnTypes</key>
        <array>
            <string>NSStringPboardType</string>
        </array>

        <!-- Context in which the service is available -->
        <key>NSRequiredContext</key>
        <dict>
            <key>NSTextContext</key>
            <array>
                <string>Word</string>
            </array>
        </dict>
    </dict>
</array>
```

### Service Handler Implementation

```swift
final class ServiceAppDelegate: NSObject, NSApplicationDelegate {
    func application(_ app: NSApplication, pasteboard: NSPasteboard, userData: String, error: AutoreleasingUnsafeMutablePointer<NSString?>) -> Bool {
        // Read input from pasteboard
        guard let input = pasteboard.string(forType: .string), !input.isEmpty else {
            error.pointee = "No text content available" as NSString
            return false
        }

        // Process the input
        let result = processText(input)

        // Write result back to pasteboard (if the service returns data)
        pasteboard.clearContents()
        pasteboard.setString(result, forType: .string)

        return true
    }

    private func processText(_ text: String) -> String {
        // Transform the text
        return text.uppercased()
    }
}
```

### Receiving Services in a Custom NSView

```swift
final class ServiceAwareTextView: NSTextView {
    // Declare which services this view can handle
    override func validRequestor(forSendType sendType: NSPasteboard.PasteboardType?, returnType: NSPasteboard.PasteboardType?) -> Any? {
        // Accept string-based services
        if let sendType = sendType, sendType == .string {
            return self
        }
        return super.validRequestor(forSendType: sendType, returnType: returnType)
    }

    // Handle the service action
    override func readSelection(from pboard: NSPasteboard, type: NSPasteboard.PasteboardType) -> Bool {
        if type == .string, let text = pboard.string(forType: .string) {
            self.insertText(text, replacementRange: self.selectedRange())
            return true
        }
        return super.readSelection(from: pboard, type: type)
    }
}
```

## Context Menu with Selection-Aware Actions

Context menus that respond to list selection state:

```swift
struct SelectableListView: View {
    @State private var items: [Item] = []
    @State private var selection: Set<Item.ID> = []

    var body: some View {
        List(items, selection: $selection) { item in
            ItemRow(item: item)
                .contextMenu {
                    // Single-selection actions
                    if selection.count == 1 {
                        Button("Edit") { editItem(with: selection.first!) }
                        Button("Duplicate") { duplicateItem(with: selection.first!) }
                        Divider()
                    }

                    // Multi-selection actions
                    if !selection.isEmpty {
                        Button("Export Selected (\(selection.count))") { exportSelected() }
                        Button("Tag…") { tagSelected() }
                        Divider()
                    }

                    // Always available
                    Button("New Item") { newItem() }

                    if !selection.isEmpty {
                        Divider()
                        Button("Delete", role: .destructive) { deleteSelected() }
                            .keyboardShortcut(.delete, modifiers: [])
                    }
                }
        }
    }

    private func editItem(with id: Item.ID) { /* ... */ }
    private func duplicateItem(with id: Item.ID) { /* ... */ }
    private func exportSelected() { /* ... */ }
    private func tagSelected() { /* ... */ }
    private func newItem() { /* ... */ }
    private func deleteSelected() { /* ... */ }
}
```

## Menu Bar Extra (Status Item) Pattern

For apps that live in the menu bar (not the dock), use `NSStatusBar`:

```swift
import SwiftUI
import AppKit

final class MenuBarController: ObservableObject {
    @Published var isConnected = false
    private var statusItem: NSStatusItem?

    func setupStatusItem() {
        statusItem = NSStatusBar.system.statusItem(withLength: NSStatusItem.variableLength)

        guard let button = statusItem?.button else { return }

        button.image = NSImage(systemSymbolName: "network", accessibilityDescription: "Status")
        button.action = #selector(showMenu)
        button.target = self
    }

    @objc private func showMenu() {
        let menu = NSMenu()
        menu.autoenablesItems = false

        let statusItem = NSMenuItem(title: isConnected ? "Connected" : "Disconnected", action: nil, keyEquivalent: "")
        statusItem.isEnabled = false
        menu.addItem(statusItem)

        menu.addItem(.separator())

        let toggleItem = NSMenuItem(
            title: isConnected ? "Disconnect" : "Connect",
            action: #selector(toggleConnection),
            keyEquivalent: ""
        )
        toggleItem.target = self
        menu.addItem(toggleItem)

        menu.addItem(.separator())

        let prefsItem = NSMenuItem(
            title: "Preferences…",
            action: #selector(showPreferences),
            keyEquivalent: ","
        )
        prefsItem.target = self
        menu.addItem(prefsItem)

        let quitItem = NSMenuItem(
            title: "Quit",
            action: #selector(quit),
            keyEquivalent: "q"
        )
        quitItem.target = self
        menu.addItem(quitItem)

        statusItem?.menu = menu
        statusItem?.button?.performClick(nil)
        menu.cancelTracking()
    }

    @objc private func toggleConnection() {
        isConnected.toggle()
    }

    @objc private func showPreferences() {
        // Show preferences window
    }

    @objc private func quit() {
        NSApp.terminate(nil)
    }
}
```

## Validation Pattern: Coordinating SwiftUI and AppKit

When mixing SwiftUI commands with AppKit menu items, coordinate validation through a shared model:

```swift
@MainActor
final class MenuValidationModel: ObservableObject {
    @Published var canSave = false
    @Published var canUndo = false
    @Published var canRedo = false
    @Published var hasSelection = false

    // Called from SwiftUI side
    func updateFromSwiftUI(save: Bool, selection: Bool) {
        canSave = save
        hasSelection = selection
    }

    // Called from AppKit side (NSDocument, etc.)
    func updateFromAppKit(undo: Bool, redo: Bool) {
        canUndo = undo
        canRedo = redo
    }
}

// SwiftUI commands read from the model:
struct DocumentCommands: View {
    @ObservedObject var validation: MenuValidationModel

    var body: some View {
        Group {
            CommandGroup(after: .saveItem) {
                Button("Export…") { export() }
                    .keyboardShortcut("e", modifiers: [.command, .shift])
                    .disabled(!validation.canSave)
            }

            CommandGroup(after: .pasteboard) {
                Button("Delete", role: .destructive) { delete() }
                    .disabled(!validation.hasSelection)
            }
        }
    }

    private func export() { /* ... */ }
    private func delete() { /* ... */ }
}

// AppKit validation reads from the same model:
extension Document: NSMenuItemValidation {
    func validateMenuItem(_ menuItem: NSMenuItem) -> Bool {
        switch menuItem.action {
        case #selector(save(_:)):
            return validationModel.canSave
        case #selector(undo(_:)):
            return validationModel.canUndo
        case #selector(redo(_:)):
            return validationModel.canRedo
        default:
            return true
        }
    }
}
```
