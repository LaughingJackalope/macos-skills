---
name: swift-api-design-macos
description: "Swift API Design Guidelines for macOS — naming conventions for AppKit-style delegates, NS-prefixed type interop, @objc protocol design, error types (POSIXError, CocoaError), Swift package design for macOS libraries. When designing Swift APIs for macOS frameworks or libraries."
sources: https://www.swift.org/documentation/api-design-guidelines/, https://developer.apple.com/documentation/foundation/posixerror
researched: 2026-06-06
platform: macOS 26+
---

# Swift API Design for macOS

Swift API design patterns for macOS 26+ frameworks and libraries. Covers the Swift API Design Guidelines with macOS-specific applications: AppKit delegate naming conventions, `@objc` protocol requirements, `NS`-prefixed type bridging, error type design with `POSIXError`/`CocoaError`, and Swift Package Manager structure for macOS-only targets. For general Swift language patterns, see `swift-language-macos`. For AppKit-specific implementation, see `appkit-foundations`. For SwiftUI API patterns, see `swiftui-macos-patterns`.

See [references/api-design-patterns.md](references/api-design-patterns.md) for complete AppKit-style delegate protocol with SwiftUI wrapper, error type design with CocoaError bridging, Swift package structure for macOS-only libraries, and `@objc` protocol with optional methods.

## Contents

