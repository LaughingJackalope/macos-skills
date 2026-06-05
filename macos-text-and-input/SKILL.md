---
name: macos-text-and-input
description: "TextKit 2 architecture, NSTextView configuration, NSAttributedString editing, find bar with NSTextFinder, keyboard event handling via NSTextInputClient, font panel integration, and wrapping NSTextView as NSViewRepresentable for SwiftUI. Use when building rich text editors, code editors, or any custom text input experience on macOS."
sources: https://developer.apple.com/videos/play/wwdc2022/10090/, https://developer.apple.com/documentation/appkit/nstextlayoutmanager, https://indiestack.com/2022/11/opting-out-of-textkit2-in-nstextview/, https://blog.krzyzanowskim.com/2025/08/14/textkit-2-the-promised-land/
researched: 2026-06-05
platform: macOS 26+
---

# macOS Text & Input

TextKit 2 on macOS 26+ with Swift 6.3. Covers the TextKit 2 architecture (NSTextLayoutManager, NSTextContentStorage, NSTextViewportLayoutController), NSTextView configuration, NSAttributedString editing with attachments, find bar via NSTextFinder, keyboard event handling through NSTextInputClient, font panel integration, and wrapping NSTextView as NSViewRepresentable for SwiftUI. For SwiftUI TextEditor basics, see `swiftui-patterns`. For AppKit fundamentals, see `appkit-foundations`. For NSViewRepresentable interop patterns, see `swiftui-appkit-interop`.

See [references/textkit2-nstext-view.md](references/textkit2-nstext-view.md) for the complete TextKit 2 architecture deep-dive, NSTextView wrapping recipe, find bar implementation, keyboard event handling, and font panel integration reference.

## Contents

