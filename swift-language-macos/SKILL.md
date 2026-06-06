---
name: swift-language-macos
description: "Modern Swift language features for macOS development: value types, generics, result builders, macros, property wrappers, Sendable, and Objective-C/Swift interop patterns. Trigger when writing Swift code that uses advanced type-system features, implements macros, or bridges to AppKit on macOS."
sources: https://docs.swift.org/swift-book/documentation/the-swift-programming-language/, https://developer.apple.com/documentation/swift, https://developer.apple.com/videos/play/wwdc2024/, https://swift.org/blog/swift-6.3-released/, https://swift.org/blog/whats-new-in-swift-march-2026/, https://developer.apple.com/documentation/Swift/applying-macros, https://developer.apple.com/videos/play/wwdc2023/10166/, https://developer.apple.com/videos/play/wwdc2023/10167/, https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md, https://github.com/apple/swift/blob/main/docs/ObjCInterop.md, https://developer.apple.com/videos/play/wwdc2022/10075/, https://chaosinmotion.com/2026/02/13/swiftui-style-recomposition-in-appkit-uikit/, https://developer.apple.com/documentation/Updates/AppKit, https://steipete/ObservationTrackingExample
researched: 2026-06-06
platform: macOS 26+
---

# Swift Language — macOS Patterns

Modern Swift language features applied to macOS development with Swift 6.3. Covers value-type semantics, generics, result builders, macros, property wrappers, Sendable concurrency, and Objective-C interop required for AppKit bridging. This is a language-focused skill; for AppKit framework patterns see `appkit-foundations`, for SwiftUI interop see `swiftui-appkit-interop`.

See [references/swift-macos-patterns.md](references/swift-macos-patterns.md) for complete KVO implementation, `@objc` protocol design, macro definition, `@UserDefaults` property wrapper, and type erasure patterns.

## Contents

