# TextKit 2 & NSTextView Reference

Deep reference for TextKit 2 architecture, NSTextView wrapping recipe, find bar implementation, keyboard event handling, and font panel integration on macOS 26+.

## TextKit 2 Architecture

### The Three Pillars

TextKit 2 separates text layout into three cooperating objects:

```
NSTextContentStorage (model)
        │
        ▼
NSTextLayoutManager (layout engine)
        │
        ▼
NSTextViewportLayoutController (viewport management)
```

**NSTextContentStorage** replaces `NSTextStorage`. It owns the `NSAttributedString` and acts as an `NSTextElementProvider`, vending `NSTextParagraph` elements to one or more layout managers. It tracks edits and invalidates layout incrementally.

```swift
let contentStorage = NSTextContentStorage()
contentStorage.attributedString = NSAttributedString(
    string: markdownText,
    attributes: [.font: NSFont.monospacedSystemFont(ofSize: 13, weight: .regular)]
)

// Access paragraphs (used by layout manager for line breaking)
for paragraph in contentStorage.textParagraphs {
    let range = paragraph.range
    // Each paragraph is a layout unit
}
```

**NSTextLayoutManager** replaces `NSLayoutManager`. It takes paragraphs from the content storage and produces `NSTextLayoutFragment` objects (one per line). Unlike TextKit 1's `NSLayoutManager` which owned both glyph generation *and* layout, TextKit 2's `NSTextLayoutManager` only does layout — glyph generation is handled internally by Core Text.

```swift
let layoutManager = NSTextLayoutManager()
layoutManager.textContainer = NSTextContainer(size: CGSize(width: 600, height: .greatestFiniteMagnitude))

// The content storage registers with the layout manager
contentStorage.addTextLayoutManager(layoutManager)

// Enumerate layout fragments (lines)
layoutManager.enumerateTextLayoutFragments(from: nil, options: []) { fragment in
    // fragment.layoutFragmentFrame — the rect of this line
    // fragment.textElements — the NSTextParagraph(s) in this fragment
    // fragment.renderingSurfaceBounds — for rendering
    return true
}
```

**NSTextViewportLayoutController** manages which fragments are visible in the current viewport. It lazily lays out only enough content to fill the viewport plus a small overdraw region. This is the key performance win for large documents.

```swift
let viewportController = NSTextViewportLayoutController(textLayoutManager: layoutManager)
viewportController.viewportBounds = scrollView.contentView.bounds

// As the user scrolls, update the viewport:
viewportController.viewportBounds = newVisibleRect

// The delegate (NSTextViewportLayoutControllerDelegate) is notified when
// layout completes for the current viewport
```

### NSTextElementProvider Protocol

`NSTextContentStorage` conforms to `NSTextElementProvider`. The protocol defines how the layout manager gets elements (paragraphs) to lay out:

```swift
protocol NSTextElementProvider: NSObjectProtocol {
    var documentRange: NSTextRange { get }
    func enumerateTextElements(from: NSTextElement?, options: NSTextElementProvider.EnumerationOptions, using: (NSTextElement) -> Bool) -> NSTextElement?
    func replaceContents(in range: NSTextRange, with textElements: [NSTextElement]) throws
    func synchronizedScrollView?(for textLayoutFragment: NSTextLayoutFragment) -> NSScrollView?
}
```

### NSTextFragment vs NSTextLineFragment

In TextKit 1, you had `NSTextFragment` (rectangular region of laid-out text). In TextKit 2, the equivalent is `NSTextLayoutFragment`. Each layout fragment corresponds to a visual line and contains one or more `NSTextElement`s (usually one paragraph, but a tall paragraph may span multiple fragments across pages).

```swift
layoutManager.enumerateTextLayoutFragments(from: nil, options: [.ensuresLayout]) { fragment in
    // These are equivalent concepts, different naming:
    // TextKit 1: NSTextFragment → enumerateLineFragments(forGlyphRange:using:)
    // TextKit 2: NSTextLayoutFragment → enumerateTextLayoutFragments(from:options:using:)

    let frame = fragment.layoutFragmentFrame  // geometry of this line
    for element in fragment.textElements {
        // The text content (paragraph(s)) that contributed to this line
        if let paragraph = element as? NSTextParagraph {
            let attributedString = paragraph.attributedString
            let elementRange = paragraph.elementRange
        }
    }
    return true
}
```

