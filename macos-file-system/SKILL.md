---
name: macos-file-system
description: "macOS file system access — sandbox entitlements, security-scoped bookmarks (startAccessingSecurityScopedResource), NSOpenPanel/NSSavePanel, NSFileCoordinator/NSFilePresenter, NSMetadataQuery (Spotlight from code), file promises (NSFilePromiseProvider), UTType document registration. Use when accessing files outside the sandbox, coordinating file access, or integrating with Spotlight."
sources: https://developer.apple.com/documentation/appkit/nsfilecoordinator, https://developer.apple.com/documentation/uniformtypeidentifiers
researched: 2026-06-06
platform: macOS 26+
---

# macOS File System Access

File system patterns for macOS 26+ with Swift 6.3. Covers sandbox entitlements for file access, security-scoped bookmarks for persistent access, NSOpenPanel/NSSavePanel and SwiftUI fileImporter/fileExporter, NSFileCoordinator/NSFilePresenter for coordinated reads and writes, NSMetadataQuery for Spotlight integration, NSFilePromiseProvider for drag-to-Finder, UTType registration, and NSFileProviderExtension basics.

See [references/filesystem-patterns.md](references/filesystem-patterns.md) for complete code patterns including the full security-scoped bookmark lifecycle, NSFileCoordinator with NSFilePresenter implementation, NSMetadataQuery for custom Spotlight integration, SwiftUI fileImporter/fileExporter patterns, and drag-to-Finder with NSFilePromiseProvider.

## Contents

