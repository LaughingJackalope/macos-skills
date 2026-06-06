# File System Patterns — Complete Reference

Complete code patterns for macOS file system access. Each section is self-contained.

## Security-Scoped Bookmark Lifecycle

### Create a Bookmark

After the user picks a file or folder via NSOpenPanel, immediately create a bookmark:

```swift
final class PersistentAccess {
    private let keychain = KeychainBookmarkStore()

    func userDidPickFile(_ url: URL) {
        // url is already security-scoped from NSOpenPanel
        bookmark = try? url.bookmarkData(
            options: [.withSecurityScope, .securityScopeAllowOnlyReadAccess],
            includingResourceValuesForKeys: [.contentModificationDateKey],
            relativeTo: nil
        )
        if let data = bookmark {
            keychain.store(data, forKey: "mainProjectBookmark")
        }
    }

    private var bookmark: Data?
}
```

`.securityScopeAllowOnlyReadAccess` creates a read-only bookmark. Omit for read/write. Including `.contentModificationDateKey` in the bookmark data lets you detect if the file changed before you resolve it.

### Store in Keychain

```swift
import Security

final class KeychainBookmarkStore {
    func store(_ data: Data, forKey key: String) {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: "com.example.myapp.bookmarks",
            kSecAttrAccount as String: key,
            kSecValueData as String: data
        ]
        SecItemDelete(query as CFDictionary) // Remove existing
        SecItemAdd(query as CFDictionary, nil)
    }

    func retrieve(forKey key: String) -> Data? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrService as String: "com.example.myapp.bookmarks",
            kSecAttrAccount as String: key,
            kSecReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]
        var result: AnyObject?
        guard SecItemCopyMatching(query as CFDictionary, &result) == errSecSuccess else {
            return nil
        }
        return result as? Data
    }
}
```

### Resolve and Access

```swift
func openBookmarkedFile() {
    guard let data = keychain.retrieve(forKey: "mainProjectBookmark") else { return }

    var isStale = false
    do {
        let url = try URL(
            resolvingBookmarkData: data,
            options: .withSecurityScope,
            relativeTo: nil,
            bookmarkDataIsStale: &isStale
        )

        guard url.startAccessingSecurityScopedResource() else {
            print("Sandbox denied access — entitlement may be missing")
            return
        }
        defer { url.stopAccessingSecurityScopedResource() }

        if isStale {
            // Re-create with fresh metadata
            let fresh = try url.bookmarkData(
                options: .withSecurityScope,
                includingResourceValuesForKeys: [.contentModificationDateKey],
                relativeTo: nil
            )
            keychain.store(fresh, forKey: "mainProjectBookmark")
        }

        // Read the file
        let content = try String(contentsOf: url, encoding: .utf8)
        display(content)

    } catch {
        if (error as NSError).code == NSFileReadNoPermissionError {
            // The file was moved to a location the sandbox can't reach
            promptUserToReopenFile()
        }
    }
}
```

## NSFileCoordinator with NSFilePresenter

### Full Implementation with Version Conflict Handling

