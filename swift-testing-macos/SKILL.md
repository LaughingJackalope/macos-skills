---
name: swift-testing-macos
description: "Testing on macOS — Swift Testing (@Test, @Suite, #expect, #require, confirmation, traits, parameterized tests), testing AppKit views/NSDocument, XCTest UI testing (XCUIApplication), testing XPC services, xcodebuild test macOS destination. Use when writing tests for macOS apps."
sources: https://developer.apple.com/documentation/testing, https://developer.apple.com/documentation/xctest, https://github.com/swiftlang/swift-testing, https://developer.apple.com/documentation/testing/testing-asynchronous-code, https://developer.apple.com/documentation/xcuiautomation/xcuielementquery/matching(identifier:), https://developer.apple.com/documentation/xcuiautomation/xcuielementquery/descendants(matching:), https://www.donnywals.com/testing-completion-handler-apis-with-swift-testing/
>
> **Updated for Swift 6.3 (2026):** Swift Testing `swift-6.3.2-RELEASE` (May 2026) introduces non-escapable types as valid suite types and disallows `@Test` on operator declarations. Generic `Self` usage within `@Suite` traits is corrected. The latest stable release is `swift-6.3.2-RELEASE`.
researched: 2026-06-06
platform: macOS 26+
---

# Swift Testing on macOS

Modern testing for macOS 26+ with Swift 6.3. Covers the Swift Testing framework (`@Test`, `@Suite`, `#expect`, `#require`, `#confirm`, traits, parameterized tests), testing AppKit components (`NSDocument`, `NSViewController`, `NSWindow`), XCTest UI testing with `XCUIApplication` on macOS, testing XPC services via mock connections, and running tests with `xcodebuild`. For xcodebuild destination syntax and archive/export workflows, see `xcode-macos-toolchain`. For NSDocument architecture, see `appkit-foundations`. For XPC service patterns, see `macos-ipc-xpc`. For debugging test failures, see `debugging-macos`.

See [references/testing-patterns.md](references/testing-patterns.md) for deep-dive examples: parameterized `@Suite`, XCTest UI test with menu bar interaction, mock XPC service with protocol mock, testing NSDocument reading/writing, and test plan configuration.

## Contents