- [Sandbox Entitlements](#sandbox-entitlements)
- [Security-Scoped Bookmarks](#security-scoped-bookmarks)
- [NSOpenPanel & NSSavePanel](#nsopenpanel--nssavepanel)
- [NSFileCoordinator & NSFilePresenter](#nsfilecoordinator--nsfilepresenter)
- [NSMetadataQuery](#nsmetadataquery)
- [File Promises](#file-promises)
- [UTType Registration](#uttype-registration)
- [NSFileProviderExtension](#nsfileproviderextension)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

Scope boundary: This skill owns file access, coordination, and Spotlight querying. Code signing, hardened runtime, and sandbox enablement are covered in `macos-security-entitlements`. NSPasteboard and drag & drop between views are covered in `appkit-foundations`.

## Sandbox Entitlements

The App Sandbox restricts file access. With sandbox enabled (`com.apple.security.app-sandbox`), every file operation requires a matching entitlement.

**Core file access entitlements:**

| Entitlement | Value | Use |
|---|---|---|
| `com.apple.security.files.user-selected.read-write` | `true` | Read/write files the user picks in an NSOpenPanel/NSSavePanel |
| `com.apple.security.files.user-selected.read-only` | `true` | Read-only access to user-selected files |
| `com.apple.security.files.downloads.read-write` | `true` | Access the user's Downloads folder without a panel |
| `com.apple.security.files.bookmarks.app-scope` | `true` | Persist security-scoped bookmarks between launches |
| `com.apple.security.temporary-exception.files.home-relative-path.read-write` | `[String]` | Specific home-relative paths — weakens sandbox; avoid if possible |

**Choosing the right entitlement:** Use `user-selected.read-write` for documents the user actively opens. Use `downloads.read-write` only for apps that genuinely operate on Downloads (download managers, media converters). Use `bookmarks.app-scope` whenever you need to remember file access across launches — without it, bookmarks resolve to nothing.

Temporary exception entitlements (`com.apple.security.temporary-exception.files.home-relative-path.read-write`) should be a last resort. They create a literal path whitelist that may break across macOS versions or user home directory changes. Prefer security-scoped bookmarks.

**Typical entitlement set for a document editor:**

```xml
<dict>
    <key>com.apple.security.app-sandbox</key>
    <true/>
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
    <key>com.apple.security.files.bookmarks.app-scope</key>
    <true/>
</dict>
```

## Security-Scoped Bookmarks

Security-scoped bookmarks let your app re-access files and directories the user previously opened, across app launches. Without bookmarks, sandboxed access is lost when the app quits.

**Full lifecycle — create, store, resolve, access:**

```swift
import Foundation

final class BookmarkStore {
    private let defaults = UserDefaults.standard

    // 1. CREATE: Called after the user picks a file/folder in NSOpenPanel
    func createBookmark(for url: URL) -> Data? {
        guard url.startAccessingSecurityScopedResource() else { return nil }
        defer { url.stopAccessingSecurityScopedResource() }
        do {
            let data = try url.bookmarkData(
                options: .withSecurityScope,
                includingResourceValuesForKeys: nil,
                relativeTo: nil
            )
            return data
        } catch {
            print("Bookmark creation failed: \(error)")
            return nil
        }
    }

    // 2. STORE: Persist the bookmark data
    func storeBookmark(for key: String, url: URL) {
        if let data = createBookmark(for: url) {
            defaults.set(data, forKey: key)
        }
    }

    // 3. RESOLVE: Retrieve a URL from stored bookmark data
    func resolveBookmark(for key: String) -> URL? {
        guard let data = defaults.data(forKey: key) else { return nil }
        var isStale = false
        do {
            let url = try URL(
                resolvingBookmarkData: data,
                options: .withSecurityScope,
                relativeTo: nil,
                bookmarkDataIsStale: &isStale
            )
            if isStale {
                // Re-create the bookmark with a fresh snapshot
                _ = storeBookmark(for: key, url: url)
            }
            return url
        } catch {
            print("Bookmark resolution failed: \(error)")
            return nil
        }
    }

    // 4. ACCESS: Wrap every file operation with these calls
    func withResolvedAccess<T>(for key: String, body: (URL) throws -> T) rethrows -> T? {
        guard let url = resolveBookmark(for: key),
              url.startAccessingSecurityScopedResource() else { return nil }
        defer { url.stopAccessingSecurityScopedResource() }
        return try body(url)
    }
}
```

**Storing in Keychain instead of UserDefaults:** For sensitive paths, store bookmark data in the Keychain. Use the `kSecClassGenericPassword` service with a meaningful service name. The access group should match your app's Keychain Access Group entitlement.

- Check `isStale` on resolution and re-create the bookmark when stale. Spotlight re-indexing or Time Machine restores can also trigger staleness.
- **Bookmark entitlements:** Enable `com.apple.security.files.bookmarks.app-scope` for general access, `com.apple.security.files.bookmarks.document-scope` for document-specific bookmarks (renamed from collection-scope in macOS 10.14+). Both can be active simultaneously.
- **Resource reference counting:** `startAccessingSecurityScopedResource()` is per-process reference-counted. Exceeding limits can deny access. Use a short-lived `FolderHandle` object that holds scope open while a tab is visible.
- **External drive handling:** Bookmarks fail when volumes are unmount. Implement "reconnect" UI to let users re-grant access gracefully.

## NSOpenPanel & NSSavePanel

### AppKit: NSOpenPanel / NSSavePanel

For AppKit apps, use the panel classes directly with a delegate for custom validation:

```swift
import AppKit

final class DocumentOpener: NSObject, NSOpenSavePanelDelegate {
    func showOpenPanel() {
        let panel = NSOpenPanel()
        panel.title = "Open Document"
        panel.allowedContentTypes = [.plainText, .pdf, .jpeg]
        panel.allowsMultipleSelection = true
        panel.canChooseDirectories = false
        panel.canChooseFiles = true
        panel.delegate = self
        panel.isExtensionHidden = false

        panel.begin { response in
            guard response == .OK else { return }
            for url in panel.urls {
                self.loadDocument(at: url)
            }
        }
    }

    func showSavePanel() {
        let panel = NSSavePanel()
        panel.title = "Export Document"
        panel.nameFieldStringValue = "Untitled.txt"
        panel.allowedContentTypes = [.plainText]
        panel.isExtensionHidden = false

        panel.begin { response in
            guard response == .OK, let url = panel.url else { return }
            self.saveDocument(to: url)
        }
    }

    // NSOpenSavePanelDelegate — validate the user's selection
    func panel(_ sender: Any, shouldEnable url: URL) -> Bool {
        // Custom logic: enable only files under 10 MB
        do {
            let values = try url.resourceValues(forKeys: [.fileSizeKey])
            return (values.fileSize ?? 0) < 10_000_000
        } catch {
            return true
        }
    }

    private func loadDocument(at url: URL) {
        url.startAccessingSecurityScopedResource()
        defer { url.stopAccessingSecurityScopedResource() }
        // Read the file…
    }

    private func saveDocument(to url: URL) {
        // Write to url — NSSavePanel already grants write access
        try? "Hello, world!".write(to: url, atomically: true, encoding: .utf8)
    }
}
```

### SwiftUI: .fileImporter / .fileExporter

SwiftUI's modifiers provide file dialogs without direct panel access:

```swift
// Importer
struct ImportButton: View {
    @State private var isImporterPresented = false

    var body: some View {
        Button("Import…") { isImporterPresented = true }
            .fileImporter(
                isPresented: $isImporterPresented,
                allowedContentTypes: [.plainText, .pdf],
                allowsMultipleSelection: true
            ) { result in
                switch result {
                case .success(let urls):
                    for url in urls {
                        if url.startAccessingSecurityScopedResource() {
                            defer { url.stopAccessingSecurityScopedResource() }
                            loadFile(at: url)
                        }
                    }
                case .failure(let error):
                    print("Import failed: \(error)")
                }
            }
    }

    private func loadFile(at url: URL) { /* … */ }
}

// Exporter
struct ExportButton: View {
    @State private var isExporterPresented = false
    let document: MyDocument

    var body: some View {
        Button("Export…") { isExporterPresented = true }
            .fileExporter(
                isPresented: $isExporterPresented,
                document: document,
                contentType: .plainText,
                defaultFilename: "Untitled.txt"
            ) { result in
                switch result {
                case .success(let url):
                    print("Exported to: \(url)")
                case .failure(let error):
                    print("Export failed: \(error)")
                }
            }
    }
}

// The document must conform to FileDocument (AppKit) or ReferenceFileDocument
struct MyDocument: FileDocument {
    static var readableContentTypes: [UTType] { [.plainText] }
    static var writableContentTypes: [UTType] { [.plainText] }

    var text: String

    init(text: String = "") { self.text = text }

    init(configuration: ReadConfiguration) throws {
        guard let data = configuration.file.regularFileContents,
              let string = String(data: data, encoding: .utf8)
        else { throw CocoaError(.fileReadCorruptFile) }
        text = string
    }

    func fileWrapper(configuration: WriteConfiguration) throws -> FileWrapper {
        let data = text.data(using: .utf8) ?? Data()
        return FileWrapper(regularFileWithContents: data)
    }
}
```

## NSFileCoordinator & NSFilePresenter

NSFileCoordinator serializes reads and writes to the same file across processes (e.g., your app and a File Provider extension). NSFilePresenter lets your app receive notifications when another process modifies a file you're presenting.

**Coordinated reading and writing:**

```swift
func coordinatedRead(from url: URL) -> Data? {
    var result: Data?
    var coordinatorError: NSError?
    var readError: NSError?

    let coordinator = NSFileCoordinator()
    coordinator.coordinate(readingItemAt: url, options: .withoutChanges, error: &coordinatorError) { coordinatedURL in
        do {
            result = try Data(contentsOf: coordinatedURL)
        } catch {
            readError = error as NSError
        }
    }

    if let error = coordinatorError ?? readError {
        print("Coordinated read failed: \(error)")
    }
    return result
}

func coordinatedWrite(to url: URL, data: Data) {
    var coordinatorError: NSError?
    var writeError: NSError?

    let coordinator = NSFileCoordinator()
    coordinator.coordinate(writingItemAt: url, options: .forReplacing, error: &coordinatorError) { coordinatedURL in
        do {
            try data.write(to: coordinatedURL)
        } catch {
            writeError = error as NSError
        }
    }

    if let error = coordinatorError ?? writeError {
        print("Coordinated write failed: \(error)")
    }
}
```

**Implementing NSFilePresenter:**

```swift
final class DocumentPresenter: NSObject, NSFilePresenter {
    let presentedItemURL: URL?
    let presentedItemOperationQueue: OperationQueue = .main

    init(documentURL: URL) {
        presentedItemURL = documentURL
        super.init()
        NSFileCoordinator.addFilePresenter(self)
    }

    deinit {
        NSFileCoordinator.removeFilePresenter(self)
    }

    // Another process is about to change the file
    func presentedItemDidChange() {
        DispatchQueue.main.async {
            self.refreshDocument()
        }
    }

    // Another process did move/rename the file
    func presentedItemDidMove(to newURL: URL) {
        DispatchQueue.main.async {
            self.updateDocumentURL(newURL)
        }
    }

    func presentedSubitem(at url: URL, didGain gainVersion: NSFileVersion) {
        handleNewChild(at: url)
    }

    private func refreshDocument() { /* reload UI */ }
    private func updateDocumentURL(_ url: URL) { /* update state */ }
    private func handleNewChild(at url: URL) { /* handle new child file */ }
}
```

**Add presenter when the document opens, remove on close. Never add a presenter for a URL you don't own.** Adding too many presenters degrades system performance because every file system change triggers notifications to all registered presenters.

## NSMetadataQuery

NSMetadataQuery exposes Spotlight search from within your app. Use it to find files by metadata attributes (creation date, content type, Finder tags, etc.).

**Basic query setup:**

```swift
import CoreServices

final class SpotlightSearch: NSObject, ObservableObject {
    @Published var results: [NSMetadataItem] = []

    private let query = NSMetadataQuery()

    override init() {
        super.init()
        query.delegate = self
        query.searchScopes = [NSMetadataQueryUserHomeScope]
        query.operationQueue = .main
    }

    func search(for searchText: String, contentType: UTType) {
        let predicate = NSMetadataItem.value(
        forAttribute: NSMetadataItemKeywordsAttribute
    ) as [String]
        // Simpler: use NSPredicate directly
        let typePredicate = NSPredicate(format: "kMDItemContentTypeTree == %@", contentType.identifier)
        let textPredicate = NSPredicate(format: "kMDItemTextContent == %@", searchText)
        query.predicate = NSCompoundPredicate(andPredicateWithSubpredicates: [typePredicate, textPredicate])
        query.start()
    }

    func stop() {
        query.stop()
        query.disableUpdates()
    }
}

extension SpotlightSearch: NSMetadataQueryDelegate {
    func metadataQuery(_ query: NSMetadataQuery, replacementObjectForResultObject result: NSMetadataItem) -> Any {
        return result
    }
}
```

**Querying with NSPredicate — common metadata attributes:**

| Attribute | Description |
|---|---|
| `kMDItemContentType` | UTType identifier (e.g., `public.plain-text`) |
| `kMDItemContentTypeTree` | Type hierarchy — use for "any image" queries |
| `kMDItemTextContent` | Full-text index of the file's content |
| `kMDItemKeywords` | User-assigned keywords |
| `kMDItemDisplayName` | File name as shown in Finder |
| `kMDItemNumberOfPages` | Page count (PDF, Word, etc.) |
| `kMDItemDurationSeconds` | Duration (audio, video) |
| `kMDItemFSSize` | File size in bytes |
| `kMDItemContentCreationDate` | Creation date |
| `kMDItemContentModificationDate` | Last-modified date |
| `kMDItemLastUsedDate` | Last-opened date |
| `kMDItemUserTags` | Finder tags (array of strings) |

**Important:** NSMetadataQuery runs asynchronously. Register for notifications (`NSMetadataQueryDidFinishGatheringNotification`, `NSMetadataQueryDidUpdateNotification`), or set a delegate. Calling `resultCount` immediately after `start()` returns 0.
```
**MDQuery for terminal-like syntax:** If you need the exact Spotlight query syntax used in Terminal (e.g., `kMDItemTextContent == '*search*'`), use the lower-level `MDQuery` C API instead of NSMetadataQuery. `MDQuery` supports the raw query string format directly. Alternatively, you can use `NSPredicate(fromMetadataQueryString:)` to convert an MDQuery string to an NSPredicate for use with NSMetadataQuery. For asynchronous search, MDQuery supports callbacks; NSMetadataQuery uses notification-based updates.

## File Promises

NSFilePromiseProvider enables dragging items from your view to the Finder — the file doesn't exist at drag time, it's created on demand when the drop completes.

```swift
final class PromiseDragSourceView: NSView {
    override func mouseDragged(with event: NSEvent) {
        let provider = NSFilePromiseProvider(fileType: kUTTypePlainText as String, delegate: self)
        provider.userInfo = "Promise data"

        let draggingItem = NSDraggingItem(pasteboardWriter: provider)
        draggingItem.setDraggingFrame(bounds, contents: snapshotView(bounds))

        beginDraggingSession(with: [draggingItem], event: event, source: self)
    }

    private func snapshotView(_ rect: NSRect) -> NSImage {
        let image = NSImage(size: rect.size)
        image.lockFocus()
        NSColor.systemBlue.setFill()
        NSBezierPath.fill(rect)
        image.unlockFocus()
        return image
    }
}

extension PromiseDragSourceView: NSFilePromiseProviderDelegate {
    func filePromiseProviderFileNameForType(
        _ filePromiseProvider: NSFilePromiseProvider,
        fileType: String
    ) -> String {
        "Dropped Document.txt"
    }

    func filePromiseProvider(
        _ filePromiseProvider: NSFilePromiseProvider,
        writePromiseTo url: URL,
        completionHandler: @escaping (Error?) -> Void
    ) {
        do {
            try "File created by drag promise".write(to: url, atomically: true, encoding: .utf8)
            completionHandler(nil)
        } catch {
            completionHandler(error)
        }
    }

    func operationQueue(for filePromiseProvider: NSFilePromiseProvider) -> OperationQueue {
        .main
    }
}
```

**File promise behavior:** The Finder shows the file icon at the cursor during drag, but the file isn't created until the drop target calls `writePromiseTo`. This is efficient for large files — no temp file is needed on the source side.

## UTType Registration

### Info.plist Declaration

**Exported types** (your app owns the type):

```xml
<key>UTExportedTypeDeclarations</key>
<array>
    <dict>
        <key>UTTypeIdentifier</key>
        <string>com.example.myapp.project</string>
        <key>UTTypeDescription</key>
        <string>MyApp Project File</string>
        <key>UTTypeConformsTo</key>
        <array>
            <string>com.apple.package</string>
        </array>
        <key>UTTypeTagSpecification</key>
        <dict>
            <key>public.filename-extension</key>
            <array>
                <string>myproj</string>
            </array>
        </dict>
    </dict>
</array>
```

**Imported types** (you handle types from other apps):

```xml
<key>UTImportedTypeDeclarations</key>
<array>
    <dict>
        <key>UTTypeIdentifier</key>
        <string>com.otherapp.document</string>
        <key>UTTypeDescription</key>
        <string>OtherApp Document</string>
        <key>UTTypeConformsTo</key>
        <array>
            <string>public.data</string>
        </array>
        <key>UTTypeTagSpecification</key>
        <dict>
            <key>public.filename-extension</key>
            <array>
                <string>othr</string>
            </array>
        </dict>
    </dict>
</array>
```

### Swift UTType Extension

```swift
import UniformTypeIdentifiers

extension UTType {
    static let myProject = UTType(exportedAs: "com.example.myapp.project")
    static let mySnippet = UTType(exportedAs: "com.example.myapp.snippet", conformingTo: .data)
    static let otherAppDocument = UTType(importedAs: "com.otherapp.document")
}
```

**Conformance hierarchy for common types:**
- Use `public.package` for bundle formats (folders that appear as files in Finder)
- Use `public.data` + `public.content` for custom data formats
- Use `public.composite-content` for formats with embedded resources
- Always specify at least one parent type in `UTTypeConformsTo` — bare types won't match in Spotlight queries
```
**UTType Swift framework (macOS 11+):** The `UniformTypeIdentifiers` framework replaces CFString-based UTI identifiers with Swift-friendly `UTType` constants. Over 120 named constants for common system types are provided. Use `UTType(exportedAs:)` for types your app owns, `UTType(importedAs:)` for third-party types. Xcode tooling detects common issues (missing UTIs in Info.plist, incorrect initializers).
**macOS 26 UTType notes:**
- If both exported and imported declarations exist for the same type, the exported declaration takes precedence.
- Custom UTIs for file formats should conform to `public.data` or `com.apple.package` at the root of the conformance hierarchy.
- Use `public.filename-extension` in `UTTypeTagSpecification` to declare file extensions. MIME types can also be declared.
- For a Swift Package shared between two apps where one exports and one imports the same type, use `UTType(importedAs:)` in the package and let each app declare its own direction in Info.plist.

### Document Type Registration (for NSDocument)

In your target's Info tab in Xcode, add a Document Type entry with the identifier, extensions, and roles (Editor/Viewer). For `CFBundleTypeRole` use `"Editor"` for read/write or `"Viewer"` for read-only. The `LSHandlerRank` controls whether your app is `Owner`, `Default`, `Alternate`, or `None` for that type.

## NSFileProviderExtension

NSFileProviderExtension is the foundation for building a File Provider that integrates with Finder. The extension runs out-of-process and provides files on demand.

**Minimal extension structure:**

```swift
import FileProvider

class FileProviderExtension: NSFileProviderExtension {
    var manager: NSFileProviderManager {
        NSFileProviderManager(for: domain)!
    }

    override func item(for identifier: NSFileProviderItemIdentifier) throws -> NSFileProviderItem {
        return FileProviderItem(identifier: identifier)
    }

    override func urlForItem(withPersistentIdentifier identifier: NSFileProviderItemIdentifier) -> URL? {
        guard let item = try? item(for: identifier) else { return nil }
        return NSFileProviderManager.defaultDocumentDirectoryURL
            .appendingPathComponent(item.filename)
    }

    override func providePlaceholder(at url: URL, completionHandler: @escaping (Error?) -> Void) {
        // Create a placeholder file at the given URL
        do {
            let item = try item(for: enumerator: .rootContainerIdentifier)
            try NSFileProviderManager.writePlaceholder(at: url, withMetadata: item)
            completionHandler(nil)
        } catch {
            completionHandler(error)
        }
    }

    override func startProvidingItem(at url: URL, completionHandler: @escaping (Error?) -> Void) {
        // Download the actual file content from your remote source
        downloadFile(to: url) { error in
            completionHandler(error)
        }
    }

    override func stopProvidingItem(at url: URL) {
        // Optional: evict local cache
    }

    private func downloadFile(to url: URL, completion: @escaping (Error?) -> Void) {
        // Fetch from your backend/cloud storage
        completion(nil)
    }
}
```

**Enrollment:** Add a File Provider target in Xcode (File Provider extension template). Configure the `NSExtension` > `NSExtensionFileProviderDomain` in the extension's Info.plist with a unique domain identifier. Call `NSFileProviderManager.add(_:completionHandler:)` from your main app to register the domain.

## Constraints & Gotchas

**Security-scoped resource lifecycle:** `startAccessingSecurityScopedResource()` returns `false` when the sandbox denies access. Always check the return value. Balanced calls are not strictly required (extra `stop` calls are no-ops), but you must call `stop` eventually — leaking scoped access wastes kernel resources.

**File Coordinator blocking the main thread:** `coordinate(readingItemAt:)` and `coordinate(writingItemAt:)` are synchronous and block the calling thread until all presenters acknowledge. If another process's presenter runs slow UI work, your thread stalls. Run coordinated access on a background queue unless you specifically need main-thread results.
```
**NSFileCoordinator threading model:** Each `NSFileCoordinator` should be used on a single thread. For cross-thread coordination, create separate instances per thread/object. Coordinated reads/writes can start a background task automatically to avoid deadlocks if the app is suspended. If the task expires, coordination may be canceled with `NSUserCancelledError`.
**NSFilePresenter background/foreground:** Remove presenters in `applicationDidEnterBackground` and re-add in `applicationWillEnterForeground` to prevent termination from deadlocks with other processes. Presenter methods should be fast — defer heavy processing asynchronously.
**NSSavePanel process isolation:** Since macOS 10.15, NSSavePanel runs in a separate process. The saved file is added to the app's sandbox automatically. Use `beginSheetModal(for:completionHandler:)` for window-attached sheets, or `begin(completionHandler:)` for app-modal presentation. The `canCreateDirectories` property controls whether users can create new directories in the panel.
**NSFileCoordinator init(filePresenter:):** When the file operation is initiated by a presenter, use `NSFileCoordinator(filePresenter: self)` instead of the plain `init()`. This avoids deadlocks and race conditions when the presenter is both the initiator and a participant in coordination.

**NSFilePresenter registration cost:** Every registered presenter receives notifications for every file system change in the system. Register only for URLs your app actively monitors, and always call `removeFilePresenter` in `deinit`.

**NSMetadataQuery results ordering:** Results are not sorted by default. Set `valueListAttributes` and `sortingAttributes` before calling `start()`. If you need relevance ranking, use `kMDItemRelatedUniqueIdentifier` or post-filter.

**UTType and Launch Services:** After adding exported type declarations, the type isn't available to Launch Services until the app is installed in `/Applications` or the user logs out and back in. During development, use `lsregister -lint` to reset the Launch Services database:
```bash
/System/Library/Frameworks/CoreServices.framework/Frameworks/LaunchServices.framework/Support/lsregister -lint -f /path/to/YourApp.app
```

## Common Mistakes

- **Calling `startAccessingSecurityScopedResource` inside `bookmarkData` options** — Pass `.withSecurityScope` to the bookmark creation call, not `startAccessingSecurityScopedResource`. The options flag captures scope in the bookmark data itself.
- **Forgetting `withoutChanges` on coordinated reads** — Using the default option on a read registers interest in the file, causing your app to receive `presentedItemDidChange` notifications even though you only wanted to read.
- **Removing NSFilePresenter while a coordinated write is in progress** — Always remove presenters on a background queue or after all operations complete. The system may crash or deadlock.
- **Querying NSMetadataQuery immediately after start()** — The query runs asynchronously. Observe `NSMetadataQueryDidFinishGatheringNotification` before reading results.
- **Using `kMDItemContentType` for type matching** — Use `kMDItemContentTypeTree` instead. Querying `kMDItemContentType == "public.jpeg"` returns only JPEGs, while `== "public.image"` also matches TIFF and PNG (they don't inherit from `public.image` via that attribute). `ContentTypeTree` walks the full conformance chain.
- **Dropping NSFilePromiseProvider operations to the background queue** — `writePromiseTo` runs on the operation queue you provide, but the dropped file must be written atomically. Use atomic writes or coordinate access to avoid partial files in Finder.

## Review Checklist

- [ ] Sandbox entitlements declared for every file access path (user-selected, downloads, bookmarks)
- [ ] `startAccessingSecurityScopedResource` / `stopAccessingSecurityScopedResource` balanced on every code path
- [ ] Stale bookmark re-creation implemented
- [ ] Coordinated access uses `.withoutChanges` for reads
- [ ] NSFilePresenter added/removed in matched pairs
- [ ] NSMetadataQuery observes notifications before reading results
- [ ] UTType conformances specify at least one parent type in the hierarchy
- [ ] Info.plist document types and exported/imported type declarations consistent with Swift UTType extension
- [ ] File Provider extension domain identifier unique and registered via `NSFileProviderManager.add`

## References

- [references/filesystem-patterns.md](references/filesystem-patterns.md) — Full security-scoped bookmark lifecycle, NSFileCoordinator with NSFilePresenter, NSMetadataQuery custom Spotlight integration, SwiftUI fileImporter/fileExporter patterns, drag-to-Finder with NSFilePromiseProvider
- "The Role of File Coordinators and Presenters" — https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/FileSystemProgrammingGuide/FileCoordinators/FileCoordinators.html
- "Designing for Documents in iCloud" — https://developer.apple.com/library/archive/documentation/General/Conceptual/iCloudDesignGuide/Chapters/DesigningForDocumentsIniCloud.html
- "Defining file and data types for your app" — https://developer.apple.com/documentation/uniformtypeidentifiers/defining-file-and-data-types-for-your-app
- "Uniform Type Identifiers — a reintroduction" (Tech Talks 10696) — https://developer.apple.com/videos/play/tech-talks/10696/
