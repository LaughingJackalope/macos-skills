---
name: swift-codable-macos
description: "Swift Codable on macOS — Codable, JSONDecoder/JSONEncoder, PropertyListDecoder, CodingKeys, custom encode/decode, NSCoding migration, NSKeyedArchiver bridge, NSUserDefaults Codable storage, NSDocument Codable model serialization. Use when serializing/deserializing data or migrating from NSCoding in macOS apps."
sources: https://developer.apple.com/documentation/foundation/jsondecoder, https://developer.apple.com/documentation/foundation/nskeyedarchiver
researched: 2026-06-06
platform: macOS 26+
---

# Swift Codable on macOS

Swift `Codable` is the unified serialization protocol for macOS 26+. It covers JSON, PropertyList, `NSUserDefaults`, `NSDocument`, and legacy `NSKeyedArchiver` migration. This skill covers the full serialization pipeline — from synthesized conformance on simple structs to custom `NSDocument` bridging and `NSCoding`-to-`Codable` migration paths.

See [references/codable-patterns.md](references/codable-patterns.md) for complete patterns including custom `CodingKey` strategy wrappers, `NSCoding` migration, `NSUserDefaults` storage adapter, `NSDocument` with `Codable` model, and PropertyList preferences encoding.

## Contents

- [Codable Basics](#codable-basics)
- [JSON Encoding/Decoding](#json-encodingdecoding)
- [PropertyList](#propertylist)
- [Custom Coding](#custom-coding)
- [Type-Safe Wrappers](#type-safe-wrappers)
- [NSCoding Migration](#nscoding-migration)
- [NSKeyedArchiver Bridge](#nskeyedarchiver-bridge)
- [NSUserDefaults + Codable](#nsuserdefaults--codable)
- [NSDocument + Codable](#nsdocument--codable)
- [Error Handling](#error-handling)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Codable Basics

`Codable` is a type alias for `Encodable & Decodable`. The compiler synthesizes conformance when all stored properties are themselves `Codable`. Supported natively: `String`, `Int`, `Double`, `Bool`, `Float`, `URL`, `Date`, `Data`, `Array`, `Dictionary`, `Optional`, `UUID`, `Decimal`, and other `Codable` value types.

```swift
struct User: Codable {
    let id: UUID
    let name: String
    let email: String
    var createdAt: Date
    var tags: [String]
}

// Decoding from JSON data
let user = try JSONDecoder().decode(User.self, from: jsonData)

// Encoding to JSON data
let data = try JSONEncoder().encode(user)
```

**Synthesized conformance rules:**
- Every stored property must be `Codable`.
- `let` and `var` stored properties are both encoded; `static` and computed properties are excluded.
- `private(set)` works normally — the encoder/decoder accesses the property, not the setter.
- A struct with no custom requirements gets `init(from:)` and `encode(to:)` for free.

**When the compiler cannot synthesize:** If any property's type does not conform to `Codable`, every enum case has an associated value that isn't `Codable`, or the hierarchy uses class inheritance without manual conformance.

**Enums:** Swift enums without associated values are `Codable` automatically. Enums with associated values require either all values to be `Codable`, or custom `CodingKeys` + `init(from:)`/`encode(to:)`.

```swift
enum Status: Codable {
    case active
    case inactive
    case archived(reason: String)  // requires manual conformance
}
```

## JSON Encoding/Decoding

`JSONDecoder` and `JSONEncoder` are the primary JSON serialization types. Their strategy properties control how dates, data, and key names are transformed.

**Date strategies:**

```swift
let decoder = JSONDecoder()
decoder.dateDecodingStrategy = .iso86601                          // "2026-01-15T10:30:00Z"
decoder.dateDecodingStrategy = .secondsSince1970                  // 1736939400
decoder.dateDecodingStrategy = .millisecondsSince1970             // 1736939400000
decoder.dateDecodingStrategy = .formatted(myDateFormatter)        // custom format
decoder.dateDecodingStrategy = .custom { decoder in               // fully custom
    let container = try decoder.singleValueContainer()
    let timestamp = try container.decode(Double.self)
    return Date(timeIntervalSince1970: timestamp)
}
```

**Data strategies:**

```swift
decoder.dataDecodingStrategy = .base64        // default — expects base64 string
decoder.dataDecodingStrategy = .hex          // no built-in; use .custom
decoder.dataDecodingStrategy = .custom { ... }
```

**Key strategies:**

```swift
let encoder = JSONEncoder()
encoder.keyEncodingStrategy = .useDefaultKeys        // property names as-is
encoder.keyEncodingStrategy = .convertToSnakeCase    // "full_name" vs "fullName"
encoder.keyDecodingStrategy = .convertFromSnakeCase  // "full_name" in JSON → fullName
encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
```

**ISO86601 with fractional seconds:** The built-in `.iso86601` does *not* handle fractional seconds. Use `.iso86601WithFractionalSeconds` is not a built-in option — use `.formatted` with an `ISO8601DateFormatter` that has `.withFractionalSeconds` in its format options:

```swift
let formatter = ISO8601DateFormatter()
formatter.formatOptions = [.withInternetDateTime, .withFractionalSeconds]
decoder.dateDecodingStrategy = .formatted(formatter)
```

## PropertyList

`PropertyListDecoder` and `PropertyListEncoder` handle XML and binary property lists. PropertyList supports a subset of types: `String`, `Int`, `Double`, `Bool`, `Date`, `Data`, `Array`, `Dictionary`. Note that `URL` and `Double.greatestFiniteMagnitude` sentinels are **not** PropertyList-native — use `String` for URLs when targeting PropertyList.

```swift
struct AppConfig: Codable {
    var windowPosition: NSPoint   // NSPoint is NOT Codable — use a wrapper
    var recentFiles: [String]     // Store file paths as strings, not URL
}

struct CodablePoint: Codable {
    var x: CGFloat
    var y: CGFloat

    init(_ point: NSPoint) { self.x = point.x; self.y = point.y }
    var nsPoint: NSPoint { NSPoint(x: x, y: y) }
}

// Encode to PropertyList
let config = AppConfig(windowPosition: NSPoint(x: 100, y: 200), recentFiles: [])
let plistData = try PropertyListEncoder().encode(config)

// Decode from PropertyList
let decoded = try PropertyListDecoder().decode(AppConfig.self, from: plistData)
```

**UserDefaults serialization via PropertyList:** UserDefaults stores PropertyList data. Any `Codable` type that uses only PropertyList-compatible types can be round-tripped via `UserDefaults.standard.set(try? PropertyListEncoder().encode(value), forKey:)`.

## Custom Coding

When synthesized conformance is insufficient, implement `init(from decoder:)` and `encode(to encoder:)` manually.

**Custom CodingKey enum — renaming keys:**

```swift
struct APIUser: Codable {
    let id: UUID
    let fullName: String
    let email: String
    let isActive: Bool

    enum CodingKeys: String, CodingKey {
        case id = "user_id"
        case fullName = "full_name"
        case email
        case isActive = "active"
    }
}
```

**Custom encode/decode with transformations:**

```swift
struct Version: Codable {
    let major: Int
    let minor: Int
    let patch: Int

    init(major: Int, minor: Int, patch: Int) {
        self.major = major; self.minor = minor; self.patch = patch
    }

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        let versionString = try container.decode(String.self)
        let parts = versionString.split(separator: ".").compactMap { Int($0) }
        guard parts.count == 3 else {
            throw DecodingError.dataCorrupted(
                DecodingError.Context(codingPath: decoder.codingPath,
                    debugDescription: "Expected semver string, got \(versionString)")
            )
        }
        self.major = parts[0]; self.minor = parts[1]; self.patch = parts[2]
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        try container.encode("\(major).\(minor).\(patch)")
    }
}
```

**Nested containers:**

```swift
struct PaginatedResponse<Item: Codable>: Codable {
    let items: [Item]
    let totalCount: Int
    let nextPageToken: String?

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        let page = try container.nestedContainer(keyedBy: PageKeys.self,
                                                  forKey: .page)
        items = try page.decode([Item].self, forKey: .items)
        totalCount = try page.decode(Int.self, forKey: .totalCount)
        nextPageToken = try page.decodeIfPresent(String.self, forKey: .nextPageToken)
    }

    enum CodingKeys: CodingKey { case page }
    enum PageKeys: CodingKey { case items, totalCount, nextPageToken }
}
```

**superEncoder / superDecoder — class inheritance:**

```swift
class BaseModel: Codable {
    let id: UUID

    init(id: UUID) { self.id = id }

    required init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        id = try container.decode(UUID.self, forKey: .id)
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(id, forKey: .id)
    }

    enum CodingKeys: CodingKey { case id }
}

class UserModel: BaseModel {
    let name: String
    let email: String

    init(id: UUID, name: String, email: String) {
        self.name = name; self.email = email
        super.init(id: id)
    }

    required init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        name = try container.decode(String.self, forKey: .name)
        email = try container.decode(String.self, forKey: .email)
        try super.init(from: decoder)
    }

    override func encode(to encoder: Encoder) throws {
        var container = encoder.container(keyedBy: CodingKeys.self)
        try container.encode(name, forKey: .name)
        try container.encode(email, forKey: .email)
        try super.encode(to: superEncoder())
    }

    enum CodingKeys: CodingKey { case name, email }
}
```

## Type-Safe Wrappers

Property wrappers can enforce coding conventions across an entire model.

```swift
@propertyWrapper
struct CompactDate: Codable {
    var wrappedValue: Date

    init(wrappedValue: Date) { self.wrappedValue = wrappedValue }

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        let seconds = try container.decode(TimeInterval.self)
        wrappedValue = Date(timeIntervalSince1970: seconds)
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        try container.encode(wrappedValue.timeIntervalSince1970)
    }
}

struct Event: Codable {
    let name: String
    @CompactDate var timestamp: Date

    // Usage:
    // let event = Event(name: "launch", timestamp: .now)
    // All encoding/decoding of timestamp uses seconds-since-1970
}
```

**ISO8601 string wrapper:**

```swift
@propertyWrapper
struct ISO8601String: Codable {
    var wrappedValue: Date

    init(wrappedValue: Date) { self.wrappedValue = wrappedValue }

    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        let raw = try container.decode(String.self)
        guard let date = ISO8601DateFormatter().date(from: raw) else {
            throw DecodingError.dataCorruptedError(in: container,
                debugDescription: "Invalid ISO8601 string: \(raw)")
        }
        wrappedValue = date
    }

    func encode(to encoder: Encoder) throws {
        var container = encoder.singleValueContainer()
        try container.encode(ISO8601DateFormatter().string(from: wrappedValue))
    }
}
```

## NSCoding Migration

`NSCoding` is the ObjC-era protocol. When migrating to `Codable`, you need a bridge layer.

**NSCoding → Codable mapping:**

```swift
final class LegacyDocument: NSObject, NSCoding {
    var title: String
    var content: String

    init(title: String, content: String) {
        self.title = title
        self.content = content
    }

    required init?(coder: NSCoder) {
        guard let title = coder.decodeObject(forKey: "title") as? String,
              let content = coder.decodeObject(forKey: "content") as? String else {
            return nil
        }
        self.title = title
        self.content = content
    }

    func encode(with coder: NSCoder) {
        coder.encode(title, forKey: "title")
        coder.encode(content, forKey: "content")
    }
}

// Codable migration target
struct DocumentModel: Codable {
    let title: String
    let content: String

    init(from legacy: LegacyDocument) {
        title = legacy.title
        content = legacy.content
    }
}

// Migration helper
func migrateLegacyDocument(data: Data) throws -> DocumentModel {
    guard let legacy = try NSKeyedUnarchiver.unarchivedObject(
        ofClass: LegacyDocument.self, from: data) else {
        throw MigrationError.unrecognizedFormat
    }
    return DocumentModel(from: legacy)
}
```

**Encoding compatibility with `NSSecureCoding`:**

```swift
final class SecureModel: NSObject, NSSecureCoding {
    static var supportsSecureCoding: Bool { true }

    var identifier: UUID
    var value: Double

    init(identifier: UUID, value: Double) {
        self.identifier = identifier
        self.value = value
    }

    required init?(coder: NSCoder) {
        guard let id = coder.decodeObject(of: NSString.self, forKey: "id") as? String,
              let uuid = UUID(uuidString: id) else { return nil }
        self.identifier = uuid
        self.value = coder.decodeDouble(forKey: "value")
    }

    func encode(with coder: NSCoder) {
        coder.encode(identifier.uuidString as NSString, forKey: "id")
        coder.encode(value, forKey: "value")
    }
}
```

## NSKeyedArchiver Bridge

Use `NSKeyedArchiver`/`NSKeyedUnarchiver` for reading legacy archived data that you cannot migrate forward. iOS 12+ / macOS 10.14+ require `NSSecureCoding` and the new Archiver API.

**Legacy (deprecated, do not use in new code):**

```swift
// Deprecated:
let data = NSKeyedArchiver.archivedData(withRootObject: obj)
let obj = NSKeyedUnarchiver.unarchiveObject(with: data)
```

**Modern Archiver API:**

```swift
// Secure archiving (iOS 12+, macOS 10.14+)
func archive<T: NSSecureCoding>(_ object: T) throws -> Data {
    let archiver = NSKeyedArchiver(requiringSecureCoding: true)
    archiver.finishEncoding()
    return try NSKeyedArchiver.archivedData(withRootObject: object,
                                              requiringSecureCoding: true)
}

func unarchive<T: NSSecureCoding>(of type: T.Type, from data: Data) throws -> T {
    guard let obj = try NSKeyedUnarchiver.unarchivedObject(ofClass: type, from: data) else {
        throw DecodingError.dataCorrupted(
            DecodingError.Context(codingPath: [],
                debugDescription: "Could not unarchive \(T.self)"))
    }
    return obj
}
```

**Class set unarchiving (mixed type archives):**

```swift
func unarchiveMixed(data: Data) throws -> NSSecureCoding {
    let classes: Set<Any> = [NSString.self, NSArray.self, NSDictionary.self, LegacyModel.self]
    guard let obj = try NSKeyedUnarchiver.unarchivedObject(
        ofClasses: classes, from: data) else {
        throw DecodingError.dataCorrupted(.init(codingPath: [], debugDescription: ""))
    }
    // Cast to the expected concrete type
    return obj
}
```

**Migration path:** Archive → extract properties → construct Codable model → encode as JSON/PropertyList. Write a one-time migration function that reads the old archive at app launch and writes the new format.

## NSUserDefaults + Codable

Store `Codable` structs in `NSUserDefaults` (UserDefaults) via JSON or PropertyList encoding. Never use `NSKeyedArchiver` data in UserDefaults on modern macOS — use `Codable` JSON instead.

```swift
extension UserDefaults {
    func setCodable<T: Codable>(_ value: T, forKey key: String) throws {
        let data = try JSONEncoder().encode(value)
        set(data, forKey: key)
    }

    func codable<T: Codable>(_ type: T.Type, forKey key: String) throws -> T? {
        guard let data = data(forKey: key) else { return nil }
        return try JSONDecoder().decode(type, from: data)
    }
}

// Usage
struct Preferences: Codable {
    var theme: String = "system"
    var fontSize: Double = 14
    var recentDocuments: [String] = []
}

let prefs = Preferences(theme: "dark", fontSize: 16)
try UserDefaults.standard.setCodable(prefs, forKey: "appPreferences")

if let loaded = try UserDefaults.standard.codable(Preferences.self, forKey: "appPreferences") {
    print(load.theme)  // "dark"
}
```

**PropertyList constraints:** If you use `PropertyListEncoder` instead of `JSONEncoder`, remember that `URL`, `Set`, and `Float` are not PropertyList-native. Stick to JSON encoding for UserDefaults unless you need binary format.

**Synchronization:** UserDefaults.synchronize() is deprecated and unnecessary. The system flushes automatically. Do not call it.

## NSDocument + Codable

Bridge `NSDocument`'s `data(ofType:)`/`read(from:ofType:)` to a `Codable` model struct for type-safe document serialization.

```swift
struct DocumentModel: Codable {
    var title: String
    var body: String
    var metadata: DocumentMetadata
}

struct DocumentMetadata: Codable {
    var createdAt: Date
    var lastModified: Date
    var tags: [String]
}

final class NoteDocument: NSDocument {
    var model = DocumentModel(
        title: "", body: "",
        metadata: DocumentMetadata(createdAt: .now, lastModified: .now, tags: [])
    )

    override class var autosavesInPlace: Bool { true }

    override func makeWindowControllers() {
        let wc = NSWindowController(window: NSWindow(
            contentRect: NSRect(x: 200, y: 200, width: 700, height: 500),
            styleMask: [.titled, .closable, .miniaturizable, .resizable],
            backing: .buffered, defer: false))
        wc.contentViewController = NoteViewController(document: self)
        addWindowController(wc)
    }

    override func data(ofType typeName: String) throws -> Data {
        let encoder = JSONEncoder()
        encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
        encoder.dateEncodingStrategy = .iso8601
        return try encoder.encode(model)
    }

    override func read(from data: Data, ofType typeName: String) throws {
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        model = try decoder.decode(DocumentModel.self, from: data)
    }

    override func revertToContents(of url: URL, ofType typeName: String) throws {
        let data = try Data(contentsOf: url)
        try read(from: data, ofType: typeName)
        // Notify view controllers to refresh
        for wc in windowControllers {
            (wc.contentViewController as? NoteViewController)?.refresh()
        }
    }
}
```

**File wrapper support (for complex documents with attachments):**

```swift
override func fileWrapper(ofType typeName: String) throws -> FileWrapper {
    let encoder = JSONEncoder()
    encoder.outputFormatting = [.prettyPrinted]
    let mainData = try encoder.encode(model)

    let mainWrapper = FileWrapper(regularFileWithContents: mainData)

    // Include attachments as a subdirectory
    let attachmentsWrapper = FileWrapper(directoryWithFileWrappers: [:])
    for attachment in attachments {
        attachmentsWrapper.addRegularFile(
            withContents: attachment.data,
            preferredFilename: attachment.fileName)
    }

    let root = FileWrapper(directoryWithFileWrappers: [
        "document.json": mainWrapper,
        "attachments": attachmentsWrapper,
    ])

    return root
}

override func read(from fileWrapper: FileWrapper, ofType typeName: String) throws {
    guard let mainData = fileWrapper.fileWrappers?["document.json"]?.regularFileContents else {
        throw DocumentError.missingFile
    }
    model = try JSONDecoder().decode(DocumentModel.self, from: mainData)
}
```

## Error Handling

`DecodingError` provides structured diagnostics. Always handle it with a typed switch for actionable messages.

```swift
do {
    let result = try JSONDecoder().decode(MyModel.self, from: data)
} catch let DecodingError.keyNotFound(key, context) {
    print("Missing key '\(key.stringValue)' at path: \(context.codingPath.map(\.stringValue).joined(separator: "."))")
} catch let DecodingError.typeMismatch(type, context) {
    print("Type mismatch for \(type) at \(context.codingPath): \(context.debugDescription)")
} catch let DecodingError.valueNotFound(type, context) {
    print("Missing value of type \(type) at \(context.codingPath)")
} catch let DecodingError.dataCorrupted(context) {
    print("Data corrupted: \(context.debugDescription)")
} catch {
    print("Unexpected error: \(error)")
}
```

**`DecodingError` anatomy:**

| Case | Trigger | Recovery hint |
|------|---------|---------------|
| `keyNotFound(_:context:)` | JSON key missing | Provide default value via custom `init(from:)` |
| `valueNotFound(_:context:)` | Present key, no value | Check for null vs missing |
| `typeMismatch(_:context:)` | Value exists, wrong type | Check number-as-string edge cases |
| `dataCorrupted(context:)` | Structural problem (bad JSON) | inspect `context.codingPath` for location |

**CodingPath:** `context.codingPath` is an array of `CodingKey` values. Traverse this to locate the exact position of the error in nested structures. The `debugDescription` includes the full path.

## Constraints & Gotchas

- **Date encoding:** `.iso8601` does NOT support fractional seconds. Use `.formatted(ISO8601DateFormatter())` with `.withFractionalSeconds` for sub-second precision. Mismatched date strategies are the #1 source of silent decode failures.

- **Class inheritance:** `Codable` synthesizes correctly for structs with no work. For class hierarchies, `super.init(from:)` must be called and `superEncoder()`/`superDecoder()` used. Subclasses must re-declare `CodingKeys` in their own enum — the compiler does not inherit them.

- **`Float` / `CGFloat`:** JSONEncoder encodes `Float` and `Double` interchangeably but some JSON parsers may round-trip `Double` to `Float` with precision loss. Be explicit about numeric width.

- **`URL`:** `URL` is `Codable` but expects a valid absolute URL string. Relative URLs or empty strings will fail. Use `URL(string:)`'s failable behavior to pre-validate if needed.

- **`Set`:** `Set` is `Codable` but JSON represents it as an array. Order is not guaranteed on decode.

- **`enum` rawValue enums:** Enums with raw values (`String`, `Int`) are `Codable` using their raw value. Enums with associated values require manual conformance.

- **Legacy archives:** `NSKeyedArchiver` data is not forward-compatible with class hierarchy changes. If you rename a class moving from `NSCoding` to `Codable`, old archives become unreadable. Always keep the old class around for migration or use `NSKeyedUnarchiver.setClass(_:forClassName:)` to remap.

- **PropertyList limitations:** Cannot encode `URL`, `Set`, `float`, `NSRange`, `NSIndexPath`, or other non-PropertyList types. Convert to `String`-based representations.

## Common Mistakes

- **Using `try?` on decode then showing no error:** Always catch `DecodingError` specifically and log the `codingPath` and `debugDescription`. `try?` discards the most useful diagnostic information in the entire Foundation framework.

- **Ignoring `dateDecodingStrategy`:** The default `.deferredToDate` works for `Date(timeIntervalSinceReferenceDate:)` — not for ISO8601 strings or Unix timestamps. Mismatch = silent nil or crash.

- **Encoding a class hierarchy without `superEncoder()`:** Subclasses that call `super.encode(to: encoder)` instead of `super.encode(to: superEncoder())` will encode superclass properties on the same key container, potentially colliding with subclass keys.

- **Using `PropertyListEncoder` with `URL` types:** `URL` is not a PropertyList type. It will crash at runtime. Convert to `String` first.

- **Storing `NSKeyedArchiver` data in UserDefaults:** This is deprecated and problematic. Use `Codable` with `JSONEncoder` instead.

- **Not calling `url.startAccessingSecurityScopedResource()`:** When round-tripping file URLs via Codable (e.g., as bookmark data), you must resolve the bookmark and call `startAccessingSecurityScopedResource()` before use.

- **Boolean-as-number in JSON:** `0` and `1` in some JSON APIs represent booleans. `JSONDecoder` rejects these — use a property wrapper or custom decode to handle `Int`-as-`Bool`. Wrap with: `try Bool(truncating: NSNumber(value: container.decode(Int.self)))`.

- **Assuming `JSONDecoder` is thread-safe across all shared state:** `JSONDecoder` instances are safe to share. Encoder/decoder stateless configuration (date strategy, key strategy) is set at initialization and not mutated during encode/decode.

## Review Checklist

- [ ] All `Date` fields have an explicit `dateDecodingStrategy` set (not relying on default)
- [ ] `CodingKey` enums use `String` raw values matching the JSON/PropertyList keys exactly
- [ ] Class hierarchies use `superEncoder()`/`superDecoder()` not shared containers
- [ ] `DecodingError` is caught with typed cases for actionable diagnostics in user-facing code
- [ ] Legacy `NSCoding` migration path archives old data, never overwrites
- [ ] UserDefaults + Codable uses JSON encoding (PropertyList has type limitations)
- [ ] `NSDocument` model is a separate struct — document controller logic stays out of the model
- [ ] File wrapper-based documents use a subdirectory or manifest structure
- [ ] `try?` is not silently discarding `DecodingError` details
- [ ] `URL` values are validated before decode (failable init)
- [ ] Tests cover at least: valid decode, missing key, type mismatch, empty data, and corrupted data

## References

- [JSONEncoder | Apple Developer Documentation](https://developer.apple.com/documentation/foundation/jsonencoder)
- [JSONDecoder | Apple Developer Documentation](https://developer.apple.com/documentation/foundation/jsondecoder)
- [PropertyListEncoder | Apple Developer Documentation](https://developer.apple.com/documentation/foundation/propertylistencoder)
- [Codable | Swift Standard Library](https://developer.apple.com/documentation/swift/codable)
- [NSKeyedArchiver | Apple Developer Documentation](https://developer.apple.com/documentation/foundation/nskeyedarchiver)
- [NSSecureCoding | Apple Developer Documentation](https://developer.apple.com/documentation/foundation/nssecurecoding)

## Research Addendum — 2026 Updates

### NSCoding Migration & NSKeyedArchiver

- **`NSKeyedUnarchiver.setClass(_:forClassName:)`** — Critical for migrations where you rename a class. Remaps the old class name in archives to the new class so old data remains readable.
- **`decodeObject(of:forKey:)` over `decodeObject(forKey:)`** — Always use the typed variant. It returns nil on type mismatch instead of a dangerous cast.
- **Modern archiver requires `requiringSecureCoding: true`** — Non-secure archiving is deprecated and will eventually be removed. All new code must use the secure path.
- **Class renaming strategy**: Keep the old `NSCoding` subclass in your project (dead code, archived-only) for one release cycle so existing user data can migrate. Then remove in a follow-up release.

### NSUserDefaults + Codable Storage

- **PropertyList-native storage**: Libraries like `KeyValueCoder` and `SwiftyUserDefaults` let you store Codable types directly as plist-native values (dictionaries, arrays, URL, Duration) instead of opaque Data blobs. This makes UserDefaults content human-readable in `defaults read` commands.
- **JSON encoding is still the default for UserDefaults Codable** — it handles all types including `URL`, `Set`, `Float`, `Duration`. PropertyList cannot encode these.
- **`SwiftyUserDefaults` approach** — Define typed `Key<T>` keys once, then access with type safety. Eliminates stringly-typed key errors.
- **Thread safety**: `UserDefaults` is thread-safe for reading/writing individual values. Avoid reading many values across threads without synchronization — snapshot consistency is not guaranteed.

### PropertyListEncoder/Decoder for macOS Preferences

- **`PropertyListEncoder.outputFormat`** — Default is `.binary`. Use `.xml` for human-readable/debuggable plists.
- **`encode(_:configuration:)` overload** — For types conforming to `EncodableWithConfiguration`, pass type-specific encoding configuration (e.g., a custom date format or unit).
- **Error handling for non-floating-point values** — `EncodingError.invalidValue` is thrown for non-conforming floating-point values (NaN, infinity) when using `.throw` strategy.
- **File-based preferences pattern**: Use `PropertyListEncoder` to write a preferences plist to `~/Library/Preferences/` or the app's container. Use `NSFileCoordinator` for coordinated reads/writes if the file might be modified externally (e.g., by `defaults` CLI).
- **OpenStep format blocked** — Writing OpenStep-format plists throws a `CocoaError`. Only XML and binary are writable.