### Custom Layout with NSTextLayoutFragment

Subclass `NSTextLayoutFragment` to customize rendering (e.g., syntax highlighting, line backgrounds):

```swift
final class SyntaxHighlightFragment: NSTextLayoutFragment {
    override func draw(at renderingOrigin: CGPoint, in CGContext: CGContext) {
        // Draw custom background
        let bounds = self.layoutFragmentFrame
        NSColor.controlBackgroundColor.setFill()
        bounds.fill()

        // Draw syntax-highlighted text
        for textLayoutFragment in textElement?.textContentManager?.enumeratedTextElements(from: nil, options: []) {
            // Custom glyph rendering
        }

        super.draw(at: renderingOrigin, in: CGContext)
    }
}

// Use a custom layout fragment class:
let layoutManager = NSTextLayoutManager()
// Set via NSTextLayoutManagerDelegate
```

### TextKit 1 Compatibility

NSTextView can run in TextKit 1 mode:

```swift
textView.usesTextKit2 = false  // opt out

// Then the TextKit 1 stack is available:
// textView.layoutManager → NSLayoutManager
// textView.textContainer → NSTextContainer
// textView.textStorage → NSTextStorage
```

**Warning:** Mixing TextKit 1 and TextKit 2 APIs on the same text view is undefined behavior. Choose one pipeline and stick with it. For new code, always use TextKit 2.

## NSTextView Wrapping Recipe

### Minimal NSViewRepresentable

```swift
import SwiftUI
import AppKit

struct CodeEditorView: NSViewRepresentable {
    @Binding var text: String
    var onCommit: (() -> Void)?

    func makeNSView(context: Context) -> NSScrollView {
        let scrollView = NSTextView.scrollableTextView()
        let textView = scrollView.documentView as! NSTextView

        // Code editor configuration
        textView.isRichText = false
        textView.isEditable = true
        textView.allowsUndo = true
        textView.font = NSFont.monospacedSystemFont(ofSize: 13, weight: .regular)
        textView.isAutomaticQuoteSubstitutionEnabled = false
        textView.isAutomaticDashSubstitutionEnabled = false
        textView.isAutomaticTextReplacementEnabled = false
        textView.isAutomaticLinkDetectionEnabled = false
        textView.isGrammarCheckingEnabled = false
        textView.isContinuousSpellCheckingEnabled = false

        // No word wrap for code
        textView.isHorizontallyResizable = true
        textView.textContainer?.widthTracksTextView = false
        textView.textContainer?.containerSize = NSSize(
            width: CGFloat.greatestFiniteMagnitude,
            height: CGFloat.greatestFiniteMagnitude
        )
        textView.textContainer?.lineFragmentPadding = 8
        textView.textContainerInset = NSSize(width: 4, height: 4)

        // TextKit 2
        textView.usesTextKit2 = true  // default, explicit for clarity

        // Find bar
        textView.usesFindBar = true
        textView.isIncrementalSearchingEnabled = true

        // Line numbers (requires ruler)
        textView.usesRuler = false  // set true if you want the default ruler

        textView.delegate = context.coordinator
        context.coordinator.textView = textView

        return scrollView
    }

    func updateNSView(_ scrollView: NSScrollView, context: Context) {
        guard let textView = scrollView.documentView as? NSTextView else { return }

        if textView.string != text {
            // Preserve selection when possible
            let selectedRange = textView.selectedRange()
            textView.string = text
            if selectedRange.location <= text.utf16.count {
                textView.selectedRange = selectedRange
            }
        }
    }

    func makeCoordinator() -> Coordinator {
        Coordinator(text: $text, onCommit: onCommit)
    }

    @MainActor
    final class Coordinator: NSObject, NSTextViewDelegate {
        @Binding var text: String
        var textView: NSTextView?
        var onCommit: (() -> Void)?

        init(text: Binding<String>, onCommit: (() -> Void)?) {
            _text = text
            self.onCommit = onCommit
        }

        func textDidChange(_ notification: Notification) {
            guard let textView = notification.object as? NSTextView else { return }
            text = textView.string
        }

        func textView(_ textView: NSTextView,
                      doCommandBy commandSelector: Selector) -> Bool {
            if commandSelector == #selector(insertNewline(_:)) {
                onCommit?()
                // Return false so the text view still processes the newline
                // Return true to swallow it
            }
            return false  // let the text view handle it
        }
    }
}
```