- [Naming Guidelines](#naming-guidelines)
- [AppKit Delegate Naming](#appkit-delegate-naming)
- [@objc Protocol Design](#objc-protocol-design)
- [NS-Prefixed Type Interop](#ns-prefixed-type-interop)
- [Error Types](#error-types)
- [Swift Package Design](#swift-package-design)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Naming Guidelines

Follow the [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/) as the baseline. macOS adds AppKit conventions on top.

**Clarity at the call site.** Every method name should read as grammatical English at the point of use. The compiler already provides type information and parameter labels; the name should not repeat it.

```swift
// Good — reads naturally, no redundancy
document.save(to: url)
windowController.presentAsModalWindow(for: document)
array.remove(at: index)
view.enclosingScrollView?.flashScrollers()

// Bad — redundant type information, awkward reads
document.saveDocument(toFilePath: url)
array.removeElementFromArrayAtIndex(index)
```

**Omit needless words.** If a type's role is obvious from the parameter type or context, do not repeat it in the method name. Remove `get` from accessors (use the property). Remove type names from methods when the type is already clear.

```swift
// Good
document.data(ofType: "public.plain-text")
color.alphaComponent   // not color.getAlphaComponent()

// Bad
document.getData(forType: "public.plain-text")
```

**Fluent naming rules:**
- Method names that mutate the receiver use imperative verbs: `append`, `remove`, `invalidate`.
- Non-mutating methods that return a new value use past participle or noun form: `sorted()`, `reversed()`, `uppercaseString`.
- Boolean properties read as assertions: `isEmpty`, `isEditable`, `hasPrefix(_:)`.
- Protocols that describe capabilities end in `-able` or `-ible`: `Equatable`, `Codable`, `Representable`. Protocols describing what something is use nouns: `Delegate`, `DataSource`.

**Initializers.** Always label every parameter — there is no unlabeled `init` pattern in Swift. Mirror the type's conceptual structure.

```swift
// Standard initializers
init(name: String)
init(contentsOf url: URL) throws
init(data: Data, encoding: String.Encoding) throws

// Failable initializers
init?(data: Data)

// NSCoding/NSSecureCoding bridge
init?(coder: NSCoder)
```

**Factory methods.** Use `make` prefixes for factory methods that create new instances without hiding the allocation: `makeWindowController()`, `makeTextStorage()`, `makeConstraint()`. Use type-named static factories when the creation method is descriptive: `URL(fileURLWithPath:)`, `NSImage(named:)`.

**Principle of least surprise.** If your framework introduces a concept that mirrors an existing Foundation type, match Foundation's naming for that concept. If your document model is similar to `NSDocument`, use the same method names for the same operations.

## AppKit Delegate Naming

AppKit delegates follow a strict naming pattern that pure Swift delegates should mirror when the delegate pattern is used.

**Protocol naming.** Suffix with `Delegate`. Do not use `Protocol` in the name. The delegate name describes *what* is being delegated, not that it is a delegate.

```swift
protocol WindowDelegate      // not WindowDelegateProtocol
protocol CompletionDelegate  // not CompletionHandler (Handler implies callback, not delegate)
protocol SelectionDelegate   // describes the domain being delegated
```

**First parameter convention.** The first parameter of every delegate method should be the delegating object, unlabeled. This matches Foundation's convention and makes delegate dispatch unambiguous in multi-object contexts.

```swift
protocol WindowDelegate: AnyObject {
    func window(_ window: NSWindow, shouldPopUpDocumentPathMenu menu: NSMenu) -> Bool
    func window(_ window: NSWindow, willPositionSheet sheet: NSWindow, using rect: NSRect) -> NSRect
}
```

**Method intent prefixes:**

| Prefix | Meaning | Returns | Default if unimplemented |
|--------|---------|---------|--------------------------|
| `will` | Something is about to happen (observable, not cancelable) | `Void` | — |
| `did` | Something already happened (notification) | `Void` | — |
| `should` | Permission request — can veto | `Bool` | `true` |
| `will/did` + progressive | Progress callbacks with a `Bool` for cancellation | `Void` (cancel via return or out-param) | — |

```swift
protocol DocumentDelegate: AnyObject {
    // The document is about to close — observe but cannot prevent
    func documentWillClose(_ document: Document)

    // The document has closed — notification only
    func documentDidClose(_ document: Document)

    // Permission: may this document save? Return false to prevent.
    func document(_ document: Document, shouldSave data: Data, to url: URL) -> Bool

    // The document encountered a non-fatal issue
    func document(_ document: Document, didEncounterIssue issue: DocumentIssue)
}
```

**When to use `will`/`did`/`should`:**
- Use `should` for any operation that can be cancelled (close, save, delete, submit).
- Use `will` for pre-observations where the delegate may prepare side effects.
- Use `did` for post-notifications where the delegate updates state or UI.
- Never use `should` for notification-only methods — a boolean return implies veto power.

**Data source naming.** Separate from delegates. Suffix with `DataSource`. Methods return requested data; they are queries, not events.

```swift
protocol SidebarDataSource: AnyObject {
    func numberOfItems(in sidebar: SidebarView) -> Int
    func sidebar(_ sidebar: SidebarView, itemAt index: Int) -> SidebarItem
    func sidebar(_ sidebar: SidebarView, titleForItemAt index: Int) -> String?
}
```

## @objc Protocol Design

Any Swift protocol that needs to be adopted by Objective-C classes (AppKit, KVO, target-action, XPC) must be marked `@objc`. This has cascading design implications.

**Mark the protocol `@objc`, not individual methods.** When a protocol is `@objc`, every member is implicitly `@objc`. Adding `@objc` to individual methods is redundant noise.

```swift
// Good
@objc protocol DraggableSource: AnyObject {
    func draggingSession(_ session: NSDraggingSession, willBeginAt screenPoint: NSPoint)
    func draggingSession(_ session: NSDraggingSession, endedAt screenPoint: NSPoint, operation: NSDragOperation)
}

// Bad — redundant @objc on methods
@objc protocol DraggableSource: AnyObject {
    @objc func draggingSession(_ session: NSDraggingSession, willBeginAt screenPoint: NSPoint)
}
```

**Optional vs. required.** This is the most consequential design decision for `@objc` protocols. An `@objc protocol` with `optional` methods must be adopted by a type that conforms to `NSObject` (either an `NSObject` subclass or a Swift class marked `@objc`). Pure Swift classes with `@objc` protocol conformance but no `NSObject` ancestry cannot satisfy `optional` requirements.

```swift
// Option 1: All methods required — any @objc-conforming type can adopt
@objc protocol RequiredDelegate: AnyObject {
    func didComplete(_ task: Task)
}

// Option 2: Has optional methods — conforming type MUST be NSObject or @objc class
@objc protocol OptionalDelegate: AnyObject {
    @objc optional func didComplete(_ task: Task)
    @objc optional func didFail(_ task: Task, with error: Error)
}
```

**The NSObject requirement for optional.** If your `@objc` protocol has `optional` methods, every conforming type must:
1. Be an `NSObject` subclass, OR
2. Be a Swift class explicitly marked `@objc` (which implies `NSObject` under the hood in most cases)

The safest requirement: document `AnyObject` conformance and note that conformers must be `NSObject` subclasses.

```swift
// Document this constraint for consumers
/// Conforming types must be `NSObject` subclasses due to `@objc optional` requirements.
@objc protocol CompletionDelegate: AnyObject {
    func completion(_ completion: CompletionHandler, didFinishWith result: Result)
    
    @objc optional func completion(_ completion: CompletionHandler, didUpdateProgress progress: Double)
    @objc optional func completionDidCancel(_ completion: CompletionHandler)
}
```

**Return types.** `@objc` protocols cannot use Swift-only types. All parameters and return types must be representable in Objective-C:
- Swift enums with associated values → **not representable**. Use `Int`-raw-value enums instead.
- Swift structs → **not representable**. Use classes.
- Swift optionals of representable types → OK (bridged as nullable).
- Tuples → **not representable**. Use separate parameters or a result class.

```swift
// Good — all ObjC-representable
@objc protocol DataSource: AnyObject {
    func itemCount() -> Int
    func item(at index: Int) -> DataSourceItem  // class, not struct
    func state() -> DataSourceState             // @objc enum: Int
}

@objc enum DataSourceState: Int {
    case idle
    case loading
    case loaded
    case error
}
```

**Method overloading.** Objective-C does not support overloading by parameter type (only by selector name). If your protocol will be used from Objective-C, all methods must have unique selector names — no two methods can share the same base name with different parameter types.

```swift
// Bad — ambiguous in ObjC: both are just func load
@objc protocol Loader {
    func load(_ url: URL)
    func load(_ data: Data)
}

// Good — unique selectors
@objc protocol Loader {
    func load(url: URL)
    func load(data: Data)
}
```

**@objc inference (Swift 6+).** In Swift 5.x, `@objc` was inferred for members of `@objc` classes. In Swift 6, this inference is removed — you must explicitly mark anything that needs ObjC visibility. This means delegate callbacks defined in your ObjC-facing classes now need explicit `@objc`.

## NS-Prefixed Type Interop

The `NS` prefix belongs to Objective-C's namespace and signals "this type bridges to AppKit/Foundation." It is not a cosmetic choice — it carries semantic meaning for interop.

**When to use NS prefix:**
- Types that directly wrap or subclass an Objective-C class (`NSView` subclass, custom `NSTableCellView`)
- Types in `@objc` protocol method signatures where the type must be ObjC-representable
- Types meant to be passed across the ObjC/Swift boundary where name clarity matters
- Types that exist *because* of an ObjC interop requirement (e.g., an `NS`-prefixed model object used inside an `NSViewRepresentable`)

```swift
// ObjC-representable model passed through NSViewRepresentable binding
class NSDocumentThumbnail: NSObject {
    let image: NSImage
    let label: String
    init(image: NSImage, label: String) {
        self.image = image
        self.label = label
    }
}
```

**When NOT to use NS prefix:**
- Pure Swift types used only within your Swift codebase
- SwiftUI views or Swift models (SwiftUI uses `View`, not `NSView`)
- Types that happen to be used alongside AppKit but do not cross the ObjC boundary
- New types that merely *conceptualize* an AppKit concept (e.g., don't call your settings model `NSSettingsModel` unless it literally inherits from `NSObject` and is passed to ObjC APIs)

**The bridging rule.** Adding `NS` prefix means you commit to NSObject conformance or ObjC representability. If a pure Swift struct needs to cross into ObjC, you must either:
1. Box it in an `NSObject` subclass with an `NS` prefix (the "bridge object" pattern)
2. Mark it with `@objcMembers` and accept the NSObject constraint

```swift
// Swift struct — pure Swift, no NS prefix
struct DocumentIdentifier: Hashable {
    let uuid: UUID
    let name: String
}

// When ObjC interop is required, box it
class NSDocumentIdentifier: NSObject {
    let identifier: DocumentIdentifier
    init(_ identifier: DocumentIdentifier) { self.identifier = identifier }
}
```

**Interoperability matrix:**

| Scenario | NS prefix? | Reason |
|----------|-----------|--------|
| Pure Swift value type | No | No ObjC boundary |
| NSObject subclass / ObjC-visible class | Yes | Signals ObjC interop |
| Swift struct used only in SwiftUI | No | SwiftUI is Swift-native |
| Swift struct boxed in NSObject for NSViewRepresentable | Yes (on box) | ObjC bridge object |
| Protocol used only in Swift | No | Swift-native |
| Protocol with `@objc` members | No (prefix is on types, not protocols) | Mark protocol `@objc`, not NS |

## Error Types

macOS APIs deal heavily with file system errors, POSIX errno codes, and Cocoa domain errors. Designing proper error types for these domains is critical.

**Conform to `LocalizedError`.** Every error type that reaches the user should conform to `LocalizedError`, not just `Error`. This provides `errorDescription`, `failureReason`, `recoverySuggestion`, and `recoveryAttempter` — the four keys that `NSAlert` and SwiftUI error presentation consume.

```swift
enum DocumentError: LocalizedError {
    case fileNotFound(URL)
    case invalidEncoding(String.Encoding)
    case readPermissionDenied(URL)
    case corruptFile(detail: String)

    var errorDescription: String? {
        switch self {
        case .fileNotFound(let url):
            return "The file at \(url.lastPathComponent) could not be found."
        case .invalidEncoding(let encoding):
            return "The file encoding \(encoding) is not supported."
        case .readPermissionDenied(let url):
            return "Permission denied when reading \(url.lastPathComponent)."
        case .corruptFile:
            return "The file appears to be corrupt."
        }
    }

    var failureReason: String? {
        switch self {
        case .fileNotFound:
            return "The file may have been moved or deleted."
        case .readPermissionDenied:
            return "Check the file's Get Info permissions or sandbox entitlements."
        default:
            return nil
        }
    }

    var recoverySuggestion: String? {
        switch self {
        case .fileNotFound:
            return "Choose a different file or create a new one."
        case .readPermissionDenied:
            return "Grant full disk access in System Settings > Privacy & Security."
        default:
            return nil
        }
    }
}
```

**POSIXError for system call failures.** When wrapping `read()`, `write()`, `open()`, `stat()`, or any POSIX C function, use `POSIXError` directly rather than inventing your own errno codes. `POSIXError` conforms to `LocalizedError` and codes to human-readable descriptions.

```swift
func readContents(of url: URL) throws -> Data {
    let fd = open(url.fileSystemRepresentation, O_RDONLY)
    if fd == -1 {
        throw POSIXError(POSIXErrorCode(rawValue: errno) ?? .ENOENT)
    }
    defer { close(fd) }

    var buffer = [UInt8](repeating: 0, count: 4096)
    let bytesRead = read(fd, &buffer, buffer.count)
    if bytesRead == -1 {
        throw POSIXError(POSIXErrorCode(rawValue: errno) ?? .EIO)
    }
    return Data(buffer.prefix(bytesRead))
}
```

**CocoaError for AppKit/Foundation domain failures.** When throwing errors in NSDocument subclasses, file coordinators, or other Foundation-integrated code, use `CocoaError` with specific `CocoaError.Code` values. This ensures proper error domain (`NSCocoaErrorDomain`) and enables the system's recovery machinery.

```swift
override func data(ofType typeName: String) throws -> Data {
    guard let data = content.data(using: .utf8) else {
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
```

**Custom error enums with associated data.** For application-specific errors that don't map to POSIX or Cocoa domains, create a local enum with `LocalizedError` conformance. Use associated values to carry context for the error description.

```swift
enum ExportError: LocalizedError, CustomNSError {
    case unsupportedFormat(String)
    case encodingFailed(String.Encoding)
    case outputWriteFailed(URL, underlying: Error)

    var errorDescription: String? {
        switch self {
        case .unsupportedFormat(let fmt):
            return "The output format '\(fmt)' is not supported."
        case .encodingFailed(let enc):
            return "Could not encode output using \(enc)."
        case .outputWriteFailed(let url, let error):
            return "Failed to write to \(url.lastPathComponent): \(error.localizedDescription)"
        }
    }

    // NSError bridging
    static let errorDomain = "com.myapp.export"
    var errorUserInfo: [String : Any] {
        var info: [String: Any] = [NSLocalizedDescriptionKey: errorDescription ?? ""]
        if case .outputWriteFailed(_, let underlying) = self {
            info[NSUnderlyingErrorKey] = underlying
        }
        return info
    }
}
```

**Bridging NSError ↔ Swift Error.** The Swift runtime automatically bridges between Swift `Error` and `NSError`. But for reliable round-tripping (especially through `@objc` delegate callbacks), implement `CustomNSError` to control the domain and userInfo. Without `CustomNSError`, the system invents a domain from your type name, which is unpredictable.

## Swift Package Design

**Target structure for macOS libraries.** A well-structured macOS library separates concerns into distinct targets:

```
MyMacOSSDK/
├── Package.swift
├── Sources/
│   ├── MyMacOSSDK/            # Public API — re-exports, umbrella types
│   ├── MyMacOSSDKCore/        # Core logic, no platform dependency
│   └── MyMacOSSDKAppKit/      # AppKit-dependent code (NSViewRepresentable, etc.)
└── Tests/
    └── MyMacOSSDKTests/
```

**Package.swift structure:**

```swift
// swift-tools-version: 6.3
import PackageDescription

let package = Package(
    name: "MyMacOSSDK",
    platforms: [
        .macOS(.v15)  // macOS 26 SDK; minimum deployment target
    ],
    products: [
        .library(
            name: "MyMacOSSDK",
            targets: ["MyMacOSSDK"]
        ),
    ],
    dependencies: [
        .package(url: "https://github.com/apple/swift-log.git", from: "1.5.0"),
    ],
    targets: [
        // Core: no platform-specific imports, fully testable on any platform
        .target(
            name: "MyMacOSSDKCore",
            dependencies: [
                .product(name: "Logging", package: "swift-log"),
            ]
        ),
        // Platform: imports AppKit, Foundation
        .target(
            name: "MyMacOSSDKAppKit",
            dependencies: ["MyMacOSSDKCore"]
        ),
        // Umbrella: re-exports public APIs
        .target(
            name: "MyMacOSSDK",
            dependencies: ["MyMacOSSDKCore", "MyMacOSSDKAppKit"]
        ),
        .testTarget(
            name: "MyMacOSSDKTests",
            dependencies: ["MyMacOSSDKCore"]
        ),
    ]
)
```

**Conditional compilation.** Use `#canImport(AppKit)` to gate AppKit-specific code in shared targets. Prefer target-level separation over conditional compilation when possible — it produces cleaner module boundaries.

```swift
// In a shared core target that you might test on Linux in CI
#if canImport(AppKit)
import AppKit

extension MyViewModel {
    func pasteboardType() -> NSPasteboard.PasteboardType {
        .fileURL
    }
}
#endif
```

**Visibility for frameworks.** Mark everything `public` that is part of your library's API surface. Mark everything else `internal` (the default). Use `package` access for symbols shared between your own targets but not exposed to consumers. Never use `open` unless you specifically intend for consumers to subclass your types — on macOS, this is rare outside of view/window classes.

```swift
// Public API — consumed by framework users
public struct Configuration { ... }
public func makeClient(config: Configuration) -> Client { ... }

// Package-internal — shared between MyMacOSSDKCore and MyMacOSSDKAppKit
package class ClientState { ... }

// Internal — not exposed at all
internal enum ParsingState { ... }
```

## Constraints & Gotchas

- **`@objc optional` requires NSObject-conforming protocol.** The most common pitfall: marking methods `optional` in an `@objc` protocol but forgetting that adopters must be `NSObject` subclasses or `@objc` classes. A pure Swift class conforming to a protocol with `optional` methods will compile but crash at runtime when the ObjC runtime checks for the method. Always document this requirement.

- **Swift 6 removes `@objc` inference.** In Swift 5, `@objc` was inferred for members of `@objc` classes and protocol conformances. In Swift 6, you must explicitly mark anything that needs ObjC visibility. Audit all delegate callbacks, IBActions, and KVO-observed properties.

- **`POSIXError` vs raw errno.** Never store raw `Int32` errno values as your error type. Use `POSIXError.Code` which maps errno codes to enum cases with human-readable descriptions. Raw errno values are unstable across platforms.

- **`CocoaError` domain matching.** When catching delegate callback errors, check both the domain and code. `CocoaError.Code` is an enum; `CocoaError` has a `init(_:userInfo:)` that constructs the right NSError domain. But errors arriving from ObjC delegate methods are bridged as `NSError` — you must cast: `if let cocoaError = error as? CocoaError, cocoaError.code == .fileReadNoSuchFile { ... }`.

- **`Sendable` conformity and `NSObject`.** `NSObject` is not implicitly `Sendable` in Swift 6. If you pass delegate objects across concurrency boundaries, explicitly mark conformances or use `@unchecked Sendable` only when you can prove thread safety.

## Common Mistakes

- **Method naming: `get` prefix on accessors.** Use properties, not `get`-prefixed methods. `var isEmpty: Bool` not `func getIsEmpty() -> Bool`.
- **Protocol name includes "Protocol" suffix.** Use `Delegate`, not `DelegateProtocol`. The role (Delegate, DataSource, etc.) is the concept, not the Swift type.
- **NS prefix on pure Swift types.** Adding `NS` to a type that has no ObjC interop requirement is misleading. It implies NSObject conformance and ObjC bridgeability where none exists.
- **Throwing raw `NSError` in Swift code.** Prefer typed Swift errors (`enum Error: LocalizedError`) that bridge to NSError automatically. Reserve raw NSError for delegate callbacks from ObjC.
- **All delegate methods marked `should`.** Only use `should` for cancellable operations. Notification-only methods (`willClose`, `didSave`) should return `Void`.
- **Omitting the delegating object as first parameter.** Every delegate method should accept the sender as the first parameter so that one object can be delegate of multiple sources.
- **Using `@objc` on individual Swift protocol members instead of the protocol itself.** Mark the `@objc` on the protocol; individual member annotations are redundant and noise.
- **Not separating AppKit-dependent code into its own target.** This inflates test time and prevents reuse of core logic on other platforms or in CLI tools.

## Review Checklist

- [ ] Every public API reads grammatically at the call site with type information omittable
- [ ] No `get` prefix on accessors; use computed properties instead
- [ ] Delegate protocols are suffixed `Delegate`, data source protocols are suffixed `DataSource`
- [ ] All delegate methods take the delegating object as the first unlabeled parameter
- [ ] Methods use correct intent prefix: `should` for cancellable, `will`/`did` for observation
- [ ] NS prefix used only on types that cross the ObjC boundary (NSObject subclasses, ObjC-visible types)
- [ ] All error types conform to `LocalizedError` with `errorDescription`, `failureReason`, and `recoverySuggestion`
- [ ] System call errors use `POSIXError`; Foundation/AppKit errors use `CocoaError`
- [ ] Custom error types implement `CustomNSError` for reliable NSError bridging
- [ ] `@objc` protocols with optional methods document the NSObject requirement
- [ ] All delegate callbacks and IBActions explicitly marked `@objc` (Swift 6 requirement)
- [ ] Package targets separate core logic from platform-specific code
- [ ] `minimum-deployments` set correctly in Package.swift for macOS versions
- [ ] Visibility uses `public` for API, `package` for inter-target, `internal` for everything else

## References

- [references/api-design-patterns.md](references/api-design-patterns.md) — Complete AppKit-style delegate protocol with SwiftUI wrapper, error type design with CocoaError bridging, Swift package structure for macOS-only library, and @objc protocol with optional methods
- [Swift API Design Guidelines](https://www.swift.org/documentation/api-design-guidelines/)
- [POSIXError Documentation](https://developer.apple.com/documentation/foundation/posixerror)
- [CocoaError Documentation](https://developer.apple.com/documentation/foundation/cocoaerror)

## Research Addendum — 2026 Updates

### AppKit Delegate Naming Evolution

- **Cocoa Encyclopedia pattern**: Classic AppKit delegates use a `delegate` outlet/property with a protocol (formal or informal). AppKit supports both formal (`@objc protocol`) and informal (category on NSObject) protocols, but Swift code should prefer formal `@objc protocols`.
- **`respondsToSelector:` before optional methods** — In mixed ObjC/Swift code, always check responds before calling optional delegate methods to avoid crash on incomplete conformances.
- **WWDC 2026 naming emphasis**: Apple continues to push clarity at the call site. The 2026 guidelines reinforce: omit type names from methods when the type is obvious; prefer parameters that read as grammatical English; avoid `get` prefix.
- **First argument convention in pure Swift**: Apple's own newer delegates (outside AppKit) are moving toward making the sender the first unlabeled argument, matching AppKit's `window(_:shouldPopUpDocumentPathMenu:)` pattern. Follow this even in pure Swift delegate protocols.

### @objc Protocol Patterns (2026)

- **Swift 6 strictness**: No `@objc` inference at all. Every symbol that crosses the ObjC boundary must be explicitly annotated. This includes protocol conformances, class inheritance, methods, and properties used via KVO, target-action, or `#selector`.
- **XPC service protocols**: `NSXPCInterface` requires `@objc` protocols. The Swift protocol must be marked `@objc` and all parameters/return types must be ObjC-representable. Optional methods require `@objc optional`.
- **Bridge crashes**: A Swift protocol marked `@objc` but with an associated-enum case or struct parameter will crash at runtime when called from ObjC. Audit all `@objc` protocol method signatures for ObjC representability.
- **NSObject descendants**: Pure Swift classes (no `NSObject` ancestry) can conform to `@objc` protocols but cannot satisfy `optional` requirements. Document this constraint clearly.

### Error Types (2026)

- **`POSIXError.Code`** maps standard POSIX errno values (`.ENOENT`, `.EPERM`, `.EEXIST`, `.ENOSPC`, `.EIO`, etc.) to Swift enum cases. Use these instead of raw `Int32` errno values — they're stable and provide `LocalizedError` descriptions.
- **`CocoaError.Code`** covers Foundation/AppKit domain errors (`.fileReadNoSuchFile`, `.fileWriteFileExists`, `.fileWriteInapplicableStringEncoding`, `.coderReadCorrupt`, etc.). Use `CocoaError(.fileReadNoSuchFile, userInfo: [...])` for proper NSError domain.
- **Bridging**: When catching errors from `@objc` delegate callbacks, cast `NSError` to `CocoaError` or `POSIXError`:
  ```swift
  if let posixError = error as? POSIXError, posixError.code == .ENOENT { ... }
  if let cocoaError = error as? CocoaError, cocoaError.code == .fileReadNoSuchFile { ... }
  ```
- **`CustomNSError`** is essential for reliable round-tripping through `@objc` boundaries. Without it, the system invents an unpredictable domain from your type name. Always implement `errorDomain`, `errorCode`, and `errorUserInfo`.
