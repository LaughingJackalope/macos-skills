# NSDocument Patterns Reference

Deep reference for NSDocument subclassing, autosave, window controller management, and file I/O on macOS 26+.

## When to Use NSDocument

**Use NSDocument when:**
- Your app edits files on disk
- You need autosave and versioning
- You want undo/redo integrated with the document lifecycle
- You need window restoration across app launches

**Don't use NSDocument when:**
- Your data isn't file-based (preferences, transient state)
- You're building a server-connected app with no local files
- You only need a single window with no file semantics

For non-file apps, use `NSWindowController` directly with a custom model layer.

## NSDocument Subclass

The minimal NSDocument subclass:

```swift
final class NoteDocument: NSDocument {
    var text: String = ""

    // MARK: - Window Controllers

    override func makeWindowControllers() {
        let wc = NSWindowController(window: makeWindow())
        wc.contentViewController = NoteViewController(document: self)
        addWindowController(wc)
    }

    private func makeWindow() -> NSWindow {
        let window = NSWindow(
            contentRect: NSRect(x: 100, y: 100, width: 600, height: 400),
            styleMask: [.titled, .closable, .miniaturizable, .resizable, .fullSizeContentView],
            backing: .buffered,
            defer: false
        )
        window.minSize = NSSize(width: 300, height: 200)
        return window
    }

    // MARK: - Reading

    override func read(from data: Data, ofType typeName: String) throws {
        guard let string = String(data: data, encoding: .utf8) else {
            throw documentError("File is not valid UTF-8 text")
        }
        text = string
    }

    // MARK: - Writing

    override func data(ofType typeName: String) throws -> Data {
        guard let data = text.data(using: .utf8) else {
            throw documentError("Could not encode text as UTF-8")
        }
        return data
    }

    // MARK: - Error Handling

    private func documentError(_ message: String) -> NSError {
        NSError(domain: "com.myapp.document", code: 1,
                userInfo: [NSLocalizedDescriptionKey: message])
    }
}
```

## File Types and UTI Configuration

Register your document types in `Info.plist`:

```xml
<key>CFBundleDocumentTypes</key>
<array>
    <dict>
        <key>CFBundleTypeName</key>
        <string>Plain Text</string>
        <key>CFBundleTypeRole</key>
        <string>Editor</string>
        <key>LSItemContentTypes</key>
        <array>
            <string>public.plain-text</string>
        </array>
        <key>CFBundleTypeExtensions</key>
        <array>
            <string>txt</string>
        </array>
    </dict>
</array>

<!-- For custom file types, declare a UTI: -->
<key>UTExportedTypeDeclarations</key>
<array>
    <dict>
        <key>UTTypeIdentifier</key>
        <string>com.myapp.note</string>
        <key>UTTypeDescription</key>
        <string>MyApp Note</string>
        <key>UTTypeConformsTo</key>
        <array>
            <string>public.data</string>
        </array>
        <key>UTTypeTagSpecification</key>
        <dict>
            <key>public.filename-extension</key>
            <array>
                <string>mnote</string>
            </array>
        </dict>
    </dict>
</array>
```

In your NSDocument subclass, override `writableTypes(for:)` to control which types you can save as:

```swift
override class func writableTypes(for saveOperation: NSDocument.SaveOperationType) -> [String] {
    switch saveOperation {
    case .saveAsOperation:
        return ["com.myapp.note", "public.plain-text"]
    default:
        return ["com.myapp.note"]
    }
}

override func fileNameExtension(forType type: String, saveOperation: NSDocument.SaveOperationType) -> String? {
    switch type {
    case "com.myapp.note": return "mnote"
    case "public.plain-text": return "txt"
    default: return nil
    }
}
```

## Autosave

Autosave is enabled by default on macOS 10.7+. The system saves documents periodically and when the app goes to the background.

**Autosave configuration:**

```swift
override class var autosavesInPlace: Bool { true }  // default on 10.7+
override class var autosavesDrafts: Bool { true }    // save drafts for untitled docs
```

**Custom autosave names:** Override to provide meaningful names for untitled documents:

```override func autosaveDocument(withDelegate delegate: Any, didAutosaveSelector: Selector?, contextInfo: UnsafeMutableRawPointer?) {
    // Set a meaningful default name before autosaving
    if fileURL == nil {
        fileURL = URL(fileURLWithPath: NSTemporaryDirectory())
            .appendingPathComponent("Untitled Note.mnote")
    }
    super.autosaveDocument(withDelegate: delegate, didAutosaveSelector: didAutosaveSelector, contextInfo: contextInfo)
}
```

**Autosave frequency:** The system decides when to autosave. You can trigger it manually:

```swift
// Request an autosave
updateChangeCount(.changeDone)  // marks document as dirty
// The system will autosave on its next cycle

// Or force immediately:
autosaveDocument(withDelegate: nil, didAutosaveSelector: nil, contextInfo: nil)
```

