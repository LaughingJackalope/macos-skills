# Swift Concurrency Patterns for macOS

Deep-dive patterns for bridging Swift concurrency with AppKit, XPC services, and background daemons on macOS 26+ / Swift 6.3.

## MainActor Bridge: Calling AppKit from Async Context

The most common pattern in a macOS app: you're on a background actor or detached task and need to update the UI. The correct way depends on where you're calling from.

### Pattern: Background Actor → MainActor

When an actor method produces a result that must be displayed, use `MainActor.run`:

```swift
actor ImageProcessor {
    /// Runs on a background actor (not main).
    func process(image: NSImage) async -> ProcessedImage {
        // Heavy work — runs off-main.
        let cgImage = image.cgImage(forProposedRect: nil, context: nil, hints: nil)!
        let filtered = await applyFilters(to: cgImage)

        // Return a value type — does not require main actor.
        return ProcessedImage(cgImage: filtered)
    }

    /// Convenience: process and update a view.
    func processAndDisplay(image: NSImage, in view: ProcessedImageView) async {
        let result = await process(image: image)

        // Hop to main actor for the UI update.
        await MainActor.run {
            view.displayedImage = NSImage(cgImage: result.cgImage, size: view.bounds.size)
            view.needsDisplay = true  // AppKit — must be on main thread.
        }
    }
}
```

### Pattern: Known-Main-Thread Callbacks (assumeIsolated)

AppKit delegate methods always fire on the main thread. When such a callback needs to access a `@MainActor`-isolated reference type, use `assumeIsolated` to avoid an unnecessary `await`:

```swift
@MainActor
final class SearchCoordinator: NSObject, NSSearchFieldDelegate {
    var results: [SearchResult] = []
    let controller: SearchResultsController

    init(controller: SearchResultsController) {
        self.controller = controller
        super.init()
    }

    func controlTextDidChange(_ notification: Notification) {
        // Always called on the main thread by AppKit.
        MainActor.assumeIsolated {
            let field = notification.object as! NSSearchField
            self.results = self.controller.filter(query: field.stringValue)
        }
    }
}
```

**Rule:** Only call `assumeIsolated` when you are certain of the calling thread. A `DispatchQueue.main.async` callback, `NSTimer` (scheduled on main run loop), or AppKit delegate is fine. A generic `DispatchQueue.global()` callback is not.

### Pattern: DispatchQueue.main.async vs MainActor.run

Prefer `MainActor.run` in Swift-written code. Use `DispatchQueue.main.async` only for:

- C/Objective-C callbacks that don't carry Swift concurrency executor information
- Code that interacts with APIs requiring a raw dispatch block (e.g., some `NSTimer` factories)

```swift
// Prefer this:
await MainActor.run {
    self.label.stringValue = "Done"
}

// Only this for C/ObjC callback bridging:
DispatchQueue.main.async {
    Task { @MainActor in
        self.label.stringValue = "Done"
    }
}
```

## AsyncStream for NSNotificationCenter Events

Bridge `NSNotificationCenter` into `AsyncStream` so notification-based events can be consumed with `for-await-in`:

```swift
extension NotificationCenter {
    /// Returns an AsyncStream of notifications for the given name.
    /// Captures the NotificationCenter weakly — safe to outlive the center.
    func notifications(named name: Notification.Name,
                       object sender: AnyObject? = nil)
        -> AsyncStream<Notification>
    {
        AsyncStream { continuation in
            let observer = self.addObserver(
                forName: name,
                object: sender,
                queue: nil  // Delivered on the posting thread.
            ) { notification in
                continuation.yield(notification)
            }

            continuation.onTermination = { @Sendable _ in
                self.removeObserver(observer)
            }
        }
    }
}

// Consumption — observe NSDocument state changes
func observeDocumentLifecycle() async {
    await withTaskGroup(of: Void.self) { group in
        // Document did open.
        group.addTask { @MainActor in
            for await notification in NotificationCenter.default
                .notifications(named: NSDocument.didOpenNotification)
            {
                guard let doc = notification.object as? Document else { continue }
                await indexDocument(doc)
            }
        }

        // Document will close.
        group.addTask { @MainActor in
            for await _ in NotificationCenter.default
                .notifications(named: NSDocument.willCloseNotification)
            {
                await flushPendingSaves()
            }
        }
    }
}
```

### Typed Notification Payloads

For type safety, map the stream:

```swift
struct ThemeChanged {
    let newTheme: NSAppearance
}

extension NotificationCenter {
    var themeChanges: AsyncStream<ThemeChanged> {
        notifications(named: NSAppearance.nameDidChangeNotification)
            .map { ThemeChanged(newTheme: $0.object as? NSAppearance ?? .current) }
            .eraseToStream()  // AsyncStream<ThemeChanged>
    }
}

// Helper: AsyncSequence -> erased AsyncStream
extension AsyncSequence {
    func eraseToStream() -> AsyncStream<Element> {
        AsyncStream { continuation in
            Task {
                for await element in self {
                    continuation.yield(element)
                }
                continuation.finish()
            }
        }
    }
}
```

## XPC Service with Async Swift: NSXPCConnection + withCheckedContinuation

Bridging XPC's callback-based API into Swift concurrency continuations. This pattern keeps the XPC client code idiomatic while preserving the security boundary of XPC.

### Client Side: Async Wrapper

```swift
/// Wraps an XPC service call with a continuation.
/// Returns the decoded result or throws the XPC error.
func callXPC<Input: Codable, Output: Codable>(
    service: String,
    method: String,
    input: Input
) async throws -> Output {
    try await withCheckedThrowingContinuation { continuation in
        let connection = NSXPCConnection(serviceName: service)
        // Define the interface with Objective-C interop for XPC.
        connection.remoteObjectInterface = NSXPCInterface(with: LegacyXPCProtocol.self)
        connection.invalidationHandler = {
            continuation.resume(throwing: XPCError.connectionInvalidated)
        }
        connection.resume()

        let proxy = connection.remoteObjectProxyWithErrorHandler { error in
            continuation.resume(throwing: error)
        } as! LegacyXPCProtocol

        // Encode input to Data for the XPC call.
        guard let inputData = try? JSONEncoder().encode(input) else {
            continuation.resume(throwing: XPCError.encodingFailure)
            return
        }

        proxy.execute(method: method, payload: inputData) { responseData, error in
            if let error {
                continuation.resume(throwing: error)
                return
            }
            guard let responseData,
                  let output = try? JSONDecoder().decode(Output.self, from: responseData)
            else {
                continuation.resume(throwing: XPCError.decodingFailure)
                return
            }
            continuation.resume(returning: output)
        }
    }
}
```

### Service Side: Concurrency-Native but XPC-Compliant

The XPC service defines `@objc` protocol methods for the XPC boundary but uses Swift concurrency internally:

```swift
// ObjC-XPC boundary protocol:
@objc protocol LegacyXPCProtocol {
    func execute(method: String, payload: Data, completion: @escaping (Data?, Error?) -> Void)
}

// Service implementation — Swift concurrency inside:
final class MyXPCService: NSObject, LegacyXPCProtocol {
    private let processor = DataProcessor()  // actor or @MainActor-isolated class

    func execute(method: String, payload: Data,
                 completion: @escaping (Data?, Error?) -> Void) {
        Task {
            do {
                let result = try await handle(method: method, payload: payload)
                let data = try JSONEncoder().encode(result)
                completion(data, nil)
            } catch {
                completion(nil, error)
            }
        }
    }

    private func handle(method: String, payload: Data) async throws -> ServiceResult {
        // Full Swift concurrency — async let, task groups, actors.
        switch method {
        case "analyze":
            let input = try JSONDecoder().decode(AnalysisInput.self, from: payload)
            return try await processor.analyze(input)
        default:
            throw XPCError.unknownMethod
        }
    }
}
```

### Lifetime and Concurrency Notes

- Always call `connection.resume()` before sending messages.
- The invalidation handler fires on the XPC connection's internal queue — do not assume main thread. Parse it, then `resume()` the continuation.
- Never store `NSXPCConnection` in an actor — it's a reference type that's not Sendable. Store the service name and create new connections per call, or wrap in an `@unchecked Sendable` with external locking.

## Task Group for Parallel I/O on macOS

Fetch or process multiple files concurrently with structured concurrency:

```swift
struct FileResult: Sendable {
    let url: URL
    let parsed: ParsedContent
}

func processFiles(at urls: [URL]) async throws -> [FileResult] {
    // Limit concurrency to avoid file descriptor exhaustion.
    // macOS default FD limit is 256 (soft); use a reasonable cap.
    let maxConcurrent = min(urls.count, 16)

    return try await withThrowingTaskGroup(of: FileResult.self) { group in
        var results: [FileResult] = []
        results.reserveCapacity(urls.count)

        // Seed the first batch.
        var index = 0
        for url in urls.prefix(maxConcurrent) {
            group.addTask { try await parseFile(at: url) }
            index += 1
        }

        // As each task completes, add the next URL.
        for try await result in group {
            results.append(result)
            if index < urls.count {
                let nextURL = urls[index]
                group.addTask { try await parseFile(at: nextURL) }
                index += 1
            }
        }

        return results
    }
}

private func parseFile(at url: URL) async throws -> FileResult {
    // File system reads are async-safe in Swift concurrency.
    let data = try Data(contentsOf: url, options: .mappedIfSafe)
    let parsed = try await parseContent(data)
    return FileResult(url: url, parsed: parsed)
}
```

