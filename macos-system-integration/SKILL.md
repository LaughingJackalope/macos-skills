---
name: macos-system-integration
description: "System-level macOS integration patterns: SMAppService login items and launch agents, NSWorkspace app lifecycle, security-scoped bookmarks, NSPasteboard custom UTType and promised data, Core Spotlight indexing, Quick Look preview, AppleScript automation, and NSFileCoordinator/NSFilePresenter coordinated file access. Use when registering daemons or agents, opening files and launching apps, persisting sandbox file access, implementing drag-and-drop with custom types, indexing content for Spotlight, presenting Quick Look previews, scripting other apps, or coordinating file reads/writes across processes."
sources: https://developer.apple.com/documentation/servicemanagement/smappservice, https://developer.apple.com/forums/thread/719862, https://developer.apple.com/forums/thread/758393, https://developer.apple.com/documentation/corespotlight, https://developer.apple.com/documentation/quicklook
researched: 2026-06-05
platform: macOS 26+
---

# macOS System Integration

System-level integration patterns for macOS 26+ with Swift 6.3. Covers login item and launch agent registration via `SMAppService`, app lifecycle observation with `NSWorkspace`, security-scoped bookmark persistence, pasteboard operations with custom UTTypes, Core Spotlight indexing, Quick Look preview, AppleScript task execution, and coordinated file access with `NSFileCoordinator`/`NSFilePresenter`.

See [references/servicemanagement-launch-agents.md](references/servicemanagement-launch-agents.md) for complete SMAppService patterns, LaunchAgent/LaunchDaemon plist format, and migration from `SMLoginItemSetEnabled`. See [references/pasteboard-spotlight.md](references/pasteboard-spotlight.md) for NSPasteboard patterns, custom UTType registration, Core Spotlight indexing, and Quick Look integration.

## Contents

