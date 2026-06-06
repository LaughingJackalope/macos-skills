---
name: swift-concurrency-macos
description: "Swift concurrency for macOS — actors, @MainActor, Task/TaskGroup, AsyncStream, Sendable, strict concurrency migration, and AppKit bridging patterns (DispatchQueue, NSRunLoop, XPC service concurrency). Use when writing async Swift code that interacts with AppKit, daemons, or XPC services on macOS."
sources: https://developer.apple.com/documentation/swift/concurrency, https://developer.apple.com/videos/play/wwdc2024/10169/, https://github.com/apple/swift-evolution/blob/main/proposals/0466-approachable-concurrency.md, https://swiftcrafted.dev/article/swift-6-2-approachable-concurrency-complete-guide, https://useyourloaf.com/blog/approachable-concurrency-in-swift-packages/, https://www.donnywals.com/exploring-concurrency-changes-in-swift-6-2/, https://developer.apple.com/videos/play/wwdc2025/268/, https://needone.app/bridging-legacy-apis-with-mainactor-assumeisolated/, https://developer.apple.com/forums/thread/788052, https://www.telefonica.com/en/communication-room/reports/survival-guide-migrate-swift-6-breaking-app/, https://forums.swift.org/t/main-syntax-crashes-services-on-language-version-6-but-not-5/79313, https://github.com/andreasley/XPCDistributedActorSystem, https://developer.apple.com/videos/play/wwdc2021/10132/
researched: 2026-06-06
platform: macOS 26+
---

# Swift Concurrency for macOS

Swift concurrency on macOS 26+ with Swift 6.3. Covers actors, `@MainActor`, structured and unstructured tasks, `AsyncStream`, `Sendable` data-race safety, strict concurrency migration from legacy AppKit codebases, and bridging patterns for `DispatchQueue`, `NSRunLoop`, XPC services, and background daemons. For AppKit-SwiftUI interop, see `swiftui-appkit-interop`. For XPC basics, see `macos-system-integration`.

See [references/concurrency-patterns.md](references/concurrency-patterns.md) for deep-dive patterns: MainActor bridging, `AsyncStream` for `NSNotificationCenter`, XPC with `withCheckedContinuation`, parallel I/O with task groups, and annotating AppKit delegate classes for strict concurrency.

## Contents

