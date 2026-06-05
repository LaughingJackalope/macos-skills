# NSPasteboard, Core Spotlight, and Quick Look

Complete reference for NSPasteboard patterns with custom UTType registration, Core Spotlight indexing, and Quick Look integration on macOS.

## NSPasteboard Patterns

### Reading Multiple Types

```swift
func readFromPasteboard() {
    let pb = NSPasteboard.general

    // Check available types
    let availableTypes = pb.types ?? []
    print("Available types: \(availableTypes)")

    // Try reading in priority order
    if let fileURLs = pb.readObjects(forClasses: [NSURL.self], options: [.urlReadingFileURLsOnly: true]) as? [URL] {
        // Handle file URLs
        for url in fileURLs {
            print("File: \(url.path)")
        }
    } else if let image = NSImage(pasteboard: pb) {
        // Handle image data
        processImage(image)
    } else if let string = pb.string(forType: .string) {
        // Handle plain text
        processString(string)
    } else if let rtfData = pb.data(forType: .rtf),
              let attributedString = NSAttributedString(rtf: rtfData, documentAttributes: nil) {
        // Handle RTF
        processAttributedString(attributedString)
    }
}
```

### Writing Multiple Representations

```swift
func writeToPasteboard(items: [PasteboardItem]) {
    let pb = NSPasteboard.general
    pb.clearContents()

    var writers: [NSPasteboardWriting] = []

    for item in items {
        switch item {
        case .text(let string):
            writers.append(string as NSString)

        case .attributedText(let attrString):
            writers.append(attrString)

        case .image(let image):
            writers.append(image)

        case .fileURL(let url):
            writers.append(url as NSURL)

        case .custom(let data, let type):
            let pbItem = NSPasteboardItem()
            pbItem.setData(data, forType: type)
            writers.append(pbItem)
        }
    }

    pb.writeObjects(writers)
}
```

### Drag and Drop with Custom Types

For `NSView`-based drag and drop:

```swift
final class DragSourceView: NSView {
    override func mouseDragged(with event: NSEvent) {
        let data = try! JSONEncoder().encode(dragPayload)
        let item = NSPasteboardItem()
        item.setData(data, forType: .myProjectItem)

        let draggingItem = NSDraggingItem(pasteboardWriter: item)
        draggingItem.setDraggingFrame(bounds, contents: snapshotImage)

        beginDraggingSession(with: [draggingItem], event: event, source: self)
    }
}

final class DropTargetView: NSView {
    override init(frame: NSRect) {
        super.init(frame: frame)
        registerForDraggedTypes([.myProjectItem, .fileURL])
    }

    required init?(coder: NSCoder) { fatalError() }

    override func draggingEntered(_ sender: NSDraggingInfo) -> NSDragOperation {
        let pb = sender.draggingPasteboard
        if pb.canReadObject(forClasses: [NSData.self], options: nil) {
            highlight()
            return .copy
        }
        return []
    }

    override func performDragOperation(_ sender: NSDraggingInfo) -> Bool {
        let pb = sender.draggingPasteboard
        guard let data = pb.data(forType: .myProjectItem),
              let item = try? JSONDecoder().decode(ProjectItem.self, from: data) else {
            return false
        }
        handleDrop(item: item)
        return true
    }

    override func draggingExited(_ sender: NSDraggingInfo?) {
        unhighlight()
    }
}
```

### NSPasteboard Filtering

Filter pasteboard content by type conformance:

```swift
let options: [NSPasteboard.ReadingOptionKey: Any] = [
    .urlReadingFileURLsOnly: true,
    .urlReadingContentsConformToTypes: [UTType.image, UTType.movie]
]

if let urls = NSPasteboard.general.readObjects(forClasses: [NSURL.self], options: options) as? [URL] {
    // Only image and movie file URLs
}
```

## Custom UTType Registration

### Info.plist Declaration

For types your app owns (exported):

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
    <dict>
        <key>UTTypeIdentifier</key>
        <string>com.example.myapp.snippet</string>
        <key>UTTypeDescription</key>
        <string>MyApp Code Snippet</string>
        <key>UTTypeConformsTo</key>
        <array>
            <string>public.data</string>
            <string>public.content</string>
        </array>
        <key>UTTypeTagSpecification</key>
        <dict>
            <key>public.filename-extension</key>
            <array>
                <string>mysnip</string>
            </array>
            <key>public.mime-type</key>
            <array>
                <string>application/x-myapp-snippet</string>
            </array>
        </dict>
    </dict>
</array>
```

For types from other apps you want to handle (imported):

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
    // Exported types
    static let myProject = UTType(exportedAs: "com.example.myapp.project")
    static let mySnippet = UTType(exportedAs: "com.example.myapp.snippet", conformingTo: .data)

    // Imported types
    static let otherAppDocument = UTType(importedAs: "com.otherapp.document")
}
```