- [Value Types & Generics](#value-types--generics)
- [Result Builders](#result-builders)
- [Macros](#macros)
- [Property Wrappers](#property-wrappers)
- [Sendable & Strict Concurrency](#sendable--strict-concurrency)
- [Objective-C & AppKit Interop](#objective-c--appkit-interop)
- [Swift 6.3 Language Updates](#swift-63-language-updates)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

---

## Value Types & Generics

### struct vs class vs actor

Choose the right tool for the job:

| Construct | Use When | Identity | Inheritance | Thread Safety |
|-----------|----------|----------|-------------|---------------|
| `struct` | Data containers, view models, configuration, delegates held by a single owner | Value (copied) | N/A | Safe — each copy is independent |
| `class` | Shared mutable state, identity matters (NSView, NSWindowController), @objc bridging | Reference (heap) | Single inheritance | Manual — needs locks/queues |
| `actor` | Shared mutable state accessed concurrently | Reference (heap) | N/A | Built in — serializes access |

**Rule of thumb for macOS:** Start with `struct`. Move to `class` only when you need reference semantics, identity comparison (`===`), or Objective-C interop. Move to `actor` when that shared state crosses concurrency boundaries.

```swift
// struct: value semantics, ideal for view models parsed from documents
struct DocumentModel {
    var title: String
    var body: NSAttributedString
    var modificationDate: Date
}

// class: identity matters — an NSWindowController is a unique object
final class InspectorController: NSWindowController {
    var observedDocument: DocumentModel?
}

// actor: shared mutable state across concurrency boundaries
actor RenderCache {
    private var cache: [URL: NSImage] = [:]

    func image(for url: URL) -> NSImage? { cache[url] }
    func store(_ image: NSImage, for url: URL) { cache[url] = image }
}
```

### Existentials: `any` vs `some`

Use `some Protocol` (opaque type) when the concrete type is known at compile time. Use `any Protocol` (existential) when you need heterogeneous collections or runtime type erasure.

```swift
// Opaque type — compiler knows the concrete type, enables specialization
func makeShape() -> some Shape { Circle(radius: 5) }

// Existential — type-erased box, needed for heterogeneous storage
var shapes: [any Shape] = [Circle(radius: 5), Rect(width: 10, height: 20)]

// macOS common case: an array of differently-typed AppKit commands
var toolbarItems: [any IToolbarItem] = [SearchItem(), ShareItem(), ExportItem()]
```

**Performance:** Existentials use an inline buffer (3 pointers) + heap allocation for large values. For hot paths, prefer `some` or generics. For configuration-time code (toolbar setup, menu building), `any` is fine.

```swift
// Generic dispatch — no existential overhead
func render<T: Shape>(_ shape: T) { shape.draw() }

// Existential dispatch — uses witness table
func render(_ shape: any Shape) { shape.draw() }
```

### Type Erasure with `any`

When a protocol has `assocatiedtype` or `Self` requirements, you can't use it directly as a type. Create an eraser with `@unchecked Sendable` if the eraser crosses concurrency boundaries (see `references/swift-macos-patterns.md` for a complete example):

```swift
protocol Validating {
    associatedtype Input
    func validate(_ input: Input) -> Bool
}

struct AnyValidator<Input>: Validating {
    private let _validate: (Input) -> Bool

    init<V: Validating>(_ validator: V) where V.Input == Input {
        _validate = validator.validate
    }

    func validate(_ input: Input) -> Bool { _validate(input) }
}
```

---

## Result Builders

Result builders let you construct domain-specific languages by composing static method calls. SwiftUI's view DSL is the canonical example, but custom builders work well for AppKit command/menu configuration, layout, and validation pipelines.

### How They Work

A result builder attaches to a parameter or function and transforms a statement sequence into a single value via static methods defined on the builder type. The key methods:

| Method | Purpose |
|--------|---------|
| `buildBlock(_:)` | Combines components into the final type |
| `buildOptional(_:)` | Handles `if` without `else` |
| `buildEither(first:)` / `buildEither(second:)` | Handles `if/else` |
| `buildArray(_:)` | Handles `for` loops |
| `buildFinalResult(_:)` | Optional post-processing step |

### Custom Builder for Menu DSL

```swift
@resultBuilder
struct MenuBuilder {
    static func buildBlock(_ components: NSMenuItem...) -> [NSMenuItem] { components }

    static func buildOptional(_ component: [NSMenuItem]?) -> [NSMenuItem] {
        component ?? []
    }

    static func buildEither(first: [NSMenuItem]) -> [NSMenuItem] { first }
    static func buildEither(second: [NSMenuItem]) -> [NSMenuItem] { second }

    static func buildArray(_ components: [[NSMenuItem]]) -> [NSMenuItem] {
        components.flatMap { $0 }
    }
}

func makeMenu(@MenuBuilder _ items: () -> [NSMenuItem]) -> NSMenu {
    let menu = NSMenu(title: "")
    items().forEach(menu.addItem)
    return menu
}

// Usage — declarative menu construction
let fileMenu = makeMenu {
    NSMenuItem(title: "New", action: #selector(newDoc), keyEquivalent: "n")
    NSMenuItem(title: "Open…", action: #selector(openDoc), keyEquivalent: "o")
    NSMenuItem.separator()

    #if DEBUG
    NSMenuItem(title: "Debug Console", action: #selector(showConsole), keyEquivalent: "D")
    #endif

    for recentURL in recentDocuments.prefix(5) {
        NSMenuItem(title: recentURL.lastPathComponent, action: #selector(openRecent), keyEquivalent: "")
    }
}
```

### SwiftUI CommandsBuilder

SwiftUI provides `CommandsBuilder` as a built-in result builder for constructing commands menus declaratively. It works like `ViewBuilder` but for `CommandMenu` and `CommandGroup`:

```swift
var body: some Commands {
    CommandMenu("File") {
        Button("New") { newDocument() }
            .keyboardShortcut("n")
        Button("Open…") { openDocument() }
            .keyboardShortcut("o")
        Divider()
        Button("Close") { closeDocument() }
            .keyboardShortcut("w")
    }

    CommandGroup(after: .newItem) {
        Button("Import…") { importFile() }
    }
}
```

**Limitations:** Result builders don't support `break`, `continue`, `return`, or `catch`. Local `let` bindings are allowed. Availability checks (`#available`) work inside builder closures.

---

## Macros

Swift macros eliminate boilerplate by generating code at compile time. The macro *definition* lives in a separate macro target (a Swift Package); the macro *usage* is in your main target.

### Freestanding Macros

Freestanding macros appear with a `#` prefix and can produce arbitrary code:

```swift
// Built-in: #warning and #error for compile-time diagnostics
#warning("TODO: migrate to new API before macOS 26 GM")
#error("This path is unreachable in production")

// Custom freestanding macro: generates a type-safe notification name
let name = #notificationName("DocumentDidChange")
// Expands to: NSNotification.Name("com.example.DocumentDidChange")
```

### Attached Macros

Attached macros modify the declaration they're attached to:

| Attachment Point | What It Can Do |
|-----------------|----------------|
| `@attached(member)` | Add new members (properties, methods) to a type |
| `@attached(extension)` | Add protocol conformances via extensions |
| `@attached(peer)` | Add sibling declarations |
| `@attached(accessor)` | Add property accessors (for property wrappers) |

```swift
// Attached macro that generates Equatable/Hashable from stored properties
@attached(member, names: named(==), named(hash))
@attached(extension, conformances: Equatable, Hashable)
public macro AutoEquatable() = #externalMacro(module: "MyMacros", type: "AutoEquatableMacro")

// Usage — the macro synthesizes == and hash(into:)
@AutoEquatable
struct DocumentConfig {
    var fontSize: Double
    var fontFamily: String
    var lineSpacing: Double
}
```

### Defining a Macro

Macro definitions require the `SwiftSyntax` package and a separate macro target in your `Package.swift`:

```swift
// In a macro target (library: .macro)
import SwiftSyntax
import SwiftSyntaxMacros

public struct ViewModelMacro: MemberMacro {
    public static func expansion(
        of node: AttributeSyntax,
        providingMembersOf declaration: some DeclGroupSyntax,
        conformingTo protocols: [TypeSyntax],
        in context: some MacroExpansionContext
    ) throws -> [DeclSyntax] {
        guard let classDecl = declaration.as(ClassDeclSyntax.self) else {
            throw MacroError.appliedToNonClass
        }

        let className = classDecl.name.text
        return [
            "static let identifier = \"\(raw: className)\"",
            "weak var delegate: \(raw: className)Delegate?",
        ]
    }
}
```

**Macro debugging:** Use `-Xfrontend -dump-macro-expansions` to see generated code. In Xcode, right-click a macro invocation → "Expand Macro." In Xcode 26.5+, use Editor → Expand Macro.

### Swift 6.3 Macro Improvements

Swift 6.3 improves macro development with better SwiftSyntax prebuilt reuse, avoiding conflicts between source-built and prebuilt macro libraries. The Swift Package Manager 6.3 release notes also include improved symbol graph generation with control over inherited documentation inclusion.

---

## Property Wrappers

Property wrappers encapsulate access patterns for stored properties. They project a `value` and a `$value` (the wrapper instance itself).

### @dynamicMemberLookup

`@dynamicMemberLookup` lets you forward member access to an underlying object. Useful for type-safe wrappers around `NSUserDefaults`, configuration dictionaries, or KVO-compliant objects:

```swift
@dynamicMemberLookup
struct AppConfig {
    private var storage: [String: Any] = [:]

    subscript<T>(dynamicMember key: String) -> T? {
        get { storage[key] as? T }
        set { storage[key] = newValue }
    }
}

var config = AppConfig()
config.defaultFontSize = 14.0  // forwarded to storage
config.fontFamily = "SF Mono"
let size: Double? = config.defaultFontSize  // 14.0
```

**Caution:** `@dynamicMemberLookup` bypasses the type checker — you lose autocomplete and compile-time safety. Use it only for bridging to dynamic systems (KVO, NSUserDefaults, scripting).

### Property Wrapper for NSUserDefaults

See `references/swift-macos-patterns.md` for a complete `@UserDefaults` property wrapper with typed access, default values, and KVO compliance.

```swift
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T
    let storage: UserDefaults

    init(wrappedValue: T, key: String, storage: UserDefaults = .standard) {
        self.key = key
        self.defaultValue = wrappedValue
        self.storage = storage
        self.storage.register(defaults: [key: wrappedValue])
    }

    var wrappedValue: T {
        get { storage.object(forKey: key) as? T ?? defaultValue }
        set { storage.set(newValue, forKey: key) }
    }

    var projectedValue: Binding<T> { /* ... */ }
}
```

### KVO-Compliant Stored Properties

For classes that need KVO, use `@objc dynamic` on stored properties. The `dynamic` keyword ensures accessors go through the Objective-C runtime:

```swift
final class Document: NSObject {
    @objc dynamic var title: String = ""
    @objc dynamic var isModified: Bool = false
    @objc dynamic var lastSavedDate: Date?
}
```

### @Observable with AppKit (macOS 15+/26+)

macOS 15+ (and default on macOS 26+) supports SwiftUI-style automatic recomposition in AppKit. When you read `@Observable` properties inside certain lifecycle methods, the framework automatically re-runs those methods when the property changes — eliminating KVO, Combine, and manual `setNeedsDisplay()` calls.

Lifecycle methods that participate in automatic observation tracking:
- `viewWillLayout()` / `viewDidLayout()`
- `updateViewConstraints()`
- `layout()`
- `draw(_:)`
- `updateConfiguration(using:)`

```swift
@MainActor
@Observable
final class DocumentModel {
    var title: String = ""
    var isModified: Bool = false
}

final class DocumentView: NSView {
    var model: DocumentModel?

    override func viewWillLayout() {
        super.viewWillLayout()
        // Reading model properties here establishes automatic observation.
        if let model {
            titleLabel.stringValue = model.title
            needsSaveIndicator.isHidden = !model.isModified
        }
    }
}
```

**Enable on macOS 15:** Set `NSObservationTrackingEnabled = true` in Info.plist. On macOS 26+, this is enabled by default.

**Best practices:**
- Place observation code in the appropriate lifecycle method (e.g., `viewWillLayout`, not `draw`)
- Cache expensive results to avoid recomputation
- Mutate observable properties on the main thread
- Use `@ObservationIgnored` for properties that shouldn't trigger updates

---

## Sendable & Strict Concurrency

Swift 6 enforces strict concurrency checking. Every value that crosses concurrency boundaries (actor boundaries, `@Sendable` closures) must conform to `Sendable`.

### Sendable Conformance

| Type | Sendable? | Notes |
|------|-----------|-------|
| `struct` with all-`Sendable` fields | ✅ Automatic | Compiler synthesizes |
| `enum` with all-`Sendable` cases | ✅ Automatic | Compiler synthesizes |
| `class` | ❌ Manual | Must be `final`, all fields `Sendable` and `let` |
| `actor` | ✅ Always | Built-in isolation |
| `NSView`, `NSWindow` | ❌ Never | Main-actor only, reference type |
| `UnsafePointer` | ❌ Never | Raw memory |

### @unchecked Sendable

Use `@unchecked Sendable` when you *know* a type is thread-safe but the compiler can't prove it. Common for classes that use internal locking or are only accessed from a known queue:

```swift
final class CachedImageStore: @unchecked Sendable {
    private var cache: [URL: NSImage] = [:]
    private let lock = NSLock()

    func image(for url: URL) -> NSImage? {
        lock.lock(); defer { lock.unlock() }
        return cache[url]
    }

    func store(_ image: NSImage, for url: URL) {
        lock.lock(); defer { lock.unlock() }
        cache[url] = image
    }
}
```

**Never** mark a type `@unchecked Sendable` without a clear thread-safety argument. If you're wrong, you get data races that are nearly impossible to debug.

### Migrating Class-Only Protocols

When a protocol needs to be adopted by actors or `Sendable` types, remove `AnyObject` conformance and add `Sendable`:

```swift
// Before: class-only, blocks actors
protocol DataSource: AnyObject {
    func fetch() async -> [Item]
}

// After: works with actors and value types
protocol DataSource: Sendable {
    func fetch() async -> [Item]
}
```

If existing code requires both `AnyObject` and `Sendable`, use a protocol composition at the use site:

```swift
func useDataSource(_ source: some DataSource & AnyObject) { }
```

### @retroactive Conformances

When you need to add a conformance to a type you don't own, use `@retroactive` to silence the "redundant conformance" warning and make the intent explicit:

```swift
// NSView doesn't conform to Sendable, but you can't add that.
// Instead, for your own types:
extension MyCustomType: @retroactive Equatable {
    static func == (lhs: MyCustomType, rhs: MyCustomType) -> Bool {
        lhs.id == rhs.id
    }
}
```

---

## Objective-C & AppKit Interop

Most of AppKit is Objective-C under the hood. Swift code that touches `NSView`, `NSWindowController`, `NSDocument`, or the responder chain needs `@objc` bridging.

### @objc and @objcMembers

Use `@objc` on individual members that need Objective-C visibility. Use `@objcMembers` on a class to export all eligible members at once:

```swift
@objcMembers
final class PreferencesViewController: NSViewController {
    // All eligible members are @objc by default
    var fontSize: Double = 14.0
    var fontFamily: String = "SF Mono"

    @objc(didChangeFont:)  // custom ObjC selector
    func didChangeFont(_ sender: Any?) { }

    // Excluded from ObjC because it uses Swift-only types
    var swiftOnlyConfig: some Shape? = nil
}
```

**What gets exported:** class types, protocols, methods, properties, subscripts, and initializers that use ObjC-compatible types. Swift-only types (`some Protocol`, `async throws`, tuples, generic types) are excluded.

### NSObject Subclassing

Subclass `NSObject` when you need:
- KVO/KVC compliance (`@objc dynamic` properties)
- Target-action patterns (`@objc func buttonClicked(_:)`)
- Nib/storyboard loading (`init?(coder:)`)
- `NSCoding` / `NSSecureCoding` for archiving

Don't subclass `NSObject` for pure-Swift classes that never touch the ObjC runtime — it adds reference-counting overhead and changes identity semantics.

```swift
// NSObject: needed for KVO + target-action
final class ToolbarController: NSObject {
    weak var document: Document?

    @objc func handleShare(_ sender: Any?) {
        guard let doc = document else { return }
        // ...
    }
}

// Pure Swift: no NSObject needed
struct DocumentSnapshot {
    let title: String
    let body: String
    let date: Date
}
```

### KVO with NSKeyValueObservation

The modern KVO pattern uses `observe(_:options:changeHandler:)` which returns an `NSKeyValueObservation` token. The observation is automatically invalidated when the token is deallocated:

```swift
final class DocumentObserver {
    private var observation: NSKeyValueObservation?

    func observe(_ document: Document) {
        observation = document.observe(\.isModified, options: [.new]) { doc, change in
            print("Modified: \(change.newValue ?? false)")
        }
    }
}
```

For manual KVO (implementing `observeValue(forKeyPath:of:change:context:)`), always use unique static context pointers and call `super` for unhandled keys:

```swift
final class ManualObserver: NSObject {
    private static var context = 0

    func startObserving(_ document: Document) {
        document.addObserver(self, forKeyPath: "title", options: [.new], context: &Self.context)
    }

    override func observeValue(
        forKeyPath keyPath: String?,
        of object: Any?,
        change: [NSKeyValueChangeKey: Any]?,
        context: UnsafeMutableRawPointer?
    ) {
        guard context == &Self.context else {
            super.observeValue(forKeyPath: keyPath, of: object, change: change, context: context)
            return
        }
        if keyPath == "title" {
            // handle title change
        }
    }
}
```

### KVO Dependent Keys

For derived/calculated properties that depend on other keys, use the class method pattern:

```swift
final class Person: NSObject {
    @objc dynamic var firstName: String = ""
    @objc dynamic var lastName: String = ""

    @objc var fullName: String { "\(firstName) \(lastName)" }

    // Declare that fullName depends on firstName and lastName
    @objc class func keyPathsForValuesAffectingFullName() -> Set<String> {
        [#keyPath(firstName), #keyPath(lastName)]
    }
}
```

For to-many relationships, automatic key-path dependencies are not supported — observe each child object's relevant keys manually, or use Core Data notifications.

### Target-Action Patterns

Target-action is AppKit's event delivery mechanism. In Swift, the action method must be `@objc` and take a single `Any?` parameter:

```swift
final class ActionHandler: NSObject {
    @objc func handleMenuItem(_ sender: NSMenuItem) {
        switch sender.action {
        case #selector(saveDocument): save()
        case #selector(closeDocument): close()
        default: break
        }
    }

    func configureMenu() {
        let item = NSMenuItem(title: "Save", action: #selector(handleMenuItem), keyEquivalent: "s")
        item.target = self
    }
}
```

For `IBAction` from storyboards/xibs, the pattern is the same but wired in Interface Builder:

```swift
@IBAction func saveDocument(_ sender: Any?) {
    // Connected via storyboard
}
```

### SwiftUI with AppKit: NSViewRepresentable Coordinator

The `NSViewRepresentable` coordinator pattern (WWDC22) is the standard way to manage AppKit delegates and KVO in SwiftUI wrappers:

```swift
struct ScriptEditorRepresentable: NSViewRepresentable {
    @Binding var sourceCode: String

    func makeCoordinator() -> Coordinator {
        Coordinator(sourceCode: $sourceCode)
    }

    func makeNSView(context: Context) -> ScriptEditorView {
        let view = ScriptEditorView()
        view.delegate = context.coordinator
        return view
    }

    func updateNSView(_ nsView: ScriptEditorView, context: Context) {
        if nsView.text != sourceCode {
            nsView.text = sourceCode
        }
    }

    final class Coordinator: NSObject, ScriptEditorViewDelegate {
        @Binding var sourceCode: String

        init(sourceCode: Binding<String>) {
            _sourceCode = sourceCode
        }

        func scriptEditor(_ editor: ScriptEditorView,DidChangeText text: String) {
            sourceCode = text
        }
    }
}
```

---

## Swift 6.3 Language Updates

Swift 6.3 (released March 2026) brings several language and tooling improvements relevant to macOS development:

### @c Attribute for C Interoperability

New `@c` attribute exposes Swift functions and enums directly to C, generating corresponding C headers:

```swift
@c("sw_compute")
func computeValue(_ x: Int32) -> Int32 {
    x * 2
}

// Generates a C header declaration:
// int32_t sw_compute(int32_t x);
```

### Module Selectors

When multiple imported modules provide the same symbol, use module selectors for disambiguation:

```swift
import Foundation
import CoreFoundation

// Disambiguate when types collide
let url: Foundation.URL = URL(string: "https://example.com")!
```

### Optimization Attributes for Library Authors

New attributes give library authors more control:

- `@specialize`: Pre-specialize generics for known types to eliminate runtime overhead
- `@inline(always)`: Guarantee inlining across module boundaries
- `@exported(implementation)`: ABI-stable library optimization hints

### Swift Build Integration

Swift 6.3 previews Swift Build as the next-generation unified cross-platform build system, integrated with Swift Package Manager. Try with `swift build --build-system swiftbuild`.

### SE-0522: Source-Level Warning Control

Fine-grained per-declaration warning control with `@warn`:

```swift
@warn("Migrate to the new API before macOS 27")
func legacyImport() { }
```

### SE-0515: Non-copyable reduce

Enables `reduce` to produce non-copyable results by consuming the initial value, reducing unnecessary copies for expensive-to-copy types.

---

## Constraints & Gotchas

1. **Swift 6 strict concurrency + AppKit:** Most AppKit classes are `@MainActor`. If you mark a class `@MainActor`, all its members run on the main actor. This is correct for view controllers but wrong for background data processing. Use `nonisolated` for pure-computation methods.

2. **`@objc` + generics:** Generic types and methods cannot be exposed to Objective-C. If you need both generics and ObjC bridging, create a generic Swift implementation and an `@objc` wrapper class.

3. **Existential opening (Swift 6.1+):** You can now open existentials inline: `func draw(shape: some Shape)` can be called with `any Shape` via implicit opening. This reduces the need for type erasure in many cases.

4. **`@retroactive` for conformances:** Always use `@retroactive` when adding a conformance to a type from another module. Without it, you get a warning, and the conformance may silently break if the upstream module adds the same conformance.

5. **Macro expansion order:** Macros expand top-down, outer-before-inner. An attached macro on a class sees the class declaration but not the members added by other attached macros on the same class.

6. **`@unchecked Sendable` is a promise:** The compiler trusts you. If the type has mutable shared state without synchronization, you get undefined behavior. Document the thread-safety invariant in a comment.

7. **KVO + Swift properties:** Only `@objc dynamic` stored properties are KVO-compliant. Computed properties, `let` constants, and non-`dynamic` properties do not trigger KVO notifications.

8. **Result builder availability:** Result builder components must resolve to the same type. Use `buildOptional`, `buildEither`, `buildArray` for conditional/loop support.

9. **Automatic AppKit observation scope:** Only specific lifecycle methods participate in `@Observable` tracking (`viewWillLayout`, `draw`, etc.). Reading a property in `mouseDown` or an action method does NOT establish automatic observation.

10. **KVO to-many dependencies:** `keyPathsForValuesAffectingValueForKey:` does not work with to-many relationships. Observe each child object's relevant keys and update the parent manually.

---

## Common Mistakes

- **Using `class` by default.** Start with `struct`. Move to `class` only for identity, ObjC interop, or shared mutable state.
- **Forgetting `@objc` on action methods.** Target-action requires ObjC dispatch. A plain Swift method won't be found by the responder chain.
- **Using `any` when `some` suffices.** If the concrete type is known at compile time, `some Protocol` avoids existential overhead.
- **Marking everything `@unchecked Sendable`.** This is not a concurrency fix — it's a concurrency promise. Use actors or locks instead.
- **KVO without `dynamic`.** `@objc` alone isn't enough. The property must be `@objc dynamic` for automatic KVO.
- **Not using `@retroactive`.** Adding conformances to foreign types without `@retroactive` produces warnings and is fragile.
- **Subclassing `NSObject` unnecessarily.** Pure Swift types don't need it. It adds ObjC runtime overhead and changes value semantics to reference semantics.
- **Ignoring macro expansion errors.** Macro errors can be cryptic. Use `-Xfrontend -dump-macro-expansions` to debug.
- **Reading `@Observable` properties outside lifecycle methods.** Only designated AppKit lifecycle methods participate in automatic observation tracking.
- **Forgetting `buildOptional`/`buildEither` in result builders.** Conditional content (`if`/`else`) without these methods causes compiler errors.

---

## Review Checklist

- [ ] Value types (`struct`) preferred over `class` unless reference semantics or ObjC interop is required
- [ ] `some Protocol` used where the concrete type is known at compile time; `any Protocol` only for heterogeneous storage
- [ ] `@unchecked Sendable` has a documented thread-safety justification
- [ ] KVO properties are `@objc dynamic`, not just `@objc`
- [ ] `@retroactive` used for all conformances on foreign types
- [ ] Result builder `buildOptional` and `buildEither` handle conditional content
- [ ] Macro targets are in a separate `.macro` library product
- [ ] `NSObject` subclassing is justified (KVO, target-action, nib loading)
- [ ] `@MainActor` isolation is correct — background work is `nonisolated` or on a separate actor
- [ ] Existential types (`any`) are not used in performance-critical paths without profiling
- [ ] `@Observable` properties read only in designated AppKit lifecycle methods for automatic recomposition
- [ ] KVO dependent keys declared via `keyPathsForValuesAffecting<Key>` for derived properties

---

## References

- [Swift Programming Language — Generics](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/generics/)
- [Swift Programming Language — Macros](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/macros/)
- [Swift Programming Language — Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
- [WWDC 2024: Write Swift macros](https://developer.apple.com/videos/play/wwdc2024/10166/)
- [WWDC 2024: Meet Swift 6](https://developer.apple.com/videos/play/wwdc2024/10136/)
- [WWDC 2023: Write Swift macros](https://developer.apple.com/videos/play/wwdc2023/10166/)
- [WWDC 2023: Expand on Swift macros](https://developer.apple.com/videos/play/wwdc2023/10167/)
- [WWDC 2022: Use SwiftUI with AppKit](https://developer.apple.com/videos/play/wwdc2022/10075/)
- [Apple Developer: Applying Macros](https://developer.apple.com/documentation/Swift/applying-macros)
- [Apple Developer: Swift Package Manager Macros](https://developer.apple.com/documentation/swift-packages/macros)
- [Swift 6.3 Released](https://swift.org/blog/swift-6.3-released/)
- [What's New in Swift — March 2026](https://swift.org/blog/whats-new-in-swift-march-2026/)
- [SwiftPM 6.3 Release Notes](https://github.com/swiftlang/swift-package-manager/blob/main/Documentation/ReleaseNotes/6.3.md)
- [SE-0289: Result Builders](https://github.com/apple/swift-evolution/blob/main/proposals/0289-result-builders.md)
- [Swift @c Attribute for C Interop](https://infoworld.media/article/4150248/swift-6-3-boosts-c-interoperability-android-sdk.html)
- [SwiftUI-Style Recomposition in AppKit/UIKit](https://chaosinmotion.com/2026/02/13/swiftui-style-recomposition-in-appkit-uikit/)
- [AppKit 2026 Updates](https://developer.apple.com/documentation/Updates/AppKit)
- [ObservationTrackingExample](https://github.com/steipete/ObservationTrackingExample)
- [Swift and Objective-C Interop (Internal)](https://github.com/apple/swift/blob/main/docs/ObjCInterop.md)
- [references/swift-macos-patterns.md](references/swift-macos-patterns.md) — Complete KVO, `@objc` protocol design, macro definition, `@UserDefaults` wrapper, type erasure