- [Actor & @MainActor Basics](#actor--mainactor-basics)
- [Task & TaskGroup](#task--taskgroup)
- [AsyncStream & AsyncSequence](#asyncstream--asyncsequence)
- [Sendable & Data Races](#sendable--data-races)
- [Strict Concurrency Migration](#strict-concurrency-migration)
- [AppKit Bridging](#appkit-bridging)
- [XPC & Daemon Concurrency](#xpc--daemon-concurrency)
- [Approachable Concurrency (SE-0461/0466/0470)](#approachable-concurrency-se-046104660470)
- [Swift 6.3 Concurrency Updates](#swift-63-concurrency-updates)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Actor & @MainActor Basics

Actors isolate their mutable state to prevent data races. Each actor processes messages (calls to its methods) sequentially — only one `async` call executes at a time. Swift 6.3 enforces this at compile time.

### Actor Declaration

```swift
actor ImageCache {
    private var cache: [String: NSImage] = [:]

    func image(for key: String) -> NSImage? {
        cache[key]
    }

    func setImage(_ image: NSImage, for key: String) {
        cache[key] = image
    }

    func removeAll() {
        cache.removeAll()
    }
}
```

All methods and mutable state in an actor are implicitly `isolated` to that actor. Callers must `await` access from outside:

```swift
let cache = ImageCache()
let img = await cache.image(for: "hero")
await cache.setImage(newImage, for: "hero")
```

### @MainActor

`@MainActor` is a global actor that isolates to the main thread. Use it for anything that touches AppKit or UI:

```swift
@MainActor
final class DocumentController: NSObject {
    var documents: [Document] = []

    func openDocument(at url: URL) async throws {
        let doc = try Document(contentsOf: url)
        documents.append(doc)
        // This is safe — we're isolated to the main actor.
    }
}
```

### nonisolated

Mark individual methods `nonisolated` when they don't access actor state:

```swift
actor ImageCache {
    private var cache: [String: NSImage] = [:]

    // Pure function — no actor state access.
    nonisolated func cacheKey(for url: URL) -> String {
        url.absoluteString
    }

    func image(for key: String) -> NSImage? {
        cache[key]
    }
}
```

### Global Actors and Actor Isolation Inference

A `@MainActor` propagates through call sites. If `@MainActor` is on a plain `func`, the entire function runs on the main thread. If it's on the type, every member is main-actor-isolated unless marked `nonisolated`. A `nonisolated async func` on an actor still runs on the actor — `nonisolated` alone doesn't remove actor isolation for `async` methods. Use `nonisolated` (non-async) for synchronous escape hatches; the compiler enforces no actor state access.

Swift 6.3 infers the executing actor context: a `@MainActor` method calling another `@MainActor` method does not need `await`. Cross-actor calls (main actor → background actor or vice versa) always require `await`.

## Task & TaskGroup

### Unstructured Task

`Task {}` creates a new top-level task on the current actor. Use it to launch async work from synchronous code:

```swift
func loadThumbnails() {
    Task { @MainActor in
        for doc in documents {
            let thumbnail = await generateThumbnail(for: doc)
            doc.thumbnail = thumbnail  // MainActor: safe
        }
    }
}
```

### Task Cancellation

Check `Task.isCancelled` or call `try Task.checkCancellation()` in cooperative cancellation points:

```swift
func processLargeFile(at url: URL) async throws {
    let handle = try FileHandle(forReadingFrom: url)
    defer { try? handle.close() }

    for try await line in handle.bytes.lines {
        // Cooperative check — throw if the task was cancelled.
        try Task.checkCancellation()
        await processLine(line)
    }
}
```

### withTaskCancellationHandler

Register a synchronous cleanup closure for when the task is cancelled:

```swift
func downloadFile(from url: URL) async throws -> Data {
    try await withTaskCancellationHandler {
        let (data, _) = try await URLSession.shared.data(from: url)
        return data
    } onCancel: {
        // Cancel the underlying request synchronously.
        // This runs on the task's executor, NOT asynchronously.
    }
}
```

### TaskGroup

`TaskGroup` runs a dynamic number of child tasks in parallel and collects results. Use `withThrowingTaskGroup` when children can fail:

```swift
func fetchAllMetadata(urls: [URL]) async throws -> [Metadata] {
    try await withThrowingTaskGroup(of: Metadata.self) { group in
        for url in urls {
            group.addTask {
                let (data, _) = try await URLSession.shared.data(from: url)
                return try JSONDecoder().decode(Metadata.self, from: data)
            }
        }

        var results: [Metadata] = []
        for try await metadata in group {
            results.append(metadata)
        }
        return results
    }
}
```

**Cancellation propagation:** When a task group is cancelled, all pending child tasks are cancelled. Errors from any child cancel the group (in the throwing variant).

### Task Value (Swift 6.2+)

Swift 6.2 introduced `@Sendable` isolated parameters and `sending` results. Swift 6.3 further stabilizes the consuming/borrowing parameter modifiers. For mutations within task work:

```swift
// Swift 6.3: consuming parameter — the closure takes ownership
func rewriteDocument(_ doc: consuming Document) async -> Document {
    doc.applyEdits()
    doc.finalize()
    doc  // returned as a sending result
}
```

## AsyncStream & AsyncSequence

`AsyncStream` creates a push-based async sequence. Use it to bridge callback-based APIs (KVO, `NSNotificationCenter`, `DispatchSource`) into `for-await-in` loops.

### Basic AsyncStream

```swift
func makeFileWatcher(at url: URL) -> AsyncStream<URL> {
    AsyncStream { continuation in
        let descriptor = open(url.path, O_EVTONLY)
        let source = DispatchSource.makeFileSystemObjectSource(
            fileDescriptor: descriptor,
            eventMask: .write,
            queue: .global()
        )

        source.setEventHandler {
            continuation.yield(url)
        }

        source.setCancelHandler {
            close(descriptor)
            continuation.finish()
        }

        source.resume()

        // Called when the consumer exits the for-await loop.
        continuation.onTermination = { @Sendable _ in
            source.cancel()
        }
    }
}

// Consumption
for await url in makeFileWatcher(at: configURL) {
    await reloadConfig(from: url)
}
```

### AsyncThrowingStream

Use `AsyncThrowingStream` when the producer can fail:

```swift
func readLines(from url: URL) -> AsyncThrowingStream<String, Error> {
    AsyncThrowingStream { continuation in
        Task {
            do {
                let handle = try FileHandle(forReadingFrom: url)
                for try await line in handle.bytes.lines {
                    continuation.yield(line)
                }
                continuation.finish()
            } catch {
                continuation.finish(throwing: error)
            }
        }
    }
}
```

### Buffering

Control what happens when the stream is faster than the consumer:

```swift
// Keep only the latest element; drop older ones.
let stream = AsyncStream<Int> { continuation in
    // ...
}.buffer(policy: .bufferingNewest(1))
```

## Sendable & Data Races

Swift 6 enforces data-race safety at compile time. Types passed across actor boundaries must conform to `Sendable`.

### Automatic Sendable Inference

The compiler infers `Sendable` for `struct` types whose stored properties are all `Sendable`, and for `final class` types with `Sendable` properties:

```swift
// Inferred Sendable: all properties are Sendable.
struct RenderConfig: Sendable {
    var width: Int
    var height: Int
    var scale: Double
}

// NOT Sendable: NSMutableData is a class with mutable state.
struct BadConfig: Sendable {
    var data: NSMutableData  // Compiler error: NSMutableData is not Sendable
}
```

### @unchecked Opt-out

Legacy or externally synchronized types that are safe to share but can't prove it to the compiler:

```swift
final class LegacyCounter: @unchecked Sendable {
    private var count: Int = 0
    private let lock = NSLock()

    func increment() {
        lock.lock()
        defer { lock.unlock() }
        count += 1
    }

    func value -> Int {
        lock.lock()
        defer { lock.unlock() }
        return count
    }
}
```

**Key rule:** `@unchecked Sendable` is a promise — if the type escapes its synchronization, you get runtime data races. Add an `NSLock` or move to an actor. Use `@unchecked Sendable` only when the type is wrapped in a lock, serial dispatch queue, or other synchronization primitive.

### Sendable Closures

Closures that capture mutable state must be `@Sendable`. In Swift 6.3, `@Sendable` is inferred for many closure contexts. When the compiler cannot infer it, annotate explicitly:

```swift
func process(in group: inout ThrowingTaskGroup<Output, Error>) {
    let config = RenderConfig(width: 1024, height: 768, scale: 2.0)
    group.addTask { @Sendable in
        // config is captured by value (value type) — safe.
        await render(config: config)
    }
}
```

## Strict Concurrency Migration

### Complete vs Enabled Checking

In Swift 6.3, strict concurrency checking has three levels:

| Level | Flag | Effect |
|-------|------|--------|
| Minimal | `minimal` (Swift 5.x compat) | Warns only on explicit `Sendable`/actor annotations |
| Targeted | `targeted` | Enforces on annotated code; warns elsewhere |
| Complete | `complete` (Swift 6.3 default) | Enforces everywhere; errors on all data-race risks |

Set in Xcode build settings under **Strict Concurrency Checking**. Swift 6.3 uses `complete` by default for new projects.

### Incremental Migration Strategy

For an existing macOS codebase:

1. **Start with `complete`** on a feature module boundary — a framework target with limited AppKit imports.
2. Annotate leaf types (`struct`s, value-holder `final class`s) with `Sendable`.
3. Wrap legacy singletons in `@unchecked Sendable` with `NSLock` backing. Refactor to actors as time allows.
4. Add `@MainActor` to `NSDocument` subclasses, `NSWindowController`s, and `NSViewController`s.
5. Use `@preconcurrency import` to suppress warnings from frameworks that haven't adopted Swift concurrency:

```swift
@preconcurrency import AppKit
```

6. Enable complete checking per-target; keep `targeted` for the AppKit-heavy target during migration.

### Suppressions

When the compiler flags a concurrency issue you need to suppress temporarily:

```swift
// Fix the root cause (annotate the callback as @Sendable, wrap in MainActor.assumeIsolated) — don't suppress.
```

### Default Actor Isolation in Xcode 26

New projects in Xcode 26 default to MainActor isolation (see [Approachable Concurrency](#approachable-concurrency-se-046104660470)). Existing projects remain nonisolated by default. You can change this in build settings under **Default Actor Isolation**.

## AppKit Bridging

### DispatchQueue.main.async from Async Context

When inside an async function and you need to touch AppKit, prefer `@MainActor` annotation over `DispatchQueue.main.async`. If you're on a background actor and the function is not `@MainActor`, use `MainActor.run`:

```swift
actor ThumbnailGenerator {
    func generate(for doc: Document) async -> NSImage? {
        // Background work...
        let rawImage = await renderImage(doc)

        // Jump to main actor for AppKit.
        await MainActor.run {
            let nsImage = NSImage(cgImage: rawImage, size: .zero)
            doc.thumbnail = nsImage  // If doc is @MainActor, await here too.
            return nsImage
        }
    }
}
```

### MainActor.assumeIsolated

`MainActor.assumeIsolated` performs a synchronous, non-hopping access to main-actor-isolated state. Use when you are **certain** you're already on the main thread — common in AppKit delegate callbacks:

```swift
func controlTextDidChange(_ notification: Notification) {
    // This is an NSTextFieldDelegate callback — always on the main thread.
    MainActor.assumeIsolated {
        searchField.stringValue = self.query
    }
}
```

**Danger:** calling `assumeIsolated` off the main actor is a runtime crash. Only use in callbacks you know are main-thread (most AppKit delegate methods, `NSNotificationCenter` observers on `.main`).

Do NOT use `@_unsafeInheritExecutor` as a substitute — it's unsafe unless you can guarantee caller executor matching.

### Bridging Completion Handler APIs

Many AppKit APIs with completion handlers are not yet annotated with `@Sendable`/`@MainActor`. Use `withCheckedContinuation` for robust bridging:

```swift
// Bridging NSAnimationContext.runAnimationGroup with completion handler
func animateChanges() async {
    await withCheckedContinuation { continuation in
        NSAnimationContext.runAnimationGroup({ context in
            context.duration = 0.3
            // Perform animations
        }, completionHandler: {
            continuation.resume()
        })
    }
    // Continuation runs — animation complete
}
```

The Swift 6.3 compiler may emit actor isolation warnings when completion handlers for APIs like `NSAnimationContext.runAnimationGroup` are not annotated. Wrap with `MainActor.assumeIsolated` or use the continuation pattern above until Apple annotates the API.

### DispatchQueue Bridging for KVO Callback Queues

When KVO observers fire on a background queue, bridge to Swift concurrency:

```swift
final class ProcessObserver: NSObject, @unchecked Sendable {
    private var observation: NSKeyValueObservation?

    func observe(process: Process) {
        observation = process.observe(\.isFinished, options: [.new]) { [weak self] _, change in
            Task { @MainActor [weak self] in
                await self?.handleCompletion(change.newValue ?? false)
            }
        }
    }

    @MainActor
    private func handleCompletion(_ finished: Bool) async { /* ... */ }
}
```

### NSRunLoop Interaction

`NSRunLoop.perform(_:)` is the AppKit equivalent of a synchronous dispatch. From async code, **avoid** `NSRunLoop.run()` or `CFRunLoopRunInMode()` — these are not async-safe and will trigger Swift 6 warnings (blocking the cooperative thread pool). Instead:

```swift
// Bridging perform from async context — fire-and-forget onto the main run loop:
func triggerMenuUpdate() async {
    await MainActor.run {
        // perform(_:) schedules on the next run loop cycle.
        _ = NSObject.perform(#selector(MenuController.refreshMenu),
                             on: .main,
                             with: nil,
                             waitUntilDone: false)
    }
}
```

For legacy code that blocks on `NSRunLoop.current.run(until:)` or `CFRunLoopRunInMode`, refactor to use async/await or `Task.sleep` rather than blocking the thread.

## XPC & Daemon Concurrency

### NSXPCConnection from Async Code

Bridge XPC reply handlers with `withCheckedContinuation`:

```swift
func fetchSystemInfo() async throws -> SystemInfo {
    try await withCheckedThrowingContinuation { continuation in
        let connection = NSXPCConnection(serviceName: "com.example.daemon")
        connection.remoteObjectInterface = NSXPCInterface(with: DaemonProtocol.self)
        connection.resume()

        let proxy = connection.remoteObjectProxyWithErrorHandler { error in
            continuation.resume(throwing: error)
        } as? DaemonProtocol

        proxy?.getSystemInfo { info in
            if let info {
                continuation.resume(returning: info)
            } else {
                continuation.resume(throwing: DaemonError.nilResponse)
            }
        }
    }
}
```

### XPC Service with Async Swift

In the XPC service, define the protocol with completion handlers so callers can use continuations:

```swift
// Shared protocol (async Swift + Objective-C interop):
@objc protocol DaemonProtocol {
    func getSystemInfo(completion: @escaping (Data?, Error?) -> Void)
}

// Service implementation — internally uses Swift concurrency:
final class DaemonService: NSObject, DaemonProtocol {
    func getSystemInfo(completion: @escaping (Data?, Error?) -> Void) {
        Task {
            do {
                let info = try await self.gatherInfo()
                let data = try JSONEncoder().encode(info)
                completion(data, nil)
            } catch {
                completion(nil, error)
            }
        }
    }

    private func gatherInfo() async throws -> SystemInfo {
        // Pure Swift concurrency inside the XPC service.
        async let cpu = getCPUInfo()
        async let memory = getMemoryInfo()
        async let disk = getDiskInfo()
        return try await SystemInfo(cpu: cpu, memory: memory, disk: disk)
    }
}
```

### @main Crash with XPC in Swift Language Version 6

A known issue: XPC services with `@main` annotated entry points crash in Swift Language Version 6 due to interaction between Swift concurrency's MainActor context and XPC's `incomingSessionHandler`.

**Workaround:** Move XPC listener setup to a separate function or `main.swift` file to avoid `@main`'s implicit MainActor context:

```swift
// Instead of:
// @main struct XPCService { ... }  // ← can crash

// Use a plain main.swift:
import Foundation

final class ServiceDelegate: NSObject, NSXPCListenerDelegate {
    func listener(_ listener: NSXPCListener,
                  shouldAcceptNewConnection newConnection: NSXPCConnection) -> Bool {
        newConnection.exportedInterface = NSXPCInterface(with: DaemonProtocol.self)
        newConnection.exportedObject = DaemonService()
        newConnection.resume()
        return true
    }
}

let delegate = ServiceDelegate()
let listener = NSXPCListener.service()
listener.delegate = delegate
listener.resume()
```

Alternatively, mark session handlers explicitly as `@Sendable`:

```swift
_ = try XPCListener.withContinuation(
    for: "com.example.service",
    { @Sendable request in
        request.accept { @Sendable (message: String) in
            // handle message
        }
    }
)
```

### Swift 6 Distributed Actors over XPC

Experimental project `andreasley/XPCDistributedActorSystem` demonstrates distributed actors using XPC as the transport layer on macOS 15+. Key types must be identical on both endpoints (place in a shared library).

```swift
// Shared library
protocol Calculator: DistributedActor {
    func add(_ a: Int, _ b: Int) async throws -> Int
}

// App side
let system = XPCDistributedActorSystem()
let calc: any Calculator = try system.resolve(id: .calculator, as: Calculator.self)
let result = try await calc.add(2, 3)  // 5

// Service side: host the actor
let system = XPCDistributedActorSystem()
try system.register(CalculatorImpl(), as: .calculator)
```

**Status:** Experimental. Known issues with actor ID assignment.

### Background Daemon Patterns (launchd + Swift Concurity)

A background daemon (`launchd` plist `LaunchDaemon` or `XPCService`) written in Swift with structured concurrency:

```swift
// main.swift for an XPC Service target
import Foundation

final class ServiceDelegate: NSObject, NSXPCListenerDelegate {
    func listener(_ listener: NSXPCListener,
                  shouldAcceptNewConnection newConnection: NSXPCConnection) -> Bool {
        newConnection.exportedInterface = NSXPCInterface(with: DaemonProtocol.self)
        let exportedObject = DaemonService()
        newConnection.exportedObject = exportedObject
        newConnection.resume()
        return true
    }
}

let delegate = ServiceDelegate()
let listener = NSXPCListener.service()
listener.delegate = listener
listener.resume()  // Blocks — runs the XPC service run loop.
```

For a `LaunchAgent`/`LaunchDaemon` that is a lifecycle-managed process (no XPC listener):

```swift
// A lifecycle-managed daemon
@main
struct BackgroundDaemon {
    static func main() async throws {
        let worker = try await Worker()
        try await worker.run()  // Async main — structured concurrency drives the process.
        // Process exits when the async main returns or is cancelled.
    }
}
```

## Approachable Concurrency (SE-0461/0466/0470)

Swift 6.2 / Xcode 26 introduces **Approachable Concurrency** — a set of features that reduce the friction of concurrency annotations, especially for UI-heavy and single-threaded code.

### SE-0466: Default Actor Isolation

Module-level setting to infer `@MainActor` by default:

```swift
// Package.swift
swiftSettings: [
    .defaultIsolation(MainActor.self)
]

// Or in Xcode 26: Build Settings → Default Actor Isolation → MainActor
```

With default MainActor isolation, all unannotated declarations in the module are implicitly `@MainActor`-isolated. This is the default for new Xcode 26 projects.

**Key details:**
- Does not affect declarations imported from other modules
- Can be overridden by explicit `nonisolated` or `@MainActor` annotations
- Members of actors are excluded from inference
- Changing default isolation is source-incompatible — existing projects retain their current default

### SE-0461: Nonisolated Nonsending by Default

Nonisolated async functions now inherit the caller's actor context by default (no more implicit hop to a background executor):

```swift
// Without SE-0461: nonisolated async func runs on a background executor
// With SE-0461: nonisolated async func runs on the caller's executor
nonisolated func computeValue() async -> Int {
    // If called from MainActor, this runs on MainActor
}
```

### SE-0470: InferIsolatedConformances

Automatically infers actor-isolated conformances, reducing manual annotations:

```swift
// Before: had to explicitly annotate
extension MyType: @MainActor SomeProtocol { }

// After: compiler infers isolation from context
extension MyType: SomeProtocol { }
```

### Enabling All Approachable Concurrency Features in Package.swift

```swift
swiftSettings: [
    .defaultIsolation(MainActor.self),
    .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
    .enableUpcomingFeature("InferIsolatedConformances"),
    .enableUpcomingFeature("InferSendableFromCaptures"),
    .enableUpcomingFeature("GlobalActorIsolatedTypesUsability"),
]
```

Or in Xcode 26, enable "Approachable Concurrency" in the target build settings dropdown. New Xcode 26 projects get it by default.

### Compiler Crash Warning

A known compiler crash exists when enabling approachable concurrency with `-O` (Release builds) under whole-module optimization. Track at [Swift Issue #88173](https://github.com/swiftlang/swift/issues/88173). Use Debug builds during migration if affected.

## Swift 6.3 Concurrency Updates

- **Default Actor Isolation (SE-0466):** Now stable and enabled by default in new Xcode 26 projects. Module-level MainActor inference reduces false-positive data-race diagnostics in sequential UI code.
- **NonisolatedNonsendingByDefault (SE-0461):** Nonisolated async functions inherit caller's executor by default — fewer unexpected hops.
- **InferSendableFromCaptures:** Reduces manual `@Sendable` annotations when the compiler can prove closure captures are safe.
- **InferIsolatedConformances:** Reduces boilerplate when conforming to protocols from different isolation contexts.
- **GlobalActorIsolatedTypesUsability:** Makes types isolated to global actors more ergonomic to use across modules.
- **XPC improvements:** Xcode 26 provides better diagnostics for XPC service concurrency setup.
- **SwiftPM 6.3:** Improved build plugin support for C interoperability with generated headers, module maps, and API notes.

---

## Constraints & Gotchas

1. **Actors do not guarantee a specific thread** — they guarantee serial access. Do not assume an actor runs on any particular thread except `@MainActor`. Use `Thread.isMainThread` sparingly.
2. **Re-entrancy in actors** — an actor method that `await`s can interleave with another call. Do not assume the actor's state is unchanged across an `await` point (the "actor re-entrancy" problem).
3. **Deadlocks** — calling back into the same actor synchronously (e.g., `actor.syncMethod()` instead of `await actor.asyncMethod()`) deadlocks. The compiler prevents most of these, but `nonisolated` methods that call isolated methods back are a risk.
4. **`NSLock` inside actors** — do not use `NSLock` inside an actor. The actor already serializes access. A lock inside an actor is redundant and can cause deadlocks with Swift's cooperative thread pool.
5. **Priority inversion with `Task {}`** — unstructured `Task {}` inherits the current priority. Use `Task(priority: .userInitiated)` for work the user is waiting on, or `Task.detached { }` to fully detach from the current actor and priority.
6. **AppKit is not Sendable** — `NSView`, `NSWindow`, `NSDocument` and their subclasses are reference types that are not Sendable. Pass them only on `@MainActor` contexts or through `@unchecked Sendable` wrappers with external synchronization (not recommended).
7. **`TaskGroup` ordering** — results arrive in completion order, not insertion order if using `next()`. Collect into an array if you need index correspondence.
8. **`AsyncStream` backpressure** — without buffering policy, `AsyncStream` suspends the producer when the consumer is slow. Specify `.unbounded` for fire-and-forget, or `.bufferingNewest(n)` for UI updates.
9. **NSRunLoop is not async-safe** — `NSRunLoop.current.run(until:)` and `CFRunLoopRunInMode()` are not safe from async contexts. Refactor blocking run loop usage to async/await patterns.
10. **XPC + @main crash** — XPC services with `@main` entry points can crash in Swift Language Version 6. Use `main.swift` or non-`@main` patterns.
11. **Completion handler Sendability** — Many AppKit completion handlers are not yet annotated with `@Sendable`/`@MainActor` in Swift 6.3. Use `MainActor.assumeIsolated` or `withCheckedContinuation` until Apple provides annotations across all APIs.

---

## Common Mistakes

- **Detached tasks for UI work.** `Task.detached { self.updateUI() }` — the detached task is not on the main actor. Use `Task { @MainActor in self.updateUI() }` or `await MainActor.run { self.updateUI() }`.
- **Forgetting `@MainActor` on `NSDocument` or `NSWindowController` callbacks.** AppKit calls these on the main thread, but the compiler doesn't know that without the annotation. Missing it causes data-race errors in Swift 6.
- **Capturing `self` strongly in `AsyncStream` continuations.** Always capture `weak self` — the continuation lives until the stream is finished or the consumer exits the loop.
- **Using `NSLock` inside an actor.** Actors already serialize. The lock is deadlock-prone under Swift's cooperative thread pool.
- **Mixing `DispatchQueue.main.async` and `MainActor.run`.** Pick one pattern per codebase. `MainActor.run` is the Swifty approach; `DispatchQueue.main.async` is needed for C/API callbacks.
- **Assuming `Task == Thread`.** Tasks are scheduled on the cooperative thread pool. A single task may execute on different threads across suspension points. Never store thread-local state in a task.
- **Ignoring `Sendable` errors on ObjC interop.** Bridged Objective-C types (e.g., from `NSXPCConnection`) are not `Sendable`. Wrap the result in a Swift value type before crossing actor boundaries.
- **Blocking `NSRunLoop.run()` from async code.** This blocks the cooperative thread pool. Use async/await alternatives.
- **Using `@main` for XPC services.** Can crash in Swift 6 Language Version. Use `main.swift` or explicit listener setup instead.

---

## Review Checklist

- [ ] All `@MainActor`-isolated types are annotated — `NSDocument`, `NSWindowController`, `NSViewController`, delegate classes
- [ ] `NSLock` is not used inside actors
- [ ] `Sendable` errors on all types passed across actor boundaries
- [ ] `withCheckedContinuation` is used for callback-based XPC and AppKit APIs
- [ ] `Task {}` used for structured concurrency (not `Task.detached` for main-thread work)
- [ ] Default actor isolation set appropriately (MainActor for UI modules, nonisolated for library targets)
- [ ] `AsyncStream` continuations capture `self` weakly
- [ ] Cooperative cancellation checked in long-running `for` loops (`Task.checkCancellation()`)
- [ ] `DispatchQueue.main.async` and `MainActor.run` do not both appear in the same module (pick one)
- [ ] Strict concurrency checking set to at least `targeted` for incremental migration
- [ ] XPC services use `main.swift` or non-`@main` patterns to avoid Swift 6 Language Version crash
- [ ] NSRunLoop blocking calls removed from async contexts
- [ ] Approach |
| `targeted` | Enforces on annotated code; warns elsewhere |
| `complete` | Enforces everywhere; errors on all data-race risks |

Set in Xcode build settings under **Strict Concurrency Checking**. Swift 6.3 uses `complete` by default for new projects.

### Incremental Migration Strategy

For an existing macOS codebase:

1. **Start with `complete`** on a feature module boundary — a framework target with limited AppKit imports.
2. Annotate leaf types (`struct`s, value-holder `final class`s) with `Sendable`.
3. Wrap legacy singletons in `@unchecked Sendable` with `NSLock` backing. Refactor to actors as time allows.
4. Add `@MainActor` to `NSDocument` subclasses, `NSWindowController`s, and `NSViewController`s.
5. Use `@preconcurrency import` to suppress warnings from frameworks that haven't adopted Swift concurrency:

```swift
@preconcurrency import AppKit
```

6. Enable complete checking per-target; keep `targeted` for the AppKit-heavy target during migration.

### Suppressions

When the compiler flags a concurrency issue you need to suppress temporarily:

```swift
// Fix the root cause (annotate the callback as @Sendable, wrap in MainActor.assumeIsolated) — don't suppress.
```

### Default Actor Isolation in Xcode 26

New projects in Xcode 26 default to MainActor isolation (see [Approachable Concurrency](#approachable-concurrency-se-046104660470)). Existing projects remain nonisolated by default. You can change this in build settings under **Default Actor Isolation**.

## AppKit Bridging

### DispatchQueue.main.async from Async Context

When inside an async function and you need to touch AppKit, prefer `@MainActor` annotation over `DispatchQueue.main.async`. If you're on a background actor and the function is not `@MainActor`, use `MainActor.run`:

```swift
actor ThumbnailGenerator {
    func generate(for doc: Document) async -> NSImage? {
        // Background work...
        let rawImage = await renderImage(doc)

        // Jump to main actor for AppKit.
        await MainActor.run {
            let nsImage = NSImage(cgImage: rawImage, size: .zero)
            doc.thumbnail = nsImage  // If doc is @MainActor, await here too.
            return nsImage
        }
    }
}
```

### MainActor.assumeIsolated

`MainActor.assumeIsolated` performs a synchronous, non-hopping access to main-actor-isolated state. Use when you are **certain** you're already on the main thread — common in AppKit delegate callbacks:

```swift
func controlTextDidChange(_ notification: Notification) {
    // This is an NSTextFieldDelegate callback — always on the main thread.
    MainActor.assumeIsolated {
        searchField.stringValue = self.query
    }
}
```

**Danger:** calling `assumeIsolated` off the main actor is a runtime crash. Only use in callbacks you know are main-thread (most AppKit delegate methods, `NSNotificationCenter` observers on `.main`).

Do NOT use `@_unsafeInheritExecutor` as a substitute — it's unsafe unless you can guarantee caller executor matching.

### Bridging Completion Handler APIs

Many AppKit APIs with completion handlers are not yet annotated with `@Sendable`/`@MainActor`. Use `withCheckedContinuation` for robust bridging:

```swift
// Bridging NSAnimationContext.runAnimationGroup with completion handler
func animateChanges() async {
    await withCheckedContinuation { continuation in
        NSAnimationContext.runAnimationGroup({ context in
            context.duration = 0.3
            // Perform animations
        }, completionHandler: {
            continuation.resume()
        })
    }
    // Continuation runs — animation complete
}
```

The Swift 6.3 compiler may emit actor isolation warnings when completion handlers for APIs like `NSAnimationContext.runAnimationGroup` are not annotated. Wrap with `MainActor.assumeIsolated` or use the continuation pattern above until Apple annotates the API.

### DispatchQueue Bridging for KVO Callback Queues

When KVO observers fire on a background queue, bridge to Swift concurrency:

```swift
final class ProcessObserver: NSObject, @unchecked Sendable {
    private var observation: NSKeyValueObservation?

    func observe(process: Process) {
        observation = process.observe(\.isFinished, options: [.new]) { [weak self] _, change in
            Task { @MainActor [weak self] in
                await self?.handleCompletion(change.newValue ?? false)
            }
        }
    }

    @MainActor
    private func handleCompletion(_ finished: Bool) async { /* ... */ }
}
```

### NSRunLoop Interaction

`NSRunLoop.perform(_:)` is the AppKit equivalent of a synchronous dispatch. From async code, **avoid** `NSRunLoop.run()` or `CFRunLoopMode()` — these are not async-safe and will trigger Swift 6 warnings (blocking the cooperative thread pool). Instead:

```swift
// Bridging perform from async context — fire-and-forget onto the main run loop:
func triggerMenuUpdate() async {
    await MainActor.run {
        // perform(_:) schedules on the next run loop cycle.
        _ = NSObject.perform(#selector(MenuController.refreshMenu),
                             on: .main,
                             with: nil,
                             waitUntilDone: false)
    }
}
```

For legacy code that blocks on `NSRunLoop.current.run(until:)` or `CFRunLoopRunInMode`, refactor to use async/await or `Task.sleep` rather than blocking the thread.

## XPC & Daemon Concurrency

### NSXPCConnection from Async Code

Bridge XPC reply handlers with `withCheckedContinuation`:

```swift
func fetchSystemInfo() async throws -> SystemInfo {
    try await withCheckedThrowingContinuation { continuation in
        let connection = NSXPCConnection(serviceName: "com.example.daemon")
        connection.remoteObjectInterface = NSXPCInterface(with: DaemonProtocol.self)
        connection.resume()

        let proxy = connection.remoteObjectProxyWithErrorHandler { error in
            continuation.resume(throwing: error)
        } as? DaemonProtocol

        proxy?.getSystemInfo { info in
            if let info {
                continuation.resume(returning: info)
            } else {
                continuation.resume(throwing: DaemonError.nilResponse)
            }
        }
    }
}
```

### XPC Service with Async Swift

In the XPC service, define the protocol with completion handlers so callers can use continuations:

```swift
// Shared protocol (async Swift + Objective-C interop):
@objc protocol DaemonProtocol {
    func getSystemInfo(completion: @escaping (Data?, Error?) -> Void)
}

// Service implementation — internally uses Swift concurrency:
final class DaemonService: NSObject, DaemonProtocol {
    func getSystemInfo(completion: @escaping (Data?, Error?) -> Void) {
        Task {
            do {
                let info = try await self.gatherInfo()
                let data = try JSONEncoder().encode(info)
                completion(data, nil)
            } catch {
                completion(nil, error)
            }
        }
    }

    private func gatherInfo() async throws -> SystemInfo {
        // Pure Swift concurrency inside the XPC service.
        async let cpu = getCPUInfo()
        async let memory = getMemoryInfo()
        async let disk = getDiskInfo()
        return try await SystemInfo(cpu: cpu, memory: memory, disk: disk)
    }
}
```

### @main Crash with XPC in Swift Language Version 6

A known issue: XPC services with `@main` annotated entry points crash in Swift Language Version 6 due to interaction between Swift concurrency's MainActor context and XPC's `incomingSessionHandler`.

**Workaround:** Move XPC listener setup to a separate function or `main.swift` file to avoid `@main`'s implicit MainActor context:

```swift
// Instead of:
// @main struct XPCService { ... }  // ← can crash

// Use a plain main.swift:
import Foundation

final class ServiceDelegate: NSObject, NSXPCListenerDelegate {
    func listener(_ listener: NSXPCListener,
                  shouldAcceptNewConnection newConnection: NSXPCConnection) -> Bool {
        newConnection.exportedInterface = NSXPCInterface(with: DaemonProtocol.self)
        newConnection.exportedObject = DaemonService()
        newConnection.resume()
        return true
    }
}

let delegate = ServiceDelegate()
let listener = NSXPCListener.service()
listener.delegate = delegate
listener.resume()
```

Alternatively, mark session handlers explicitly as `@Sendable`:

```swift
_ = try XPCListener.withContinuation(
    for: "com.example.service",
    { @Sendable request in
        request.accept { @Sendable (message: String) in
            // handle message
        }
    }
)
```

### Swift 6 Distributed Actors over XPC

Experimental project `andreasley/XPCDistributedActorSystem` demonstrates distributed actors using XPC as the transport layer on macOS 15+. Key types must be identical on both endpoints (place in a shared library).

```swift
// Shared library
protocol Calculator: DistributedActor {
    func add(_ a: Int, _ b: Int) async throws -> Int
}

// App side
let system = XPCDistributedActorSystem()
let calc: any Calculator = try system.resolve(id: .calculator, as: Calculator.self)
let result = try await calc.add(2, 3)  // 5

// Service side: host the actor
let system = XPCDistributedActorSystem()
try system.register(CalculatorImpl(), as: .calculator)
```

**Status:** Experimental. Known issues with actor ID assignment.

### Background Daemon Patterns (launchd + Swift Concurrency)

A background daemon (`launchd` plist `LaunchDaemon` or `XPCService`) written in Swift with structured concurrency:

```swift
// main.swift for an XPC Service target
import Foundation

final class ServiceDelegate: NSObject, NSXPCListenerDelegate {
    func listener(_ listener: NSXPCListener,
                  shouldAcceptNewConnection newConnection: NSXPCConnection) -> Bool {
        newConnection.exportedInterface = NSXPCInterface(with: DaemonProtocol.self)
        let exportedObject = DaemonService()
        newConnection.exportedObject = exportedObject
        newConnection.resume()
        return true
    }
}

let delegate = ServiceDelegate()
let listener = NSXPCListener.service()
listener.delegate = delegate
listener.resume()  // Blocks — runs the XPC service run loop.
```

For a `LaunchAgent`/`LaunchDaemon` that is a lifecycle-managed process (no XPC listener):

```swift
// A lifecycle-managed daemon
@main
struct BackgroundDaemon {
    static func main() async throws {
        let worker = try await Worker()
        try await worker.run()  // Async main — structured concurrency drives the process.
        // Process exits when the async main returns or is cancelled.
    }
}
```

## Approachable Concurrency (SE-0461/0466/0470)

Swift 6.2 / Xcode 26 introduces **Approachable Concurrency** — a set of features that reduce the friction of concurrency annotations, especially for UI-heavy and single-threaded code.

### SE-0466: Default Actor Isolation

Module-level setting to infer `@MainActor` by default:

```swift
// Package.swift
swiftSettings: [
    .defaultIsolation(MainActor.self)
]

// Or in Xcode 26: Build Settings → Default Actor Isolation → MainActor
```

With default MainActor isolation, all unannotated declarations in the module are implicitly `@MainActor`-isolated. This is the default for new Xcode 26 projects.

**Key details:**
- Does not affect declarations imported from other modules
- Can be overridden by explicit `nonisolated` or `@MainActor` annotations
- Members of actors are excluded from inference
- Changing default isolation is source-incompatible — existing projects retain their current default

### SE-0461: Nonisolated Nonsending by Default

Nonisolated async functions now inherit the caller's actor context by default (no more implicit hop to a background executor):

```swift
// Without SE-0461: nonisolated async func runs on a background executor
// With SE-0461: nonisolated async func runs on the caller's executor
nonisolated func computeValue() async -> Int {
    // If called from MainActor, this runs on MainActor
}
```

### SE-0470: InferIsolatedConformances

Automatically infers actor-isolated conformances, reducing manual annotations:

```swift
// Before: had to explicitly annotate
extension MyType: @MainActor SomeProtocol { }

// After: compiler infers isolation from context
extension MyType: SomeProtocol { }
```

### Enabling All Approachable Concurrency Features in Package.swift

```swift
swiftSettings: [
    .defaultIsolation(MainActor.self),
    .enableUpcomingFeature("NonisolatedNonsendingByDefault"),
    .enableUpcomingFeature("InferIsolatedConformances"),
    .enableUpcomingFeature("InferSendableFromCaptures"),
    .enableUpcomingFeature("GlobalActorIsolatedTypesUsability"),
]
```

Or in Xcode 26, enable "Approachable Concurrency" in the target build settings dropdown. New Xcode 26 projects get it by default.

### Compiler Crash Warning

A known compiler crash exists when enabling approachable concurrency with `-O` (Release builds) under whole-module optimization. Track at [Swift Issue #88173](https://github.com/swiftlang/swift/issues/88173). Use Debug builds during migration if affected.

## Swift 6.3 Concurrency Updates

- **Default Actor Isolation (SE-0466):** Now stable and enabled by default in new Xcode 26 projects. Module-level MainActor inference reduces false-positive data-race diagnostics in sequential UI code.
- **NonisolatedNonsendingByDefault (SE-0461):** Nonisolated async functions inherit caller's executor by default — fewer unexpected hops.
- **InferSendableFromCaptures:** Reduces manual `@Sendable` annotations when the compiler can prove closure captures are safe.
- **InferIsolatedConformances:** Reduces boilerplate when conforming to protocols from different isolation contexts.
- **GlobalActorIsolatedTypesUsability:** Makes types isolated to global actors more ergonomic to use across modules.
- **XPC improvements:** Xcode 26 provides better diagnostics for XPC service concurrency setup.
- **SwiftPM 6.3:** Improved build plugin support for C interoperability with generated headers, module maps, and API notes.

---

## Constraints & Gotchas

1. **Actors do not guarantee a specific thread** — they guarantee serial access. Do not assume an actor runs on any particular thread except `@MainActor`. Use `Thread.isMainThread` sparingly.
2. **Re-entrancy in actors** — an actor method that `await`s can interleave with another call. Do not assume the actor's state is unchanged across an `await` point (the "actor re-entrancy" problem).
3. **Deadlocks** — calling back into the same actor synchronously (e.g., `actor.syncMethod()` instead of `await actor.asyncMethod()`) deadlocks. The compiler prevents most of these, but `nonisolated` methods that call isolated methods back are a risk.
4. **`NSLock` inside actors** — do not use `NSLock` inside an actor. The actor already serializes access. A lock inside an actor is redundant and can cause deadlocks with Swift's cooperative thread pool.
5. **Priority inversion with `Task {}`** — unstructured `Task {}` inherits the current priority. Use `Task(priority: .userInitiated)` for work the user is waiting on, or `Task.detached { }` to fully detach from the current actor and priority.
6. **AppKit is not Sendable** — `NSView`, `NSWindow`, `NSDocument` and their subclasses are reference types that are not Sendable. Pass them only on `@MainActor` contexts or through `@unchecked Sendable` wrappers with external synchronization (not recommended).
7. **`TaskGroup` ordering** — results arrive in completion order, not insertion order if using `next()`. Collect into an array if you need index correspondence.
8. **`AsyncStream` backpressure** — without buffering policy, `AsyncStream` suspends the producer when the consumer is slow. Specify `.unbounded` for fire-and-forget, or `.bufferingNewest(n)` for UI updates.
9. **NSRunLoop is not async-safe** — `NSRunLoop.current.run(until:)` and `CFRunLoopRunInMode()` are not safe from async contexts. Refactor blocking run loop usage to async/await patterns.
10. **XPC + @main crash** — XPC services with `@main` entry points can crash in Swift Language Version 6. Use `main.swift` or non-`@main` patterns.
11. **Completion handler Sendability** — Many AppKit completion handlers are not yet annotated with `@Sendable`/`@MainActor` in Swift 6.3. Use `MainActor.assumeIsolated` or `withCheckedContinuation` until Apple provides annotations across all APIs.

---

## Common Mistakes

- **Detached tasks for UI work.** `Task.detached { self.updateUI() }` — the detached task is not on the main actor. Use `Task { @MainActor in self.updateUI() }` or `await MainActor.run { self.updateUI() }`.
- **Forgetting `@MainActor` on `NSDocument` or `NSWindowController` callbacks.** AppKit calls these on the main thread, but the compiler doesn't know that without the annotation. Missing it causes data-race errors in Swift 6.
- **Capturing `self` strongly in `AsyncStream` continuations.** Always capture `weak self` — the continuation lives until the stream is finished or the consumer exits the loop.
- **Using `NSLock` inside an actor.** Actors already serialize. The lock is deadlock-prone under Swift's cooperative thread pool.
- **Mixing `DispatchQueue.main.async` and `MainActor.run`.** Pick one pattern per codebase. `MainActor.run` is the Swifty approach; `DispatchQueue.main.async` is needed for C/API callbacks.
- **Assuming `Task == Thread`.** Tasks are scheduled on the cooperative thread pool. A single task may execute on different threads across suspension points. Never store thread-local state in a task.
- **Ignoring `Sendable` errors on ObjC interop.** Bridged Objective-C types (e.g., from `NSXPCConnection`) are not `Sendable`. Wrap the result in a Swift value type before crossing actor boundaries.
- **Blocking `NSRunLoop.run()` from async code.** This blocks the cooperative thread pool. Use async/await alternatives.
- **Using `@main` for XPC services.** Can crash in Swift 6 Language Version. Use `main.swift` or explicit listener setup instead.

---

## Review Checklist

- [ ] All `@MainActor`-isolated types are annotated — `NSDocument`, `NSWindowController`, `NSViewController`, delegate classes
- [ ] `NSLock` is not used inside actors
- [ ] `Sendable` errors on all types passed across actor boundaries
- [ ] `withCheckedContinuation` is used for callback-based XPC and AppKit APIs
- [ ] `Task {}` used for structured concurrency (not `Task.detached` for main-thread work)
- [ ] Default actor isolation set appropriately (MainActor for UI modules, nonisolated for library targets)
- [ ] `AsyncStream` continuations capture `self` weakly
- [ ] Cooperative cancellation checked in long-running `for` loops (`Task.checkCancellation()`)
- [ ] `DispatchQueue.main.async` and `MainActor.run` do not both appear in the same module (pick one)
- [ ] Strict concurrency checking set to at least `targeted` for incremental migration
- [ ] XPC services use `main.swift` or non-`@main` patterns to avoid Swift 6 Language Version crash
- [ ] NSRunLoop blocking calls removed from async contexts
- [ ] Completion handler-based AppKit APIs bridged via `MainActor.assumeIsolated` or `withCheckedContinuation`

---

## References

- [The Swift Programming Language — Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [SE-0304: Structured Concurrency](https://github.com/apple/swift-evolution/blob/main/proposals/0304-structured-concurrency.md)
- [SE-0338: Clarify Actor Isolation](https://github.com/apple/swift-evolution/blob/main/proposals/0338-clarify-actor-isolation.md)
- [SE-0466: Default Actor Isolation (Stable in Swift 6.2)](https://github.com/apple/swift-evolution/blob/main/proposals/0466-approachable-concurrency.md)
- [SE-0461: Nonisolated Nonsending By Default](https://github.com/apple/swift-evolution/blob/main/proposals/0461-nonisolated-nonsending-by-default.md)
- [Protecting Mutable State with Swift Actors (WWDC21)](https://developer.apple.com/videos/play/wwdc2021/10133/)
- [Visualize and Optimize Swift Concurrency (WWDC22)](https://developer.apple.com/videos/play/wwdc2022/10170/)
- [Embracing Swift Concurrency (WWDC25)](https://developer.apple.com/videos/play/wwdc2025/268/)
- [What's New in Swift (WWDC24 Session 10169)](https://developer.apple.com/videos/play/wwdc2024/10169/)
- [Meet async/await in Swift (WWDC21)](https://developer.apple.com/videos/play/wwdc2021/10132/)
- [Bridging Legacy APIs with MainActor.assumeIsolated](https://needone.app/bridging-legacy-apis-with-mainactor-assumeisolated/)
- [Xcode 26 Concurrency Sendable Closure Problems](https://developer.apple.com/forums/thread/788052)
- [Survival Guide to Migrate to Swift 6](https://www.telefonica.com/en/communication-room/reports/survival-guide-migrate-swift-6-breaking-app/)
- [Concorde Changes in Swift 6.2 — Swift Crafted](https://swiftcrafted.dev/article/swift-6-2-approachable-concurrency-complete-guide)
- [Approachable Concurrency in Swift Packages](https://useyourloaf.com/blog/approachable-concurrency-in-swift-packages/)
- [Exploring Concurrency Changes in Swift 6.2 — Donny Wals](https://www.donnywals.com/exploring-concurrency-changes-in-swift-6-2/)
- [Setting Default Actor Isolation in Xcode 26 — Donny Wals](https://www.donnywals.com/setting-default-actor-isolation-in-xcode-26/)
- [Swift 6.2: A First Look at How It's Changing Concurrency — SwiftLee](https://www.avanderlee.com/concurrency/swift-6-2-concurrency-changes/)
- [@main Syntax Crashes Services in Swift 6](https://forums.swift.org/t/main-syntax-crashes-services-on-language-version-6-but-not-5/79313)
- [XPCDistributedActorSystem (Experimental)](https://github.com/andreasley/XPCDistributedActorSystem)
- [XPC Apple Developer Documentation](https://developer.apple.com/documentation/xpc)