```swift
final class CoordinatedDocumentStore {
    private var presenters: [URL: DocumentPresenter] = [:]

    // MARK: - Coordinated Read

    func readDocument(at url: URL) -> String? {
        let coordinator = NSFileCoordinator(filePresenter: presenters[url])
        var result: String?
        var coordinatorError: NSError?

        coordinator.coordinate(readingItemAt: url, options: .withoutChanges, error: &coordinatorError) { coordinatedURL in
            do {
                result = try String(contentsOf: coordinatedURL, encoding: .utf8)
            } catch {
                print("Read error: \(error)")
            }
        }

        return result
    }

    // MARK: - Coordinated Write with Conflict Handling

    func writeDocument(at url: URL, content: String) throws {
        let coordinator = NSFileCoordinator(filePresenter: presenters[url])
        var writeError: NSError?

        try coordinator.coordinate(writingItemAt: url, options: .forReplacing) {
            var coordinatorError: NSError?
            coordinator.coordinate(writingItemAt: $0, options: .forReplacing, error: &coordinatorError) { coordinatedURL in
                do {
                    try content.write(to: coordinatedURL, atomically: true, encoding: .utf8)
                } catch {
                    writeError = error as NSError
                }
            }
        }

        if let error = writeError { throw error }
    }

    // MARK: - Registering a Presenter

    func monitorDocument(at url: URL) {
        let presenter = DocumentPresenter(fileURL: url)
        presenters[url] = presenter
    }

    func stopMonitoring(url: URL) {
        presenters.removeValue(forKey: url)
    }
}

// MARK: - NSFilePresenter Implementation

final class DocumentPresenter: NSObject, NSFilePresenter {
    let presentedItemURL: URL
    let presentedItemOperationQueue: OperationQueue

    // Track save versions for conflict detection
    private var lastKnownVersion: NSFileVersion?

    init(fileURL: URL) {
        presentedItemURL = fileURL
        presentedItemOperationQueue = OperationQueue()
        presentedItemOperationQueue.maxConcurrentOperationCount = 1
        presentedItemOperationQueue.qualityOfService = .userInitiated
        super.init()
        NSFileCoordinator.addFilePresenter(self)
        saveCurrentVersion()
    }

    deinit {
        NSFileCoordinator.removeFilePresenter(self)
    }

    private func saveCurrentVersion() {
        lastKnownVersion = NSFileVersion.currentVersionOfItem(at: presentedItemURL)
    }

    // Called when another process modifies the file
    func presentedItemDidChange() {
        let newVersion = NSFileVersion.currentVersionOfItem(at: presentedItemURL)
        guard newVersion != lastKnownVersion else { return }
        lastKnownVersion = newVersion

        // On the operation queue — not on whatever thread called this
        presentedItemOperationQueue.addOperation { [weak self] in
            guard let self else { return }
            DispatchQueue.main.async {
                self.notifyDocumentChanged()
            }
        }
    }

    // Called by the coordinator before a coordinated write
    func relinquishPresentedItem(toReader reader: @escaping ((() -> Void)?) -> Void) {
        // Flush any pending UI reads before the writer proceeds
        saveCurrentVersion()
        reader(nil) // nil completion = no need to wait
    }

    func presentedItemDidMove(to newURL: URL) {
        presentedItemOperationQueue.addOperation { [weak self] in
            guard let self else { return }
            DispatchQueue.main.async {
                self.handleFileMoved(to: newURL)
            }
        }
    }

    private func notifyDocumentChanged() {
        NotificationCenter.default.post(name: .documentDidChangeRemotely, object: presentedItemURL)
    }

    private func handleFileMoved(to newURL: URL) {
        NotificationCenter.default.post(name: .documentDidMove, object: nil, userInfo: [
            "oldURL": presentedItemURL,
            "newURL": newURL
        ])
    }
}

extension Notification.Name {
    static let documentDidChangeRemotely = Notification.Name("documentDidChangeRemotely")
    static let documentDidMove = Notification.Name("documentDidMove")
}
```

**Critical detail:** The `operationQueue` you provide to `NSFilePresenter` determines which thread receives `presentedItemDidChange`. Use a dedicated serial queue, not `.main`, to avoid blocking the presenter callback chain. Dispatch UI updates to main yourself.

## NSMetadataQuery for Custom Spotlight Integration

### Finding Files by Multiple Criteria

