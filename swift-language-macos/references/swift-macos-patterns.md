# Swift macOS Patterns — Deep Reference

Complete code patterns for Swift language features in macOS development. Each section is self-contained and copy-paste ready.

---

## Complete KVO: Modern and Manual

### Modern KVO with NSKeyValueObservation

The preferred KVO pattern. The observation token manages the lifecycle — when it deallocates, the observation is removed automatically.

```swift
import Cocoa

final class Document: NSObject {
    @objc dynamic var title: String = "Untitled"
    @objc dynamic var isModified: Bool = false
    @objc dynamic var wordCount: Int = 0
}

final class DocumentInspector: NSViewController {
    private var observations: [NSKeyValueObservation] = []
    private var document: Document?

    func bind(to document: Document) {
        self.document = document
        observations.removeAll()

        // Observe a single key path with change handler
        observations.append(
            document.observe(\.title, options: [.initial, .new]) { [weak self] doc, change in
                self?.titleField.stringValue = doc.title
            }
        )

        // Observe with old+new values for diffing
        observations.append(
            document.observe(\.isModified, options: [.old, .new]) { [weak self] doc, change in
                let wasModified = change.oldValue ?? false
                let isModified = change.newValue ?? false
                if wasModified != isModified {
                    self?.updateWindowModifiedState(isModified)
                }
            }
        )

        // Observe multiple key paths on the same object
        observations.append(
            document.observe(\.wordCount, options: [.new]) { [weak self] doc, change in
                self?.statusLabel.stringValue = "\(doc.wordCount) words"
            }
        )
    }

    func unbind() {
        observations.removeAll()  // Invalidates all observations
        document = nil
    }

    deinit {
        observations.removeAll()
    }

    // Stub UI outlets
    @IBOutlet var titleField: NSTextField!
    @IBOutlet var statusLabel: NSTextField!

    private func updateWindowModifiedState(_ modified: Bool) {
        view.window?.isDocumentEdited = modified
    }
}
```

**Key rules:**
- Always use `[weak self]` in change handlers to avoid retain cycles.
- Store tokens in an array for batch invalidation.
- The `.initial` option fires the handler immediately with the current value.
- Observations are automatically removed when the token is deallocated.

### Manual KVO with observeValue

Use manual KVO when you need context-based dispatch or when observing many objects of the same class:

```swift
final class WindowTitleObserver: NSObject {
    private static var context = 0  // Unique pointer for this observer

    weak var window: NSWindow?
    var onTitleChange: ((String) -> Void)?

    func startObserving(_ window: NSWindow) {
        self.window = window
        window.addObserver(self, forKeyPath: #keyPath(NSWindow.title),
                           options: [.new], context: &Self.context)
    }

    func stopObserving() {
        window?.removeObserver(self, forKeyPath: #keyPath(NSWindow.title),
                               context: &Self.context)
        window = nil
    }

    override func observeValue(
        forKeyPath keyPath: String?,
        of object: Any?,
        change: [NSKeyValueChangeKey: Any]?,
        context: UnsafeMutableRawPointer?
    ) {
        // Always check context first — this method may be called for key paths
        // observed by superclasses or other observers on the same object
        guard context == &Self.context else {
            super.observeValue(forKeyPath: keyPath, of: object,
                               change: change, context: context)
            return
        }

        if keyPath == #keyPath(NSWindow.title),
           let title = change?[.newKey] as? String {
            onTitleChange?(title)
        }
    }

    deinit {
        stopObserving()
    }
}
```

**Critical:** Always call `super.observeValue(...)` for unhandled contexts. Failing to do so crashes on macOS 14+.

---

## @objc Protocol Design: AppKit Delegate Patterns

### NSApplicationDelegate in Swift

```swift
import Cocoa

@MainActor
@objc
final class AppDelegate: NSObject, NSApplicationDelegate {
    var mainWindowController: NSWindowController?

    func applicationDidFinishLaunching(_ notification: Notification) {
        let window = NSWindow(
            contentRect: NSRect(x: 100, y: 100, width: 800, height: 600),
            styleMask: [.titled, .closable, .miniaturizable, .resizable],
            backing: .buffered,
            defer: false
        )
        window.title = "MyApp"

        let wc = NSWindowController(window: window)
        wc.contentViewController = MainViewController()
        wc.showWindow(nil)
        mainWindowController = wc
    }

    func applicationShouldOpenUntitledFile(_ sender: NSApplication) -> Bool {
        return true
    }

    func applicationSupportsSecureRestorableState(_ app: NSApplication) -> Bool {
        return true
    }
}
```

### NSWindowDelegate in Swift