## Window Controller Management

Each document owns its window controllers. For single-window documents, create one controller. For multi-window documents (e.g., main + inspector), create multiple.

```swift
final class ProjectDocument: NSDocument {
    var project: Project!

    override func makeWindowControllers() {
        // Main editor window
        let editorWC = EditorWindowController()
        editorWC.contentViewController = EditorViewController(document: self)
        addWindowController(editorWC)
    }

    // Inspector is created on demand, not in makeWindowControllers
    var inspectorWC: InspectorWindowController?

    func showInspector() {
        if let wc = inspectorWC {
            wc.showWindow(nil)
            return
        }
        let wc = InspectorWindowController()
        wc.contentViewController = InspectorViewController(document: self)
        addWindowController(wc)
        inspectorWC = wc
        wc.showWindow(nil)
    }
}
```

**Window controller from view controller:** Access the document from any view controller in the chain:

```swift
// In a view controller:
guard let document = view.window?.windowController?.document as? MyDocument else { return }
```

## Reading and Writing with Complex Data

For documents with structured data (not just a single string), use `Codable` or `NSKeyedArchiver`:

```swift
struct DocumentData: Codable {
    var title: String
    var items: [Item]
    var metadata: [String: String]
}

final class StructuredDocument: NSDocument {
    var documentData = DocumentData(title: "", items: [], metadata: [:])

    override func data(ofType typeName: String) throws -> Data {
        let encoder = JSONEncoder()
        encoder.outputFormatting = [.prettyPrinted, .sortedKeys]
        return try encoder.encode(documentData)
    }

    override func read(from data: Data, ofType typeName: String) throws {
        let decoder = JSONDecoder()
        documentData = try decoder.decode(DocumentData.self, from: data)
    }
}
```

**For binary data or legacy formats:**

```swift
override func read(from url: URL, ofType typeName: String) throws {
    // Read directly from URL instead of Data
    let fileHandle = try FileHandle(forReadingFrom: url)
    defer { try? fileHandle.close() }
    // Parse binary format...
}

override func write(to url: URL, ofType typeName: String, saveOperation: NSDocument.SaveOperationType) throws {
    // Write directly to URL
    let data = try data(ofType: typeName)
    try data.write(to: url, options: .atomic)  // atomic write for safety
}
```

## Save Panel Customization

Override `prepareSavePanel(_:)` to customize the save panel:

```swift
override func prepareSavePanel(_ savePanel: NSSavePanel) -> Bool {
    super.prepareSavePanel(savePanel)
    savePanel.nameFieldStringValue = documentData.title.isEmpty ? "Untitled" : documentData.title
    savePanel.allowedContentTypes = [UTType(filenameExtension: "mnote")!]
    savePanel.isExtensionHidden = false
    return true
}
```

## Reverting Documents

Implement revert to reload from disk:

```swift
override func revertToContents(of url: URL, ofType typeName: String) throws {
    // Reset state
    documentData = DocumentData(title: "", items: [], metadata: [:])
    // Re-read from disk
    let data = try Data(contentsOf: url)
    try read(from: data, ofType: typeName)
    // Notify view controllers to refresh
    for wc in windowControllers {
        (wc.contentViewController as? DocumentAwareViewController)?.documentDidRevert()
    }
}
```

## Document State and Change Tracking

```swift
// Mark document as dirty (triggers autosave and dirty indicator)
updateChangeCount(.changeDone)

// Mark document as saved
updateChangeCount(.changeCleared)

// Check if document has unsaved changes
if document.isDocumentEdited {
    // Show save prompt
}

// Disable change tracking temporarily (for programmatic updates)
disableUndoRegistration()
documentData.title = "Auto-generated title"
enableUndoRegistration()
```

## Undo/Redo Integration

NSDocument automatically manages an `NSUndoManager`. Use it for all user actions:

```swift
func addItem(_ item: Item, at index: Int) {
    documentData.items.insert(item, at: index)

    undoManager?.registerUndo(withTarget: self) { target in
        target.removeItem(at: index)
    }
    undoManager?.setActionName("Add Item")
}

func removeItem(at index: Int) {
    let item = documentData.items.remove(at: index)

    undoManager?.registerUndo(withTarget: self) { target in
        target.addItem(item, at: index)
    }
    undoManager?.setActionName("Remove Item")
}
```

**Key points:**

- Always call `updateChangeCount(.changeDone)` after mutations that should dirty the document.
- Set the undo action name for meaningful menu items ("Undo Add Item" vs "Undo").
- The undo manager is automatically cleared when the document is saved.
- For KVO-compliant changes, wrap in `willChangeValue`/`didChangeValue` or use the undo manager's grouping.
