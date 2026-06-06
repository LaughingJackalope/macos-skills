# API Design Patterns Reference

Complete working patterns for macOS API design: AppKit-style delegate protocol with SwiftUI wrapper, error type design with CocoaError bridging, Swift package structure for a macOS-only library, and `@objc` protocol with optional methods.

---

## Contents

- [1. AppKit-Style Delegate Protocol with SwiftUI Wrapper](#1-appkit-style-delegate-protocol-with-swiftui-wrapper)
- [2. Error Type Design with CocoaError Bridging](#2-error-type-design-with-cocoaerror-bridging)
- [3. Swift Package Structure for macOS-Only Library](#3-swift-package-structure-for-macos-only-library)
- [4. @objc Protocol with Optional Methods](#4-objc-protocol-with-optional-methods)

---

## 1. AppKit-Style Delegate Protocol with SwiftUI Wrapper

This pattern shows a complete AppKit-style delegate protocol, its implementation by an `NSViewController`, and a SwiftUI wrapper using `NSViewControllerRepresentable`.

### The Delegate Protocol

```swift
import AppKit

// MARK: - Delegate Protocol

/// Delegate for a source list sidebar that manages item selection and reordering.
/// Conforming types must be `NSObject` subclasses due to `@objc optional` requirements.
@objc protocol SourceListDelegate: AnyObject {
    // Required: selection change notification
    func sourceList(_ sourceList: SourceListView, didSelectItem item: SourceListItem)

    // Required: permission to reorder
    func sourceList(_ sourceList: SourceListView, shouldMoveItem item: SourceListItem, to index: Int) -> Bool

    // Required: reordering completed
    func sourceList(_ sourceList: SourceListView, didMoveItem item: SourceListItem, to index: Int)

    // Optional: context menu provider
    @objc optional func sourceList(_ sourceList: SourceListView, menuForItem item: SourceListItem) -> NSMenu?

    // Optional: drag source validation
    @objc optional func sourceList(_ sourceList: SourceListView, canDragItem item: SourceListItem) -> Bool
}
```

### The AppKit Implementation

```swift
// MARK: - Data Model

final class SourceListItem: NSObject {
    let id: UUID
    var title: String
    var icon: NSImage?
    var badgeCount: Int

    init(id: UUID = UUID(), title: String, icon: NSImage? = nil, badgeCount: Int = 0) {
        self.id = id
        self.title = title
        self.icon = icon
        self.badgeCount = badgeCount
    }
}

// MARK: - View Controller

final class SourceListViewController: NSViewController {
    weak var delegate: SourceListDelegate?

    private var items: [SourceListItem] = []
    private var tableView: NSTableView!
    private var scrollView: NSScrollView!

    // MARK: - Items Management

    func setItems(_ items: [SourceListItem]) {
        self.items = items
        tableView?.reloadData()
    }

    func selectItem(at index: Int) {
        guard index >= 0 && index < items.count else { return }
        tableView.selectRowIndexes(IndexSet(integer: index), byExtendingSelection: false)
    }

    // MARK: - View Lifecycle

    override func loadView() {
        scrollView = NSScrollView()
        scrollView.hasVerticalScroller = true
        scrollView.autohidesScrollers = true
        scrollView.borderType = .noBorder

        tableView = NSTableView()
        tableView.allowsEmptySelection = false
        tableView.allowsMultipleSelection = false
        tableView.delegate = self
        tableView.dataSource = self
        tableView.style = .sourceList
        tableView.rowHeight = 28

        let column = NSTableColumn(identifier: NSUserInterfaceItemIdentifier("main"))
        column.title = ""
        tableView.addTableColumn(column)

        scrollView.documentView = tableView
        view = scrollView
    }
}

// MARK: - NSTableViewDataSource

extension SourceListViewController: NSTableViewDataSource {
    func numberOfRows(in tableView: NSTableView) -> Int {
        items.count
    }

    func tableView(_ tableView: NSTableView, objectValueFor tableColumn: NSTableColumn?, row: Int) -> Any? {
        items[row]
    }

    func tableView(_ tableView: NSTableView, validateDrop info: NSDraggingInfo, proposedRow row: Int, proposedDropOperation dropOperation: NSTableView.DropOperation) -> NSDragOperation {
        guard row >= 0 && row < items.count else { return [] }
        let item = items[row]
        if delegate?.sourceList?(self.sourceListView, canDragItem: item) == true {
            return .move
        }
        return []
    }

    private var sourceListView: SourceListView { view as! SourceListView }
}

// MARK: - NSTableViewDelegate

extension SourceListViewController: NSTableViewDelegate {
    func tableView(_ tableView: NSTableView, viewFor tableColumn: NSTableColumn?, row: Int) -> NSView? {
        let item = items[row]
        let cell = NSTableCellView()
        // Configure cell with item.title, item.icon, item.badgeCount
        return cell
    }

    func tableViewSelectionDidChange(_ notification: Notification) {
        let selectedRow = tableView.selectedRow
        guard selectedRow >= 0 && selectedRow < items.count else { return }
        delegate?.sourceList(view as! SourceListView, didSelectItem: items[selectedRow])
    }

    func tableView(_ tableView: NSTableView, rowActionsForRow: Int, edge: NSTableView.RowActionEdge) -> [NSTableViewRowAction] {
        []
    }
}

// MARK: - Custom View (for delegate parameter)

final class SourceListView: NSView {
    var embeddedController: SourceListViewController? {
        (subviews.first as? NSScrollView)?.documentView?.enclosingController as? SourceListViewController
    }
}

extension NSView {
    var enclosingController: NSViewController? {
        var responder: NSResponder? = self
        while let r = responder {
            if let vc = r as? NSViewController { return vc }
            responder = r.nextResponder
        }
        return nil
    }
}
```

### The SwiftUI Wrapper

```swift
import SwiftUI
import AppKit

struct SourceList: NSViewControllerRepresentable {
    let items: [SourceListItem]
    @Binding var selectedItem: SourceListItem?
    var onMove: ((SourceListItem, Int) -> Void)?
    var menuProvider: ((SourceListItem) -> NSMenu?)?

    func makeNSViewController(context: Context) -> SourceListViewController {
        let controller = SourceListViewController()
        controller.delegate = context.coordinator
        return controller
    }

    func updateNSViewController(_ controller: SourceListViewController, context: Context) {
        controller.setItems(items)
        context.coordinator.items = items
        context.coordinator.selectedItem = $selectedItem
        context.coordinator.onMove = onMove
        context.coordinator.menuProvider = menuProvider
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(selectedItem: $selectedItem, onMove: onMove, menuProvider: menuProvider)
    }

    // MARK: - Coordinator (NSObject subclass for @objc optional support)

    final class Coordinator: NSObject, SourceListDelegate {
        var items: [SourceListItem] = []
        var selectedItem: Binding<SourceListItem?>?
        var onMove: ((SourceListItem, Int) -> Void)?
        var menuProvider: ((SourceListItem) -> NSMenu?)?

        init(selectedItem: Binding<SourceListItem?>?, onMove: ((SourceListItem, Int) -> Void)?, menuProvider: ((SourceListItem) -> NSMenu?)?) {
            self.selectedItem = selectedItem
            self.onMove = onMove
            self.menuProvider = menuProvider
        }

        func sourceList(_ sourceList: SourceListView, didSelectItem item: SourceListItem) {
            selectedItem?.wrappedValue = item
        }

        func sourceList(_ sourceList: SourceListView, shouldMoveItem item: SourceListItem, to index: Int) -> Bool {
            true
        }

        func sourceList(_ sourceList: SourceListView, didMoveItem item: SourceListItem, to index: Int) {
            onMove?(item, index)
        }

        func sourceList(_ sourceList: SourceListView, menuForItem item: SourceListItem) -> NSMenu? {
            menuProvider?(item)
        }

        func sourceList(_ sourceList: SourceListView, canDragItem item: SourceListItem) -> Bool {
            true
        }
    }
}
```

### SwiftUI Usage

```swift
struct ContentView: View {
    @State private var items: [SourceListItem] = [
        SourceListItem(title: "Inbox", badgeCount: 3),
        SourceListItem(title: "Sent"),
        SourceListItem(title: "Archive"),
    ]
    @State private var selectedItem: SourceListItem?

    var body: some View {
        NavigationSplitView {
            SourceList(
                items: items,
                selectedItem: $selectedItem,
                onMove: { item, newIndex in
                    // Handle reorder
                    items.move(fromOffsets: IndexSet([items.firstIndex(where: { $0.id == item.id })!]),
                               toOffset: newIndex)
                },
                menuProvider: { item in
                    let menu = NSMenu()
                    menu.addItem(NSMenuItem(title: "Rename", action: #selector(_rename), keyEquivalent: ""))
                    menu.addItem(NSMenuItem(title: "Delete", action: #selector(_delete), keyEquivalent: ""))
                    return menu
                }
            )
        } detail: {
            if let selectedItem {
                DetailView(item: selectedItem)
            } else {
                Text("Select an item")
            }
        }
    }
}
```

**Key design decisions:**
- The Coordinator is an `NSObject` subclass because the delegate protocol has `@objc optional` methods.
- The delegate protocol uses the first-parameter convention (the delegating object as the first unlabeled argument).
- The SwiftUI wrapper uses closures for callbacks, bridging the delegate pattern into SwiftUI's declarative model.
- `SourceListItem` is an `NSObject` subclass because it crosses the ObjC boundary (used in `NSTableView` data source).

---

## 2. Error Type Design with CocoaError Bridging

This pattern shows a layered error design: a typed Swift enum for the public API, with internal bridging to `CocoaError` and `POSIXError` for system-level failures.

### Public Error Enum

```swift
import Foundation

/// Errors that can occur during document synchronization.
/// Conforms to `LocalizedError` for user-facing presentation and
/// `CustomNSError` for reliable ObjC bridging.
enum SyncError: LocalizedError, CustomNSError {
    case fileNotFound(URL)
    case permissionDenied(URL)
    case networkUnavailable
    case serverError(statusCode: Int, message: String)
    case conflict(localVersion: String, remoteVersion: String)
    case quotaExceeded(used: Int64, limit: Int64)
    case underlying(Error)

    // MARK: - LocalizedError

    var errorDescription: String? {
        switch self {
        case .fileNotFound(let url):
            return "The file '\(url.lastPathComponent)' could not be found."
        case .permissionDenied(let url):
            return "Permission denied when accessing '\(url.lastPathComponent)'."
        case .networkUnavailable:
            return "The network is not available. Check your connection."
        case .serverError(let code, _):
            return "Server error (HTTP \(code)). Please try again."
        case .conflict:
            return "The document has been modified in two places."
        case .quotaExceeded:
            return "Storage quota exceeded. Free up space and try again."
        case .underlying(let error):
            return error.localizedDescription
        }
    }

    var failureReason: String? {
        switch self {
        case .fileNotFound:
            return "The file may have been moved or deleted."
        case .permissionDenied:
            return "The app may not have access to this location."
        case .networkUnavailable:
            return "A network connection is required to sync documents."
        case .serverError(_, let message):
            return message.isEmpty ? "The server returned an error." : message
        case .conflict(let local, let remote):
            return "Local version '\(local)' conflicts with remote version '\(remote)'."
        case .quotaExceeded(let used, let limit):
            return "Using \(byteCount(used)) of \(byteCount(limit))."
        case .underlying:
            return nil
        }
    }

    var recoverySuggestion: String? {
        switch self {
        case .fileNotFound:
            return "Choose a different file or create a new one."
        case .permissionDenied:
            return "Grant access in System Settings > Privacy & Security > Files and Folders."
        case .networkUnavailable:
            return "Connect to Wi-Fi or check your network settings."
        case .quotaExceeded:
            return "Delete unused documents or upgrade your storage plan."
        default:
            return nil
        }
    }

    // MARK: - CustomNSError

    static let errorDomain = "com.myapp.sync"

    var errorCode: Int {
        switch self {
        case .fileNotFound:       1001
        case .permissionDenied:   1002
        case .networkUnavailable: 1003
        case .serverError:        1004
        case .conflict:           1005
        case .quotaExceeded:      1006
        case .underlying:         1000
        }
    }

    var errorUserInfo: [String: Any] {
        var info: [String: Any] = [
            NSLocalizedDescriptionKey: errorDescription ?? "",
        ]
        if let reason = failureReason {
            info[NSLocalizedFailureReasonErrorKey] = reason
        }
        if let suggestion = recoverySuggestion {
            info[NSLocalizedRecoverySuggestionErrorKey] = suggestion
        }
        if case .underlying(let error) = self {
            info[NSUnderlyingErrorKey] = error
        }
        if case .fileNotFound(let url) = self {
            info[NSFilePathErrorKey] = url.path
        }
        if case .permissionDenied(let url) = self {
            info[NSFilePathErrorKey] = url.path
        }
        return info
    }

    // MARK: - NSError bridging

    static func from(_ error: NSError) -> SyncError {
        guard error.domain == Self.errorDomain else {
            return .underlying(error)
        }
        switch error.code {
        case 1001:
            let url = (error.userInfo[NSFilePathErrorKey] as? String)
                .map(URL.init(fileURLWithPath:)) ?? URL(fileURLWithPath: "/")
            return .fileNotFound(url)
        case 1002:
            let url = (error.userInfo[NSFilePathErrorKey] as? String)
                .map(URL.init(fileURLWithPath:)) ?? URL(fileURLWithPath: "/")
            return .permissionDenied(url)
        case 1003: return .networkUnavailable
        case 1004:
            let message = error.userInfo[NSLocalizedFailureReasonErrorKey] as? String ?? ""
            return .serverError(statusCode: 500, message: message)
        case 1005:
            let local = error.userInfo["localVersion"] as? String ?? ""
            let remote = error.userInfo["remoteVersion"] as? String ?? ""
            return .conflict(localVersion: local, remoteVersion: remote)
        case 1006:
            let used = error.userInfo["usedBytes"] as? Int64 ?? 0
            let limit = error.userInfo["limitBytes"] as? Int64 ?? 0
            return .quotaExceeded(used: used, limit: limit)
        default:
            return .underlying(error)
        }
    }

    private func byteCount(_ bytes: Int64) -> String {
        let formatter = ByteCountFormatter()
        formatter.countStyle = .file
        return formatter.string(fromByteCount: bytes)
    }
}
```

### Internal System Error Bridging

```swift
/// Internal function that bridges POSIX and Cocoa errors to the public SyncError type.
/// This keeps system-level error handling isolated from the public API.
enum SystemErrorBridging {

    /// Wraps a file read operation, converting POSIX and Cocoa errors to SyncError.
    static func readFile(at url: URL) throws -> Data {
        do {
            return try Data(contentsOf: url)
        } catch let error as CocoaError {
            switch error.code {
            case .fileReadNoSuchFile:
                throw SyncError.fileNotFound(url)
            case .fileReadNoPermission:
                throw SyncError.permissionDenied(url)
            default:
                throw SyncError.underlying(error)
            }
        } catch let error as POSIXError {
            switch error.code {
            case .ENOENT:
                throw SyncError.fileNotFound(url)
            case .EACCES, .EPERM:
                throw SyncError.permissionDenied(url)
            default:
                throw SyncError.underlying(error)
            }
        } catch {
            throw SyncError.underlying(error)
        }
    }

    /// Wraps a URLSession data task, converting network errors to SyncError.
    static func performSync(request: URLRequest) async throws -> Data {
        do {
            let (data, response) = try await URLSession.shared.data(for: request)
            guard let httpResponse = response as? HTTPURLResponse else {
                throw SyncError.underlying(URLError(.badServerResponse))
            }
            guard (200...299).contains(httpResponse.statusCode) else {
                let message = String(data: data, encoding: .utf8) ?? ""
                throw SyncError.serverError(statusCode: httpResponse.statusCode, message: message)
            }
            return data
        } catch let error as SyncError {
            throw error  // already wrapped
        } catch let error as URLError {
            switch error.code {
            case .notConnectedToInternet, .networkConnectionLost:
                throw SyncError.networkUnavailable
            default:
                throw SyncError.underlying(error)
            }
        } catch {
            throw SyncError.underlying(error)
        }
    }
}
```

### Usage in NSDocument

```swift
final class SyncDocument: NSDocument {
    var content: String = ""

    override func data(ofType typeName: String) throws -> Data {
        guard let data = content.data(using: .utf8) else {
            // Use CocoaError for document I/O — it's the expected domain for NSDocument
            throw CocoaError(.fileWriteInapplicableStringEncoding,
                             userInfo: [NSDebugDescriptionErrorKey: "Content cannot be encoded as UTF-8"])
        }
        return data
    }

    override func read(from data: Data, ofType typeName: String) throws {
        guard let string = String(data: data, encoding: .utf8) else {
            throw CocoaError(.fileReadUnknown,
                             userInfo: [NSFilePathErrorKey: fileURL?.path ?? "",
                                        NSDebugDescriptionErrorKey: "File is not valid UTF-8"])
        }
        content = string
    }

    func syncToCloud() async throws {
        guard let fileURL else {
            throw CocoaError(.fileWriteNoPermission,
                             userInfo: [NSFilePathErrorKey: "Untitled"])
        }
        let data = try SystemErrorBridging.readFile(at: fileURL)
        let request = URLRequest(url: URL(string: "https://api.example.com/sync")!)
        _ = try await SystemErrorBridging.performSync(request: request)
    }
}
```

**Key design decisions:**
- `SyncError` is the public-facing typed enum. It never exposes `POSIXError` or `CocoaError` directly.
- `SystemErrorBridging` is an internal enum (no cases) that acts as a namespace for conversion functions. It catches system errors and re-throws as `SyncError`.
- `NSDocument` methods throw `CocoaError` because that's what the NSDocument architecture expects — the error domain and codes are part of the AppKit contract.
- `CustomNSError` is implemented on `SyncError` so that when it crosses the ObjC boundary (e.g., through an `@objc` completion handler), the domain and code are preserved.

---

## 3. Swift Package Structure for macOS-Only Library

A complete `Package.swift` and file layout for a macOS-only library that provides a document scanning framework.

### Package.swift

```swift
// swift-tools-version: 6.3
import PackageDescription

let package = Package(
    name: "DocumentScanKit",
    platforms: [
        .macOS(.v15)
    ],
    products: [
        .library(
            name: "DocumentScanKit",
            targets: ["DocumentScanKit"]
        ),
    ],
    dependencies: [
        .package(url: "https://github.com/apple/swift-log.git", from: "1.6.0"),
    ],
    targets: [
        // Core: pure Swift, no platform dependency, fully testable
        .target(
            name: "DocumentScanKitCore",
            dependencies: [
                .product(name: "Logging", package: "swift-log"),
            ],
            path: "Sources/DocumentScanKitCore",
            swiftSettings: [
                .enableExperimentalFeature("StrictConcurrency")
            ]
        ),

        // Vision: depends on Vision framework (macOS 14+)
        .target(
            name: "DocumentScanKitVision",
            dependencies: ["DocumentScanKitCore"],
            path: "Sources/DocumentScanKitVision"
        ),

        // AppKit: NSViewRepresentable wrappers, NSDocument integration
        .target(
            name: "DocumentScanKitAppKit",
            dependencies: ["DocumentScanKitCore", "DocumentScanKitVision"],
            path: "Sources/DocumentScanKitAppKit"
        ),

        // Umbrella: re-exports all public APIs
        .target(
            name: "DocumentScanKit",
            dependencies: [
                "DocumentScanKitCore",
                "DocumentScanKitVision",
                "DocumentScanKitAppKit",
            ],
            path: "Sources/DocumentScanKit"
        ),

        .testTarget(
            name: "DocumentScanKitTests",
            dependencies: ["DocumentScanKitCore", "DocumentScanKitVision"],
            path: "Tests/DocumentScanKitTests"
        ),
    ]
)
```

### File Layout

```
DocumentScanKit/
├── Package.swift
├── Sources/
│   ├── DocumentScanKit/
│   │   └── DocumentScanKit.swift          # Umbrella re-exports
│   ├── DocumentScanKitCore/
│   │   ├── Models/
│   │   │   ├── ScannedDocument.swift      # Pure Swift model
│   │   │   ├── ScanRegion.swift           # Value type
│   │   │   └── ScanOptions.swift          # Configuration struct
│   │   ├── Processing/
│   │   │   ├── DocumentProcessor.swift    # Core processing logic
│   │   │   └── ImageFilter.swift          # Pure Swift image ops
│   │   └── Errors/
│   │       └── ScanError.swift            # LocalizedError enum
│   ├── DocumentScanKitVision/
│   │   ├── VisionTextRecognizer.swift     # VNRecognizeTextRequest wrapper
│   │   └── VisionBarcodeDetector.swift    # VNDetectBarcodesRequest wrapper
│   └── DocumentScanKitAppKit/
│       ├── ScanPreviewView.swift          # NSViewRepresentable
│       ├── ScanDocument.swift             # NSDocument subclass
│       └── ScanWindowController.swift     # NSWindowController
└── Tests/
    └── DocumentScanKitTests/
        ├── DocumentProcessorTests.swift
        ├── ScanErrorTests.swift
        └── ScanOptionsTests.swift
```

### Umbrella Re-Export

```swift
// Sources/DocumentScanKit/DocumentScanKit.swift

@_exported import DocumentScanKitCore
@_exported import DocumentScanKitVision
@_exported import DocumentScanKitAppKit

/// The current version of DocumentScanKit.
public let documentScanKitVersion = "1.0.0"
```

### Core Model (Pure Swift)

```swift
// Sources/DocumentScanKitCore/Models/ScannedDocument.swift

import Foundation

public struct ScannedDocument: Identifiable, Sendable {
    public let id: UUID
    public var pages: [ScannedPage]
    public var createdAt: Date
    public var metadata: DocumentMetadata

    public init(id: UUID = UUID(), pages: [ScannedPage] = [], metadata: DocumentMetadata = DocumentMetadata()) {
        self.id = id
        self.pages = pages
        self.createdAt = Date()
        self.metadata = metadata
    }
}

public struct ScannedPage: Identifiable, Sendable {
    public let id: UUID
    public var recognizedText: String
    public var confidence: Float
    public var regions: [ScanRegion]

    public init(id: UUID = UUID(), recognizedText: String = "", confidence: Float = 0, regions: [ScanRegion] = []) {
        self.id = id
        self.recognizedText = recognizedText
        self.confidence = confidence
        self.regions = regions
    }
}

public struct ScanRegion: Sendable {
    public let text: String
    public let boundingBox: NormalizedRect
    public let confidence: Float

    public init(text: String, boundingBox: NormalizedRect, confidence: Float) {
        self.text = text
        self.boundingBox = boundingBox
        self.confidence = confidence
    }
}

public struct NormalizedRect: Sendable {
    public let x: Double
    public let y: Double
    public let width: Double
    public let height: Double

    public init(x: Double, y: Double, width: Double, height: Double) {
        self.x = x
        self.y = y
        self.width = width
        self.height = height
    }
}

public struct DocumentMetadata: Sendable {
    public var title: String
    public var author: String?
    public var tags: [String]

    public init(title: String = "", author: String? = nil, tags: [String] = []) {
        self.title = title
        self.author = author
        self.tags = tags
    }
}
```

**Key design decisions:**
- Core models are pure Swift value types (`struct`, `Sendable`) with no platform dependency.
- Vision framework code is in a separate target so the core can be tested without Vision.
- AppKit code is in a separate target so the library could theoretically add a SwiftUI-only target later.
- The umbrella target uses `@_exported import` so consumers write `import DocumentScanKit` and get everything.
- `StrictConcurrency` is enabled in the core target to catch `Sendable` violations early.

---

## 4. @objc Protocol with Optional Methods

A complete `@objc` protocol with optional methods, demonstrating the NSObject requirement, proper selector naming, and Swift usage.

### The Protocol

```swift
import Foundation

/// Delegate for a background task manager.
/// **Important:** Conforming types must be `NSObject` subclasses because
/// this protocol contains `@objc optional` methods.
@objc protocol BackgroundTaskDelegate: AnyObject {
    // MARK: - Required Methods

    /// Called when a task begins execution.
    func taskManager(_ manager: BackgroundTaskManager, didStartTask task: BackgroundTask)

    /// Called when a task completes successfully.
    func taskManager(_ manager: BackgroundTaskManager, didCompleteTask task: BackgroundTask, with result: TaskResult)

    /// Called when a task fails.
    func taskManager(_ manager: BackgroundTaskManager, didFailTask task: BackgroundTask, with error: Error)

    // MARK: - Optional Methods

    /// Called periodically with progress updates. Return `false` to cancel the task.
    @objc optional func taskManager(_ manager: BackgroundTaskManager, didUpdateProgress progress: Double, for task: BackgroundTask) -> Bool

    /// Called when the task manager is about to retry a failed task.
    @objc optional func taskManager(_ manager: BackgroundTaskManager, willRetryTask task: BackgroundTask, attempt: Int)

    /// Called when all tasks in a batch have completed.
    @objc optional func taskManagerDidFinishAllTasks(_ manager: BackgroundTaskManager)

    /// Called to determine if a task should be retried after failure. Default is `true`.
    @objc optional func taskManager(_ manager: BackgroundTaskManager, shouldRetry task: BackgroundTask, after error: Error) -> Bool
}
```

### The Task Manager

```swift
import Foundation

final class BackgroundTask: NSObject {
    let id: UUID
    let name: String
    var progress: Double = 0.0
    var state: TaskState = .pending

    init(name: String) {
        self.id = UUID()
        self.name = name
    }
}

enum TaskState {
    case pending
    case running
    case completed(TaskResult)
    case failed(Error)
    case cancelled
}

struct TaskResult {
    let data: Data
    let duration: TimeInterval
}

final class BackgroundTaskManager: NSObject {
    weak var delegate: BackgroundTaskDelegate?
    private var tasks: [BackgroundTask] = []

    func addTask(_ task: BackgroundTask) {
        tasks.append(task)
    }

    func startAll() {
        for task in tasks where task.state == .pending {
            startTask(task)
        }
    }

    private func startTask(_ task: BackgroundTask) {
        task.state = .running
        delegate?.taskManager(self, didStartTask: task)

        // Simulate work with progress updates
        Task { @MainActor in
            for i in 1...10 {
                try? await Task.sleep(for: .milliseconds(100))
                task.progress = Double(i) / 10.0

                // Call optional progress delegate — use responds(to:) for safety
                let shouldContinue = delegate?.taskManager?(self, didUpdateProgress: task.progress, for: task) ?? true
                if !shouldContinue {
                    task.state = .cancelled
                    return
                }
            }

            let result = TaskResult(data: Data(), duration: 1.0)
            task.state = .completed(result)
            delegate?.taskManager(self, didCompleteTask: task, with: result)

            if tasks.allSatisfy({ isCompleted($0) }) {
                delegate?.taskManagerDidFinishAllTasks?(self)
            }
        }
    }

    private func isCompleted(_ task: BackgroundTask) -> Bool {
        switch task.state {
        case .completed, .failed, .cancelled:
            return true
        default:
            return false
        }
    }
}
```

### NSObject Delegate Implementation

```swift
/// Example delegate that implements all methods.
final class TaskDelegate: NSObject, BackgroundTaskDelegate {
    func taskManager(_ manager: BackgroundTaskManager, didStartTask task: BackgroundTask) {
        print("Started: \(task.name)")
    }

    func taskManager(_ manager: BackgroundTaskManager, didCompleteTask task: BackgroundTask, with result: TaskResult) {
        print("Completed: \(task.name) in \(result.duration)s")
    }

    func taskManager(_ manager: BackgroundTaskManager, didFailTask task: BackgroundTask, with error: Error) {
        print("Failed: \(task.name) — \(error.localizedDescription)")
    }

    func taskManager(_ manager: BackgroundTaskManager, didUpdateProgress progress: Double, for task: BackgroundTask) -> Bool {
        print("Progress: \(Int(progress * 100))% — \(task.name)")
        return true  // don't cancel
    }

    func taskManager(_ manager: BackgroundTaskManager, willRetryTask task: BackgroundTask, attempt: Int) {
        print("Retrying: \(task.name) (attempt \(attempt))")
    }

    func taskManagerDidFinishAllTasks(_ manager: BackgroundTaskManager) {
        print("All tasks finished")
    }

    func taskManager(_ manager: BackgroundTaskManager, shouldRetry task: BackgroundTask, after error: Error) -> Bool {
        // Don't retry permission errors
        if let posixError = error as? POSIXError,
           posixError.code == .EACCES || posixError.code == .EPERM {
            return false
        }
        return true
    }
}
```

### Minimal Delegate (Required Methods Only)

```swift
/// Example delegate that only implements required methods.
/// This is valid because the protocol methods are `@objc optional`.
final class MinimalTaskDelegate: NSObject, BackgroundTaskDelegate {
    func taskManager(_ manager: BackgroundTaskManager, didStartTask task: BackgroundTask) {
        print("Started: \(task.name)")
    }

    func taskManager(_ manager: BackgroundTaskManager, didCompleteTask task: BackgroundTask, with result: TaskResult) {
        print("Completed: \(task.name)")
    }

    func taskManager(_ manager: BackgroundTaskManager, didFailTask task: BackgroundTask, with error: Error) {
        print("Failed: \(task.name)")
    }
    // Optional methods are not implemented — this is fine
}
```

### Calling Optional Methods Safely

```swift
// When calling optional delegate methods from Swift, use optional chaining.
// The `?` before the method name handles the case where the delegate
// doesn't implement the method.

// Pattern 1: Optional return value with default
let shouldContinue = delegate?.taskManager?(self, didUpdateProgress: 0.5, for: task) ?? true

// Pattern 2: Void return — just call with optional chaining
delegate?.taskManagerDidFinishAllTasks?(self)

// Pattern 3: Check with responds(to:) for ObjC interop or when you need
// to distinguish "not implemented" from "implemented but returned nil"
if delegate?.responds(to: #selector(BackgroundTaskDelegate.taskManager(_:shouldRetry:after:))) == true {
    let shouldRetry = delegate?.taskManager?(self, shouldRetry: task, after: someError) ?? true
    // handle retry decision
}
```

**Key design decisions:**
- The protocol is marked `@objc` at the protocol level, not on individual methods.
- Optional methods use `@objc optional` — the double annotation is required.
- The conforming delegate (`TaskDelegate`) is an `NSObject` subclass. This is mandatory for `@objc optional` to work.
- When calling optional methods from Swift, use `delegate?.methodName?(...)` — the `?` after the method name is the optional chaining for the optional method.
- The `responds(to:)` check is available when you need to distinguish "method not implemented" from "method returned nil/false."
- All method parameters are ObjC-representable types (`Double`, `Int`, `Error` which bridges to `NSError`).