### Line Numbers

For line numbers with NSTextView, the standard approach on macOS 14+ is to use a custom `NSTextLayoutFragment` subclass that draws a gutter alongside each line. An alternative is to track the visible range and overlay an `NSRulerView`:

```swift
// Using NSRulerView for line numbers (simplest approach)
class LineNumberRulerView: NSRulerView {
    weak var textView: NSTextView?

    override init(scrollView: UNScrollView, orientation: NSRulerView.Orientation) {
        super.init(scrollView: scrollView, orientation: .verticalRuler)
        clientView = scrollView.documentView
        ruleThickness = 40
    }

    required init(coder: NSCoder) { fatalError() }

    override func drawHashMarksAndLabels(in rect: NSRect) {
        guard let textView,
              let layoutManager = textView.textLayoutManager else { return }

        let visibleRect = textView.visibleRect
        var y = visibleRect.minY

        layoutManager.enumerateTextLayoutFragments(from: nil, options: []) { fragment in
            let frame = fragment.layoutFragmentFrame
            if frame.maxY < y { return true }  // skip fragments above viewport
            if frame.minY > visibleRect.maxY { return false }  // past viewport

            // Draw line number
            let lineNumber = fragment.textElements.first?.elementRange.location ?? 0
            let attrs: [NSAttributedString.Key: Any] = [
                .font: NSFont.monospacedSystemFont(ofSize: 10, weight: .regular),
                .foregroundColor: NSColor.secondaryLabelColor
            ]
            let string = NSAttributedString(string: "\(lineNumber + 1)", attributes: attrs)
            string.draw(at: NSPoint(x: 4, y: frame.minY - visibleRect.minY))

            return true
        }
    }
}

// Attach to scroll view:
let ruler = LineNumberRulerView(scrollView: scrollView, orientation: .verticalRuler)
ruler.textView = textView
scrollView.verticalRulerView = ruler
scrollView.hasVerticalRuler = true
scrollView.rulersVisible = true
```

## Find Bar Implementation

### Basic Setup

```
NSTextFinder (controller)
    ├── client: NSTextFinderClient  (the text view or coordinator)
    └── findBarContainer: NSScrollView  (must be the scroll view)
```

```swift
// The text view self-wires when usesFindBar = true and it's in a scroll view.
// No manual NSTextFinder setup needed for basic use.

textView.usesFindBar = true
textView.isIncrementalSearchingEnabled = true

// The system creates the NSTextFinder automatically.
```

### Custom NSTextFinderClient

When you need custom find behavior (e.g., searching a virtual document, regex support):

