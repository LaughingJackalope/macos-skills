# Codable Patterns Reference

Deep reference for Codable strategies on macOS 26+: custom key strategies, NSCoding migration, NSUserDefaults storage, NSDocument bridging, and PropertyList preferences encoding.

## Complete Codable with Custom Key Strategy

When an API uses snake_case keys but your Swift model uses camelCase, `convertToSnakeCase` / `convertFromSnakeCase` handle the mapping automatically. For per-field overrides, supplement with a `CodingKeys` enum.

```swift
struct ServerResponse: Codable {
    let userId: UUID
    let fullName: String
    let emailAddress: String
    let isActive: Bool
    let profileImageUrl: URL?
}

// Automatic snake_case ↔ camelCase conversion
let decoder = JSONDecoder()
decoder.keyDecodingStrategy = .convertFromSnakeCase
// JSON keys "user_id", "full_name", "email_address", "is_active", "profile_image_url"
// map to: userId, fullName, emailAddress, isActive, profileImageUrl

let encoder = JSONEncoder()
encoder.keyEncodingStrategy = .convertToSnakeCase
// Reverse: camelCase → snake_case
```

When automatic conversion is insufficient (e.g., keys that are Swift reserved words, or API keys that don't follow a pattern), use an explicit `CodingKeys` enum. Mix both approaches: set the automatic strategy, then override specific keys in the enum.

```swift
struct ConfigEntry: Codable {
    let defaultValue: String
    let computedValue: String
    let `class`: SwiftParsedClass  // reserved word needs backticks

    enum CodingKeys: String, CodingKey {
        case defaultValue = "default"    // "default" is reserved in JSON config formats
        case computedValue = "computed"
        case `class` = "class"          // explicit mapping for reserved word
    }
}
```

**Custom key encoding strategy via `KeyEncodingStrategy` closure (macOS 12+):** For fully dynamic key renaming (e.g., prefix stripping):

```swift
let encoder = JSONEncoder()
encoder.keyEncodingStrategy = .custom { codingPath in
    let key = codingPath.last!.stringValue
    // Prefix all keys for namespace isolation
    return AnyKey(stringValue: "myapp_\(key)")!
}
```

## NSCoding Migration Pattern for Existing Classes

When migrating an `NSCoding`-based class to `Codable`, follow a phased approach: read old archives, transfer to new struct, write new format.

### Phase 1: Keep the Old Class for Reading Only

```swift
// Do NOT delete this class — it's still needed to unarchive old data
final class LegacyPreferences: NSObject, NSCoding {
    var fontSize: Double
    var themeName: String
    var windowFrame: String  // NSRect as serialized string (legacy)

    init(fontSize: Double = 14, themeName: String = "default",
         windowFrame: String = "") {
        self.fontSize = fontSize
        self.themeName = themeName
        self.windowFrame = windowFrame
    }

    required init?(coder: NSCoder) {
        fontSize = coder.decodeDouble(forKey: "fontSize")
        themeName = coder.decodeObject(forKey: "themeName") as? String ?? ""
        windowFrame = coder.decodeObject(forKey: "windowFrame") as? String ?? ""
    }

    func encode(with coder: NSCoder) {
        coder.encode(fontSize, forKey: "fontSize")
        coder.encode(themeName, forKey: "themeName")
        coder.encode(windowFrame, forKey: "windowFrame")
    }
}
```

### Phase 2: Define the Codable Model

```swift
struct PreferencesV2: Codable, Equatable {
    var fontSize: Double = 14.0
    var theme: Theme = .default
    var windowPosition: WindowPosition?

    enum Theme: String, Codable {
        case `default`, dark, light, system
    }

    struct WindowPosition: Codable, Equatable {
        var x: Double
        var y: Double
        var width: Double
        var height: Double

        init(_ rect: NSRect) {
            x = rect.origin.x; y = rect.origin.y
            width = rect.size.width; height = rect.size.height
        }

        var nsRect: NSRect {
            NSRect(x: x, y: y, width: width, height: height)
        }
    }
}
```

### Phase 3: Migration Function

```swift
func loadPreferences() -> PreferencesV2 {
    let oldKey = "preferences"   // NSCoding archive key
    let newKey = "preferences_v2" // Codable JSON key

    // Try new format first
    if let data = UserDefaults.standard.data(forKey: newKey),
       let prefs = try? JSONDecoder().decode(PreferencesV2.self, from: data) {
        return prefs
    }

    // Fall back to NSCoding archive migration
    if let oldData = UserDefaults.standard.data(forKey: oldKey) {
        if let legacy = try? NSKeyedUnarchiver.unarchivedObject(
            ofClass: LegacyPreferences.self, from: oldData) {

            // Transform legacy data to new format
            let position = parseWindowFrame(legacy.windowFrame)
            let theme = PreferencesV2.Theme(rawValue: legacy.themeName) ?? .default

            let migrated = PreferencesV2(
                fontSize: legacy.fontSize,
                theme: theme,
                windowPosition: position.map(PreferencesV2.WindowPosition.init)
            )

            // Save in new format and remove old data
            if let newData = try? JSONEncoder().encode(migrated) {
                UserDefaults.standard.set(newData, forKey: newKey)
                UserDefaults.standard.removeObject(forKey: oldKey)
            }

            return migrated
        }
    }

    // Return defaults for fresh installs
    return PreferencesV2()
}

private func parseWindowFrame(_ string: String) -> NSRect? {
    // Parse legacy NSRect string: "{{x, y}, {w, h}}"
    let scanner = Scanner(string: string)
    var x = 0.0, y = 0.0, w = 0.0, h = 0.0
    guard scanner.scanString("{{") != nil,
          scanner.scanDouble(&x),
          scanner.scanString(",") != nil,
          scanner.scanDouble(&y),
          scanner.scanString("},{") != nil,
          scanner.scanDouble(&w),
          scanner.scanDouble(&h),
          scanner.scanString("}}") != nil else { return nil }
    return NSRect(x: x, y: y, width: w, height: h)
}
```

## NSUserDefaults Codable Storage Wrapper

A reusable property wrapper that stores any `Codable` value in `UserDefaults`:

```swift
@propertyWrapper
struct UserDefault<T: Codable> {
    let key: String
    let defaultValue: T

    var wrappedValue: T {
        get {
            guard let data = UserDefaults.standard.data(forKey: key) else {
                return defaultValue
            }
            do {
                return try JSONDecoder().decode(T.self, from: data)
            } catch {
                // Log the error for debugging
                print("[UserDefault] Failed to decode \(T.self) for key '\(key)': \(error.localizedDescription)")
                return defaultValue
            }
        }
        set {
            do {
                let data = try JSONEncoder().encode(newValue)
                UserDefaults.standard.set(data, forKey: key)
            } catch {
                print("[UserDefault] Failed to encode \(T.self) for key '\(key)': \(error.localizedDescription)")
            }
        }
    }

    init(wrappedValue: T, key: String) {
        self.key = key
        self.defaultValue = wrappedValue
        self._placeholder = wrappedValue      // keep compiler happy
    }

    private var _placeholder: T

    init(wrappedValue: T, key: String, defaultValue: T) {
        self.key = key
        self.defaultValue = defaultValue
        self._placeholder = wrappedValue
    }

    static subscript<EnclosingSelf: ObservableObject>(
        _enclosingInstance observed: EnclosingSelf,
        wrapped wrappedKeyPath: ReferenceWritableKeyPath<EnclosingSelf, T>,
        storage storageKeyPath: ReferenceWritableKeyPath<EnclosingSelf, Self>
    ) -> T {
        get {
            observed[keyPath: storageKeyPath].wrappedValue
        }
        set {
            observed[keyPath: storageKeyPath].wrappedValue = newValue
            (observed as? ObservableObjectWillChangePublisher)?
                .objectWillChange.send()
        }
    }
}

// Usage with @Observable (Swift 5.9+ / macOS 14+):
@Observable
class AppSettings {
    @UserDefault(key: "preferences.theme")
    var theme: String = "system"

    @UserDefault(key: "preferences.fontSize")
    var fontSize: Double = 14.0

    @UserDefault(key: "preferences.recentFiles")
    var recentFiles: [String] = []
}
```

**For ObservableObject (pre-iOS 17 / macOS 14):** ObjectiveC KVO observation of `UserDefaults.didChangeNotification` works, or use `Defaults` library pattern with Combine publisher.

## NSDocument with Codable Model (Read/Write)

The complete pattern for an `NSDocument` backed by a `Codable` model, supporting save, autosave, and file coordination.

```swift
// The pure model — no AppKit dependency
struct ProjectModel: Codable, Equatable {
    var name: String
    var items: [ProjectItem]
    var created: Date
    var lastSaved: Date

    struct ProjectItem: Codable, Equatable {
        let id: UUID
        var title: String
        var isCompleted: Bool
        var notes: String
    }
}

final class ProjectDocument: NSDocument {
    var model: ProjectModel

    override convenience init() {
        let model = ProjectModel(
            name: "Untitled", items: [],
            created: .now, lastSaved: .now
        )
        self.init(model: model)
    }

    init(model: ProjectModel) {
        self.model = model
        super.init()
    }

    // MARK: - Document Type Configuration

    override class var autosavesInPlace: Bool { true }

    override class func readableTypes() -> [String] {
        ["com.myapp.project"]
    }

    override class func writableTypes(for saveOperation: NSDocument.SaveOperationType) -> [String] {
        ["com.myapp.project", "public.json"]
    }

    override class var nativeType: String { "com.myapp.project" }

    override func fileNameExtension(forType type: String,
                                      saveOperation: NSDocument.SaveOperationType) -> String? {
        switch type {
        case "com.myapp.project": return "myproj"
        case "public.json": return "json"
        default: return nil
        }
    }

    // MARK: - Window Controllers

    override func makeWindowControllers() {
        let wc = ProjectWindowController()
        wc.contentViewController = ProjectViewController(document: self)
        addWindowController(wc)
    }

    // MARK: - Writing

    override func data(ofType typeName: String) throws -> Data {
        var writeModel = model
        writeModel.lastSaved = .now

        let encoder = JSONEncoder()
        encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
        encoder.dateEncodingStrategy = .iso8601
        return try encoder.encode(writeModel)
    }

    // MARK: - Reading

    override func read(from data: Data, ofType typeName: String) throws {
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        model = try decoder.decode(ProjectModel.self, from: data)
    }

    // MARK: - Version Backwards Compatibility

    // Override to support reading older model formats
    override class func read(from data: Data, ofType typeName: String) throws -> Self {
        // Attempt current format
        let decoder = JSONDecoder()
        decoder.dateDecodingStrategy = .iso8601
        if let model = try? decoder.decode(ProjectModel.self, from: data) {
            return self.init(model: model)
        }

        // Fallback: try legacy format (v1 schema with different keys)
        struct LegacyV1Model: Codable {
            var projectName: String
            var entries: [LegacyEntry]
        }
        struct LegacyEntry: Codable {
            var name: String
            var done: Bool
        }
        if let legacy = try? decoder.decode(LegacyV1Model.self, from: data) {
            let migrated = ProjectModel(
                name: legacy.projectName,
                items: legacy.entries.map { entry in
                    ProjectModel.ProjectItem(
                        id: UUID(), title: entry.name,
                        isCompleted: entry.done, notes: ""
                    )
                },
                created: .now, lastSaved: .now
            )
            return self.init(model: migrated)
        }

        throw DocumentError.incompatibleFormat
    }

    // MARK: - Revert

    override func revertToContents(of url: URL, ofType typeName: String) throws {
        let fileData = try Data(contentsOf: url)
        try read(from: fileData, ofType: typeName)
        for wc in windowControllers {
            (wc.contentViewController as? ProjectViewController)?.reloadData()
        }
    }

    // MARK: - Change Tracking

    func modelDidChange() {
        updateChangeCount(.changeDone)
    }
}

enum DocumentError: LocalizedError {
    case incompatibleFormat
    case missingFile

    var errorDescription: String? {
        switch self {
        case .incompatibleFormat:
            return "The document format is not recognized."
        case .missingFile:
            return "A required file is missing."
        }
    }
}
```

## PropertyList Encoding for Preferences

For simple preferences that map directly to PropertyList types, `PropertyListEncoder` produces compact binary output accessible via `defaults(1)` command-line tool.

```swift
struct AppPreferences: Codable {
    // PropertyList-compatible types only
    var windowWidth: Double = 800
    var windowHeight: Double = 600
    var selectedTabIndex: Int = 0
    var sidebarVisible: Bool = true
    var recentFilePaths: [String] = []       // File paths as strings, not URL
    var accentColorName: String = "blue"
    var lastOpenedDirectory: String? = nil   // Path string, not URL

    // NOT PropertyList compatible:
    // var someURL: URL          → use String path
    // var color: NSColor        → use String name or RGB components
    // var someSet: Set<String>  → use Array
}

extension AppPreferences {
    private static let storageKey = "appPreferences"

    static func load() -> AppPreferences {
        guard let data = UserDefaults.standard.data(forKey: storageKey),
              let prefs = try? PropertyListDecoder().decode(Self.self, from: data) else {
            return AppPreferences()
        }
        return prefs
    }

    func save() {
        if let data = try? PropertyListEncoder().encode(self) {
            UserDefaults.standard.set(data, forKey: Self.storageKey)
        }
    }
}

// To make NSColor PropertyList-compatible, encode as components:
struct CodableColor: Codable {
    var red: Double
    var green: Double
    var blue: Double
    var alpha: Double

    init(color: NSColor) {
        let rgb = color.usingColorSpace(.sRGB) ?? .clear
        red = rgb.redComponent
        green = rgb.greenComponent
        blue = rgb.blueComponent
        alpha = rgb.alphaComponent
    }

    var nsColor: NSColor {
        NSColor(red: red, green: green, blue: blue, alpha: alpha)
    }
}
```

**Binary PropertyList format:** Use `PropertyListSerialization` directly for control over format:

```swift
let data = try PropertyListSerialization.data(
    fromPropertyList: dictionary,
    format: .binary,        // .xml for human-readable, .binary for compact
    options: 0
)
```