- [TextKit 2 Architecture](#textkit-2-architecture)
- [NSTextView Setup](#nstextview-setup)
- [NSAttributedString Editing](#nsattributedstring-editing)
- [SwiftUI TextEditor vs NSTextView](#swiftui-texteditor-vs-nstextview)
- [Wrapping NSTextView as NSViewRepresentable](#wrapping-nstextview-as-nsviewrepresentable)
- [Find Bar: NSTextFinder](#find-bar-nstextfinder)
- [Keyboard Event Handling](#keyboard-event-handling)
- [NSFont and Font Panel](#nsfont-and-font-panel)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## TextKit 2 Architecture

TextKit 2 is a complete rewrite of the text system. It replaces TextKit 1's `NSLayoutManager`/`NSTextContainer` pipeline with three cooperating objects that separate concerns cleanly:

| Component | Role |
|---|---|
| `NSTextContentStorage` | Owns the attributed string and notifies the layout manager of edits. Replaces `NSTextStorage`. |
| `NSTextLayoutManager` | Performs line layout, glyph generation, and geometry. Replaces `NSLayoutManager`. |
| `NSTextViewportLayoutController` | Manages which portion of the layout is visible and drives viewport-based rendering. |

**Key difference from TextKit 1:** TextKit 1 used a single `NSLayoutManager` connected to multiple `NSTextContainer`s (for multi-column or multi-page layouts). TextKit 2 inverts this: each `NSTextLayoutManager` has exactly one `NSTextElementProvider` (the content storage) and one viewport controller. Multi-column layouts use multiple layout managers sharing one content storage.

```swift
// TextKit 2 pipeline setup
let textContentStorage = NSTextContentStorage()
textContentStorage.attributedString = NSAttributedString(string: "Hello, world!")

let textLayoutManager = NSTextLayoutManager()
textContentStorage.addTextLayoutManager(textLayoutManager)

let viewportController = NSTextViewportLayoutController(textLayoutManager: textLayoutManager)
viewportController.viewportBounds = CGRect(x: 0, y: 0, width: 600, height: 800)
```

**NSTextView and TextKit 2:** On macOS, `NSTextView` uses TextKit 2 by default. You can access the TextKit 2 objects through `NSTextView.textLayoutManager`, `NSTextView.textContentManager`, and `NSTextView.viewportLayoutController`. If you need TextKit 1 compatibility (e.g., for legacy `NSLayoutManager` delegate code), set `NSTextView.usesTextKit2 = false` — but this is discouraged for new code.

**NSTextFragment vs. NSTextLineFragment:** In TextKit 2, layout is element-based. `NSTextParagraph` represents a paragraph (delimited by newlines or paragraph separators). Each paragraph produces `NSTextLineFragment` objects during layout. You access these through `NSTextLayoutManager.enumerateTextLayoutFragments(from:options:using:)` — never by index, since layout is lazy and viewport-dependent.

```swift
// Enumerate visible line fragments
textLayoutManager.enumerateTextLayoutFragments(
    from: nil,
    options: [.ensuresLayout, .ensuresExtraLineFragment]
) { fragment in
    let bounds = fragment.layoutFragmentFrame
    let textElements = fragment.textElements
    // Each fragment corresponds to a visible line
    return true // continue enumeration
}
```

**Rendering:** TextKit 2 uses `NSTextLayoutFragment` for rendering. Each layout fragment draws its content via `draw(at:in:)`. For custom rendering (e.g., syntax highlighting), subclass `NSTextLayoutFragment` and override `draw(at:in:)`.

## NSTextView Setup

`NSTextView` is the full-featured text editing view. Configure it through its delegate and a handful of critical properties.

```swift
final class TextViewController: NSViewController {
    var textView: NSTextView!

    override func loadView() {
        // Create a scroll view containing the text view
        let scrollView = NSTextView.scrollableTextView()
        textView = scrollView.documentView as? NSTextView
        view = scrollView
    }

    override func viewDidLoad() {
        super.viewDidLoad()
        configureTextView()
    }

    private func configureTextView() {
        textView.delegate = self

        // Core behavior flags
        textView.isRichText = true               // allow NSAttributedString attributes
        textView.allowsUndo = true               // enable built-in undo/redo
        textView.isEditable = true
        textView.isSelectable = true

        // Find bar (requires a scroll view ancestor)
        textView.usesFindBar = true
        textView.isIncrementalSearchingEnabled = true

        // Appearance
        textView.font = NSFont.systemFont(ofSize: 14)
        textView.textColor = .labelColor
        textView.backgroundColor = .textBackgroundColor
        textView.insertionPointColor = .controlAccentColor

        // Line wrapping
        textView.isHorizontallyResizable = false
        textView.textContainer?.widthTracksTextView = true
        textView.textContainer?.containerSize = NSSize(
            width: CGFloat.greatestFiniteMagnitude,
            height: CGFloat.greatestFiniteMagnitude
        )

        // Smart substitutions
        textView.isSmartInsertDeleteEnabled = true
        textView.isAutomaticQuoteSubstitutionEnabled = true
        textView.isAutomaticDashSubstitutionEnabled = true
        textView.isAutomaticTextReplacementEnabled = true

        // Grammar checking (macOS 14+)
        textView.isGrammarCheckingEnabled = false  // opt-in, not default

        // Ruler
        textView.usesRuler = false
        textView.isRulerVisible = false
    }
}
```

**Delegate:** `NSTextViewDelegate` (inherits from `NSTextDelegate`) provides callbacks for text changes, selection changes, and typing attributes. Use `textDidChange(_:)` for reacting to edits, `textView:shouldChangeTextIn:replacementString:` for validation, and `textViewDidChangeSelection(_:)` for selection tracking.

```swift
extension TextViewController: NSTextViewDelegate {
    func textDidChange(_ notification: Notification) {
        guard let textView = notification.object as? NSTextView else { return }
        // React to text changes
        updateWordCount(textView.string)
    }

    func textView(_ textView: NSTextView,
                  shouldChangeTextIn affectedCharRange: NSRange,
                  replacementString: String?) -> Bool {
        // Return false to reject the edit
        return true
    }

    func textViewDidChangeSelection(_ notification: Notification) {
        guard let textView = notification.object as? NSTextView else { return }
        let range = textView.selectedRange()
        updateSelectionUI(range)
    }
}
```

**isRichText:** When `false`, the text view strips all attributes on input and stores plain `String`. When `true`, it preserves `NSAttributedString` attributes. Set this before setting content — toggling it after the fact does not convert existing content.

**allowsUndo:** Always `true` unless you have a specific reason to disable it. The undo manager is per-text-view via `textView.undoManager`. For custom undo grouping, wrap edits in `shouldChangeTextIn`/`didChangeText` and use `undoManager?.beginUndoGrouping()`.

## NSAttributedString Editing

`NSAttributedString` is the fundamental document model for rich text. On macOS, it is always mutable through `NSMutableAttributedString` for in-place editing.

**Common attributes:**

```swift
let attributes: [NSAttributedString.Key: Any] = [
    .font: NSFont.systemFont(ofSize: 14),
    .foregroundColor: NSColor.labelColor,
    .backgroundColor: NSColor.textBackgroundColor,
    .underlineStyle: NSUnderlineStyle.single.rawValue,
    .strikethroughStyle: NSUnderlineStyle.single.rawValue,
    .link: URL(string: "https://example.com")!,
    .paragraphStyle: {
        let style = NSMutableParagraphStyle()
        style.lineSpacing = 4
        style.paragraphSpacing = 8
        style.alignment = .left
        return style
    }()
]
```

**NSTextAttachment:** Use `NSTextAttachment` to embed images or custom views inline with text. On macOS, `NSTextAttachment` uses an `NSTextAttachmentCell` for rendering.

```swift
// Image attachment
let attachment = NSTextAttachment()
attachment.image = NSImage(named: "icon")

let attachmentString = NSAttributedString(attachment: attachment)
textView.textStorage?.append(attachmentString)

// Custom attachment cell
final class CustomAttachmentCell: NSTextAttachmentCell {
    override func cellSize() -> NSSize {
        return NSSize(width: 200, height: 100)
    }

    override func draw(withFrame cellFrame: NSRect, in controlView: NSView?) {
        // Custom drawing
        NSColor.systemBlue.setFill()
        cellFrame.fill()
    }

    override func highlight(_ flag: Bool, withFrame cellFrame: NSRect, in controlView: NSView?) {
        if flag {
            NSColor.systemBlue.withAlphaComponent(0.3).setFill()
            cellFrame.fill()
        }
    }
}

let customAttachment = NSTextAttachment()
customAttachment.attachmentCell = CustomAttachmentCell()
```

**Editing the text storage:** Always edit through `textStorage` (an `NSTextStorage` subclass in TextKit 2, which bridges to `NSTextContentStorage`). Use `beginEditing()`/`endEditing()` to batch changes and coalesce undo operations.

```swift
guard let storage = textView.textStorage else { return }
storage.beginEditing()
storage.replaceCharacters(in: NSRange(location: 0, length: 5),
                          with: NSAttributedString(string: "Hello",
                                                   attributes: [.font: NSFont.boldSystemFont(ofSize: 16)]))
storage.endEditing()  // coalesces notifications and undo
```

**Paragraph style:** Use `NSMutableParagraphStyle` for alignment, line spacing, headings, and indentation. For code editors, set `lineBreakMode` via `NSParagraphStyle`:

```swift
let codeStyle = NSMutableParagraphStyle()
codeStyle.lineBreakMode = .byCharWrapping  // no word wrapping for code
codeStyle.tabStops = (1...20).map { i in
    NSTextTab(type: .leftTabStopType, location: CGFloat(i) * 28)
}
codeStyle.defaultTabInterval = 28
```

## SwiftUI TextEditor vs NSTextView

SwiftUI's `TextEditor` on macOS is a thin wrapper around `NSTextView` with significant limitations:

| Capability | SwiftUI TextEditor | NSTextView |
|---|---|---|
| Rich text (NSAttributedString) | macOS 15+ only (AttributedString) | Full support |
| Find bar | Not available | `usesFindBar = true` |
| Font panel | Not available | Full integration |
| Custom text attachments | Not available | NSTextAttachment |
| Line numbers | Not available | Custom layout fragment |
| Grammar/spell checking | Limited | Full control |
| Incremental search | Not available | `isIncrementalSearchingEnabled` |
| Ruler | Not available | `usesRuler` |
| Custom key handling | Not available | NSTextInputClient |

**When to drop to NSTextView:**
- Rich text editing with attributes, attachments, or custom paragraph styles
- Code editors (line numbers, syntax highlighting, no word wrap)
- Find/replace UI
- Font panel integration
- Custom keyboard handling (e.g., Emacs keybindings, Vim mode)
- Any text view that needs `NSTextViewDelegate` callbacks beyond basic change notification

**When SwiftUI TextEditor is sufficient:**
- Plain text input (notes, comments, forms)
- macOS 15+ `AttributedString`-based rich text without attachments
- Simple multi-line text fields where AppKit interop is overkill

## Wrapping NSTextView as NSViewRepresentable

To use `NSTextView` in SwiftUI, wrap it in `NSViewRepresentable`. The key challenge is managing the text view's lifecycle and bridging state changes bidirectionally.

```swift
import SwiftUI
import AppKit

struct TextViewRepresentable: NSViewRepresentable {
    @Binding var text: String
    var font: NSFont = .systemFont(ofSize: 14)
    var isEditable: Bool = true
    var onTextChange: ((String) -> Void)?

    func makeNSView(context: Context) -> NSScrollView {
        let scrollView = NSTextView.scrollableTextView()
        let textView = scrollView.documentView as! NSTextView

        textView.delegate = context.coordinator
        textView.font = font
        textView.isEditable = isEditable
        textView.isRichText = false
        textView.allowsUndo = true
        textView.usesFindBar = true
        textView.isIncrementalSearchingEnabled = true

        // Configure text container for word wrap
        textView.isHorizontallyResizable = false
        textView.textContainer?.widthTracksTextView = true
        textView.textContainer?.containerSize = NSSize(
            width: CGFloat.greatestFiniteMagnitude,
            height: CGFloat.greatestFiniteMagnitude
        )

        context.coordinator.textView = textView
        return scrollView
    }

    func updateNSView(_ scrollView: NSScrollView, context: Context) {
        guard let textView = scrollView.documentView as? NSTextView else { return }

        // Only update if text actually changed to avoid cursor jumps
        if textView.string != text {
            textView.string = text
        }
        textView.font = font
        textView.isEditable = isEditable
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(text: $text, onTextChange: onTextChange)
    }

    @MainActor
    final class Coordinator: NSObject, NSTextViewDelegate {
        @Binding var text: String
        var textView: NSTextView?
        var onTextChange: ((String) -> Void)?

        init(text: Binding<String>, onTextChange: ((String) -> Void)?) {
            _text = text
            self.onTextChange = onTextChange
        }

        func textDidChange(_ notification: Notification) {
            guard let textView = notification.object as? NSTextView else { return }
            let newText = textView.string
            text = newText
            onTextChange?(newText)
        }
    }
}

// SwiftUI usage
struct ContentView: View {
    @State private var text = "Type here..."

    var body: some View {
        TextViewRepresentable(text: $text, font: .monospacedSystemFont(ofSize: 13, weight: .regular))
            .frame(minWidth: 400, minHeight: 300)
    }
}
```

**Critical detail:** Always check `textView.string != text` in `updateNSView` before assigning. Without this guard, every keystroke triggers `updateNSView`, which resets the entire string and moves the cursor to the end — making the editor unusable.

**For rich text (NSAttributedString):** Replace the `String` binding with `NSAttributedString` and use `textView.textStorage?.setAttributedString(...)` in `updateNSView`. Be aware that `NSAttributedString` is a reference type — use a custom binding or `onChange` to detect actual content changes.

## Find Bar: NSTextFinder

`NSTextFinder` provides the standard macOS find bar (activated with ⌘F). It requires a scroll view ancestor and a find bar client.

```swift
final class FindableTextView: NSTextView {
    override func awakeFromNib() {
        super.awakeFromNib()
        usesFindBar = true
        isIncrementalSearchingEnabled = true
    }
}

// The find bar client (usually the view controller or the text view itself)
final class FindClient: NSObject, NSTextFinderClient {
    weak var textView: NSTextView?

    init(textView: NSTextView) {
        self.textView = textView
    }

    // NSTextFinderClient
    var string: String {
        textView?.string ?? ""
    }

    var firstSelectedRange: NSRange {
        textView?.selectedRange() ?? NSRange(location: 0, length: 0)
    }

    func shouldReplaceCharacters(inRanges ranges: [NSValue], withStrings strings: [String]) -> Bool {
        return true
    }

    func replaceCharacters(in range: NSRange, with string: String) {
        guard let textView,
              let storage = textView.textStorage,
              textView.shouldChangeText(in: range, replacementString: string) else { return }
        storage.replaceCharacters(in: range, with: string)
        textView.didChangeText()
    }

    func scrollRangeToVisible(_ range: NSRange) {
        textView?.scrollRangeToVisible(range)
    }

    func contentString(for range: NSRange, attributes effectiveAttributes: AutoreleasingUnsafeMutablePointer<NSDictionary?>?) -> NSAttributedString? {
        return textView?.textStorage?.attributedSubstring(from: range)
    }

    func rects(for range: NSRange) -> [NSValue]? {
        guard let textView else { return nil }
        // Convert text range to rects for find highlight
        let layoutManager = textView.layoutManager
        let textContainer = textView.textContainer
        var rects: [NSValue] = []
        let glyphRange = layoutManager?.glyphRange(forCharacterRange: range, actualCharacterRange: nil) ?? NSRange()
        layoutManager?.enumerateEnclosingRects(forGlyphRange: glyphRange,
                                                withinSelectedGlyphRange: NSRange(location: NSNotFound, length: 0),
                                                in: textContainer!) { rect, _ in
            rects.append(NSValue(rect: rect))
        }
        return rects.isEmpty ? nil : rects
    }

    var allowsMultipleSelection: Bool { false }
    var isEditable: Bool { textView?.isEditable ?? false }
    var visibleCharacterRanges: [NSValue] {
        guard let textView else { return [] }
        let visibleRect = textView.visibleRect
        guard let layoutManager = textView.layoutManager,
              let textContainer = textView.textContainer else { return [] }
        let glyphRange = layoutManager.glyphRange(forBoundingRect: visibleRect, in: textContainer)
        let charRange = layoutManager.characterRange(forGlyphRange: glyphRange, actualGlyphRange: nil)
        return [NSValue(range: charRange)]
    }
}

// Wiring it up
let textView = scrollView.documentView as? NSTextView
let findClient = FindClient(textView: textView!)
let textFinder = NSTextFinder()
textFinder.findBarContainer = scrollView
textFinder.client = findClient
```

**Incremental search:** When `isIncrementalSearchingEnabled = true`, the find bar highlights matches as the user types. The `NSTextFinder` handles this automatically when the client is properly configured.

**Custom find bar:** If you need a custom find bar (not the system one), set `usesFindBar = false` and implement your own UI. Use `NSTextFinder` programmatically:

```swift
textFinder.performAction(.showFindInterface)
textFinder.performAction(.nextMatch)
textFinder.performAction(.previousMatch)
textFinder.performAction(.replaceAndFind)
```

## Keyboard Event Handling

`NSTextView` handles most keyboard input automatically through its `NSTextInputClient` conformance. For custom keyboard behavior, you have three levels of control:

**Level 1: `performKeyEquivalent(_:)`** — Intercept key equivalents (⌘S, ⌘Z, etc.) before the text view processes them. Override in a custom `NSTextView` subclass:

```swift
final class CustomTextView: NSTextView {
    override func performKeyEquivalent(with event: NSEvent) -> Bool {
        // Intercept Cmd+Return for custom action
        if event.modifierFlags.contains(.command) && event.keyCode == 36 {
            sendAction(#selector(delegate?.customTextViewDidPressCmdReturn(_:)), to: delegate)
            return true  // consumed
        }
        return super.performKeyEquivalent(with: event)
    }
}
```

**Level 2: `keyDown(with:)` and `interpretKeyEvents(_:)`** — Handle raw key events. Call `interpretKeyEvents` for keys the text view should process normally (character input, arrow keys, etc.):

```swift
override func keyDown(with event: NSEvent) {
    if event.modifierFlags.contains(.command) && event.charactersIgnoringModifiers == "j" {
        // Custom Cmd+J handling
        moveToNextPlaceholder()
        return
    }
    // Pass everything else to the input context for standard processing
    interpretKeyEvents([event])
}
```

**Level 3: `NSTextInputClient`** — The full protocol for text input. Implement this when you need complete control over text input (e.g., for a custom text view that doesn't inherit from `NSTextView`). Key methods:

```swift
// NSTextInputClient protocol methods you must implement:

func insertText(_ string: Any, replacementRange: NSRange) {
    // Insert text at the given range (or selection if range is NSNotFound)
    let str = string as? String ?? (string as! NSAttributedString).string
    let range = replacementRange.location == NSNotFound ? selectedRange() : replacementRange
    textStorage?.replaceCharacters(in: range, with: str)
    setSelectedRange(NSRange(location: range.location + str.count, length: 0))
}

func setMarkedText(_ string: Any, selectedRange: NSRange, replacementRange: NSRange) {
    // For IME input (CJK, etc.) — mark provisional text
    let str = (string as? String) ?? ""
    // Display str as marked (underlined) text at the replacement range
    // Store for later confirmation via insertText
}

func unmarkText() {
    // Confirm or discard marked text
}

func selectedRange() -> NSRange {
    return currentSelectionRange
}

func markedRange() -> NSRange {
    return markedTextRange  // NSRange(location: NSNotFound, length: 0) if none
}

func hasMarkedText() -> Bool {
    return markedRange().location != NSNotFound
}

func attributedSubstring(forProposedRange: NSRange, actualRange: NSRangePointer) -> NSAttributedString? {
    return textStorage?.attributedSubstring(from: forProposedRange)
}

func firstRect(forCharacterRange: NSRange, actualRange: NSRangePointer) -> NSRect {
    // Return the rect for the first character of the range (for IME candidate window)
    return caretRect
}

func characterIndex(for point: NSPoint) -> Int {
    // Convert a point to a character index
    return layoutManager?.glyphIndex(for: point, in: textContainer!) ?? 0
}
```

**Key codes reference:** Common key codes for `NSEvent.keyCode`:
- 53: Escape
- 36: Return/Enter
- 48: Tab
- 51: Delete (backspace)
- 123-126: Arrow keys (left, right, down, up)
- 115: Home
- 119: End
- 117: Forward delete
- 121: Page down
- 116: Page up

## NSFont and Font Panel

`NSFontManager` and `NSFontPanel` provide the standard macOS font management UI. `NSTextView` integrates with them automatically when the responder chain is intact.

**Showing the font panel:**

```swift
// From any responder in the chain:
NSFontManager.shared.orderFrontFontPanel(nil)

// Or from a menu item:
@objc func showFontPanel(_ sender: Any?) {
    let fontManager = NSFontManager.shared
    fontManager.target = self
    fontManager.orderFrontFontPanel(nil)
}
```

**Responding to font changes:** Implement `changeFont(_:)` in a responder. The font panel sends this up the responder chain:

```swift
// In your NSTextView subclass or view controller:
@objc func changeFont(_ sender: NSFontManager?) {
    guard let fontManager = sender,
          let textView,
          let storage = textView.textStorage else { return }

    let range = textView.selectedRange()
    guard range.length > 0 else {
        // No selection — change the typing attributes (font for next input)
        let newFont = fontManager.convert(textView.font ?? NSFont.systemFont(ofSize: 14))
        textView.typingAttributes[.font] = newFont
        return
    }

    // Apply to selection
    let newFont = fontManager.convert(storage.attribute(.font, at: range.location, effectiveRange: nil) as? Font
                                      ?? NSFont.systemFont(ofSize: 14))
    storage.addAttribute(.font, value: newFont, range: range)
}
```

**NSFont common patterns:**

```swift
// System fonts
let body = NSFont.systemFont(ofSize: NSFont.systemFontSize)
let bold = NSFont.boldSystemFont(ofSize: NSFont.systemFontSize)
let monospaced = NSFont.monospacedSystemFont(ofSize: 13, weight: .regular)
let monospacedDigit = NSFont.monospacedDigitSystemFont(ofSize: 13, weight: .regular)

// Dynamic type (macOS 11+)
let largeTitle = NSFont.preferredFont(forTextStyle: .largeTitle)
let bodyText = NSFont.preferredFont(forTextStyle: .body)
let caption = NSFont.preferredFont(forTextStyle: .caption1)

// Custom font
if let customFont = NSFont(name: "SF Mono", size: 12) {
    textView.font = customFont
}

// Font descriptor for precise control
let descriptor = NSFontDescriptor(fontAttributes: [
    .family: "SF Mono",
    .traits: [NSFontTraitMask: NSFontWeightRegular]
])
let font = NSFont(descriptor: descriptor, size: 12)
```

**Validating the font panel:** Implement `validModesForFontPanel(_:)` to control which font panel modes are available:

```swift
override func validModesForFontPanel(_ fontPanel: NSFontPanel) -> NSFontPanel.ModeMask {
    return [.collectionMask, .faceMask, .sizeMask]
    // Exclude .underlineMask, .strikethroughMask, .textColorMask, etc.
}
```

**NSFontPanel target:** The font panel's target must respond to `changeFont(_:)`. If your text view is in a scroll view inside a view controller, ensure the view controller is in the responder chain and implements `changeFont(_:)`, or subclass `NSTextView` to handle it directly.

## Constraints & Gotchas

- **TextKit 2 is the default.** `NSTextView` uses TextKit 2 on macOS. Access `textLayoutManager` for TextKit 2 APIs. Do not mix TextKit 1 (`NSLayoutManager`) and TextKit 2 (`NSTextLayoutManager`) APIs on the same text view.
- **NSTextView requires a scroll view for find bar.** `usesFindBar = true` only works when the text view is inside an `NSScrollView`. The scroll view is the `findBarContainer`.
- **Undo manager is per-text-view.** `textView.undoManager` returns the text view's undo manager. Do not replace it — it is tightly integrated with the text storage's edit tracking.
- **AttributedString bridging.** SwiftUI's `AttributedString` and AppKit's `NSAttributedString` bridge automatically, but not all attributes survive the round-trip. Test custom attributes carefully.
- **Font panel requires responder chain.** The font panel sends `changeFont(_:)` up the responder chain. If your SwiftUI `NSViewRepresentable` coordinator is not in the chain, the font panel will not work. Override `viewDidMoveToWindow()` to ensure the text view becomes first responder.
- **Text container insets.** `NSTextView.textContainerInset` (macOS 14+) adds padding inside the text view. Set this for comfortable margins: `textView.textContainerInset = NSSize(width: 8, height: 8)`.
- **Line fragment padding.** `textContainer.lineFragmentPadding` adds padding to each line. Set to 0 for code editors that need text to reach the edge.

## Common Mistakes

- **Setting text in `updateNSView` without a guard.** This causes cursor to jump to end on every keystroke. Always compare before assigning.
- **Not calling `super` in `performKeyEquivalent`.** If you override and return `false` for unhandled keys, the text view never sees them. Always call `super` for keys you don't handle.
- **Editing `NSTextStorage` outside `beginEditing()`/`endEditing()`.** This causes inconsistent state and crashes. Always wrap edits.
- **Using `NSLayoutManager` APIs with TextKit 2.** If `usesTextKit2` is true (the default), use `NSTextLayoutManager` instead. The TextKit 1 APIs will fail or produce undefined behavior.
- **Forgetting `isEditable`.** A text view that is not editable still shows the insertion point and accepts keyboard focus by default. Set `isEditable = false` for read-only views.
- **Not setting `textContainer?.widthTracksTextView = true`.** Without this, the text container has infinite width and text never wraps.

## Review Checklist

- [ ] TextKit 2 APIs used (NSTextLayoutManager, NSTextContentStorage) — not mixing with TextKit 1
- [ ] NSTextView delegate set for text/selection change callbacks
- [ ] `usesFindBar = true` only when text view is inside an NSScrollView
- [ ] `updateNSView` guards against unnecessary text assignment (cursor position preserved)
- [ ] `beginEditing()`/`endEditing()` wraps all NSTextStorage mutations
- [ ] `isRichText` set correctly for the use case
- [ ] Font panel target is in the responder chain and implements `changeFont(_:)`
- [ ] `performKeyEquivalent` calls `super` for unhandled keys
- [ ] `textContainerInset` and `lineFragmentPadding` configured for the layout
- [ ] `allowsUndo = true` for editable text views

## References

- [WWDC 2022: What's new in TextKit 2](https://developer.apple.com/videos/play/wwdc2022/10090/)
- [NSTextLayoutManager Documentation](https://developer.apple.com/documentation/appkit/nstextlayoutmanager)
- [Opting out of TextKit 2 in NSTextView](https://indiestack.com/2022/11/opting-out-of-textkit2-in-nstextview/)
- [TextKit 2: The Promised Land](https://blog.krzyzanowskim.com/2025/08/14/textkit-2-the-promised-land/)
