# NSViewRepresentable Recipes

Complete working recipes for common AppKit wrapping scenarios. Each recipe includes the full `NSViewRepresentable` struct, the Coordinator with delegate methods, a SwiftUI usage example, and gotchas specific to that wrapper.

---

## Contents

- [1. NSTextView Wrapper (Rich Text Editor)](#1-nstextview-wrapper-rich-text-editor)
- [2. NSProgressIndicator Wrapper](#2-nsprogressindicator-wrapper)
- [3. NSScrollView Wrapper](#3-nsscrollview-wrapper)
- [4. NSVisualEffectView Wrapper](#4-nsvisualeffectview-wrapper)
- [5. NSTableView Wrapper (Source-List)](#5-nstableview-wrapper-source-list)
- [6. NSStackView Wrapper](#6-nsstackview-wrapper)
- [7. NSTokenField Wrapper](#7-nstokenfield-wrapper)
- [8. NSPathControl Wrapper](#8-nspathcontrol-wrapper)

---

## 1. NSTextView Wrapper (Rich Text Editor)

Wrap `NSTextView` for rich text editing with `NSAttributedString` binding and placeholder support. `NSTextView` requires a proper text system setup when created outside of an `NSScrollView`.

```swift
import SwiftUI
import AppKit

struct RichTextEditor: NSViewRepresentable {
    @Binding var attributedText: NSAttributedString
    var placeholder: String = ""
    @Binding var isFirstResponder: Bool

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    func makeNSView(context: Context) -> NSScrollView {
        // NSTextView needs to be embedded in an NSScrollView for scrolling.
        let scrollView = NSScrollView()
        scrollView.hasVerticalScroller = true
        scrollView.autohidesScrollers = true
        scrollView.borderType = .noBorder

        let textView = NSTextView()
        textView.delegate = context.coordinator
        textView.isRichText = true
        textView.allowsUndo = true
        textView.isEditable = true
        textView.isSelectable = true
        textView.font = .preferredFont(forTextStyle: .body)
        textView.textContainerInset = NSSize(width: 8, height: 8)
        textView.drawsBackground = false
        textView.isAutomaticQuoteSubstitutionEnabled = false
        textView.isAutomaticDashSubstitutionEnabled = false

        // Placeholder text field
        let placeholderField = NSTextField(labelWithString: placeholder)
        placeholderField.textColor = .placeholderTextColor
        placeholderField.font = .preferredFont(forTextStyle: .body)
        placeholderField.translatesAutoresizingMaskIntoConstraints = false
        textView.addSubview(placeholderField)
        NSLayoutConstraint.activate([
            placeholderField.topAnchor.constraint(equalTo: textView.topAnchor, constant: 8),
            placeholderField.leadingAnchor.constraint(equalTo: textView.leadingAnchor, constant: 12),
        ])
        placeholderField.tag = 999

        scrollView.documentView = textView
        return scrollView
    }

    func updateNSView(_ nsView: NSScrollView, context: Context) {
        guard let textView = nsView.documentView as? NSTextView else { return }

        if textView.attributedString() != attributedText {
            textView.textStorage?.setAttributedString(attributedText)
        }

        // Placeholder visibility
        if let placeholderField = textView.viewWithTag(999) as? NSTextField {
            placeholderField.isHidden = !textView.string.isEmpty
        }

        // First responder management
        if isFirstResponder && textView.window?.firstResponder !== textView {
            textView.window?.makeFirstResponder(textView)
        } else if !isFirstResponder && textView.window?.firstResponder === textView {
                textView.window?.makeFirstResponder(nil)
        }
    }

    static func dismantleNSView(_ nsView: NSScrollView, coordinator: Coordinator) {
        coordinator.cancellables.removeAll()
    }

    @MainActor
    final class Coordinator: NSObject, NSTextViewDelegate {
        var parent: RichTextEditor
        var cancellables: Set<AnyCancellable> = []

        init(_ parent: RichTextEditor) { self.parent = parent }

        func textDidBeginEditing(_ notification: Notification) {
            parent.isFirstResponder = true
        }

        func textDidEndEditing(_ notification: Notification) {
            parent.isFirstResponder = false
        }

        func textDidChange(_ notification: Notification) {
            guard let textView = notification.object as? NSTextView else { return }
            parent.attributedText = textView.attributedString()

            if let placeholderField = textView.viewWithTag(999) as? NSTextField {
                placeholderField.isHidden = !textView.string.isEmpty
            }
        }
    }
}
```

### Usage

```swift
struct DocumentEditor: View {
    @State private var text = NSAttributedString(string: "")
    @State private var isFocused = false

    var body: some View {
        RichTextEditor(
            attributedText: $text,
            placeholder: "Start writing...",
            isFirstResponder: $isFocused
        )
        .frame(minHeight: 200)
    }
}
```

### Gotchas

- **Text system setup.** `NSTextView` requires an `NSTextContainer` and `NSLayoutManager`. When created programmatically, the text system is set up automatically. When embedding in `NSScrollView`, assign the text view as `documentView`.
- **Attributed string comparison.** Guard in `updateNSView` prevents re-render loops from keystroke-triggered delegate callbacks.
- **macOS 26 alternative.** `TextEditor` in macOS 26 supports `AttributedString` natively. Prefer it unless you need `NSAttributedString` or delegate-level control.

---

## 2. NSProgressIndicator Wrapper

Wrap `NSProgressIndicator` for determinate and indeterminate progress display with animation support.

```swift
import SwiftUI
import AppKit

struct ProgressIndicator: NSViewRepresentable {
    var progress: Double // 0.0 to 1.0
    var isIndeterminate: Bool = false
    var style: NSProgressIndicator.Style = .bar

    func makeNSView(context: Context) -> NSProgressIndicator {
        let indicator = NSProgressIndicator()
        indicator.style = style
        indicator.isIndeterminate = isIndeterminate
        indicator.minValue = 0
        indicator.maxValue = 1
        indicator.doubleValue = progress
        indicator.translatesAutoresizingMaskIntoConstraints = false
        return indicator
    }

    func updateNSView(_ nsView: NSProgressIndicator, context: Context) {
        nsView.isIndeterminate = isIndeterminate
        if !isIndeterminate {
            nsView.doubleValue = progress
        }
    }
}
```

### Usage

```swift
struct DownloadView: View {
    @State private var downloadProgress: Double = 0

    var body: some View {
        VStack(spacing: 12) {
            ProgressIndicator(progress: downloadProgress)
                .frame(height: 20)

            Text("\(Int(downloadProgress * 100))%")
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding()
        .onAppear {
            // Simulate progress
            Timer.scheduledTimer(withTimeInterval: 0.1, repeats: true) { timer in
                downloadProgress = min(downloadProgress + 0.02, 1.0)
                if downloadProgress >= 1.0 { timer.invalidate() }
            }
        }
    }
}
```

### Gotchas

- **Animation.** `NSProgressIndicator` animates value changes by default. To disable, call `nsView.animator().doubleValue = progress` only when you want animation; set `nsView.doubleValue` directly for instant updates.
- **Indeterminate mode.** When `isIndeterminate = true`, the `doubleValue` is ignored. The indicator shows a spinning animation (bar style) or pulsing animation (spinning style).
- **Threading.** Always update progress on the main thread. `NSProgressIndicator` is not thread-safe.

---

## 3. NSScrollView Wrapper

Wrap `NSScrollView` to provide a scrollable container for custom content views, with programmatic scroll control.

```swift
import SwiftUI
import AppKit

struct ScrollableView: NSViewRepresentable {
    var content: NSView
    @Binding var scrollPosition: NSPoint?
    var onScroll: ((NSPoint) -> Void)?

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    func makeNSView(context: Context) -> NSScrollView {
        let scrollView = NSScrollView()
        scrollView.hasVerticalScroller = true
        scrollView.hasHorizontalScroller = true
        scrollView.autohidesScrollers = true
        scrollView.drawsBackground = false

        // Configure the content view
        let clipView = NSClipView()
        clipView.drawsBackground = false
        scrollView.contentView = clipView

        scrollView.documentView = content

        // Observe content view bounds for scroll notifications
        context.coordinator.observeScrollView(scrollView)

        return scrollView
    }

    func updateNSView(_ nsView: NSScrollView, context: Context) {
        if nsView.documentView !== content {
            nsView.documentView = content
        }

        // Programmatic scroll
        if let position = scrollPosition {
            nsView.contentView.scroll(to: position)
            scrollPosition = nil // Consume the scroll request
        }
    }

    @MainActor
    final class Coordinator: NSObject {
        var parent: ScrollableView
        var observer: NSObjectProtocol?

        init(_ parent: ScrollableView) { self.parent = parent }

        func observeScrollView(_ scrollView: NSScrollView) {
            observer = NotificationCenter.default.addObserver(
                forName: NSView.boundsDidChangeNotification,
                object: scrollView.contentView,
                queue: .main
            ) { [weak self] _ in
                guard let self else { return }
                let point = scrollView.contentView.bounds.origin
                self.parent.onScroll?(point)
            }
        }

        deinit {
            if let observer {
                NotificationCenter.default.removeObserver(observer)
            }
        }
    }
}
```

### Usage

```swift
struct LogViewer: View {
    @State private var scrollToBottom = false
    @State private var logContent: NSView = {
        let text = NSTextView()
        text.isEditable = false
        text.font = .monospacedSystemFont(ofSize: 11, weight: .regular)
        text.string = ""
        return text
    }()
    @State private var scrollPosition: NSPoint?

    var body: some View {
        VStack {
            ScrollableView(
                content: logContent,
                scrollPosition: $scrollPosition,
                onScroll: { point in
                    // Track scroll position if needed
                }
            )

            Button("Scroll to Bottom") {
                if let docView = (logContent as? NSTextView) {
                    let maxY = max(0, docView.bounds.height - 100)
                    scrollPosition = NSPoint(x: 0, y: maxY)
                }
            }
        }
    }
}
```

### Gotchas

- **Document view sizing.** The document view must have an explicit frame or `intrinsicContentSize` for scrolling to work. Without it, the scroll view cannot compute the scrollable area.
- **Clip view notifications.** Use `NSView.boundsDidChangeNotification` on the content view (not the scrollView) to track scroll position changes.
- **Elastic scrolling.** AppKit enables elastic (bounce) scrolling by default. Disable with `scrollView.usesPredominantAxisScrolling = true` and `scrollView.allowsMagnification = false` if not needed.

---

## 4. NSVisualEffectView Wrapper

Wrap `NSVisualEffectView` for vibrancy, sidebar blur, and material backgrounds. Essential for modern macOS window chrome.

```swift
import SwiftUI
import AppKit

struct VisualEffectBackground: NSViewRepresentable {
    var material: NSVisualEffectView.Material = .sidebar
    var blendingMode: NSVisualEffectView.BlendingMode = .behindWindow
    var state: NSVisualEffectView.State = .followsWindowActiveState
    var isEmphasized: Bool = false

    func makeNSView(context: Context) -> NSVisualEffectView {
        let view = NSVisualEffectView()
        view.material = material
        view.blendingMode = blendingMode
        view.state = state
        view.isEmphasized = isEmphasized
        view.wantsLayer = true
        return view
    }

    func updateNSView(_ nsView: NSVisualEffectView, context: Context) {
        nsView.material = material
        nsView.blendingMode = blendingMode
        nsView.state = state
        nsView.isEmphasized = isEmphasized
    }
}
```

### Usage

```swift
struct SidebarView: View {
    var body: some View {
        VisualEffectBackground(material: .sidebar)
            .overlay {
                VStack(alignment: .leading, spacing: 4) {
                    Text("Sidebar")
                        .font(.headline)
                    Text("Content here")
                        .foregroundStyle(.secondary)
                }
                .padding()
                .frame(maxWidth: .infinity, alignment: .leading)
            }
    }
}
```

### Gotchas

- **Layer backing required.** `NSVisualEffectView` requires `wantsLayer = true`. Set it in `makeNSView`, not `updateNSView`, to avoid layer flash.
- **Blending mode.** `.behindWindow` blends with content behind the window (requires the window to be non-opaque). `.withinWindow` blends with content inside the window. Use `.withinWindow` for in-window blur effects.
- **Material availability.** Not all materials are available in every context. `.sidebar` and `.menu` are always available. `.hudWindow` and `.fullScreenUI` require specific window styles.
- **Vibrancy with text.** For text that should appear vibrant over a visual effect view, use `NSAttributedString` with the `.foregroundColor` set to `.labelColor` or `.secondaryLabelColor` -- AppKit handles vibrancy automatically.

---

## 5. NSTableView Wrapper (Source-List)

Wrap `NSTableView` for a source-list style outline with selection binding and delegate callbacks.

```swift
import SwiftUI
import AppKit

struct SourceList<Item: Identifiable & Hashable>: NSViewRepresentable {
    var items: [Item]
    @Binding var selection: Item.ID?
    var titleKeyPath: KeyPath<Item, String>
    var onDoubleClick: ((Item) -> Void)?

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    func makeNSView(context: Context) -> NSScrollView {
        let scrollView = NSScrollView()
        scrollView.hasVerticalScroller = true
        scrollView.autohidesScrollers = true
        scrollView.borderType = .noBorder

        let tableView = NSTableView()
        tableView.headerView = nil
        tableView.allowsEmptySelection = true
        tableView.allowsMultipleSelection = false
        tableView.style = .sourceList
        tableView.delegate = context.coordinator
        tableView.dataSource = context.coordinator

        let column = NSTableColumn(identifier: NSUserInterfaceItemIdentifier("main"))
        column.resizingMask = .autoresizingMask
        tableView.addTableColumn(column)

        scrollView.documentView = tableView
        return scrollView
    }

    func updateNSView(_ nsView: NSScrollView, context: Context) {
        guard let tableView = nsView.documentView as? NSTableView else { return }

        // Reload if items changed
        tableView.reloadData()

        // Sync selection
        if let selection,
           let index = items.firstIndex(where: { $0.id == selection }) {
            let indexSet = IndexSet(integer: index)
            if tableView.selectedRowIndexes != indexSet {
                tableView.selectRowIndexes(indexSet, byExtendingSelection: false)
            }
        } else if !tableView.selectedRowIndexes.isEmpty {
            tableView.deselectAll(nil)
        }
    }

    @MainActor
    final class Coordinator: NSObject, NSTableViewDelegate, NSTableViewDataSource {
        var parent: SourceList

        init(_ parent: SourceList) { self.parent = parent }

        func numberOfRows(in tableView: NSTableView) -> Int {
            parent.items.count
        }

        func tableView(
            _ tableView: NSTableView,
            viewFor tableColumn: NSTableColumn?,
            row: Int
        ) -> NSView? {
            let item = parent.items[row]
            let cell = NSTableCellView()
            let textField = NSTextField(labelWithString: item[keyPath: parent.titleKeyPath])
            textField.translatesAutoresizingMaskIntoConstraints = false
            cell.addSubview(textField)
            NSLayoutConstraint.activate([
                textField.leadingAnchor.constraint(equalTo: cell.leadingAnchor, constant: 4),
                textField.trailingAnchor.constraint(equalTo: cell.trailingAnchor, constant: -4),
                textField.centerYAnchor.constraint(equalTo: cell.centerYAnchor),
            ])
            cell.textField = textField
            return cell
        }

        func tableViewSelectionDidChange(_ notification: Notification) {
            guard let tableView = notification.object as? NSTableView else { return }
            if tableView.selectedRow >= 0 && tableView.selectedRow < parent.items.count {
                parent.selection = parent.items[tableView.selectedRow].id
            } else {
                parent.selection = nil
            }
        }

        func tableView(
            _ tableView: NSTableView,
            rowViewForRow row: Int
        ) -> NSTableRowView? {
            let rowView = SourceListRowView()
            rowView.isEmphasized = true
            return rowView
        }
    }
}

final class SourceListRowView: NSTableRowView {
    override var isEmphasized: Bool {
        get { true }
        set { }
    }
}
```

### Usage

```swift
struct ContentView: View {
    struct FileItem: Identifiable, Hashable {
        let id = UUID()
        let name: String
    }

    @State private var items = [
        FileItem(name: "Document.txt"),
        FileItem(name: "Notes.md"),
        FileItem(name: "Report.pdf"),
    ]
    @State private var selectedID: UUID?

    var body: some View {
        HStack(spacing: 0) {
            SourceList(
                items: items,
                selection: $selectedID,
                titleKeyPath: \.name
            )
            .frame(width: 200)

            Divider()

            if let id = selectedID,
               let item = items.first(where: { $0.id == id }) {
                Text("Selected: \(item.name}")
                    .frame(maxWidth: .infinity, maxHeight: .infinity)
            } else {
                Text("No selection")
                    .frame(maxWidth: .infinity, maxHeight: .infinity)
            }
        }
    }
}
```

### Gotchas

- **Source list style.** Setting `tableView.style = .sourceList` enables the blue highlight and rounded selection appearance. This must be set before the table is added to a scroll view.
- **Selection sync.** The coordinator writes to the `@Binding` on selection change; `updateNSView` reads it back. Guard against feedback loops by checking if the selection already matches.
- **Diffing.** For large lists, use `tableView.reloadData()` sparingly. Consider `NSDiffableDataSourceSnapshot` equivalent or batch updates for performance.

---

## 6. NSStackView Wrapper

Wrap `NSStackView` for Auto Layout-driven horizontal or vertical stacks that don't map cleanly to SwiftUI's `HStack`/`VStack` (e.g., when you need AppKit-specific alignment or distribution).

```swift
import SwiftUI
import AppKit

struct AppKitStack: NSViewRepresentable {
    var orientation: NSUserInterfaceLayoutOrientation = .horizontal
    var spacing: CGFloat = 8
    var distribution: NSStackView.Distribution = .fill
    var alignment: NSLayoutConstraint.Attribute = .centerY
    var views: [NSView]

    func makeNSView(context: Context) -> NSStackView {
        let stack = NSStackView()
        stack.orientation = orientation
        stack.spacing = spacing
        stack.distribution = distribution
        stack.alignment = alignment
        stack.translatesAutoresizingMaskIntoConstraints = false

        for view in views {
            stack.addArrangedSubview(view)
        }

        return stack
    }

    func updateNSView(_ nsView: NSStackView, context: Context) {
        nsView.orientation = orientation
        nsView.spacing = spacing
        nsView.distribution = distribution
        nsView.alignment = alignment

        // Diff arranged subviews
        let current = Set(nsView.arrangedSubviews)
        let incoming = Set(views)

        for view in current.subtracting(incoming) {
            nsView.removeArrangedSubview(view)
            view.removeFromSuperview()
        }

        for (index, view) in views.enumerated() {
            if !current.contains(view) {
                nsView.insertArrangedSubview(view, at: index)
            }
        }
    }
}
```

### Gotchas

- **Remove vs removeFromSuperview.** Always call `removeArrangedSubview` before `removeFromSuperview`. The former removes from the arranged list; the latter removes from the view hierarchy.
- **Intrinsic content size.** `NSStackView` has no intrinsic content size by default. If used in a scroll view, add explicit constraints or use `.intrinsicContentSize` on the stack.

---

## 7. NSTokenField Wrapper

Wrap `NSTokenField` for tokenized input (e.g., tags, email recipients) with binding support.

```swift
import SwiftUI
import AppKit

struct TokenField: NSViewRepresentable {
    @Binding var tokens: [String]
    var placeholder: String = ""
    var tokenizingCharacterSet: CharacterSet = .newlines

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    func makeNSView(context: Context) -> NSTokenField {
        let field = NSTokenField()
        field.delegate = context.coordinator
        field.placeholderString = placeholder
        field.tokenStyle = .rounded
        field.stringValue = tokens.joined(separator: "\n")
        return field
    }

    func updateNSView(_ nsView: NSTokenField, context: Context) {
        let currentTokens = nsView.objectValue as? [String] ?? []
        if currentTokens != tokens {
            nsView.objectValue = tokens
        }
    }

    @MainActor
    final class Coordinator: NSObject, NSTokenFieldDelegate {
        var parent: TokenField

        init(_ parent: TokenField) { self.parent = parent }

        func tokenField(
            _ tokenField: NSTokenField,
            representedObjectForEditing editingString: String
        ) -> Any? {
            editingString.trimmingCharacters(in: .whitespaces)
        }

        func tokenField(
            _ tokenField: NSTokenField,
            displayStringForRepresentedObject representedObject: Any
        ) -> String? {
            representedObject as? String
        }

        func tokenField(
            _ tokenField: NSTokenField,
            completionsForSubstring substring: String,
            indexOfToken tokenIndex: Int,
            indexOfSelectedItem selectedIndex: UnsafeMutablePointer<Int>?
        ) -> [Any]? {
            // Return completion candidates based on substring
            []
        }

        func controlTextDidChange(_ obj: Notification) {
            guard let field = obj.object as? NSTokenField else { return }
            parent.tokens = field.objectValue as? [String] ?? []
        }
    }
}
```

### Gotchas

- **Object value vs string value.** `NSTokenField` uses `objectValue` (an array of represented objects) rather than `stringValue` for its content. Always use `objectValue` for token manipulation.
- **Tokenizing characters.** The default tokenizing character is newline. Customize `tokenizingCharacterSet` for comma-separated or space-separated tokens.

---

## 8. NSPathControl Wrapper

Wrap `NSPathControl` for a clickable file path display with navigation support.

```swift
import SwiftUI
import AppKit

struct PathControl: NSViewRepresentable {
    var url: URL?
    var onPathChange: ((URL) -> Void)?

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    func makeNSView(context: Context) -> NSPathControl {
        let control = NSPathControl()
        control.delegate = context.coordinator
        control.isEditable = false
        control.pathStyle = .standard
        control.translatesAutoresizingMaskIntoConstraints = false
        return control
    }

    func updateNSView(_ nsView: NSPathControl, context: Context) {
        if nsView.url != url {
            nsView.url = url
        }
    }

    @MainActor
    final class Coordinator: NSObject, NSPathControlDelegate {
        var parent: PathControl

        init(_ parent: PathControl) { self.parent = parent }

        func pathControl(
            _ pathControl: NSPathControl,
            willDisplay openPanel: NSOpenPanel
        ) {
            openPanel.canChooseDirectories = true
            openPanel.canChooseFiles = false
        }

        func pathControl(
            _ pathControl: NSPathControl,
            validateDrop info: any NSDraggingInfo
        ) -> NSDragOperation {
            .copy
        }

        func pathControl(
            _ pathControl: NSPathControl,
            acceptDrop info: any NSDraggingInfo
        ) -> Bool {
            guard let urls = info.draggingPasteboard.readObjects(forClasses: [NSURL.self]) as? [URL],
                  let url = urls.first else { return false }
            parent.onPathChange?(url)
            return true
        }
    }
}
```

### Usage

```swift
struct FileBrowser: View {
    @State private var currentURL: URL?

    var body: some View {
        VStack {
            PathControl(url: currentURL) { newURL in
                currentURL = newURL
            }
            .padding()

            if let url = currentURL {
                Text("Current: \(url.path)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }
        }
    }
}
```

### Gotchas

- **Drag and drop.** `NSPathControl` supports drag and drop natively. Implement `validateDrop` and `acceptDrop` in the coordinator to handle dropped files/folders.
- **Path style.** `.standard` shows the full path with clickable components. `.popUp` shows a dropdown menu. `.bar` shows a compact bar style.