### Common Conformance Hierarchy

```
public.item (base)
├── public.content
│   ├── public.text
│   │   ├── public.plain-text
│   │   ├── public.utf8-plain-text
│   │   ├── public.utf16-external-plain-text
│   │   ├── public.utf16-plain-text
│   │   ├── public.rtf
│   │   ├── public.html
│   │   ├── public.xml
│   │   └── public.source-code
│   ├── public.image
│   │   ├── public.jpeg
│   │   ├── public.png
│   │   └── public.tiff
│   ├── public.audio
│   ├── public.movie
│   └── public.presentation
├── public.data
│   ├── public.composite-content
│   ├── public.archive
│   └── public.database
├── public.directory
│   ├── com.apple.package
│   └── public.folder
└── public.url
    └── public.file-url
```

## Core Spotlight Indexing

### CSSearchableItemAttributeSet Common Properties

```swift
func createAttributeSet(for document: Document) -> CSSearchableItemAttributeSet {
    let attributes = CSSearchableItemAttributeSet(contentType: .text)

    // Display
    attributes.title = document.title
    attributes.displayName = document.title
    attributes.contentDescription = document.summary
    attributes.comment = document.comment
    attributes.thumbnailData = document.thumbnailPNG
    attributes.thumbnailURL = document.thumbnailFileURL

    // Metadata
    attributes.keywords = document.tags
    attributes.subject = document.subject
    attributes.theme = document.category
    attributes.contentCreationDate = document.createdAt
    attributes.contentModificationDate = document.modifiedAt
    attributes.lastUsedDate = document.lastOpenedAt
    attributes.addedDate = Date()

    // Content
    attributes.pageCount = document.pageCount
    attributes.pageSize = CGSize(width: 612, height: 792)  // US Letter
    attributes.numberOfPages = document.pageCount
    attributes.creator = document.author
    attributes.contributors = document.contributors
    attributes.languages = document.languages
    attributes.rights = document.rights
    attributes.organizations = [document.organization]
    attributes.publishers = [document.publisher]

    // Ratings and stars
    attributes.rating = document.starRating
    attributes.ratingDescription = "\(document.starRating) stars"

    // Media
    attributes.duration = document.duration  // for audio/video
    attributes.audioSampleRate = document.sampleRate
    attributes.audioChannelCount = document.channelCount
    attributes.tempo = document.bpm
    attributes.keySignature = document.musicalKey
    attributes.musicalGenre = document.genre

    // Files
    attributes.fileSize = document.fileSizeInBytes
    attributes.totalBitrate = document.bitrate

    // Custom — use the generic userInfo dictionary for app-specific data
    attributes.userInfo = ["projectId": document.projectID]

    return attributes
}
```

### Domain Identifiers for Organization

Use domain identifiers to group items for batch operations:

```swift
extension SpotlightIndexer {
    enum Domain: String {
        case documents = "com.example.myapp.documents"
        case projects = "com.example.myapp.projects"
        case snippets = "com.example.myapp.snippets"
    }

    func index(_ document: Document) async throws {
        let attributes = createAttributeSet(for: document)
        let item = CSSearchableItem(
            uniqueIdentifier: document.id,
            domainIdentifier: Domain.documents.rawValue,
            attributeSet: attributes
        )
        try await CSSearchableIndex.default().indexSearchableItems([item])
    }

    func removeAllDocuments() async throws {
        try await CSSearchableIndex.default()
            .deleteSearchableItems(withDomainIdentifiers: [Domain.documents.rawValue])
    }

    func removeSnippet(id: String) async throws {
        try await CSSearchableIndex.default()
            .deleteSearchableItems(withIdentifiers: [id])
    }
}
```

### Reindexing After App Update

When your data model changes, reindex existing content:

```swift
func reindexAllDocuments() async throws {
    let index = CSSearchableIndex.default()

    // Delete all existing items
    try await index.deleteAllSearchableItems()

    // Re-fetch and reindex
    let documents = try await fetchAllDocuments()
    let items = documents.map { doc -> CSSearchableItem in
        let attrs = createAttributeSet(for: doc)
        return CSSearchableItem(
            uniqueIdentifier: doc.id,
            domainIdentifier: "com.example.myapp.documents",
            attributeSet: attrs
        )
    }

    // Batch in groups of 1000 to avoid timeouts
    for batch in items.chunked(into: 1000) {
        try await index.indexSearchableItems(batch)
    }
}
```

### Spotlight Privacy

Exclude files from Spotlight indexing by setting the `kMDItemIsInvisible` attribute or by placing a `.noindex` extension on directories:

```swift
// Programmatically exclude from Spotlight
attributes.isInvisible = true
```

## Quick Look Integration

### QLPreviewPanel in a Window Controller

```swift
final class DocumentWindowController: NSWindowController {
    var previewItems: [QLPreviewItem] = []
    var currentPreviewIndex = 0

    var canPreview: Bool {
        !previewItems.isEmpty
    }

    @IBAction func toggleQuickLook(_ sender: Any?) {
        guard let panel = QLPreviewPanel.shared() else { return }

        if panel.isVisible {
            panel.orderOut(sender)
        } else {
            panel.makeKeyAndOrderFront(sender)
        }
    }

    // The window controller must be in the responder chain
    override func acceptsPreviewPanelControl(_ panel: QLPreviewPanel!) -> Bool {
        canPreview
    }

    override func beginPreviewPanelControl(_ panel: QLPreviewPanel!) {
        panel.dataSource = self
        panel.delegate = self
    }

    override func endPreviewPanelControl(_ panel: QLPreviewPanel!) {
        panel.dataSource = nil
        panel.delegate = nil
    }
}

extension DocumentWindowController: QLPreviewPanelDataSource {
    func numberOfPreviewItems(in panel: QLPreviewPanel!) -> Int {
        previewItems.count
    }

    func previewPanel(_ panel: QLPreviewPanel!, previewItemAt index: Int) -> QLPreviewItem! {
        previewItems[index]
    }
}

extension DocumentWindowController: QLPreviewPanelDelegate {
    func previewPanel(_ panel: QLPreviewPanel!, handle event: NSEvent!) -> Bool {
        // Handle keyboard events in the preview panel
        if event.type == .keyDown {
            switch event.keyCode {
            case 123:  // left arrow
                panel.currentPreviewItemIndex = max(0, panel.currentPreviewItemIndex - 1)
                return true
            case 124:  // right arrow
                panel.currentPreviewItemIndex = min(previewItems.count - 1, panel.currentPreviewItemIndex + 1)
                return true
            default:
                break
            }
        }
        return false
    }

    func previewPanel(_ panel: QLPreviewPanel!, sourceFrameOnScreenFor item: QLPreviewItem!) -> NSRect {
        // Return the rect of the item in the source view for zoom animation
        return .zero  // simplified
    }

    func previewPanel(_ panel: QLPreviewPanel!, transitionImageFor item: QLPreviewItem!, contentRect: UnsafeMutablePointer<NSRect>) -> Any {
        // Return an image for the zoom transition animation
        return NSImage()
    }
}
```

### SwiftUI Quick Look with Multiple Items

```swift
struct FileBrowserView: View {
    @State private var files: [URL] = []
    @State private var selection: Set<URL> = []
    @State private var previewURL: URL?

    var body: some View {
        Table(files, selection: $selection) {
            TableColumn("Name") { url in
                Text(url.lastPathComponent)
            }
            TableColumn("Size") { url in
                Text(byteCountFormatter.string(fromByteCount: (try? url.resourceValues(forKeys: [.fileSizeKey]).fileSize) ?? 0))
            }
        }
        .onChange(of: selection) { _, newValue in
            previewURL = newValue.first
        }
        .quickLookPreview($previewURL)
    }
}
```

### Custom Preview for In-Memory Data

```swift
final class DataPreviewItem: NSObject, QLPreviewItem {
    let previewItemURL: URL
    let previewItemTitle: String?

    init(data: Data, fileName: String, mimeType: String, title: String?) throws {
        let tempDir = FileManager.default.temporaryDirectory
            .appendingPathComponent("quicklook-previews", isDirectory: true)
        try FileManager.default.createDirectory(at: tempDir, withIntermediateDirectories: true)

        let tempURL = tempDir.appendingPathComponent(fileName)
        try data.write(to: tempURL)
        self.previewItemURL = tempURL
        self.previewItemTitle = title
    }

    deinit {
        // Clean up temp file
        try? FileManager.default.removeItem(at: previewItemURL)
    }
}

// Usage
func showPreview(for document: Document) throws {
    let data = try document.renderToPDF()
    let previewItem = try DataPreviewItem(
        data: data,
        fileName: "\(document.title).pdf",
        mimeType: "application/pdf",
        title: document.title
    )
    // Add to preview panel's items array
}
```

### QLPreviewPanel Keyboard Shortcuts

| Key | Action |
|-----|--------|
| Space | Open/close preview panel |
| ← → | Navigate between items |
| ↑ ↓ | Navigate between items (if multiple) |
| Escape | Close preview panel |
| Cmd+R | Reload preview |
| Cmd+I | Show info (if supported by preview) |