```swift
final class TextFinderClient: NSObject, NSTextFinderClient {
    weak var textView: NSTextView?
    private var findPattern = ""
    private var matches: [NSRange] = []
    private var currentMatchIndex = 0

    // MARK: - Required methods

    var string: String {
        textView?.string ?? ""
    }

    var firstSelectedRange: NSRange {
        textView?.selectedRange() ?? NSRange(location: 0, length: 0)
    }

    var isEditable: Bool {
        textView?.isEditable ?? false
    }

    var allowsMultipleSelection: Bool { false }

    var visibleCharacterRanges: [NSValue] {
        guard let textView else { return [] }
        let visibleRect = textView.visibleRect
        // Convert visible rect to character range
        guard let layoutManager = textView.layoutManager,
              let textContainer = textView.textContainer else {
            return [NSValue(range: NSRange(location: 0, length: string.utf16.count))]
        }
        let glyphRange = layoutManager.glyphRange(forBoundingRect: visibleRect, in: textContainer)
        let charRange = layoutManager.characterRange(forGlyphRange: glyphRange, actualGlyphRange: nil)
        return [NSValue(range: charRange)]
    }

    // MARK: - Find operations

    func performAction(_ action: NSTextFinder.Action) {
        switch action {
        case .showFindInterface:
            findPattern = ""
            matches = []
            currentMatchIndex = 0
        case .nextMatch:
            findNext()
        case .previousMatch:
            findPrevious()
        case .replace:
            replaceCurrent()
        case .replaceAll:
            replaceAll()
        case .replaceAndFind:
            replaceCurrent()
            findNext()
        case .setSearchString:
            // Called when the user types in the find bar
            break
        case .replaceAllInSelection:
            replaceAllInSelection()
        case .selectAll:
            selectAllMatches()
        case .selectAllInSelection:
            break
        case .showReplaceInterface:
            break
        case .hideFindInterface:
            break
        @unknown default:
            break
        }
    }

    // MARK: - Replace operations

    func shouldReplaceCharacters(inRanges ranges: [NSValue], withStrings strings: [String]) -> Bool {
        return true
    }

    func replaceCharacters(in range: NSRange, with string: String) {
        guard let storage = textView?.textStorage else { return }
        storage.replaceCharacters(in: range, with: string)
        // Don't call shouldChangeText/didChangeText here — NSTextFinder manages that
    }

    func scrollRangeToVisible(_ range: NSRange) {
        textView?.scrollRangeToVisible(range)
    }

    func contentString(for range: NSRange, attributes: AutoreleasingUnsafeMutablePointer<NSDictionary?>?) -> NSAttributedString? {
        return textView?.textStorage?.attributedSubstring(from: range)
    }

    func rects(for range: NSRange) -> [NSValue]? {
        guard let textView else { return nil }
        guard let layoutManager = textView.layoutManager,
              let textContainer = textView.textContainer else { return nil }

        let glyphRange = layoutManager.glyphRange(forCharacterRange: range, actualCharacterRange: nil)
        var rects: [NSValue] = []
        layoutManager.enumerateEnclosingRects(forGlyphRange: glyphRange,
                                               withinSelectedGlyphRange: NSRange(location: NSNotFound, length: 0),
                                               in: textContainer) { rect, _ in
            rects.append(NSValue(rect: rect))
        }
        return rects.isEmpty ? nil : rects
    }

    // MARK: - Find logic

    func search(for pattern: String, searchRange: NSRange, options: NSString.CompareOptions = []) {
        findPattern = pattern
        matches = []
        currentMatchIndex = 0

        let nsString = string as NSString
        var searchStart = searchRange.location

        while searchStart < NSMaxRange(searchRange) {
            let range = nsString.range(of: pattern, options: options, range: NSRange(location: searchStart, length: NSMaxRange(searchRange) - searchStart))
            if range.location == NSNotFound { break }
            matches.append(range)
            searchStart = NSMaxRange(range)
        }
    }

    private func findNext() {
        guard !matches.isEmpty else { return }
        currentMatchIndex = (currentMatchIndex + 1) % matches.count
        let range = matches[currentMatchIndex]
        textView?.selectedRange = range
        textView?.scrollRangeToVisible(range)
    }

    private func findPrevious() {
        guard !matches.isEmpty else { return }
        currentMatchIndex = (currentMatchIndex - 1 + matches.count) % matches.count
        let range = matches[currentMatchIndex]
        textView?.selectedRange = range
        textView?.scrollRangeToVisible(range)
    }

    private func replaceCurrent() {
        guard currentMatchIndex < matches.isEmpty else { return }
        let range = matches[currentMatchIndex]
        // Replacement text comes from NSTextFinder's replace field
    }

    private func replaceAll() {
        // Replace all matches
    }

    private func replaceAllInSelection() {
        // Replace within selection only
    }

    private func selectAllMatches() {
        // Highlight all matches
    }
}

// Wiring:
let client = TextFinderClient()
let finder = NSTextFinder()
finder.client = client
finder.findBarContainer = scrollView
textView.textFinder = finder
```

### Programmatic Find

```swift
// Show find bar
NSApp.sendAction(#selector(NSTextFinderAction.showFindInterface), to: nil, from: nil)

// Or directly:
textFinder.performAction(.showFindInterface)
textFinder.performAction(.nextMatch)

// Set search string programmatically:
textFinder.searchString = "search term"
textFinder.performAction(.showFindInterface)
```

## Keyboard Event Handling

### performKeyEquivalent(_:)

Override in `NSTextView` subclass for menu key equivalents and special key chords:

```swift
final class EditorTextView: NSTextView {
    var onSave: (() -> Void)?
    var onFind: (() -> Void)?

    override func performKeyEquivalent(with event: NSEvent) -> Bool {
        let flags = event.modifierFlags.intersection(.deviceIndependentFlagsMask)

        // Custom: Cmd+Shift+S for Save As
        if flags == [.command, .shift],
           event.charactersIgnoringModifiers == "s" {
            onSave?()
            return true
        }

        // Custom: Cmd+G for Find Next (Emacs-style)
        if flags == .command,
           event.charactersIgnoringModifiers == "g" {
            onFind?()
            return true
        }

        return super.performKeyEquivalent(with: event)
    }
}
```

### doCommandBySelector(_:)

`NSTextView` sends itself `doCommandBySelector:` for key actions. Override in an `NSTextViewDelegate` to intercept:

```swift
extension EditorCoordinator: NSTextViewDelegate {
    func textView(_ textView: NSTextView, doCommandBy commandSelector: Selector) -> Bool {
        switch commandSelector {
        case #selector(insertTab(_:)):
            // Custom tab behavior: insert 4 spaces
            textView.insertText("    ", replacementRange: textView.selectedRange())
            return true  // consumed — don't call default

        case #selector(insertNewline(_:)):
            // Auto-indent: match the indentation of the current line
            let currentLine = currentLineNumber(for: textView)
            let indent = indentationOfLine(currentLine, in: textView)
            textView.insertText("\n" + String(repeating: " ", count: indent), replacementRange: textView.selectedRange())
            return true

        case #selector(deleteBackward(_:)):
            // Custom backspace: delete 4 spaces at once if at indent boundary
            let range = textView.selectedRange()
            if range.length == 0 && range.location >= 4 {
                let str = textView.string as NSString
                let preceding = str.substring(with: NSRange(location: range.location - 4, length: 4))
                if preceding == "    " {
                    textView.replaceCharacters(in: NSRange(location: range.location - 4, length: 4), with: "")
                    return true
                }
            }
            return false  // let default handle it

        case #selector(moveUp(_:)):
            // Custom: move to matching bracket
            if let matchPos = findMatchingBracket(from: selectedPosition, in: textView) {
                textView.selectedRange = NSRange(location: matchPos, length: 0)
                return true
            }
            return false

        default:
            return false  // let the text view handle it
        }
    }
}
```

### NSTextInputClient

The full text input protocol. You only need to implement this when building a **custom view that doesn't inherit from NSTextView**. Key methods and their roles:

```swift
final class CustomTextInputView: NSView, NSTextInputClient {
    // State
    private var textStorage = NSTextStorage()
    private var selectedRange = NSRange(location: 0, length: 0)
    private var markedRange = NSRange(location: NSNotFound, length: 0)

    // MARK: - Drawing

    override var isFlipped: Bool { true }

    override func draw(_ dirtyRect: NSRect) {
        super.draw(dirtyRect)
        // Draw text, selection, and marked text
        let glyphRange = layoutManager.glyphRange(forBoundingRect: bounds, in: textContainer)
        layoutManager.drawGlyphs(forGlyphRange: glyphRange, at: .zero)

        // Draw selection highlight
        if selectedRange.length > 0 {
            let selGlyphRange = layoutManager.glyphRange(forCharacterRange: selectedRange, actualCharacterRange: nil)
            layoutManager.fillBackgroundGlyphRange(selGlyphRange, with: NSColor.selectedTextBackgroundColor, for: bounds.origin)
        }

        // Draw insertion point
        let caretGlyphIndex = layoutManager.glyphIndexForCharacter(at: selectedRange.location)
        if let caretRect = layoutManager.boundingRect(forGlyphRange: NSRange(location: caretGlyphIndex, length: 1), in: textContainer) {
            NSColor.textColor.setFill()
            NSRect(x: caretRect.minX, y: caretRect.minY, width: 1, height: caretRect.height).fill()
        }

        // Draw marked text underline
        if markedRange.location != NSNotFound {
            let markGlyphRange = layoutManager.glyphRange(forCharacterRange: markedRange, actualCharacterRange: nil)
            layoutManager.enumerateLineFragments(forGlyphRange: markGlyphRange) { rect, _, _, _, _ in
                NSColor.systemGray.setStroke()
                let path = NSBezierPath()
                path.move(to: NSPoint(x: rect.minX, y: rect.maxY))
                path.line(to: NSPoint(x: rect.maxX, y: rect.maxY))
                path.lineWidth = 2
                path.stroke()
            }
        }
    }

    // MARK: - Events

    override func mouseDown(with event: NSEvent) {
        window?.makeFirstResponder(self)
        let point = convert(event.locationInWindow, from: nil)
        let glyphIndex = layoutManager.glyphIndex(for: point, in: textContainer)
        let charIndex = layoutManager.characterIndexForGlyph(at: glyphIndex)
        selectedRange = NSRange(location: charIndex, length: 0)
        needsDisplay = true
    }

    override func keyDown(with event: NSEvent) {
        interpretKeyEvents([event])
    }

    // MARK: - NSTextInputClient

    func insertText(_ string: Any, replacementRange: NSRange) {
        let text = string as? String ?? (string as! NSAttributedString).string
        let range = replacementRange.location == NSNotFound ? selectedRange : replacementRange

        textStorage.replaceCharacters(in: range, with: text)
        let newLocation = range.location + text.utf16.count
        selectedRange = NSRange(location: newLocation, length: 0)
        unmarkText()
        needsDisplay = true
    }

    func setMarkedText(_ string: Any, selectedRange selRange: NSRange, replacementRange: NSRange) {
        let text = string as? String ?? ""
        if markedRange.location == NSNotFound {
            markedRange = selectedRange
        }
        textStorage.replaceCharacters(in: markedRange, with: text)
        markedRange.length = text.utf16.count
        selectedRange = NSRange(location: markedRange.location + selRange.location, length: selRange.length)
        needsDisplay = true
    }

    func unmarkText() {
        if markedRange.location != NSNotFound {
            selectedRange = NSRange(location: markedRange.location + markedRange.length, length: 0)
            markedRange = NSRange(location: NSNotFound, length: 0)
        }
        needsDisplay = true
    }

    func selectedRange() -> NSRange { selectedRange }

    func markedRange() -> NSRange { markedRange }

    func hasMarkedText() -> Bool { markedRange.location != NSNotFound }

    func attributedSubstring(forProposedRange range: NSRange, actualRange: NSRangePointer) -> NSAttributedString? {
        let actual = NSIntersectionRange(range, NSRange(location: 0, length: textStorage.length))
        actualRange.pointee = actual
        return textStorage.attributedSubstring(from: actual)
    }

    func firstRect(forCharacterRange range: NSRange, actualRange: NSRangePointer) -> NSRect {
        let actual = NSIntersectionRange(range, NSRange(location: 0, length: textStorage.length))
        actualRange.pointee = actual
        let glyphRange = layoutManager.glyphRange(forCharacterRange: actual, actualCharacterRange: nil)
        return layoutManager.boundingRect(forGlyphRange: glyphRange, in: textContainer)
    }

    func characterIndex(for point: NSPoint) -> Int {
        let glyphIndex = layoutManager.glyphIndex(for: point, in: textContainer)
        return layoutManager.characterIndexForGlyph(at: glyphIndex)
    }

    // MARK: - NSResponder

    override var acceptsFirstResponder: Bool { true }
    override var canBecomeKeyView: Bool { true }

    override func becomeFirstResponder() -> Bool {
        // Register for text input
        inputContext?.selectedKeyboardInputMode = NSTextInputContext.currentKeyboardInputMode
        return true
    }

    override func resignFirstResponder() -> Bool {
        unmarkText()
        return true
    }
}
```

### interpretKeyEvents

The bridge between `keyDown` and `NSTextInputClient`. When you call `interpretKeyEvents`, the text input context translates standard keystrokes into `NSTextInputClient` method calls:

```swift
override func keyDown(with event: NSEvent) {
    // Handle custom keys first
    if shouldHandleCustomKey(event) {
        handleCustomKey(event)
        return
    }

    // Pass standard keys to the text input context for processing
    // This handles: character input, arrow keys, delete, home/end, etc.
    interpretKeyEvents([event])

    // If interpretKeyEvents didn't handle it, try performKeyEquivalent
    interpretKeyEvents returns void — if the event wasn't handled,
    you may need to call super or handle it yourself
}
```

## Font Panel Integration

### Complete Font Panel Setup