```swift
final class WindowStateTracker: NSObject, NSWindowDelegate {
    var onWillClose: (() -> Void)?
    var onDidResize: ((NSSize) -> Void)?

    func windowWillClose(_ notification: Notification) {
        onWillClose?()
    }

    func windowDidResize(_ notification: Notification) {
        guard let window = notification.object as? NSWindow else { return }
        onDidResize?(window.frame.size)
    }

    func windowDidBecomeKey(_ notification: Notification) {
        // Window is now the key window
    }

    func windowDidResignKey(_ notification: Notification) {
        // Window is no longer the key window
    }
}

// Usage:
final class MyWindowController: NSWindowController {
    private let stateTracker = WindowStateTracker()

    override func windowDidLoad() {
        super.windowDidLoad()
        window?.delegate = stateTracker

        stateTracker.onWillClose = { [weak self] in
            self?.saveWindowState()
        }
    }

    private func saveWindowState() {
        guard let window else { return }
        let frame = NSStringFromRect(window.frame)
        UserDefaults.standard.set(frame, forKey: "mainWindowFrame")
    }
}
```

### Designing Custom @objc Protocols

When designing a protocol that needs to be used from Objective-C (e.g., for delegate patterns):

```swift
// The protocol must be @objc to be used from Objective-C or for target-action
@objc protocol DocumentEditorDelegate: AnyObject {
    // Required method
    func editor(_ editor: DocumentEditor, didChange text: String)

    // Optional method — must be marked @objc optional
    @objc optional func editorDidSave(_ editor: DocumentEditor)
    @objc optional func editor(_ editor: DocumentEditor, didEncounterError error: NSError)
}

final class DocumentEditor: NSObject {
    weak var delegate: DocumentEditorDelegate?

    func updateText(_ text: String) {
        delegate?.editor(self, didChange: text)
    }

    func save() {
        // ... save logic ...
        delegate?.editorDidSave?(self)  // Optional chaining for @objc optional
    }
}
```

**Rules for @objc protocols:**
- The protocol must be marked `@objc`.
- Optional methods require `@objc optional`.
- Only `@objc` types can conform — not structs, enums, or actors.
- Associated types are not supported.

---

## Macro Definition: ViewModel Boilerplate Generator

This example defines an attached macro that generates common boilerplate for view model classes — a `static identifier`, a `delegate` property, and a default `init`.

### Macro Declaration (in a `.macro` target)

```swift
// Package.swift: .macro(name: "MyMacros", targets: ["MyMacros"])

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
            throw MacroError("@ViewModel can only be applied to classes")
        }

        let className = classDecl.name.text

        return [
            """
            static let identifier = "\(raw: className)"
            """,
            """
            weak var delegate: \(raw: className)Delegate?
            """,
            """
            private var cancellables = Set<AnyCancellable>()
            """,
            """
            deinit {
                cancellables.removeAll()
            }
            """
        ]
    }
}

enum MacroError: String, Error {
    case wrongTarget = "@ViewModel can only be applied to classes"
}
```

### Macro Usage (in your main target)

```swift
@ViewModel
final class ProfileViewModel: NSObject {
    @Published var name: String = ""
    @Published var email: String = ""
}

// Expands to:
// final class ProfileViewModel: NSObject {
//     @Published var name: String = ""
//     @Published var email: String = ""
//
//     static let identifier = "ProfileViewModel"
//     weak var delegate: ProfileViewModelDelegate?
//     private var cancellables = Set<AnyCancellable>()
//     deinit { cancellables.removeAll() }
// }
```

---

## Property Wrapper: @UserDefaults

A complete property wrapper for typed access to `NSUserDefaults` with default values, KVO compliance, and projected value binding support.