- [SMAppService: Login Items and Launch Agents](#smappservice-login-items-and-launch-agents)
- [NSWorkspace: App Lifecycle and File Opening](#nsworkspace-app-lifecycle-and-file-opening)
- [Security-Scoped Bookmarks](#security-scoped-bookmarks)
- [NSPasteboard: Custom Types and Promised Data](#nspasteboard-custom-types-and-promised-data)
- [Core Spotlight on macOS](#core-spotlight-on-macos)
- [Quick Look](#quick-look)
- [AppleScript Automation](#applescript-automation)
- [NSFileCoordinator and NSFilePresenter](#nsfilecoordinator-and-nsfilepresenter)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

**Scope boundary:** SwiftUI state management patterns are covered in the `swiftui-patterns` skill. Keychain and credential storage are in the `swift-security` skill. AppKit window/view fundamentals are in the `appkit-foundations` skill.

## SMAppService: Login Items and Launch Agents

`SMAppService` (macOS 13+) is the modern, unified API for registering login items, launch agents, and launch daemons. It replaces the deprecated `SMLoginItemSetEnabled` and the manual `LSSharedFileList` approach.

### Service Types

| Type | Purpose | Runs As |
|------|---------|---------|
| `.loginItem` | Helper app launched at user login | User context, GUI allowed |
| `.agent` | LaunchAgent — loaded by launchd at login | User context, no GUI by default |
| `.daemon` | LaunchDaemon — loaded at system boot | Root or specified user, no GUI |

### Registering a Login Item

The helper bundle must be a separate `.app` bundle inside the main app's `Contents/Library/LoginItems/` directory. The helper's `Info.plist` sets `LSBackgroundOnly` or `LSUIElement` to control GUI access.

```swift
import ServiceManagement

final class LoginItemManager: ObservableObject {
    let service = SMAppService.loginItem(identifier: "com.example.MyApp.Helper")

    @Published var status: SMAppService.Status = .notRegistered

    init() {
        status = service.status
    }

    func register() async throws {
        try await service.register()
        status = service.status
    }

    func unregister() async throws {
        try await service.unregister()
        status = service.status
    }

    func toggle() async throws {
        switch service.status {
        case .notRegistered, .notFound:
            try await register()
        case .enabled, .requiresApproval:
            try await unregister()
        @unknown default:
            break
        }
    }
}
```

**Status values:**

| Status | Meaning |
|--------|---------|
| `.notRegistered` | Not registered with the system |
| `.enabled` | Registered and currently loaded |
| `.requiresApproval` | Registered but needs user approval in System Settings > Login Items |
| `.notFound` | The helper bundle could not be found at the expected path |

### Registering a Launch Agent

Place the agent's `LaunchAgent` plist in `~/Library/LaunchAgents/` (per-user) or `/Library/LaunchAgents/` (system-wide). The agent is a separate executable bundled inside the app.

```swift
let agent = SMAppService.agent(plistName: "com.example.MyApp.Agent.plist")

// Register (installs the plist and loads it)
try await agent.register()

// Check status
switch agent.status {
case .enabled:
    print("Agent is running")
case .requiresApproval:
    print("User must approve in System Settings")
case .notRegistered:
    print("Not installed")
case .notFound:
    print("Plist not found at expected path")
@unknown default:
    break
}

// Unregister (unloads and removes the plist)
try await agent.unregister()
```

### Migration from SMLoginItemSetEnabled

The old `SMLoginItemSetEnabled` API is deprecated. Migration path:

1. Replace `SMLoginItemSetEnabled("com.example.Helper", true)` with `try await SMAppService.loginItem(identifier: "com.example.Helper").register()`.
2. Replace `SMLoginItemSetEnabled("com.example.Helper", false)` with `try await SMAppService.loginItem(identifier: "com.example.Helper").unregister()`.
3. The helper bundle location (`Contents/Library/LoginItems/`) is unchanged.
4. `SMAppService` adds async/await support and proper error handling — the old API returned `Bool` with no error details.
5. The new API reports `.requiresApproval` status, which the old API did not surface. Handle this by directing users to System Settings.

### Approval Flow

On macOS 13+, newly registered agents and login items appear in System Settings > General > Login Items. The status is `.requiresApproval` until the user toggles them on. There is no API to programmatically approve — the user must do it. Check status after `register()` and prompt the user to approve if needed.

```swift
try await service.register()
if service.status == .requiresApproval {
    // Direct user to System Settings
    if let url = URL(string: "x-apple.systempreferences:com.apple.LoginItems-Settings.extension") {
        NSWorkspace.shared.open(url)
    }
}
```

## NSWorkspace: App Lifecycle and File Opening

`NSWorkspace` (replacing the deprecated `NSWorkspace.shared` singleton pattern) is the primary interface for launching apps, opening files, and observing the app lifecycle on macOS.

### Opening Files and URLs

```swift
// Open a file with its default handler
let config = NSWorkspace.OpenConfiguration()
config.activates = true  // bring the target app to front
config.createsNewApplicationInstance = false

try await NSWorkspace.shared.open(
    fileURL,
    configuration: config
)

// Open a URL in the default browser
try await NSWorkspace.shared.open(URL(string: "https://example.com")!)

// Open a file with a specific application
try await NSWorkspace.shared.open(
    [fileURL],
    withApplicationAt: appURL,
    configuration: config
)
```

### Launching Applications

```swift
// Launch an application by bundle identifier
let config = NSWorkspace.OpenConfiguration()
config.environment = ["MY_APP_MODE": "debug"]
config.hides = false

let runningApp = try await NSWorkspace.shared.openApplication(
    at: appBundleURL,
    configuration: config
)

// Check if an app is already running
let runningApps = NSWorkspace.shared.runningApplications
let isRunning = runningApps.contains { $0.bundleIdentifier == "com.example.TargetApp" }
```

### Observing App Lifecycle

Use `NSWorkspace.shared.notificationCenter` to observe app state changes. These notifications are essential for reacting to app activation, deactivation, termination, and device sleep/wake.

```swift
final class AppLifecycleObserver {
    private var cancellables = Set<AnyCancellable>()

    init() {
        let center = NSWorkspace.shared.notificationCenter

        center.publisher(for: NSWorkspace.didLaunchApplicationNotification)
            .compactMap { $0.userInfo?[NSWorkspace.applicationUserInfoKey] as? NSRunningApplication }
            .sink { app in
                print("Launched: \(app.localizedName ?? "unknown")")
            }
            .store(in: &cancellables)

        center.publisher(for: NSWorkspace.didTerminateApplicationNotification)
            .compactMap { $0.userInfo?[NSWorkspace.applicationUserInfoKey] as? NSRunningApplication }
            .sink { app in
                print("Terminated: \(app.localizedName ?? "unknown")")
            }
            .store(in: &cancellables)

        center.publisher(for: NSWorkspace.didActivateApplicationNotification)
            .compactMap { $0.userInfo?[NSWorkspace.applicationUserInfoKey] as? NSRunningApplication }
            .sink { app in
                print("Activated: \(app.localizedName ?? "unknown")")
            }
            .store(in: &cancellables)

        center.publisher(for: NSWorkspace.willSleepNotification)
            .sink { _ in
                // Save state, release resources
            }
            .store(in: &cancellables)

        center.publisher(for: NSWorkspace.didWakeNotification)
            .sink { _ in
                // Reconnect, refresh state
            }
            .store(in: &cancellables)
    }
}
```

**Key notifications:**

| Notification | When |
|---|---|
| `didLaunchApplicationNotification` | An app finishes launching |
| `didTerminateApplicationNotification` | An app has quit |
| `didActivateApplicationNotification` | An app becomes active (frontmost) |
| `didDeactivateApplicationNotification` | An app resigns active status |
| `didHideApplicationNotification` | An app is hidden |
| `didUnhideApplicationNotification` | An app is unhidden |
| `willSleepNotification` | System is about to sleep |
| `didWakeNotification` | System woke from sleep |
| `screensDidSleepNotification` | Displays went to sleep |
| `screensDidWakeNotification` | Displays woke up |

## Security-Scoped Bookmarks

Security-scoped bookmarks let sandboxed apps persist access to user-selected files and directories across launches. Without bookmarks, a sandboxed app loses file access when it restarts.

### Creating a Bookmark

```swift
func createSecurityScopedBookmark(for url: URL) throws -> Data {
    let bookmarkData = try url.bookmarkData(
        options: .withSecurityScope,
        includingResourceValuesForKeys: nil,
        relativeTo: nil
    )
    // Persist bookmarkData in UserDefaults, a file, or your data store
    UserDefaults.standard.set(bookmarkData, forKey: "savedFolderBookmark")
    return bookmarkData
}
```

### Resolving and Accessing

```swift
func accessBookmarkedFolder() throws -> URL {
    guard let bookmarkData = UserDefaults.standard.data(forKey: "savedFolderBookmark") else {
        throw BookmarkError.noBookmarkStored
    }

    var isStale = false
    let url = try URL(
        resolvingBookmarkData: bookmarkData,
        options: .withSecurityScope,
        relativeTo: nil,
        bookmarkDataIsStale: &isStale
    )

    if isStale {
        // Re-create the bookmark with fresh data
        let freshData = try createSecurityScopedBookmark(for: url)
        UserDefaults.standard.set(freshData, forKey: "savedFolderBookmark")
    }

    // MUST call startAccessingSecurityScopedResource before accessing the file
    guard url.startAccessingSecurityScopedResource() else {
        throw BookmarkError.accessDenied
    }

    return url
}

// Usage
func processFiles() throws {
    let url = try accessBookmarkedFolder()
    defer { url.stopAccessingSecurityScopedResource() }

    // Access files within the bookmarked scope
    let contents = try FileManager.default.contentsOfDirectory(at: url, includingPropertiesForKeys: nil)
    // ... process files ...
}
```

### Critical Rules

- **Always call `startAccessingSecurityScopedResource()`** before accessing the file. It returns `Bool` — check it.
- **Always balance with `stopAccessingSecurityScopedResource()`** when done. Use `defer` to guarantee cleanup.
- **Check `isStale`** when resolving. If stale, re-create the bookmark from the resolved URL.
- **Bookmarks are per-user, per-volume.** They break if the user moves the file or the volume is remounted.
- **Store bookmark data, not the path.** Paths change; bookmarks track the file by inode.

## NSPasteboard: Custom Types and Promised Data

`NSPasteboard` is the macOS clipboard and drag-and-drop transport. For custom data types, register a `UTType` and use it as the pasteboard type.

### Reading and Writing Standard Types

```swift
// Write a string to the general pasteboard
let pb = NSPasteboard.general
pb.clearContents()
pb.setString("Hello, macOS!", forType: .string)

// Read a string
if let string = pb.string(forType: .string) {
    print("Clipboard: \(string)")
}

// Write multiple representations
pb.clearContents()
pb.writeObjects([attributedString])  // NSAttributedString conforms to NSPasteboardWriting
pb.setData(jsonData, forType: .json) // if UTType.json is available
```

### Custom UTType Registration

Define custom types in your app's `Info.plist` under `UTExportedTypeDeclarations` (for types your app owns) or `UTImportedTypeDeclarations` (for types from other apps you want to handle):

```swift
// In code, reference your custom type
extension UTType {
    static let myDocument = UTType(exportedAs: "com.example.myapp.document", conformingTo: .data)
    static let projectItem = UTType(exportedAs: "com.example.myapp.project-item", conformingTo: .item)
}

// Write custom type
let data = try JSONEncoder().encode(myProjectItem)
pb.clearContents()
pb.setData(data, forType: .myDocument)

// Read custom type
if let data = pb.data(forType: .myDocument),
   let item = try? JSONDecoder().decode(ProjectItem.self, from: data) {
    // use item
}
```

### Promised Data (Lazy Delivery)

For large data, promise the data and deliver it only when the destination requests it. Implement `NSPasteboardItemDataProvider`:

```swift
final class LargeFileDataProvider: NSObject, NSPasteboardItemDataProvider {
    func pasteboard(_ pasteboard: NSPasteboard?, item: NSPasteboardItem, dataForType type: NSPasteboard.PasteboardType) {
        // Called when the destination actually requests the data
        guard type == .myDocument else { return }
        do {
            let data = try generateLargeFileData()  // expensive operation
            item.setData(data, forType: type)
        } catch {
            // If this fails, the pasteboard operation fails
        }
    }

    func pasteboard(_ pasteboard: NSPasteboard?, item: NSPasteboardItem, dataForType type: NSPasteboard.PasteboardType) {
        // Deprecated but may still be called on older systems
    }
}

// Promise the data
let item = NSPasteboardItem()
item.setDataProvider(LargeFileDataProvider(), forTypes: [.myDocument])
let pb = NSPasteboard.general
pb.clearContents()
pb.writeObjects([item])
```

## Core Spotlight on macOS

Core Spotlight lets your app index content so it appears in Spotlight search results. On macOS, users expect deep search into document content, metadata, and app-specific data.

### Indexing Items

```swift
import CoreSpotlight

final class SpotlightIndexer {
    let index = CSSearchableIndex.default()

    func indexDocument(id: String, title: String, content: String, thumbnail: Data? = nil) async throws {
        let attributeSet = CSSearchableItemAttributeSet(contentType: .text)
        attributeSet.title = title
        attributeSet.contentDescription = content
        attributeSet.thumbnailData = thumbnail
        attributeSet.keywords = ["document", "project"]
        attributeSet.displayName = title

        // CSLocalizedString for localized display
        let localizedTitle = CSLocalizedString(localizedStrings: [
            "en": title,
            "ja": japaneseTitle
        ])
        attributeSet.title = localizedTitle

        let item = CSSearchableItem(
            uniqueIdentifier: id,
            domainIdentifier: "com.example.myapp.documents",
            attributeSet: attributeSet
        )

        try await index.indexSearchableItems([item])
    }

    func removeDocument(id: String) async throws {
        try await index.deleteSearchableItems(withIdentifiers: [id])
    }

    func removeAllDocuments() async throws {
        try await index.deleteSearchableItems(withDomainIdentifiers: ["com.example.myapp.documents"])
    }
}
```

### Handling Spotlight Selection

When the user clicks a Spotlight result, your app receives the unique identifier via `NSUserActivity`:

```swift
// In your App struct or scene delegate
.onContinueUserActivity(CSSearchableItemActionType) { activity in
    if let uniqueIdentifier = activity.userInfo?[CSSearchableItemActivityIdentifier] as? String {
        // Navigate to the document identified by uniqueIdentifier
        openDocument(id: uniqueIdentifier)
    }
}
```

### Batch Indexing

For large datasets, use batch indexing with completion handlers:

```swift
func batchIndex(items: [(id: String, title: String, content: String)]) async throws {
    let searchableItems = items.map { item in
        let attributeSet = CSSearchableItemAttributeSet(contentType: .text)
        attributeSet.title = item.title
        attributeSet.contentDescription = item.content
        return CSSearchableItem(
            uniqueIdentifier: item.id,
            domainIdentifier: "com.example.myapp.documents",
            attributeSet: attributeSet
        )
    }

    try await index.indexSearchableItems(searchableItems)
}
```

## Quick Look

Quick Look provides file previews without opening the full app. Integrate it via `QLPreviewPanel` (AppKit) or `.quickLookPreview` (SwiftUI).

### QLPreviewPanel (AppKit)

```swift
final class PreviewController: NSObject, QLPreviewPanelDataSource, QLPreviewPanelDelegate {
    var previewItems: [URL] = []

    func showPreviewPanel() {
        guard let panel = QLPreviewPanel.shared() else { return }
        if panel.isVisible {
            panel.orderOut(nil)
        } else {
            panel.makeKeyAndOrderFront(nil)
        }
    }

    // QLPreviewPanelDataSource
    func numberOfPreviewItems(in panel: QLPreviewPanel!) -> Int {
        previewItems.count
    }

    func previewPanel(_ panel: QLPreviewPanel!, previewItemAt index: Int) -> QLPreviewItem! {
        previewItems[index] as NSURL
    }

    // Make the controller the panel's data source and delegate
    override func acceptsPreviewPanelControl(_ panel: QLPreviewPanel!) -> Bool {
        true
    }

    override func beginPreviewPanelControl(_ panel: QLPreviewPanel!) {
        panel.dataSource = self
        panel.delegate = self
    }

    override func endPreviewPanelControl(_ panel: QLPreviewPanel!) {
        panel.dataSource = nil
        panel.delegate = nil
    }
}
```

### SwiftUI .quickLookPreview

```swift
struct DocumentListView: View {
    let documents: [URL]
    @State private var selectedURL: URL?

    var body: some View {
        List(documents, id: \.self, selection: $selectedURL) { url in
            Text(url.lastPathComponent)
        }
        .quickLookPreview($selectedURL)
    }
}
```

### Custom QLPreviewItem

For in-memory data or non-file previews, implement `QLPreviewItem`:

```swift
final class InMemoryPreviewItem: NSObject, QLPreviewItem {
    let previewItemURL: URL
    let previewItemTitle: String?

    init(data: Data, fileName: String, title: String?) throws {
        let tempURL = FileManager.default.temporaryDirectory
            .appendingPathComponent(fileName)
        try data.write(to: tempURL)
        self.previewItemURL = tempURL
        self.previewItemTitle = title
    }
}
```

## AppleScript Automation

`NSUserAppleScriptTask` runs AppleScript files bundled within your app, with access to the app's scripting dictionary. This is the sandboxed alternative to `NSAppleScript`.

### Running a Bundled AppleScript

Place `.scpt` files in your app's `Contents/Resources/Scripts/` directory:

```swift
final class ScriptRunner {
    enum ScriptError: Error {
        case scriptNotFound
        case executionFailed(String)
    }

    func runScript(named name: String, withArguments arguments: [String] = []) async throws -> NSAppleEventDescriptor {
        guard let scriptURL = Bundle.main.url(forResource: name, withExtension: "scpt", subdirectory: "Scripts") else {
            throw ScriptError.scriptNotFound
        }

        let task = try NSUserAppleScriptTask(url: scriptURL)

        let event = NSAppleEventDescriptor(eventClass: coreAppleEventClass,
                                           eventID: kAEOpenApplication,
                                           targetDescriptor: nil,
                                           returnID: AEReturnID(kAutoGenerateReturnID),
                                           transactionID: AETransactionID(kAnyTransactionID))

        // Add arguments as a list parameter
        let argumentList = NSAppleEventDescriptor.list()
        for (index, arg) in arguments.enumerated() {
            argumentList.insert(NSAppleEventDescriptor(string: arg), at: index + 1)
        }
        event.setParam(argumentList, forKeyword: keyDirectObject)

        return try await withCheckedThrowingContinuation { continuation in
            task.execute(withAppleEvent: event) { result, error in
                if let error {
                    continuation.resume(throwing: ScriptError.executionFailed(error.localizedDescription))
                } else if let result {
                    continuation.resume(returning: result)
                } else {
                    continuation.resume(returning: NSAppleEventDescriptor.null())
                }
            }
        }
    }
}
```

### NSAppleEventDescriptor for Inter-App Communication

For sending Apple events directly to other apps:

```swift
func tellFinderToReveal(fileURL: URL) throws {
    let target = NSAppleEventDescriptor(
        bundleIdentifier: "com.apple.finder"
    )

    let event = NSAppleEventDescriptor(
        eventClass: kCoreEventClass,
        eventID: kAEOpenDocuments,
        targetDescriptor: target,
        returnID: AEReturnID(kAutoGenerateReturnID),
        transactionID: AETransactionID(kAnyTransactionID)
    )

    let fileList = NSAppleEventDescriptor.list()
    let fileDesc = NSAppleEventDescriptor(fileURL: fileURL)
    fileList.insert(fileDesc, at: 1)
    event.setParam(fileList, forKeyword: keyDirectObject)

    var error: NSDictionary?
    event.sendEvent(options: [.noReply],
                    timeout: TimeInterval(kAEDefaultTimeout),
                    error: &error)

    if let error {
        throw ScriptError.executionFailed(error.description)
    }
}
```

### Scripting Dictionary

Define your app's scripting interface in `sdef` files (Scripting Definition). Add the `NSAppleScriptEnabled` key to `Info.plist`. The scripting dictionary exposes your app's objects and commands to AppleScript and Automator.

## NSFileCoordinator and NSFilePresenter

`NSFileCoordinator` serializes file access across processes to prevent data corruption. `NSFilePresenter` lets your app react to changes made by other processes.

### Coordinated Reading and Writing

```swift
final class CoordinatedFileReader {
    private let fileCoordinator = NSFileCoordinator()

    func read(from url: URL) throws -> Data {
        var result: Data?
        var readError: Error?

        fileCoordinator.coordinate(readingItemAt: url, options: .withoutChanges, error: &readError) { coordinatedURL in
            do {
                result = try Data(contentsOf: coordinatedURL)
            } catch {
                readError = error
            }
        }

        if let readError { throw readError }
        return result!
    }

    func write(_ data: Data, to url: URL) throws {
        var writeError: Error?

        fileCoordinator.coordinate(writingItemAt: url, options: .forReplacing, error: &writeError) { coordinatedURL in
            do {
                try data.write(to: coordinatedURL)
            } catch {
                writeError = error
            }
        }

        if let writeError { throw writeError
        }
    }
}
```

### Implementing NSFilePresenter

```swift
final class DocumentFilePresenter: NSObject, NSFilePresenter {
    let presentedItemURL: URL
    let presentedItemOperationQueue = OperationQueue.main

    var onDidChange: (() -> Void)?
    var onDidMove: ((URL) -> Void)?

    init(url: URL) {
        self.presentedItemURL = url
        super.init()
        NSFileCoordinator.addFilePresenter(self)
    }

    deinit {
        NSFileCoordinator.removeFilePresenter(self)
    }

    // Another process is about to change the file
    func relinquishPresentedItem(toReader reader: @escaping @Sendable ((() -> Void)?) -> Void) {
        // Save any unsaved state, then allow the reader
        saveIfNeeded()
        reader()  // call the completion handler when ready
    }

    func relinquishPresentedItem(toWriter writer: @escaping @Sendable ((() -> Void)?) -> Void) {
        saveIfNeeded()
        writer()
    }

    // The file was changed by another process
    func presentedItemDidChange() {
        onDidChange?()
    }

    func presentedItemDidMove(to newURL: URL) {
        onDidMove?(newURL)
    }

    func presentedItemDidChangeUbiquityAttributes(_ attributes: Set<URLResourceKey>) {
        // Handle iCloud attribute changes
    }

    private func saveIfNeeded() {
        // Persist any in-memory changes before the other process writes
    }
}
```

### Coordination Options

| Option | Use When |
|--------|----------|
| `.forReading` | Reading a file that others may also read |
| `.forWriting` | Writing to a file |
| `.forDeleting` | Deleting a file |
| `.forMoving` | Moving/renaming a file |
| `.forMerging` | Merging changes after a coordinated write |
| `.withoutChanges` | Reading without triggering change notifications |
| `.forReplacing` | Writing that replaces the entire file |

## Constraints & Gotchas

- **SMAppService is macOS 13+ only.** For macOS 12 and below, use `SMLoginItemSetEnabled` (deprecated but functional) or `LSSharedFileList`.
- **SMAppService.register() is async.** It can take seconds if the system needs to validate the helper bundle. Never call it on the main thread without `await`.
- **Security-scoped bookmarks require user interaction.** The user must explicitly select the file/folder (via `NSOpenPanel` or drag-and-drop). You cannot bookmark arbitrary paths.
- **NSPasteboard operations must be on the main thread.** The pasteboard server connection is main-thread bound.
- **Core Spotlight indexing is rate-limited.** Don't index every keystroke. Batch updates and use `CSSearchableIndex.default().indexSearchableItems()`.
- **QLPreviewPanel requires a key window.** It won't display if no window is key. Call `window.makeKeyAndOrderFront(nil)` first.
- **NSUserAppleScriptTask requires the `NSAppleEventsUsageDescription` key** in `Info.plist with a usage description string.
- **NSFileCoordinator is process-local.** It coordinates between threads and `NSFilePresenter` instances within your process, and with `NSFileCoordinator` instances in other processes. It does not coordinate with raw `FileManager` calls.
- **Always use `defer { url.stopAccessingSecurityScopedResource() }`** after `startAccessingSecurityScopedResource()`. Forgetting this leaks a kernel resource.
- **NSWorkspace notifications are posted to `NSWorkspace.shared.notificationCenter`,** not `NotificationCenter.default`. Using the wrong center means you never receive them.

## Common Mistakes

1. **Calling `SMAppService.register()` synchronously.** It's an `async` method. Calling it without `await` (or from a non-async context without proper wrapping) silently fails or blocks.

2. **Not handling `.requiresApproval` status.** After `register()`, the service may not be enabled until the user approves it in System Settings. Always check the status and guide the user.

3. **Forgetting `stopAccessingSecurityScopedResource()`.** This leaks a kernel resource. The system may revoke your access. Always use `defer`.

4. **Storing file paths instead of bookmark data.** Paths break across relaunches, volume remounts, and file moves. Always use security-scoped bookmarks for persistent access.

5. **Using `NSPasteboard.general` from a background thread.** The pasteboard is main-thread bound. Dispatch to main or use `@MainActor`.

6. **Indexing too frequently with Core Spotlight.** The system throttles rapid updates. Debounce or batch your indexing operations.

7. **Not implementing `relinquishPresentedItem(toWriter:)` in `NSFilePresenter`.** Without this, the file coordinator cannot serialize writes properly, leading to data races.

8. **Using `NSAppleScript` directly in a sandboxed app.** It's blocked by the sandbox. Use `NSUserAppleScriptTask` with bundled scripts instead.

9. **Assuming `NSWorkspace` notifications arrive on the main thread.** They arrive on the thread that posted them. Dispatch UI updates to the main queue.

10. **Not setting `presentedItemOperationQueue` in `NSFilePresenter`.** The default queue is the main queue, but if you set a custom queue, ensure it's serial to avoid re-entrancy issues.

## Review Checklist

- [ ] SMAppService registration uses `await` and handles all status cases including `.requiresApproval`
- [ ] Security-scoped bookmarks use `startAccessingSecurityScopedResource()` / `stopAccessingSecurityScopedResource()` in balanced pairs with `defer`
- [ ] Bookmark staleness is checked and refreshed when `isStale == true`
- [ ] NSPasteboard operations run on the main thread
- [ ] Custom UTType is declared in `Info.plist` under `UTExportedTypeDeclarations`
- [ ] Core Spotlight indexing is batched, not per-keystroke
- [ ] `CSSearchableItem` uses both `uniqueIdentifier` and `domainIdentifier`
- [ ] Spotlight selection is handled via `CSSearchableItemActionType` user activity
- [ ] QLPreviewPanel data source/delegate are set in `beginPreviewPanelControl` and cleared in `endPreviewPanelControl`
- [ ] `NSUserAppleScriptTask` is used instead of `NSAppleScript` in sandboxed apps
- [ ] `NSAppleEventsUsageDescription` is set in `Info.plist`
- [ ] `NSFilePresenter` implements `relinquishPresentedItem(toReader:)` and `relinquishPresentedItem(toWriter:)`
- [ ] `NSFileCoordinator` is used for all file I/O that may be accessed by multiple processes
- [ ] `NSWorkspace` notifications are observed on `NSWorkspace.shared.notificationCenter`, not `NotificationCenter.default`
- [ ] All code targets macOS 26+ with Swift 6.3 concurrency (`async/await`, `@MainActor` where needed)

## References

- [references/servicemanagement-launch-agents.md](references/servicemanagement-launch-agents.md) — Complete SMAppService patterns, LaunchAgent/LaunchDaemon plist format, migration from `SMLoginItemSetEnabled`
- [references/pasteboard-spotlight.md](references/pasteboard-spotlight.md) — NSPasteboard patterns, custom UTType registration, Core Spotlight indexing, Quick Look integration
- SwiftUI state management: see `swiftui-patterns` skill
- Keychain and credential storage: see `swift-security` skill
- AppKit fundamentals: see `appkit-foundations` skill