### Ordered Output

If output order must match input order, collect into an indexed dictionary:

```swift
func processFilesOrdered(at urls: [URL]) async throws -> [FileResult] {
    try await withThrowingTaskGroup(of: (Int, FileResult).self) { group in
        for (i, url) in urls.enumerated() {
            group.addTask {
                let result = try await parseFile(at: url)
                return (i, result)
            }
        }

        var indexed: [Int: FileResult] = [:]
        for try await (i, result) in group {
            indexed[i] = result
        }

        return urls.indices.map { indexed[$0]! }
    }
}
```

### Cancellation of I/O Task Groups

File I/O tasks should check cancellation cooperatively. `Data(contentsOf:)` is not automatically cancellation-aware. Wrap in a check:

```swift
private func parseFile(at url: URL) async throws -> FileResult {
    // .mappedIfSafe may block; check cancellation first.
    try Task.checkCancellation()
    let data = try Data(contentsOf: url, options: .mappedIfSafe)
    try Task.checkCancellation()  // Check between I/O and parse.
    let parsed = try await parseContent(data)
    return FileResult(url: url, parsed: parsed)
}
```

## Strict Concurrency: Annotating AppKit Delegate Classes

Annotating `NSDocument`, `NSWindowController`, and delegate classes for strict concurrency checking.

### NSDocument Subclass

```swift
@MainActor
final class Document: NSDocument {
    @Published var content: DocumentContent = .empty

    override class var autosavesInPlace: Bool { true }

    override func makeWindowControllers() {
        let storyboard = NSStoryboard(name: "Main", bundle: nil)
        let ctrl = storyboard.instantiateController(
            withIdentifier: "DocumentWindowController"
        ) as! NSWindowController
        ctrl.document = self
        addWindowController(ctrl)
    }

    override func data(ofType typeName: String) throws -> Data {
        try JSONEncoder().encode(content)
    }

    override func read(from data: Data, ofType typeName: String) throws {
        content = try JSONDecoder().decode(DocumentContent.self, from: data)
    }
}
```

### NSWindowController Subclass

```swift
@MainActor
final class DocumentWindowController: NSWindowController {
    override func windowDidLoad() {
        super.windowDidLoad()
        // Already on @MainActor — safe to access all AppKit ivars.
        window?.titlebarAppearsTransparent = true
        window?.styleMask.insert(.fullSizeContentView)
    }
}
```

### Delegate Objects (often non-actor-isolated)

Many delegate objects are created on the main thread but aren't `@MainActor`-annotated by default. For strict concurrency:

```swift
/// A menu delegate that must be on the main thread.
@MainActor
final class AppMenuDelegate: NSObject, NSMenuDelegate {
    var itemCount = 0

    func menuWillOpen(_ menu: NSMenu) {
        // Safe: @MainActor isolation.
        itemCount = menu.numberOfItems
    }

    func menuDidClose(_ menu: NSMenu) {
        itemCount = 0
    }
}
```

### Incremental Migration with @preconcurrency

When adopting strict concurrency in a codebase where some modules use legacy dispatch:

```swift
// In the legacy module:
@preconcurrency import AppKit

class LegacyBridge: NSObject, NSApplicationDelegate {
    // Suppress warnings for the entire delegate class.
    func applicationDidFinishLaunching(_ notification: Notification) {
        DispatchQueue.global().async { [weak self] in
            self?.performBackgroundSetup()
        }
    }

    /// Manually annotate for cross-boundary calls.
    @MainActor
    private func updateUIAfterSetup() {
        // ...
    }

    private func performBackgroundSetup() {
        // Background work...
        Task { @MainActor in
            await updateUIAfterSetup()
        }
    }
}
```

Enable `complete` strict concurrency checking on the module containing the `@MainActor` classes; keep `targeted` on modules that import the legacy code until they are migrated.

### Summary of Annotation Rules for AppKit

| Type | Annotation |
|------|-----------|
| `NSDocument` subclass | `@MainActor` |
| `NSWindowController` subclass | `@MainActor` |
| `NSViewController` subclass | `@MainActor` |
| `NSMenuDelegate` (always main-thread) | `@MainActor` |
| `NSTableViewDataSource` (always main-thread) | `@MainActor` |
| `NSApplicationDelegate` | `@MainActor` |
| Dispatched-to-main delegate callbacks | Add `MainActor.assumeIsolated` inside |
| Background work in delegate callbacks | `Task { @MainActor in ... }` for UI updates |