- [Swift Testing Basics](#swift-testing-basics)
- [Test Traits](#test-traits)
- [Parameterized Tests](#parameterized-tests)
- [Testing AppKit](#testing-appkit)
- [XCTest UI Testing on macOS](#xctest-ui-testing-on-macos)
- [Testing XPC Services](#testing-xpc-services)
- [xcodebuild test](#xcodebuild-test)
- [Test Plan Configuration](#test-plan-configuration)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Swift Testing Basics

Swift Testing is Apple's modern test framework, shipped with Swift 6+. It replaces XCTest's `XCTAssert*` macros with expressive, type-safe macros and introduces suite-based test organization.

### @Test and @Suite

Every test is a function annotated with `@Test`. Tests are grouped into suites — any type, class, or actor conforming to `Suite`. Suites organize tests, hold shared setup/teardown, and enable traits.

```swift
import Testing

@Suite("Document Model")
struct DocumentModelTests {

    @Test("New document starts empty")
    func newDocumentIsEmpty() {
        let doc = NoteDocument()
        #expect(doc.text.isEmpty)
    }

    @Test("Document text is mutable")
    func documentTextIsMutable() {
        var doc = NoteDocument()
        doc.text = "Hello"
        #expect(doc.text == "Hello")
    }

    @Test("Save round-trip preserves content", arguments: ["Hello", "café", "🎉"])
    func saveRoundTrip(content: String) throws {
        let doc = NoteDocument()
        doc.text = contents
        let data = try doc.data(ofType: "public.plain-text")
        try doc.read(from: data, ofType: "public.plain-text")
        #expect(doc.text == contents)
    }
}
```

**Suite types:** Use `struct` for most suites. Use `class` when you need reference semantics (e.g., shared non-copyable resources). Use `actor` for tests that require actor isolation.

### #expect and #require

`#expect` evaluates a Boolean expression. If it's `false`, the test **records a failure and continues** — subsequent assertions in the same test still run.

`#require` evaluates a Boolean expression or optional. If it fails, the test **aborted immediately** — no further code in the test runs. Use it for preconditions that, if false, would make the rest meaningless or crash.

```swift
#expect(result == 42)            // Soft: records failure, continues
#expect(count >= 0)              // Common pattern: validate invariant

guard let data = try? loadFile() else {
    // In Swift Testing:
}
#require(data != nil)            // Hard: aborts if nil
#expect(data!.count > 0)         // Safe to force-unwrap after #require

// #require also works with optionals directly:
let value = try #require(optionalResult)  // throws TestFailure if nil
```

**Expression capture:** `#expect` captures source expressions, so failure messages show the actual expression and values — no string interpolation needed:

```
failed: value — Expected true (result == 42)  // actual: result=7
```

You can attach messages:

```swift
#expect(result == 42, "After applying the transform, result should match expected output")
```

### #confirm (Confirmation)

`#confirm` is for non-deterministic or asynchronous testing — it confirms that a condition becomes true within a timeout, without polling loops.

```swift
@Test("File watcher detects changes")
func fileWatcherDetectsChanges() async throws {
    let watcher = FileWatcher(url: testFile)
    try watcher.start()

    // Trigger the change
    try "updated".write(to: testFile, atomically: true, encoding: .utf8)

    // Confirm the callback fires within the timeout
    await #confirm { watcher.lastChange != nil }

    try watcher.stop()
}
```

**Timeout:** Confirmation has a default timeout (configurable). If the condition isn't met in time, the test fails. This replaces `XCTestExpectation` / `waitForExpectations` from XCTest for many use cases.

>
> ### Async Testing Patterns for AppKit
>
> Swift Testing natively supports `async` tests — no `XCTestExpectation` needed. For code mixing legacy completion handlers with async/await, use `withCheckedContinuation`:
>
> ```swift
> @Test("NSDocument async open completes")
> func asyncOpenCompletes() async throws {
>     let doc = NoteDocument()
>     let tempURL = temporaryURL(for: "test.txt")
>     try "content".write(to: tempURL, atomically: true, encoding: .utf8)
>
>     // Bridge NSDocument's completion handler to async/await
>     let openedDoc = try await withCheckedThrowingContinuation { continuation in
>         let controller = NoteDocumentController()
>         controller.openDocument(withContentsOf: tempURL, display: false) { doc, wasOpen, error in
>             if let error = error {
>                 continuation.resume(throwing: error)
>             } else if let doc = doc {
>                 continuation.resume(returning: doc)
>             } else {
>                 continuation.resume(throwing: NSError(domain: "test", code: -1))
>             }
>         }
>     }
>     #expect(openedDoc.text == "content")
> }
> ```
>
> **Key rule:** Confirmations (`#confirm`) must be fulfilled before the test closure returns. For completion-handler bridging, the continuation resume *is* the fulfillment — never use `#confirm` and continuations together on the same callback.
## Test Traits

Traits annotate tests with metadata — tags, conditions, time limits, and platform expectations.

### Tag-Based Traits

Tags let you filter tests at runtime. Define tags as types conforming to `Tag`, then apply them:

```swift
extension Tag {
    @Tag static var networking: Tag
    @Tag static var ui: Tag
    @Tag static var document: Tag
    @Tag static var slow: Tag
}

@Suite(.tags(.document))
struct DocumentTests {

    @Test("Save works", .tags(.networking)) // overlaps with suite tag
    func saveWorks() { }

    @Test("Autosave triggers", .disabled("Flaky on CI"))
    func autosaveTriggers() { }
}
```

**Filtering with tags in xcodebuild:**

```bash
# Run only tests tagged with .networking
xcodebuild test -scheme MyApp -only-testing:MyAppTests/DocumentTests/saveWorks

# Exclude slow tests via -skip-testing
xcodebuild test -scheme MyApp -skip-testing:MyAppTests/*.slow
```

Tag filtering is more precise at the xcodebuild level via `-only-testing` / `-skip-testing` with test identifiers. In Xcode's test navigator and Test Plan editor, you can include/exclude tag categories.

### Conditional Execution

```swift
@Test(.enabled(if: ProcessInfo.processInfo.environment["CI"] == nil),
      "Local-only test")
func localOnly() { }

@Test(.disabled("Fails on macOS 15 due to rdar://12345678"))
func knownFailure() { }

@Suite(.enabled(if: !isRunningOnCI()))
struct LocalOnlySuite { }
```

`.enabled(if:)` and `.disabled(_:)` are mutually exclusive — don't apply both to the same test.

### Time Limit Trait

`TimeLimitTrait` (or `.timeLimit()`) caps how long a test can run. If exceeded, the test fails:

```swift
@Test(.timeLimit(.minutes(2)),
      "Large file import")
func largeFileImport() throws {
    let data = generateLargeData() // 500MB
    let doc = NoteDocument()
    try doc.read(from: data, ofType: "public.data")
}
```

### Platform Traits

```swift
@Test(.bug("rdar://12345678", "Crash on save"))
func reportedBug() { }
```

`.bug()` associates a test with a radar/FB number. It doesn't enable or disable — it's metadata for the test report.

## Parameterized Tests

Parameterized tests run the same test logic over a collection of arguments. Swift Testing validates each argument set independently — one failure doesn't stop the others.

### Basic Parameterized Test

```swift
@Test("Valid file extensions", arguments: ["txt", "md", "swift", "json"])
func validExtensions(ext: String) throws {
    let doc = NoteDocument()
    let typeName = "public.plain-text"
    let result = doc.fileNameExtension(forType: typeName, saveOperation: .saveAsOperation)
    #expect(result != nil || ext == "unknown")
}
```

### Collections of Arguments

```swift
@Test("Whitespace trimming", arguments: [
    ("  hello  ", "hello"),
    ("\n  world  \n", "world"),
    ("\tfoobar\t", "foobar"),
    ("no-op", "no-op"),
])
func whitespaceTrimming(input: String, expected: String) {
    #expect(input.trimmingCharacters(in: .whitespacesAndNewlines) == expected)
}
```

Tuples let you pass multiple arguments per case.

### Custom Argument Types

For complex argument sets, conform a type to `Arg`:

```swift
struct FileRoundTripCase: Arg {
    let name: String
    let content: String
    let encoding: String.Encoding

    var description: String { "\(name) (\(encoding.description))" }
}

extension FileRoundTripCase {
    static let cases: [FileRoundTripCase] = [
        .init(name: "ascii", content: "Hello World", encoding: .utf8),
        .init(name: "emoji", content: "🎉🐶🍕", encoding: .utf8),
        .init(name: "utf16", content: "café", encoding: .utf16),
    ]
}

@Test("Round-trip encoding", arguments: FileRoundTripCase.cases)
func roundTripEncoding(_ roundTrip: FileRoundTripCase) throws {
    let doc = NoteDocument()
    doc.content = roundTrip.content
    let data = try doc.data(ofType: "public.plain-text")
    try doc.read(from: data, ofType: "public.plain-text")
    #expect(doc.content == roundTrip.content)
}
```

The `description` property controls how the argument appears in test reports.

### Parameterized Suites

For running an entire suite over parameterized inputs, see [references/testing-patterns.md](references/testing-patterns.md).

## Testing AppKit

Testing AppKit components requires care: many classes need a run loop, window server connection, or main-thread execution.

### Testing NSDocument Subclasses

The most reliable pattern: test the model I/O methods directly without instantiating the full UI:

```swift
import Testing
@testable import MyApp

@Suite("NoteDocument")
struct NoteDocumentTests {

    @Test("Round-trip plain text")
    func roundTripPlainText() async throws {
        let doc = NoteDocument()
        doc.text = "Hello, World"

        let data = try doc.data(ofType: "public.plain-text")
        let doc2 = NoteDocument()
        try doc2.read(from: data, ofType: "public.plain-text")

        #expect(doc2.text == "Hello, World")
    }

    @Test("Empty document produces empty data")
    func emptyDocument() throws {
        let doc = NoteDocument()
        let data = try doc.data(ofType: "public.plain-text")
        #expect(data.isEmpty)
    }

    @Test("Read throws on invalid data", arguments: [
        Data([0xFF, 0xFE]),      // invalid UTF-8
        Data([0x80, 0x81]),       // continuation bytes without start
    ])
    func invalidDataThrows(invalidData: Data) {
        let doc = NoteDocument()
        #expect(throws: NSError.self) {
            try doc.read(from: invalidData, ofType: "public.plain-text")
        }
    }

    @Test("Change count tracking")
    func changeCountTracking() {
        let doc = NoteDocument()
        #expect(!doc.isDocumentEdited)
        doc.updateChangeCount(.changeDone)
        #expect(doc.isDocumentEdited)
    }
}
```

**Key principle:** Test `data(ofType:)` and `read(from:ofType:)` in isolation. Don't test through `NSWindow` or `NSFileCoordinator` — that's integration testing territory.

### Testing NSViewController

For view controller tests that need a view hierarchy:

```swift
@Suite("NoteViewController")
@MainActor  // AppKit requires main actor
struct NoteViewControllerTests {

    @Test("View loads")
    func viewLoads() {
        let doc = NoteDocument()
        doc.text = "test"
        let vc = NoteViewController(document: doc)
        _ = vc.view  // Forces loadView + viewDidLoad
        #expect(vc.view != nil)
    }

    @Test("TextView displays document content")
    @MainActor
    func textViewDisplaysContent() {
        let doc = NoteDocument()
        doc.text = "Hello, Document"
        let vc = NoteViewController(document: doc)
        _ = vc.view

        // Access the text view via IBOutlet or view hierarchy
        let textView = vc.view.subviews
            .compactMap { $0 as? NSTextView }
            .first
        #expect(textView?.string == "Hello, Document")
    }
}
```

**Requirements:**
- Annotate suite or test with `@MainActor` — AppKit UI objects must be accessed on the main thread
- Call `_ = vc.view` to trigger the lazy view lifecycle before accessing outlets
- Don't test window presentation in unit tests — use XCTest UI tests for that

### Testing NSWindow Presentation

For window-level behavior, use a coordinator pattern and mock the window:

```swift
@MainActor
@Suite("Window Controller")
struct WindowControllerTests {

    @Test("Window is created with correct style mask")
    func windowStyleMask() {
        let doc = NoteDocument()
        let wc = DocumentWindowController(document: doc)
        wc.loadWindow()

        #expect(wc.window?.styleMask.contains(.titled) == true)
        #expect(wc.window?.styleMask.contains(.closable) == true)
    }

    @Test("Content view controller is set")
    func contentViewControllerSet() {
        let doc = NoteDocument()
        let wc = DocumentWindowController(document: doc)
        let expectation = Expectation()
        wc.onContentLoaded = { expectation.fulfill() }
        wc.windowDidLoad()

        #expect(wc.contentViewController != nil)
    }
}
```

## XCTest UI Testing on macOS

XCTest UI tests drive the app through `XCUIApplication`. On macOS, this uses the Accessibility API — every UI element you want to interact with must have an accessibility role.

### Basic Setup

```swift
import XCTest

final class AppUITests: XCTestCase {

    var app: XCUIApplication!

    override func setUpWithError() throws {
        continueAfterFailure = false
        app = XCUIApplication()
        app.launch()
    }

    override func tearDownWithError() throws {
        app = nil
    }
}
```

### Menu Bar Queries

```swift
@Test("New document via File > New")
func newDocumentMenu() {
    // Access menu bar items — macOS-specific
    let fileMenu = app.menuBars.menuBarsItems["File"]
    fileMenu.click()

    let newItem = fileMenu.menuItems["New"]
    #expect(newItem.exists)
    newItem.click()

    // Verify a new window appeared
    let windows = app.windows
    #expect(windows.count >= 1)
}
```

```swift
// More robust pattern for menu bar interaction:
func clickMenuItem(_ menu: String, _ item: String) {
    let menuItem = app.menuBars.buttons[menu]
    if menuItem.exists {
        menuItem.click()
        app.menuBars.buttons[item].click()
    }
}

@Test("Save via menu")
func saveViaMenu() {
    clickMenuItem("File", "Save…")
    // Handle save panel
    let savePanel = app.sheets.firstMatch
    #expect(savePanel.waitForExistence(timeout: 5))
}
```

### Window Queries

```swift
@Test("Window title reflects document name")
func windowTitle() {
    // Wait for the main window
    let mainWindow = app.windows.firstMatch
    #expect(mainWindow.waitForExistence(timeout: 10))

    // Initial title for untitled document
    #expect(mainWindow.title.contains("Untitled"))
}
```

### Element Existence Assertions

```swift
@Test("Inspector button exists in toolbar")
func inspectorButton() {
    let inspectorButton = app.windows.firstMatch
        .toolbar.buttons["Inspector"]

    #expect(inspectorButton.waitForExistence(timeout: 5))
    #expect(inspectorButton.isEnabled)
}

@Test("Sidebar is present")
func sidebarPresent() {
    // macOS-specific: NSSplitView sections
    // Use accessibility identifier set in code:
    //   splitView.accessibilityIdentifier = "main-split"
    //   sidebarView.accessibilityIdentifier = "sidebar"

    let sidebar = app.windows.firstMatch
        .splitGroups["main-split"]
        .groups["sidebar"]

    #expect(sidebar.waitForExistence(timeout: 5))
}
```

**Key macOS UI testing differences from iOS:**
- Use `menuBars` instead of navigation bars
- `app.sheets` captures modal panels (save panel, open panel)
- Accessibility identifiers on `NSView` require `accessibilityIdentifier` — not all AppKit views expose one by default
- Window ordering: `app.windows` is ordered front to back; `firstMatch` gets the key window

>
> ### Advanced Element Queries
>
> Beyond basic queries, `XCUIElementQuery` supports composable filtering:
>
> ```swift
> // matching(identifier:) — matches on any identifying property (identifier, title, label, value, placeholderValue)
> let button = app.buttons.matching(identifier: "save-button").firstMatch
>
> // descendants(matching:) — find all descendants of a specific element type
> let allTextFields = app.windows.firstMatch.descendants(matching: .textField)
> let count = allTextFields.count  // total text fields in window
>
> // containing(_:identifier:) — find elements that have a descendant matching
> let cells = app.tables.firstMatch.cells.containing(.textField, identifier: "name-field")
>
> // Chaining queries — narrow results step by step
> let enabledSaveButtons = app.buttons.matching(identifier: "Save")
>     .matching(NSPredicate(format: "isEnabled == true"))
> ```
>
> Use `matching(identifier:)` over direct subscript queries like `app.buttons["Save"]` when you want to match any identifying property (across title, label, or accessibilityIdentifier). This is more robust across localisation changes.
## Testing XPC Services

XPC services are tested by mocking the `NSXPCConnection` and your `@objc` protocol interface.

### Protocol Mock Pattern

```swift
import Testing

// Your XPC service protocol (must be @objc for NSXPCInterface)
@objc protocol DataSourceServiceProtocol {
    func fetchItems(matching query: String,
                    completion: @escaping ([String]?, Error?) -> Void)
    func cancelAllRequests()
}

@Suite("DataSourceXPCClient")
struct DataSourceXPCClientTests {

    // Mock that conforms to the same protocol
    final class MockDataSourceService: NSObject, DataSourceServiceProtocol {
        var fetchItemsHandler: ((String) -> ([String]?, Error?))?
        var cancelCallCount = 0

        func fetchItems(matching query: String,
                        completion: @escaping ([String]?, Error?) -> Void) {
            let (results, error) = fetchItemsHandler?(query) ?? ([], nil)
            completion(results, error)
        }

        func cancelAllRequests() {
            cancelCallCount += 1
        }
    }

    @Test("Successful fetch returns items")
    func successfulFetch() async throws {
        let mock = MockDataSourceService()
        mock.fetchItemsHandler = { query in
            (["item1", "item2"], nil)
        }

        let client = DataSourceXPCClient(service: mock)

        let items = try await client.search(query: "test")
        #expect(items.count == 2)
        #expect(items[0] == "item1")
    }

    @Test("Service error is propagated")
    func serviceError() async throws {
        let mock = MockDataSourceService()
        mock.fetchItemsHandler = { _ in
            (nil, NSError(domain: "test", code: 1))
        }

        let client = DataSourceXPCClient(service: mock)

        #expect(throws: NSError.self) {
            try await client.search(query: "test")
        }
    }

    @Test("Cancellation delegates to service")
    func cancellation() async throws {
        let mock = MockDataSourceService()
        let client = DataSourceXPCClient(service: mock)

        await client.cancel()
        #expect(mock.cancelCallCount == 1)
    }
}
```

### Wrapping the Mock as an XPC Connection

For integration tests closer to the real XPC machinery:

```swift
func createMockConnection(with service: DataSourceServiceProtocol) -> NSXPCConnection {
    let interface = NSXPCInterface(with: DataSourceServiceProtocol.self)

    // Configure allowed classes for collections
        interface.setClasses(NSSet(array: [NSArray.self, NSString.self]) as Set<NSObject>,
                         for: #selector(DataSourceServiceProtocol.fetchItems(matching:completion:)),
                         argumentIndex: 0,
                         ofReply: true)

    let connection = NSXPCConnection()
    connection.remoteObjectInterface = interface
    // In a test, you'd inject the service endpoint differently.
    // In practice, the mock pattern above (direct injection) is preferred
    // for unit testing the client logic.
    return connection
}
```

**Testing service in isolation:** Test the XPC service's implemention object directly (not through the connection) whenever possible. The connection plumbing is Apple's responsibility — test your business logic, not `NSXPCConnection`.

## xcodebuild test

### Running Tests

```bash
# Run all tests for a scheme on the local Mac
xcodebuild test \
  -scheme MyApp \
  -destination "platform=macOS"

# Run a specific test target
xcodebuild test \
  -scheme MyApp \
  -destination "platform=macOS" \
  -only-testing MyAppTests/NoteDocumentTests

# Run specific test
xcodebuild test \
  -scheme MyApp \
  -destination "platform=macOS" \
  -only-testing MyAppTests/NoteDocumentTests/roundTripPlainText
```

### Test-Without-Building (CI Split)

```bash
# Build for testing
xcodebuild build-for-testing \
  -scheme MyApp \
  -destination "platform=macOS" \
  -derivedDataPath /tmp/build

# Later, test without rebuilding — useful for splitting across CI workers
xcodebuild test-without-building \
  -scheme MyApp \
  -destination "platform=macOS" \
  -derivedDataPath /tmp/build
```
>
> ### macOS CI Destination Notes
>
> For GitHub Actions and other CI runners, destination specification must match the runner's OS version:
>
> ```bash
> # GitHub Actions macos-26 runner reports macOS 26.3
> # If your deployment target is macOS 26.4, the test destination will fail to resolve.
> # Workaround: set deployment target ≤ runner OS version, or use macos-15 (more stable)
> xcodebuild test \
>   -scheme MyApp \
>   -destination 'platform=macOS,arch=arm64' \
>   -skipPackagePluginValidation
>
> # For universal testing (arm64 + x86_64), chain destinations:
> xcodebuild test \
>   -scheme MyApp \
>   -destination 'platform=macOS,arch=arm64' \
>   -destination 'platform=macOS,arch=x86_64'
> ```
>
> **`-skipPackagePluginValidation`** is required in all CI environments — without it, the build hangs waiting for interactive plugin trust approval.

### Code Coverage

```bash
xcodebuild test \
  -scheme MyApp \
  -destination "platform=macOS" \
  -enableCodeCoverage YES \
  -resultBundlePath /tmp/MyApp.xcresult
```

### Result Bundle

The `-resultBundlePath` flag produces an `.xcresult` bundle containing test results, coverage, and logs. Open it with Xcode or parse with `xcrun xcresulttool`.

## Test Plan Configuration

Test plans (`.xctestplan`) give you fine-grained control over test execution — per-configuration settings, environment variables, and test selection.

### xctestplan Format

```json
{
  "configurations": [
    {
      "id": "Debug",
      "name": "Debug",
      "options": {
        "testExecutionOrdering": "random",
        "testTimeoutsEnabled": true
      }
    },
    {
      "id": "Release",
      "name": "Release",
      "options": {
        "codeCoverage": {
          "enabled": true,
          "targets": [
            { "containerPath": "container:MyApp.xcodeproj", "identifier": "MyApp", "name": "MyApp" }
          ]
        }
      }
    }
  ],
  "defaultOptions": {
    "commandLineArgumentEntries": [
      { "argument": "-com.apple.SystemLogger" }
    ],
    "environmentVariableEntries": [
      { "key": "TEST_DATA_DIR", "value": "$(SRCROOT)/TestData" },
      { "key": "MOCK_SERVER_URL", "value": "http://localhost:8080" }
    ],
    "targetForVariableExpansion": {
      "containerPath": "container:MyApp.xcodeproj",
      "identifier": "MyApp",
      "name": "MyApp"
    },
    "testRepetitionMode": "none",
    "testTimeoutsEnabled": false
  },
  "testTargets": [
    {
      "target": {
        "containerPath": "container:MyApp.xcodeproj",
        "identifier": "MyAppTests",
        "name": "MyAppTests"
      },
      "enabled": true
    },
    {
      "target": {
        "containerPath": "container:MyApp.xcodeproj",
        "identifier": "MyAppUITests",
        "name": "MyAppUITests"
      },
      "enabled": false,
      "skippedTests": [
        "AppUITests/newDocumentMenu"
      ]
    }
  ],
  "version": 1
}
```

### Configuration Options

| Option | Description |
|--------|-------------|
| `testExecutionOrdering` | `"random"` (default) or `"lexicographic"` — random catches order-dependent bugs |
| `testTimeoutsEnabled` | Globally enable time limits for tests |
| `codeCoverage.enabled` | Collect code coverage data |
| `testRepetitionMode` | `"none"`, `"retryOnFailure"`, or `"fixedIterations:N"` — for flaky test detection |
| `environmentVariableEntries` | Set variables accessible via `ProcessInfo.processInfo.environment` |

### Using a Test Plan

```bash
# Specify test plan via xcodebuild
xcodebuild test \
  -scheme MyApp \
  -destination "platform=macOS" \
  -testPlan MyTestPlan
```

Or set the test plan in the scheme editor in Xcode (Product > Scheme > Edit > Test > Test Plans).

## Constraints & Gotchas

- **Main actor isolation:** AppKit requires main-thread access. Tests that touch `NSView`, `NSViewController`, `NSWindow`, or `NSDocument` must run on the main actor. Use `@MainActor` on the suite or individual test functions. Swift Testing respects actor-isolation — a non-isolated test cannot call a main-actor method.

- **Async testing natively:** Swift Testing tests can be `async` directly — no `XCTestExpectation` needed. Just mark the test `async` and `await` results. For `NSDocument` async save operations:

  ```swift
  @Test("Async save completes") func asyncSave() async throws {
      let doc = NoteDocument()
      doc.text = "async content"
      try await doc.write(to: tempURL, ofType: "public.plain-text", saveOperation: .saveOperation)
      #expect(FileManager.default.fileExists(atPath: tempURL.path))
  }
  ```

- **Test ordering is non-deterministic by default.** Swift Testing shuffles test execution order. If tests share mutable state, you have a bug. Each test must set up and tear down its own environment.

- **No per-test class-level setup/teardown.** XCTest has `setUp`/`tearDown`. In Swift Testing, do setup in the test body or in a helper called at the start of each test. The `init`/`deinit` of a `class` suite can serve this role.

- **Temporary files in tests.** Use `temporaryDirectory()` or create files in `$TMPDIR`:

  ```swift
  func temporaryURL(for filename: String) -> URL {
      URL(fileURLWithPath: NSTemporaryDirectory())
          .appendingPathComponent(UUID().uuidString)
          .appendingPathComponent(filename)
  }
  ```

- **NSDocument requires no window server** for basic I/O testing. `data(ofType:)` and `read(from:ofType:)` don't instantiate windows or views. You can test document model logic without a GUI.

- **XCTest UI tests require accessibility.** Every element you query must have an `accessibilityIdentifier` set in code. NSToolbar buttons, NSMenu items, and standard NSView controls are partially accessible, but custom views need explicit identifiers.

## Common Mistakes

- **Testing through the UI what should be unit tested.** Use `XCTest UI tests` only for verifying user workflows (menu clicks, window flow). Test document I/O, model logic, and controller behavior in Swift Testing unit tests.

- **Forgetting @MainActor on AppKit tests.** A test that accesses `NSView` or `NSViewController` without `@MainActor` will compile but crash at runtime or trigger strict concurrency warnings in Swift 6.

- **Using #expect when #require is appropriate.** If a nil value would cause a crash on the next line, use `#require` to abort early. `#expect(value != nil)` followed by force-unwrap is an anti-pattern.

- **Sharing state between tests.** Don't use instance variables on a `class` suite to share data across tests. The execution order is nondeterministic. Use the suite's `init` to create fresh state, or better yet, create everything inside each test.

- **Not cleaning up temporary files.** Tests that create files in `$TMPDIR` should remove them. Swift Testing has `defer` — use it:

  ```swift
  @Test("File operation") func fileOp() throws {
      let tempURL = temporaryURL(for: "test.txt")
      defer { try? FileManager.default.removeItem(at: tempURL) }
      // ... test code
  }
  ```

- **Skipping flaky CI tests with `.disabled()` permanently.** `.disabled()` should have a comment with a radar number and a plan to fix. Prefer `.enabled(if: ProcessInfo.processInfo.environment["CI"] == nil)` for environment-specific issues while you investigate.

## Review Checklist

- [ ] Tests use `@Suite` for grouping, not bare top-level functions
- [ ] AppKit tests are annotated `@MainActor`
- [ ] `#require` is used for preconditions that would crash if false
- [ ] Parameterized tests use `arguments:` with clear case descriptions
- [ ] Tags are applied to tests for CI filtering
- [ ] Temporary files are cleaned up with `defer`
- [ ] No shared mutable state between tests
- [ ] `.disabled()` includes a reason and radar/issue number
- [ ] XCTest UI tests set `accessibilityIdentifier` on custom views
- [ ] XPC service tests mock the protocol directly, not the connection plumbing
- [ ] Test plans use `testExecutionOrdering: "random"` to catch order-dependent bugs

## References

- [references/testing-patterns.md](references/testing-patterns.md) — Parameterized suites, XCTest UI tests, mock XPC services, NSDocument testing, and test plan configuration patterns
- [Apple: Swift Testing Documentation](https://developer.apple.com/documentation/testing)
- [Apple: XCTest Documentation](https://developer.apple.com/documentation/xctest)
- [Apple: Testing Swift Code with XCTest (WWDC)](https://developer.apple.com/videos/play/wwdc2024/10195/)
- [Testing asynchronous code | Apple Developer Documentation](https://developer.apple.com/documentation/testing/testing-asynchronous-code)
- [Testing completion handler APIs with Swift Testing — Donny Wals](https://www.donnywals.com/testing-completion-handler-apis-with-swift-testing/)
- [XCUIElementQuery matching(identifier:) | Apple Developer Documentation](https://developer.apple.com/documentation/xcuiautomation/xcuielementquery/matching(identifier:))
- [XCUIElementQuery descendants(matching:) | Apple Developer Documentation](https://developer.apple.com/documentation/xcuiautomation/xcuielementquery/descendants(matching:))
- [Swift 6.3 Release Notes — swift-testing](https://github.com/swiftlang/swift-testing/releases/tag/swift-6.3-RELEASE)