```swift
import CoreServices
import UniformTypeIdentifiers

final class FileFinder: NSObject, ObservableObject {
    @Published var foundFiles: [FoundFile] = []
    private let query = NSMetadataQuery()

    struct FoundFile: Identifiable {
        var id: String { url.path }
        let url: URL
        let displayName: String
        let modificationDate: Date?
        let fileSize: Int64
    }

    override init() {
        super.init()
        query.delegate = self
        query.operationQueue = .main

        // Observe query lifecycle
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(queryDidFinishGathering),
            name: .NSMetadataQueryDidFinishGathering,
            object: query
        )
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(queryDidUpdate),
            name: .NSMetadataQueryDidUpdate,
            object: query
        )
    }

    // MARK: - Search Methods

    func findImages(modifiedAfter date: Date) {
        let typePredicate = NSPredicate(format: "kMDItemContentTypeTree == %@", UTType.image.identifier)
        let datePredicate = NSPredicate(format: "kMDItemContentModificationDate >= %@", date as NSDate)
        query.predicate = NSCompoundPredicate(andPredicateWithSubpredicates: [typePredicate, datePredicate])
        query.searchScopes = [NSMetadataQueryUserHomeScope]
        query.sortDescriptors = [
            NSSortDescriptor(key: NSMetadataItemContentModificationDateKey, ascending: false)
        ]
        query.valueListAttributes = [
            NSMetadataItemDisplayNameKey,
            NSMetadataItemContentModificationDateKey,
            NSMetadataItemFSSizeKey
        ]
        query.start()
    }

    func findLargeFiles(minimumBytes: Int64 = 100_000_000) {
        query.predicate = NSPredicate(format: "kMDItemFSSize >= %lld", minimumBytes)
        query.searchScopes = [NSMetadataQueryUserHomeScope]
        query.sortDescriptors = [
            NSSortDescriptor(key: NSMetadataItemFSSizeKey, ascending: false)
        ]
        query.start()
    }

    func findRecentlyUsed(limit: Int = 20) {
        query.predicate = NSPredicate(format: "kMDItemLastUsedDate >= %@", NSDate.distantPast)
        query.searchScopes = [NSMetadataQueryUserHomeScope]
        query.sortDescriptors = [
            NSSortDescriptor(key: NSMetadataItemLastUsedDateKey, ascending: false)
        ]
        query.start()
    }

    func searchTaggedFiles(tag: String) {
        // Finder tags appear in kMDItemUserTags as ["TagName", ...]
        query.predicate = NSPredicate(format: "kMDItemUserTags == %@", tag)
        query.searchScopes = [NSMetadataQueryUserHomeScope]
        query.start()
    }

    func stop() {
        query.stop()
        query.disableUpdates()
    }

    // MARK: - Notifications

    @objc private func queryDidFinishGathering() {
        query.disableUpdates()
        updateResults()
    }

    @objc private func queryDidUpdate() {
        updateResults()
    }

    private func updateResults() {
        let items = (0..<query.resultCount).compactMap { index -> FoundFile? in
            guard let item = query.result(at: index) as? NSMetadataItem else { return nil }
            guard let url = item.value(forAttribute: NSMetadataItemURLKey) as? URL else { return nil }
            return FoundFile(
                url: url,
                displayName: item.value(forAttribute: NSMetadataItemDisplayNameKey) as? String ?? "",
                modificationDate: item.value(forAttribute: NSMetadataItemContentModificationDateKey) as? Date,
                fileSize: item.value(forAttribute: NSMetadataItemFSSizeKey) as? Int64 ?? 0
            )
        }
        DispatchQueue.main.async {
            self.foundFiles = items
        }
    }
}

extension FileFinder: NSMetadataQueryDelegate {
    func metadataQuery(_ query: NSMetadataQuery, replacementObjectForResultObject result: NSMetadataItem) -> Any {
        return result
    }
}
```

**Performance tip:** Set `searchScopes` to the narrowest possible path. `NSMetadataQueryUserHomeScope` searches the entire home directory. For better performance, pass a specific `file://` URL. Use `NSMetadataQueryIndexedLocalComputerScope` for the entire indexed volume.

## SwiftUI fileImporter/fileExporter with UTType

### Advanced Importer with Validation

```swift
struct DocumentImportView: View {
    @State private var isImporterPresented = false
    @State private var importError: String?
    @State private var documents: [ImportedDocument] = []

    var body: some View {
        VStack {
            Button("Import Documents") {
                isImporterPresented = true
            }

            if let error = importError {
                Text(error).foregroundStyle(.red)
            }

            List(documents) { doc in
                VStack(alignment: .leading) {
                    Text(doc.name).font(.headline)
                    Text("\(doc.size) bytes").font(.caption).foregroundStyle(.secondary)
                }
            }
        }
        .fileImporter(
            isPresented: $isImporterPresented,
            allowedContentTypes: [.plainText, .pdf, .jpeg, .png, .data],
            allowsMultipleSelection: true
        ) { result in
            handleImport(result: result)
        }
    }

    private func handleImport(result: Result<[URL], Error>) {
        switch result {
        case .success(let urls):
            importError = nil
            for url in urls {
                guard url.startAccessingSecurityScopedResource() else {
                    importError = "Access denied for \(url.lastPathComponent)"
                    continue
                }
                defer { url.stopAccessingSecurityScopedResource() }

                do {
                    let values = try url.resourceValues(forKeys: [.fileSizeKey, .contentTypeKey])
                    let doc = ImportedDocument(
                        name: url.lastPathComponent,
                        url: url,
                        size: values.fileSize ?? 0,
                        contentType: values.contentType
                    )
                    documents.append(doc)
                } catch {
                    importError = "Failed to read \(url.lastPathComponent): \(error.localizedDescription)"
                }
            }
        case .failure(let error):
            importError = error.localizedDescription
        }
    }
}

struct ImportedDocument: Identifiable {
    let id = UUID()
    let name: String
    let url: URL
    let size: Int
    let contentType: UTType?
}
```

### Exporter with ReferenceFileDocument