```swift
import Foundation
import Combine

@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T
    let storage: UserDefaults

    init(wrappedValue: T, key: String, storage: UserDefaults = .standard) {
        self.key = key
        self.defaultValue = wrappedValue
        self.storage = storage
    }

    var wrappedValue: T {
        get {
            switch T.self {
            case is any Optional.Type:
                // For optional types, return nil if key doesn't exist
                guard storage.object(forKey: key) != nil else { return defaultValue as! T }
                fallthrough
            default:
                return storage.object(forKey: key) as? T ?? defaultValue
            }
        }
        nonmutating set {
            if let optional = newValue as? AnyOptional, optional.isNil {
                storage.removeObject(forKey: key)
            } else {
                storage.set(newValue, forKey: key)
            }
        }
    }

    /// Projected value provides a Combine publisher for KVO-style observation
    var projectedValue: AnyPublisher<T, Never> {
        NotificationCenter.default
            .publisher(for: UserDefaults.didChangeNotification, object: storage)
            .compactMap { [weak storage] _ in
                storage?.object(forKey: key) as? T ?? self.defaultValue
            }
            .prepend(wrappedValue)
            .eraseToAnyPublisher()
    }
}

// Helper protocol to detect Optional at runtime
private protocol AnyOptional {
    var isNil: Bool { get }
}

extension Optional: AnyOptional {
    var isNil: Bool { self == nil }
}

// MARK: - Usage

final class AppPreferences: ObservableObject {
    @UserDefault(key: "defaultFontSize")
    var defaultFontSize: Double = 14.0

    @UserDefault(key: "fontFamily")
    var fontFamily: String = "SF Mono"

    @UserDefault(key: "showLineNumbers")
    var showLineNumbers: Bool = true

    @UserDefault(key: "lastOpenedDirectory")
    var lastOpenedDirectory: URL? = nil

    @UserDefault(key: "recentDocumentCount")
    var recentDocumentCount: Int = 10
}

// Observing changes via Combine:
final class PreferencesController {
    private let prefs = AppPreferences()
    private var cancellables = Set<AnyCancellable>()

    init() {
        // Observe a specific default via the projected value ($)
        prefs.$defaultFontSize
            .sink { newSize in
                print("Font size changed to \(newSize)")
            }
            .store(in: &cancellables)
    }
}
```

**Key design decisions:**
- `nonmutating set` because `UserDefaults` is global mutable state — the wrapper itself doesn't mutate.
- The projected value (`$`) exposes a Combine publisher for reactive binding.
- Optional values are handled by removing the key when set to `nil`.

---

## Type Erasure: AnyShape-Style for AppKit

AppKit protocols like `NSAnimatablePropertyContainer` and custom drawing protocols often have associated types or Self requirements. Type erasure lets you store heterogeneous conforming types.

### The Problem

```swift
protocol Drawable {
    associatedtype RenderContext
    func draw(in context: RenderContext)
}

// Can't use this directly:
// var items: [Drawable] = []  // Error: protocol 'Drawable' has associated type
```

### The Erasure Pattern

```swift
// Step 1: Define the protocol
protocol Drawable {
    func draw(in rect: NSRect)
}

// Step 2: Create a type-erased wrapper
struct AnyDrawable: Drawable {
    private let _draw: (NSRect) -> Void

    init<D: Drawable>(_ drawable: D) {
        _draw = drawable.draw(in:)
    }

    func draw(in rect: NSRect) {
        _draw(rect)
    }
}

// Step 3: Concrete implementations
struct CircleDrawable: Drawable {
    var radius: CGFloat
    var color: NSColor

    func draw(in rect: NSRect) {
        let path = NSBezierPath(ovalIn: NSRect(
            x: rect.midX - radius,
            y: rect.midY - radius,
            width: radius * 2,
            height: radius * 2
        ))
        color.setFill()
        path.fill()
    }
}

struct TextDrawable: Drawable {
    var string: String
    var attributes: [NSAttributedString.Key: Any]

    func draw(in rect: NSRect) {
        (string as NSString).draw(in: rect, withAttributes: attributes)
    }
}

// Step 4: Use heterogeneous collection
final class CanvasView: NSView {
    var drawables: [AnyDrawable] = []

    override func draw(_ dirtyRect: NSRect) {
        super.draw(dirtyRect)
        NSColor.white.setFill()
        dirtyRect.fill()

        for drawable in drawables {
            drawable.draw(in: bounds)
        }
    }
}

// Usage:
let canvas = CanvasView()
canvas.drawables = [
    AnyDrawable(CircleDrawable(radius: 50, color: .systemBlue)),
    AnyDrawable(TextDrawable(string: "Hello", attributes: [.font: NSFont.systemFont(ofSize: 24)])),
]
```

### Erasure with Sendable

When the erasure crosses concurrency boundaries, add `@unchecked Sendable` with proper synchronization:

```swift
protocol Validating: Sendable {
    associatedtype Input
    func validate(_ input: Input) -> ValidationResult
}

struct AnyValidator<Input>: Validating, @unchecked Sendable {
    private let _validate: @Sendable (Input) -> ValidationResult

    init<V: Validating>(_ validator: V) where V.Input == Input {
        _validate = { validator.validate($0) }
    }

    func validate(_ input: Input) -> ValidationResult {
        _validate(input)
    }
}
```

**When to erase vs. use generics:**
- Use generics (`<T: Drawable>`) when the concrete type is known at compile time (single-type collections, direct calls).
- Use type erasure (`AnyDrawable`) when you need heterogeneous storage, protocol-typed properties, or runtime type selection.
- In Swift 6.1+, existential opening reduces the need for erasure in function parameters — `func draw(shape: any Drawable)` now works where you previously needed generics.
