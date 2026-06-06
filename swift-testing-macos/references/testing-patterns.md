# Testing Patterns Reference

Deep-dive patterns for testing macOS apps with Swift Testing and XCTest. Targets macOS 26+ with Swift 6.3.

## Contents

- [@Suite with Parameterized Swift Testing](#suite-with-parameterized-swift-testing)
- [XCTest UI Test for macOS with Menu Bar Interaction](#xctest-ui-test-for-macos-with-menu-bar-interaction)
- [Mock XPC Service with Protocol Mock](#mock-xpc-service-with-protocol-mock)
- [Testing NSDocument Reading/Writing](#testing-nsdocument-readingwriting)
- [xcodebuild Test Plan Configuration](#xcodebuild-test-plan-configuration)

## @Suite with Parameterized Swift Testing

When you need to run an entire suite of tests against multiple configurations, use a parameterized suite pattern. Each test in the suite receives the shared parameter via the suite's initializer.

```swift
import Testing
@testable import MyApp

// The parameter type
struct DocumentFormat {
    let name: String
    let typeName: String
    let fileExtension: String
    let sampleContent: String
}

extension DocumentFormat {
    static let plainText = DocumentFormat(
        name: "Plain Text",
        typeName: "public.plain-text",
        fileExtension: "txt",
        sampleContent: "Hello, World!"
    )
    static let markdown = DocumentFormat(
        name: "Markdown",
        typeName: "net.daringfireball.markdown",
        fileExtension: "md",
        sampleContent: "# Heading\n\nSome *emphasized* text."
    )
    static let json = DocumentFormat(
        name: "JSON",
        typeName: "public.json",
        fileExtension: "json",
        sampleContent: #"{"key": "value", "count": 42}"#
    )
}

// Suite that takes a parameter — each test in the suite uses it
@Suite("Document I/O")
struct DocumentIOTests {
    let format: DocumentFormat

    init(format: DocumentFormat) {
        self.format = format
    }

    @Test("Round-trip preserves content")
    func roundTrip() throws {
        let doc = NoteDocument()
        doc.text = format.sampleContent

        let data = try doc.data(ofType: format.typeName)
        let doc2 = NoteDocument()
        try doc2.read(from: data, ofType: format.typeName)

        #expect(doc2.text == format.sampleContent)
    }

    @Test("File extension is correct")
    func fileExtension() {
        let doc = NoteDocument()
        let ext = doc.fileNameExtension(forType: format.typeName, saveOperation: .saveAsOperation)
        #expect(ext == format.fileExtension)
    }

    @Test("Empty content produces valid data")
    func emptyContent() throws {
        let doc = NoteDocument()
        doc.text = ""
        let data = try doc.data(ofType: format.typeName)
        #expect(!data.isEmpty || format.typeName == "public.plain-text")
    }
}

// Instantiate the suite for each format — Swift Testing discovers these
@Suite("Document I/O — Plain Text")
struct PlainTextIOTests: DocumentIOTests {
    init() { super.init(format: .plainText) }
}

@Suite("Document I/O — Markdown")
struct MarkdownIOTests: DocumentIOTests {
    init() { super.init(format: .markdown) }
}

@Suite("Document I/O — JSON")
struct JSONIOTests: DocumentIOTests {
    init() { super.init(format: .json) }
}
```

**Alternative: Single parameterized test across formats.** For simpler cases, a single `@Test(arguments:)` is cleaner:

```swift
@Test("All formats round-trip", arguments: [
    DocumentFormat.plainText,
    DocumentFormat.markdown,
    DocumentFormat.json,
])
func allFormatsRoundTrip(format: DocumentFormat) throws {
    let doc = NoteDocument()
    doc.text = format.sampleContent
    let data = try doc.data(ofType: format.typeName)
    let doc2 = NoteDocument()
    try doc2.read(from: data, ofType: format.typeName)
    #expect(doc2.text == format.sampleContent)
}
```

**When to use suite-level parameterization:** When you have 5+ tests that all need the same configuration, and you want them grouped under a shared name in the test navigator. When to use test-level parameterization: when only 1-2 tests need the parameter, or when different tests need different parameter sets.

## XCTest UI Test for macOS with Menu Bar Interaction

This pattern tests a complete user workflow: creating a new document, typing content, and saving via the menu bar.

```swift
import XCTest

final class DocumentWorkflowUITests: XCTestCase {

    var app: XCUIApplication!

    override func setUpWithError() throws {
        continueAfterFailure = false
        app = XCUIApplication()
        // Pass a launch arg to use a temp directory for test data
        app.launchArguments = ["--test-mode", "--test-data-dir", NSTemporaryDirectory()]
        app.launch()
    }

    override func tearDownWithError() throws {
        // Terminate cleanly
        app.terminate()
        app = nil
    }

    // MARK: - Menu Bar Helpers

    private func clickMenuItem(menu: String, item: String) {
        let menuBarItem = app.menuBars.buttons[menu]
        guard menuBarItem.waitForExistence(timeout: 5) else {
            XCTFail("Menu '\(menu)' not found")
            return
        }
        menuBarItem.click()

        let menuItem = app.menuBars.buttons[item]
        guard menuItem.waitForExistence(timeout: 3) else {
            XCTFail("Menu item '\(item)' not found in '\(menu)' menu")
            // Dismiss the menu by pressing Escape
            app.typeKey(.escape, modifierFlags: [])
            return
        }
        menuItem.click()
    }

    private func dismissOpenPanel() {
        let cancelButton = app.sheets.buttons["Cancel"]
        if cancelButton.waitForExistence(timeout: 2) {
            cancelButton.click()
        }
    }

    // MARK: - Tests

    func testNewDocumentCreatesWindow() throws {
        let initialWindowCount = app.windows.count

        clickMenuItem(menu: "File", item: "New")

        // Wait for new window
        let newWindow = app.windows.element(boundBy: initialWindowCount)
        XCTAssertTrue(newWindow.waitForExistence(timeout: 5), "New window should appear")
    }

    func testTypeAndSaveDocument() throws {
        // Create new document
        clickMenuItem(menu: "File", item: "New")
        let window = app.windows.firstMatch
        XCTAssertTrue(window.waitForExistence(timeout: 5))

        // Find the text view — set accessibilityIdentifier in your NSView subclass
        let textView = window.textViews["document-text-view"]
        XCTAssertTrue(textView.waitForExistence(timeout: 5), "Text view should exist")

        // Type content
        textView.click()
        textView.typeText("UI test content")

        // Verify content appeared
        XCTAssertTrue(textView.value as? String == "UI test content" ||
                      textView.stringValue.contains("UI test content"))

        // Save via menu
        clickMenuItem(menu: "File", item: "Save…")

        // Handle save panel
        let savePanel = app.sheets.firstMatch
        XCTAssertTrue(savePanel.waitForExistence(timeout: 5), "Save panel should appear")

        // Type filename
        let filenameField = savePanel.textFields["Save As:"]
        if filenameField.waitForExistence(timeout: 3) {
            filenameField.click()
            filenameField.typeText("UITestDoc")
        }

        // Click Save
        let saveButton = savePanel.buttons["Save"]
        if saveButton.waitForExistence(timeout: 3) {
            saveButton.click()
        }

        // Handle overwrite confirmation if it appears
        let overwriteSheet = app.sheets.firstMatch
        let replaceButton = overwriteSheet.buttons["Replace"]
        if replaceButton.waitForExistence(timeout: 2) {
            replaceButton.click()
        }

        // Verify window title updated
        XCTAssertTrue(window.title.contains("UITestDoc") ||
                      window.waitForExistence(timeout: 3))
    }

    func testCloseAndReopenDocument() throws {
        // Create and save a document first
        clickMenuItem(menu: "File", item: "New")
        let window = app.windows.firstMatch
        XCTAssertTrue(window.waitForExistence(timeout: 5))

        let textView = window.textViews["document-text-view"]
        if textView.waitForExistence(timeout: 3) {
            textView.click()
            textView.typeText("persistent content")
        }

        // Save
        clickMenuItem(menu: "File", item: "Save…")
        let savePanel = app.sheets.firstMatch
        _ = savePanel.waitForExistence(timeout: 5)
        dismissOpenPanel() // Simplified: just cancel for this example

        // Close
        clickMenuItem(menu: "File", item: "Close")
        // Handle save changes dialog if it appears
        let dontSave = app.sheets.buttons["Don't Save"]
        if dontSave.waitForExistence(timeout: 2) {
            dontSave.click()
        }
    }
}
```

**Key patterns:**
- Always use `waitForExistence(timeout:)` before interacting with elements — macOS UI has animation delays
- Set `accessibilityIdentifier` on custom `NSView` subclasses to make them queryable
- Handle sheets (modal panels) explicitly — they block interaction with the main window
- Use `app.typeKey(.escape, modifierFlags: [])` to dismiss menus that get stuck

## Mock XPC Service with Protocol Mock

This pattern shows a complete mock XPC service setup, including the async wrapper that bridges the callback-based XPC protocol to Swift concurrency.

```swift
import Testing
@testable import MyApp

// The XPC protocol (must be @objc)
@objc protocol PreferencesServiceProtocol {
    func getPreference(for key: String, completion: @escaping (Data?, Error?) -> Void)
    func setPreference(for key: String, value: Data, completion: @escaping (Error?) -> Void)
    func resetAllPreferences(completion: @escaping (Error?) -> Void)
}

// The production client that wraps the XPC connection
final class PreferencesService: Sendable {
    private let connection: NSXPCConnection

    init(connection: NSXPCConnection) {
        self.connection = connection
        let interface = NSXPCInterface(with: PreferencesServiceProtocol.self)
        connection.remoteObjectInterface = interface
        connection.resume()
    }

    func getPreference(for key: String) async throws -> Data {
        try await withCheckedThrowingContinuation { continuation in
            let proxy = connection.remoteObjectProxyWithErrorHandler { error in
                continuation.resume(throwing: error)
            } as! PreferencesServiceProtocol

            proxy.getPreference(for: key) { data, error in
                if let error {
                    continuation.resume(throwing: error)
                } else if let data {
                    continuation.resume(returning: data)
                } else {
                    continuation.resume(throwing: PreferencesError.noData)
                }
            }
        }
    }

    func setPreference(for key: String, value: Data) async throws {
        try await withCheckedThrowingContinuation { (continuation: CheckedContinuation<Void, Error>) in
            let proxy = connection.remoteObjectProxyWithErrorHandler { error in
                continuation.resume(throwing: error)
            } as! PreferencesServiceProtocol

            proxy.setPreference(for: key, value: value) { error in
                if let error {
                    continuation.resume(throwing: error)
                } else {
                    continuation.resume()
                }
            }
        }
    }
}

enum PreferencesError: Error {
    case noData
}

// MARK: - Mock

@Suite("PreferencesService")
struct PreferencesServiceTests {

    final class MockPreferencesService: NSObject, PreferencesServiceProtocol {
        var preferences: [String: Data] = [:]
        var errorToReturn: Error?
        var setCallCount = 0
        var resetCallCount = 0

        func getPreference(for key: String, completion: @escaping (Data?, Error?) -> Void) {
            if let error = errorToReturn {
                completion(nil, error)
                return
            }
            completion(preferences[key], nil)
        }

        func setPreference(for key: String, value: Data, completion: @escaping (Error?) -> Void) {
            setCallCount += 1
            if let error = errorToReturn {
                completion(error)
                return
            }
            preferences[key] = value
            completion(nil)
        }

        func resetAllPreferences(completion: @escaping (Error?) -> Void) {
            resetCallCount += 1
            if let error = errorToReturn {
                completion(error)
                return
            }
            preferences.removeAll()
            completion(nil)
        }
    }

    // Test the mock directly (no XPC connection needed)
    @Test("Get returns stored preference")
    func getPreference() async throws {
        let mock = MockPreferencesService()
        let expected = "test-value".data(using: .utf8)!
        mock.preferences["username"] = expected

        let result = try await mock.getPreferenceSync(for: "username")
        #expect(result == expected)
    }

    @Test("Get returns nil for missing key")
    func getMissingKey() async throws {
        let mock = MockPreferencesService()
        let result = try await mock.getPreferenceSync(for: "nonexistent")
        #expect(result == nil)
    }

    @Test("Set stores the value")
    func setPreference() async throws {
        let mock = MockPreferencesService()
        let value = "new-value".data(using: .utf8)!

        try await mock.setPreferenceSync(for: "theme", value: value)

        #expect(mock.setCallCount == 1)
        #expect(mock.preferences["theme"] == value)
    }

    @Test("Error is propagated from get")
    func getError() async throws {
        let mock = MockPreferencesService()
        mock.errorToReturn = NSError(domain: "test", code: 42)

        #expect(throws: NSError.self) {
            try await mock.getPreferenceSync(for: "any")
        }
    }

    @Test("Reset clears all preferences")
    func resetAll() async throws {
        let mock = MockPreferencesService()
        mock.preferences["a"] = Data([1])
        mock.preferences["b"] = Data([2])

        try await mock.resetAllPreferencesSync()

        #expect(mock.resetCallCount == 1)
        #expect(mock.preferences.isEmpty)
    }
}

// Helper extensions to bridge callback-based mock to async/await for testing
extension MockPreferencesService {
    func getPreferenceSync(for key: String) async throws -> Data? {
        await withCheckedContinuation { continuation in
            getPreference(for: key) { data, error in
                if let error {
                    continuation.resume(throwing: error)
                } else {
                    continuation.resume(returning: data)
                }
            }
        }
    }

    func setPreferenceSync(for key: String, value: Data) async throws {
        try await withCheckedThrowingContinuation { (continuation: CheckedContinuation<Void, Error>) in
            setPreference(for: key, value: value) { error in
                if let error { continuation.resume(throwing: error) }
                else { continuation.resume() }
            }
        }
    }

    func resetAllPreferencesSync() async throws {
        try await withCheckedThrowingContinuation { (continuation: CheckedContinuation<Void, Error>) in
            resetAllPreferences { error in
                if let error { continuation.resume(throwing: error) }
                else { continuation.resume() }
            }
        }
    }
}
```

**Testing strategy:** Test the client logic (the `withCheckedThrowingContinuation` wrapper) separately from the mock. The mock verifies your protocol implementation; the client tests verify the async bridging. In practice, the mock pattern above tests the service implementation directly — which is where the business logic lives.

## Testing NSDocument Reading/Writing

Complete pattern for testing an `NSDocument` subclass with multiple file types, error handling, and edge cases.

```swift
import Testing
@testable import MyApp

@Suite("ProjectDocument I/O")
struct ProjectDocumentTests {

    // MARK: - Helpers

    func makeDocument(with project: Project? = nil) -> ProjectDocument {
        let doc = ProjectDocument()
        if let project {
            doc.project = project
        }
        return doc
    }

    func temporaryURL(for filename: String) -> URL {
        URL(fileURLWithPath: NSTemporaryDirectory())
            .appendingPathComponent(UUID().uuidString)
            .appendingPathComponent(filename)
    }

    // MARK: - Round-Trip Tests

    @Test("Round-trip JSON project")
    func roundTripJSON() throws {
        let doc = makeDocument(with: Project(
            title: "Test Project",
            items: [
                Item(name: "Task 1", done: false),
                Item(name: "Task 2", done: true),
            ],
            metadata: ["version": "1.0"]
        ))

        let data = try doc.data(ofType: "com.myapp.project")
        let doc2 = makeDocument()
        try doc2.read(from: data, ofType: "com.myapp.project")

        #expect(doc2.project.title == "Test Project")
        #expect(doc2.project.items.count == 2)
        #expect(doc2.project.items[0].name == "Task 1")
        #expect(doc2.project.items[1].done == true)
        #expect(doc2.project.metadata["version"] == "1.0")
    }

    @Test("Round-trip plain text export")
    func roundTripPlainText() throws {
        let doc = makeDocument(with: Project(title: "Export Test", items: [], metadata: [:]))
        let data = try doc.data(ofType: "public.plain-text")
        let text = String(data: data, encoding: .utf8)

        #expect(text != nil)
        #expect(text!.contains("Export Test"))
    }

    // MARK: - Error Handling

    @Test("Read throws on corrupted data")
    func corruptedData() {
        let doc = makeDocument()
        let garbage = Data([0x00, 0x01, 0x02, 0xFF, 0xFE])

        #expect(throws: NSError.self) {
            try doc.read(from: garbage, ofType: "com.myapp.project")
        }
    }

    @Test("Read throws on empty data")
    func emptyData() {
        let doc = makeDocument()

        #expect(throws: NSError.self) {
            try doc.read(from: Data(), ofType: "com.myapp.project")
        }
    }

    @Test("Read throws on unknown type")
    func unknownType() throws {
        let doc = makeDocument(with: Project(title: "x", items: [], metadata: [:]))
        let data = try doc.data(ofType: "com.myapp.project")

        #expect(throws: NSError.self) {
            try doc.read(from: data, ofType: "completely.unknown.type")
        }
    }

    // MARK: - File Type Support

    @Test("Writable types for save-as", arguments: [
        NSDocument.SaveOperationType.saveAsOperation,
        NSDocument.SaveOperationType.saveToOperation,
    ])
    func writableTypes(for operation: NSDocument.SaveOperationType) {
        let doc = makeDocument()
        let types = doc.writableTypes(for: operation)
        #expect(types.contains("com.myapp.project"))
    }

    @Test("File extension for type", arguments: [
        ("com.myapp.project", "mproject"),
        ("public.plain-text", "txt"),
        ("public.json", "json"),
    ])
    func fileExtension(typeName: String, expectedExtension: String) {
        let doc = makeDocument()
        let ext = doc.fileNameExtension(forType: typeName, saveOperation: .saveOperation)
        #expect(ext == expectedExtension)
    }

    // MARK: - Change Tracking

    @Test("Document starts clean")
    func startsClean() {
        let doc = makeDocument()
        #expect(!doc.isDocumentEdited)
    }

    @Test("Modifying project marks document dirty")
    func modificationMarksDirty() {
        let doc = makeDocument(with: Project(title: "Original", items: [], metadata: [:]))
        doc.updateChangeCount(.changeCleared) // Reset

        doc.project.title = "Modified"
        doc.updateChangeCount(.changeDone)

        #expect(doc.isDocumentEdited)
    }

    // MARK: - Direct URL I/O

    @Test("Write to URL and read back")
    func writeToURL() throws {
        let doc = makeDocument(with: Project(title: "URL Test", items: [], metadata: [:]))
        let url = temporaryURL(for: "test.mproject")
        defer { try? FileManager.default.removeItem(at: url.deletingLastPathComponent()) }

        try doc.write(to: url, ofType: "com.myapp.project", saveOperation: .saveOperation)

        let doc2 = makeDocument()
        try doc2.read(from: url, ofType: "com.myapp.project")

        #expect(doc2.project.title == "URL Test")
    }

    @Test("Revert reloads from disk")
    func revert() throws {
        let doc = makeDocument(with: Project(title: "Original", items: [], metadata: [:]))
        let url = temporaryURL(for: "revert-test.mproject")
        defer { try? FileManager.default.removeItem(at: url.deletingLastPathComponent()) }

        // Save original
        try doc.write(to: url, ofType: "com.myapp.project", saveOperation: .saveOperation)

        // Modify in memory
        doc.project.title = "Modified"
        doc.updateChangeCount(.changeDone)
        #expect(doc.isDocumentEdited)

        // Revert
        try doc.revertToContents(of: url, ofType: "com.myapp.project")

        #expect(doc.project.title == "Original")
        #expect(!doc.isDocumentEdited)
    }
}
```

**Key patterns:**
- Test each file type your document supports
- Test error paths (corrupted data, empty data, unknown type) — not just the happy path
- Test change tracking (`isDocumentEdited`) if your UI depends on it
- Test URL-based I/O (`read(from:ofType:)` / `write(to:ofType:saveOperation:)`) in addition to Data-based I/O
- Always clean up temp files with `defer`

## xcodebuild Test Plan Configuration

A complete test plan for a macOS app with CI and local configurations.

```json
{
  "configurations": [
    {
      "id": "Local",
      "name": "Local Development",
      "options": {
        "testExecutionOrdering": "random",
        "testTimeoutsEnabled": false,
        "codeCoverage": {
          "enabled": false
        },
        "uiTestingScreenshotsLifetime": "keepAlways"
      }
    },
    {
      "id": "CI",
      "name": "CI Pipeline",
      "options": {
        "testExecutionOrdering": "random",
        "testTimeoutsEnabled": true,
        "testTimeouts": {
          "defaultTimeout": 300,
          "perTestTimeout": 600
        },
        "codeCoverage": {
          "enabled": true,
          "targets": [
            {
              "containerPath": "container:MyApp.xcodeproj",
              "identifier": "MyApp",
              "name": "MyApp"
            }
          ]
        },
        "testRepetitionMode": "retryOnFailure",
        "maximumTestRepetitions": 3,
        "uiTestingScreenshotsLifetime": "deleteOnSuccess"
      }
    },
    {
      "id": "Release",
      "name": "Release Validation",
      "options": {
        "testExecutionOrdering": "random",
        "testTimeoutsEnabled": true,
        "codeCoverage": {
          "enabled": true
        },
        "testRepetitionMode": "fixedIterations",
        "maximumTestRepetitions": 5
      }
    }
  ],
  "defaultOptions": {
    "commandLineArgumentEntries": [],
    "environmentVariableEntries": [
      {
        "key": "TEST_DATA_DIR",
        "value": "$(SRCROOT)/Tests/TestData"
      },
      {
        "key": "ENABLE_MOCK_SERVICES",
        "value": "1"
      }
    ],
    "targetForVariableExpansion": {
      "containerPath": "container:MyApp.xcodeproj",
      "identifier": "MyApp",
      "name": "MyApp"
    },
    "testTimeoutsEnabled": false,
    "testRepetitionMode": "none"
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
      "enabled": true,
      "skippedTests": [
        "DocumentWorkflowUITests/testCloseAndReopenDocument"
      ]
    },
    {
      "target": {
        "containerPath": "container:MyApp.xcodeproj",
        "identifier": "MyAppPerformanceTests",
        "name": "MyAppPerformanceTests"
      },
      "enabled": false
    }
  ],
  "version": 1
}
```

**Running with a specific configuration:**

```bash
# Use the CI configuration
xcodebuild test \
  -scheme MyApp \
  -destination "platform=macOS" \
  -testPlan MyTestPlan \
  -test-configuration CI

# Use the Local configuration (default)
xcodebuild test \
  -scheme MyApp \
  -destination "platform=macOS" \
  -testPlan MyTestPlan
```

**CI pipeline integration:**

```yaml
# GitHub Actions example
jobs:
  test:
    runs-on: macos-15
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.4.app

      - name: Run Tests
        run: |
          xcodebuild test \
            -scheme MyApp \
            -destination "platform=macOS" \
            -testPlan MyTestPlan \
            -test-configuration CI \
            -resultBundlePath ${{ runner.temp }}/TestResults.xcresult \
            -quiet

      - name: Upload Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: ${{ runner.temp }}/TestResults.xcresult
```

**Configuration selection rationale:**

| Config | When to use | Key settings |
|--------|-------------|--------------|
| `Local` | Day-to-day development in Xcode | No timeouts, screenshots kept, no coverage overhead |
| `CI` | Automated pipeline | Timeouts enabled, retry on failure (3x), coverage on, delete screenshots on success |
| `Release` | Pre-release validation | Fixed 5 iterations to catch flaky tests, full coverage |

**Test repetition modes:**
- `none` — run each test once (default)
- `retryOnFailure` — retry failed tests up to `maximumTestRepetitions` times. A test passes if any repetition passes. Good for CI where flaky tests waste build minutes.
- `fixedIterations` — run every test exactly N times. Good for stress-testing and catching intermittent bugs before release.