```swift
struct ProjectExporter: View {
    @State private var isExporterPresented = false
    @ObservedObject var project: Project

    var body: some View {
        Button("Export Project") {
            isExporterPresented = true
        }
        .fileExporter(
            isPresented: $isExporterPresented,
            document: ProjectDocument(project: project),
            contentType: .myProject,
            defaultFilename: project.name
        ) { result in
            switch result {
            case .success(let url):
                // Create a security-scoped bookmark for the exported location
                _ = try? url.bookmarkData(
                    options: .withSecurityScope,
                    includingResourceValuesForKeys: nil,
                    relativeTo: nil
                )
                print("Exported to: \(url)")
            case .failure(let error):
                print("Export failed: \(error)")
            }
        }
    }
}

struct ProjectDocument: ReferenceFileDocument {
    static var readableContentTypes: [UTType] { [.myProject] }
    static var writableContentTypes: [UTType] { [.myProject] }

    var project: Project

    init(project: Project) {
        self.project = project
    }

    init(configuration: ReadConfiguration) throws {
        guard let data = configuration.file.regularFileContents,
              let project = try? JSONDecoder().decode(Project.self, from: data)
        else { throw CocoaError(.fileReadCorruptFile) }
        self.project = project
    }

    func fileWrapper(configuration: WriteConfiguration) throws -> FileWrapper {
        let data = try JSONEncoder().encode(project)
        return FileWrapper(regularFileWithContents: data)
    }
}
```

## Drag-to-Finder with NSFilePromiseProvider

### Complete Pattern with Progress

```swift
final class PromiseDragSourceView: NSView {
    var itemsToExport: [ExportItem] = []

    struct ExportItem {
        let name: String
        let type: UTType
        let data: () throws -> Data
    }

    override func mouseDragged(with event: NSEvent) {
        guard !itemsToExport.isEmpty else { return }

        let providers = itemsToExport.map { item in
            let provider = NSFilePromiseProvider(fileType: item.type.identifier, delegate: self)
            provider.userInfo = item.name
            let dragItem = NSDraggingItem(pasteboardWriter: provider)
            dragItem.setDraggingFrame(
                NSRect(x: 0, y: 0, width: 64, height: 64),
                contents: iconFor(type: item.type)
            )
            return dragItem
        }

        beginDraggingSession(with: providers, event: event, source: self)
    }

    private func iconFor(type: UTType) -> NSImage {
        NSImage(systemSymbolName: "doc.fill", accessibilityDescription: nil) ?? NSImage()
    }
}

extension PromiseDragSourceView: NSFilePromiseProviderDelegate {
    func filePromiseProviderFileNameForType(
        _ filePromiseProvider: NSFilePromiseProvider,
        fileType: String
    ) -> String {
        (filePromiseProvider.userInfo as? String) ?? "Untitled"
    }

    func filePromiseProvider(
        _ filePromiseProvider: NSFilePromiseProvider,
        writePromiseTo url: URL,
        completionHandler: @escaping (Error?) -> Void
    ) {
        let fileName = filePromiseProvider.userInfo as? String ?? ""

        // Find the matching export item
        guard let item = itemsToExport.first(where: { $0.name == fileName }) else {
            completionHandler(NSError(domain: "ExportError", code: 1, userInfo: [
                NSLocalizedDescriptionKey: "No export item for \(fileName)"
            ]))
            return
        }

        // Write on a background queue — the delegate method may be called off-main
        DispatchQueue.global(qos: .userInitiated).async {
            do {
                let data = try item.data()
                try data.write(to: url, options: .atomic)
                DispatchQueue.main.async {
                    completionHandler(nil)
                }
            } catch {
                DispatchQueue.main.async {
                    completionHandler(error)
                }
            }
        }
    }

    func operationQueue(for filePromiseProvider: NSFilePromiseProvider) -> OperationQueue {
        .main
    }
}
```

**Drag-to-Finder behavior:** When the user drops onto the Finder, the system calls `writePromiseTo` with a URL in the drop location. The file is created at that URL. If the user cancels the drag (Esc key), none of the delegate methods are called. The `userInfo` property on `NSFilePromiseProvider` is the standard way to pass context from the drag session to the write callback — it survives the pasteboard round-trip.

### Multiple File Promises in One Drag Session

When dragging multiple items, each `NSFilePromiseProvider` gets its own `writePromiseTo` call. The system calls them sequentially, not in parallel. If one fails, the remaining items are still written — there is no atomic all-or-nothing guarantee. For transactional behavior, write to a temporary directory first, then move all files atomically on success.