```swift
final class FontAwareViewController: NSViewController, NSTextViewDelegate {
    @IBOutlet var textView: NSTextView!

    override func viewDidLoad() {
        super.viewDidLoad()
        setupFontIntegration()
    }

    private func setupFontIntegration() {
        // The font panel sends changeFont: up the responder chain.
        // The first responder that responds to changeFont: handles it.
        // NSTextView responds by default, applying the font to the selection.

        // Set the initial font
        textView.font = NSFont.systemFont(ofSize: 14)

        // Configure which font panel modes are available
        // (override validModesForFontPanel in a custom NSTextView subclass)
    }

    // MARK: - NSTextViewDelegate

    func textViewDidChangeSelection(_ notification: Notification) {
        updateFontPanel()
    }

    private func updateFontPanel() {
        let fontManager = NSFontManager.shared
        let range = textView.selectedRange()

        if range.length > 0 {
            // Selection: show the font of the selection
            let font = textView.textStorage?.attribute(.font, at: range.location, effectiveRange: nil) as? NSFont
            fontManager.setSelectedFont(font ?? textView.font ?? .systemFont(ofSize: 14), isMultiple: false)
        } else {
            // No selection: show the typing font
            let font = textView.typingAttributes[.font] as? NSFont ?? textView.font ?? .systemFont(ofSize: 14)
            fontManager.setSelectedFont(font, isMultiple: false)
        }
    }
}

// MARK: - Custom NSTextView with Font Panel Control

final class ConfigurableTextView: NSTextView {
    var allowedFontPanelModes: NSFontPanel.ModeMask = [.collectionMask, .faceMask, .sizeMask]

    override func validModesForFontPanel(_ fontPanel: NSFontPanel) -> NSFontPanel.ModeMask {
        return allowedFontPanelModes
    }

    override func changeFont(_ sender: NSFontManager?) {
        guard let fontManager = sender,
              let storage = textStorage else { return }

        let range = selectedRange()
        let currentFont: NSFont

        if range.length > 0 {
            // Apply to selection
            currentFont = storage.attribute(.font, at: range.location, effectiveRange: nil) as? NSFont
                        ?? font ?? .systemFont(ofSize: 14)
            let newFont = fontManager.convert(currentFont)
            storage.beginEditing()
            storage.addAttribute(.font, value: newFont, range: range)
            storage.endEditing()
        } else {
            // Update typing attributes (for next input)
            currentFont = typingAttributes[.font] as? NSFont ?? font ?? .systemFont(ofSize: 14)
            let newFont = fontManager.convert(currentFont)
            typingAttributes[.font] = newFont
        }
    }

    // MARK: - Bold/Italic/Underline actions

    @objc func makeTextBold(_ sender: Any?) {
        let fontManager = NSFontManager.shared
        let range = selectedRange()
        let currentFont: NSFont

        if range.length > 0, let f = textStorage?.attribute(.font, at: range.location, effectiveRange: nil) as? NSFont {
            currentFont = f
        } else {
            currentFont = typingAttributes[.font] as? NSFont ?? font ?? .systemFont(ofSize: 14)
        }

        let newFont = fontManager.convert(currentFont, toHaveTrait: .boldFontMask)
        applyFont(newFont, toSelection: range)
    }

    @objc func makeTextItalic(_ sender: Any?) {
        let fontManager = NSFontManager.shared
        let range = selectedRange()
        let currentFont: NSFont

        if range.length > 0, let f = textStorage?.attribute(.font, at: range.location, effectiveRange: nil) as? NSFont {
            currentFont = f
        } else {
            currentFont = typingAttributes[.font] as? NSFont ?? font ?? .systemFont(ofSize: 14)
        }

        let newFont = fontManager.convert(currentFont, toHaveTrait: .italicFontMask)
        applyFont(newFont, toSelection: range)
    }

    private func applyFont(_ font: NSFont, toSelection range: NSRange) {
        if range.length > 0 {
            textStorage?.addAttribute(.font, value: font, range: range)
        } else {
            typingAttributes[.font] = font
        }
    }
}
```

### NSFontPanel Modes

```swift
// Available modes (combine with .union or array literal):
// .collectionMask    — font collections sidebar
// .faceMask          — font face (Regular, Bold, Italic, etc.)
// .sizeMask          — font size picker
// .underlineMask     — underline toggle (managed by text view)
// .strikethroughMask — strikethrough toggle
// .textColorMask     — text color well
// .backgroundColorMask — background color well
// .shadowMask        — text shadow (legacy)
// .allEffectsMask    — all typographic effects
// .standardModesMask — collection + face + size + underline + strikethrough + colors
// .allModesMask      — everything
```
