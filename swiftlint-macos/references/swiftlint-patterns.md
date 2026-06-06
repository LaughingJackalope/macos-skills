# SwiftLint Patterns Reference

Deep reference for SwiftLint configuration on macOS 26+ projects. Complete `.swiftlint.yml`, custom rule definitions for AppKit anti-patterns, SPM build tool plugin setup, GitHub Actions CI workflow, and baseline management commands.

## Complete .swiftlint.yml for macOS Project

```yaml
# .swiftlint.yml — macOS 26+ project configuration
# Place at project root. SwiftLint walks up from each file's directory to find this.

# ─── Paths ───────────────────────────────────────────────────────────────────
included:
  - Sources
  - App
  - MyMacOSApp
  - MyMacOSAppTests

excluded:
  - "**/Generated"
  - "**/.build"
  - "**/Package.swift"
  - "**/Carthage"
  - "**/Pods"
  - "**/*+SwiftSyntax.swift"
  - Sources/Generated

# ─── Rule Toggles ────────────────────────────────────────────────────────────
disabled_rules:
  - trailing_comma
  - opening_brace
  - legacy_objc_type

opt_in_rules:
  - array_init
  - capture_variable
  - discarded_observation
  - discouraged_none_name
  - discouraged_object_literal
  - discouraged_optional_collection
  - empty_collection_literal
  - empty_count
  - explicit_self
  - legacy_random
  - literal_expression_end_indentation
  - modifier_order
  - multiline_arguments_brackets
  - multiline_parameters_brackets
  - operator_usage_whitespace
  - prefer_self_in_static_references
  - redundant_type_annotation
  - strict_fileprivate
  - toggle_bool
  - typesafe_array_init
  - unneeded_override
  - unused_capture_list
  - vertical_whitespace_closing_braces

analyzer_rules:
  - unused_declaration
  - unused_import

# ─── Rule Configuration ──────────────────────────────────────────────────────
line_length:
  warning: 140
  error: 200
  ignores_urls: true
  ignores_function_declarations: false
  ignores_comments: true

type_body_length:
  warning: 300
  error: 500

function_body_length:
  warning: 60
  error: 120

cyclomatic_complexity:
  warning: 10
  error: 20
  ignores_case_statements: true

nesting:
  type_level:
    warning: 2
    error: 4
  function_level:
    warning: 4
    error: 8

large_tuple:
  warning: 3
  error: 4

identifier_name:
  min_length:
    warning: 2
    error: 1
  max_length:
    warning: 50
    error: 60
  excluded:
    - id
    - x
    - y
    - vc
    - wc
    - db
    - to
  allowed_symbols: ["_"]

type_name:
  min_length: 3
  max_length: 50
  excluded: []

attributes:
  always_on_same_line: []
  always_on_own_line: ["@objc", "@IBOutlet", "@IBSegueAction", "@MainActor", "@StateObject", "@Published"]

force_unwrapping:
  severity: error

force_try:
  severity: error

prohibited_interface_builder:
  severity: warning

# ─── Custom Rules ────────────────────────────────────────────────────────────
custom_rules:
  # Disallow NSLog — use os.log Logger
  no_nslog:
    name: "Use os_log or Logger"
    regex: 'NSLog\('
    message: "Use Logger from os.log instead of NSLog for structured logging"
    severity: error
    match_kinds:
      - identifier

  # Disallow NSLocalizedString — use LocalizedStringResource
  no_nslocalizedstring:
    name: "Use LocalizedStringResource"
    regex: 'NSLocalizedString\('
    message: "Use LocalizedStringResource or #localized instead of NSLocalizedString"
    severity: warning
    match_kinds:
      - identifier

  # Enforce completion handler naming
  completion_handler_name:
    name: "Completion Handler Naming"
    regex: 'completion(?:Handler)?:\s*\((?:[^)]*)\)\s*->\s*Void'
    message: "Completion handlers should be named 'completion:' or 'onCompletion:'"
    severity: warning
    match_kinds:
      - argument

  # Detect unwrapped IBOutlets
  unwrapped_iboutlet:
    name: "Unwrapped IBOutlet"
    regex: '@IBOutlet\s+(?:weak\s+)?var\s+\w+!\s*:\s*(?:NS|UI)'
    message: "IBOutlets should use Optional (?) or ImplicitlyUnwrappedOptional only when necessary. Prefer Optional with guard-let in viewDidLoad."
    severity: error

  # Disallow fatalError in non-test code
  no_fatal_error:
    name: "No fatalError"
    regex: 'fatalError\('
    message: "fatalError crashes in production. Use graceful error handling."
    severity: error
    excluded:
      - "**/*Tests.swift"
      - "**/*Test.swift"
    match_kinds:
      - identifier

  # Enforce weak self in AppKit closures
  weak_self_in_closure:
    name: "Weak Self in Closure"
    regex: '\{[^}]*\bself\.[a-z]'
    message: "Use [weak self] in AppKit closures to avoid retain cycles with NSWindowController/NSDocument"
    severity: warning
    excluded:
      - "**/Tests/**"

  # Disallow print() in production code
  no_print:
    name: "No print()"
    regex: 'print\('
    message: "Use Logger instead of print() for debug output"
    severity: warning
    excluded:
      - "**/*Tests.swift"
      - "**/*Test.swift"
      - "**/Preview Content/**"
    match_kinds:
      - identifier

  # Enforce NS prefix only for AppKit wrapper types
  ns_prefix_misuse:
    name: "NS Prefix Misuse"
    regex: 'class\s+NS(?!View|Window|Controller|Document|Application|Responder|Array|Dictionary|Set|String|Data|Number|Date|URL|Request|Response|Error|Notification|Timer|Operation|Lock|Thread|Port|Socket|Pipe|Handle|Image|Color|Font|BezierPath|Gradient|Shadow|Appearance|Menu|MenuItem|TableView|CollectionView|ScrollView|SplitView|StackView|VisualEffectView|TouchBar|GestureRecognizer|PanGestureRecognizer|TapGestureRecognizer|PressGestureRecognizer|RotationGestureRecognizer|MagnificationGestureRecognizer)\w+'
    message: "Only use NS prefix when wrapping or extending an existing AppKit type"
    severity: warning
    match_kinds:
      - typeidentifier

  # Discourage NSObject subclassing in pure Swift
  nsobject_subclass:
    name: "NSObject Subclass"
    regex: 'class\s+\w+\s*:\s*NSObject\b'
    message: "Avoid subclassing NSObject unless required for Objective-C interop, KVO, or Core Data"
    severity: warning
    match_kinds:
      - typeidentifier

  # Enforce @objc naming conventions
  objc_selector_naming:
    name: "@objc Selector Naming"
    regex: '@objc\s*\(\s*selector:\s*"([a-z]+[A-Z][^"]*)"'
    message: "@objc selectors should use Objective-C naming (camelCase starting with lowercase)"
    severity: warning
    match_kinds:
      - attribute

  # Detect empty catch blocks
  empty_catch:
    name: "Empty Catch Block"
    regex: 'catch\s*\{[\s\n]*\}'
    message: "Empty catch blocks silently swallow errors. Log or handle the error."
    severity: error

  # Enforce MARK comments for organization
  missing_mark_sections:
    name: "Missing MARK Comments"
    regex: 'class\s+\w+\s*:\s*\w+\s*\{[\s\n]*(?!// MARK:)'
    message: "Large types should use // MARK: comments to organize code sections"
    severity: warning
    match_kinds:
      - typeidentifier

# ─── Cache ───────────────────────────────────────────────────────────────────
cache_path: .swiftlint_cache

# ─── Strict Mode (config-level) ──────────────────────────────────────────────
# Uncomment to make all runs strict by default:
# strict: true
```

## Custom Rule Definitions for AppKit Anti-Patterns

### Rule: `no_nslog`

**Purpose:** Replace `NSLog` with structured logging via `os.log`.

```yaml
no_nslog:
  name: "Use os_log or Logger"
  regex: 'NSLog\('
  message: "Use Logger from os.log instead of NSLog for structured logging"
  severity: error
  match_kinds:
    - identifier
```

**Why:** `NSLog` writes to the Apple System Log but lacks structured metadata, log levels, and signposts. `Logger` from `os.log` integrates with Instruments, supports `.debug`/`.info`/`.error`/`.fault` levels, and respects privacy redaction.

**Migration:**

```swift
// Before:
NSLog("User tapped button: %@", button.title)

// After:
import os.log
let logger = Logger(subsystem: "com.myapp", category: "UI")
logger.info("User tapped button: \(button.title, privacy: .public)")
```

### Rule: `unwrapped_iboutlet`

**Purpose:** Prevent force-unwrapped `IBOutlet` references that crash on nil.

```yaml
unwrapped_iboutlet:
  name: "Unwrapped IBOutlet"
  regex: '@IBOutlet\s+(?:weak\s+)?var\s+\w+!\s*:\s*(?:NS|UI)'
  message: "IBOutlets should use Optional (?) or ImplicitlyUnwrappedOptional only when necessary"
  severity: error
```

**Why:** When a XIB/storyboard fails to connect an outlet, accessing a force-unwrapped `IBOutlet` crashes. Using `Optional` with `guard-let` in `viewDidLoad` provides a clear failure point.

### Rule: `nsobject_subclass`

**Purpose:** Discourage unnecessary `NSObject` subclassing in pure Swift code.

```yaml
nsobject_subclass:
  name: "NSObject Subclass"
  regex: 'class\s+\w+\s*:\s*NSObject\b'
  message: "Avoid subclassing NSObject unless required for Objective-C interop, KVO, or Core Data"
  severity: warning
  match_kinds:
    - typeidentifier
```

**Why:** `NSObject` subclassing adds Objective-C runtime overhead, enables KVO by default (which can cause crashes if observers aren't removed), and implies reference semantics. Pure Swift classes are lighter and safer. Exceptions: Core Data `NSManagedObject` subclasses, classes exposed to Objective-C via `@objc`, classes using KVO or target-action.

### Rule: `weak_self_in_closure`

**Purpose:** Enforce `[weak self]` capture lists in AppKit closures to prevent retain cycles.

```yaml
weak_self_in_closure:
  name: "Weak Self in Closure"
  regex: '\{[^}]*\bself\.[a-z]'
  message: "Use [weak self] in AppKit closures to avoid retain cycles with NSWindowController/NSDocument"
  severity: warning
  excluded:
    - "**/Tests/**"
```

**Why:** AppKit patterns like `NSWindowController` closures, `NSDocument` callbacks, and `DispatchQueue` blocks inside view controllers commonly create retain cycles. The closure captures `self`, which owns the closure (directly or indirectly), preventing deallocation.

### Rule: `empty_catch`

**Purpose:** Detect empty catch blocks that silently swallow errors.

```yaml
empty_catch:
  name: "Empty Catch Block"
  regex: 'catch\s*\{[\s\n]*\}'
  message: "Empty catch blocks silently swallow errors. Log or handle the error."
  severity: error
```

**Why:** Empty catch blocks hide bugs. At minimum, log the error. In macOS apps, unhandled errors in document reading, file coordination, or XPC calls can cause data loss.

## SPM Build Tool Plugin Configuration

### Package.swift

```swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "MyMacOSApp",
    platforms: [.macOS(.v14)],
    products: [
        .executable(name: "MyMacOSApp", targets: ["MyMacOSApp"]),
    ],
    dependencies: [
        .package(url: "https://github.com/realm/SwiftLint", from: "0.57.0"),
    ],
    targets: [
        .executableTarget(
            name: "MyMacOSApp",
            plugins: [
                .plugin(name: "SwiftLintBuildToolPlugin", package: "SwiftLint")
            ]
        ),
        .testTarget(
            name: "MyMacOSAppTests",
            dependencies: ["MyMacOSApp"]
            // No plugin on test target — tests have different lint rules
        ),
    ]
)
```

### Xcode Project Integration

For Xcode projects (not SPM packages), add SwiftLint as a package dependency:

1. **File → Add Package Dependencies** → `https://github.com/realm/SwiftLint`
2. Select the **SwiftLintPlugin** product
3. In target settings → **Build Phases** → **Run Build Tool Plug-ins** → add `SwiftLintBuildToolPlugin`

### Manual Plugin Commands

```bash
# Lint without building
swift package swiftlint-lint

# Auto-fix violations
swift package swiftlint-fix

# List active rules
swift package swiftlint-rules

# Show SwiftLint version
swift package swiftlint-version
```

### Plugin Limitations

- The plugin uses the `.swiftlint.yml` at the package root. Custom config paths are not supported via the plugin.
- The plugin cannot pass custom arguments (like `--strict`). Use the config-level `strict: true` instead.
- The plugin runs per-target. For cross-target analysis, use a build phase or CLI invocation.

## CI Integration with GitHub Actions

### Basic Lint Job

```yaml
# .github/workflows/swiftlint.yml
name: SwiftLint

on:
  pull_request:
    paths:
      - '**.swift'
  push:
    branches: [main, develop]

jobs:
  lint:
    name: SwiftLint
    runs-on: macos-15
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install SwiftLint
        run: brew install swiftlint

      - name: Lint
        run: |
          swiftlint lint \
            --strict \
            --config .swiftlint.yml \
            --reporter github-actions-logging \
            --cache-path .swiftlint_cache

      - name: Upload cache
        if: github.ref == 'refs/heads/main'
        uses: actions/cache@v4
        with:
          path: .swiftlint_cache
          key: swiftlint-${{ hashFiles('.swiftlint.yml') }}-${{ runner.os }}
```

### Full Pipeline with Analyze

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
  push:
    branches: [main]

jobs:
  swiftlint:
    name: Lint
    runs-on: macos-15
    steps:
      - uses: actions/checkout@v4

      - name: Install SwiftLint
        run: brew install swiftlint

      - name: Run SwiftLint
        run: swiftlint lint --strict --reporter github-actions-logging

  analyze:
    name: Analyze
    runs-on: macos-15
    needs: swiftlint
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.2.app

      - name: Install SwiftLint
        run: brew install swiftlint

      - name: Clean DerivedData
        run: rm -rf /tmp/dd

      - name: Build for compiler log
        run: |
          xcodebuild clean build \
            -scheme MyMacOSApp \
            -destination 'platform=macOS' \
            -derivedDataPath /tmp/dd \
            -skipPackagePluginValidation \
            | tee /tmp/build.log

      - name: SwiftLint Analyze
        run: |
          swiftlint analyze \
            --compiler-log-path /tmp/build.log \
            --strict \
            --reporter json > /tmp/analysis.json

      - name: Upload analysis
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: swiftlint-analysis
          path: /tmp/analysis.json
```

### Using action-swiftlint

```yaml
  lint:
    runs-on: macos-15
    steps:
      - uses: actions/checkout@v4

      - name: SwiftLint
        uses: step-security/action-swiftlint@v3
        with:
          args: --strict
        env:
          DIFF_BASE: ${{ github.event.pull_request.base.ref }}
          WORKING_DIRECTORY: .
```

### Failing the Build

The build fails when:
- `--strict` is passed and any violation (warning or error) exists
- Any `error`-severity violation exists (without `--strict`)
- SwiftLint itself crashes (missing config, invalid YAML)

**Recommended:** Always use `--strict` in CI. Warnings in CI become warnings in PRs, which are easy to ignore. `--strict` forces action.

## Baseline Management Workflow

### Initial Setup (Existing Codebase)

```bash
# 1. Generate baseline of all current violations
swiftlint --write-baseline swiftlint_baseline.json

# 2. Commit the baseline
git add swiftlint_baseline.json
git commit -m "Add SwiftLint baseline"

# 3. CI uses baseline to only catch NEW violations
swiftlint lint --baseline swiftlint_baseline.json --strict
```

### Cleanup Sprint Workflow

```bash
# 1. Fix a batch of violations (manually or with --fix)
swiftlint --fix

# 2. Regenerate baseline (now smaller)
swiftlint --write-baseline swiftlint_baseline.json

# 3. Verify: lint should pass with the new baseline
swiftlint lint --baseline swiftlint_baseline.json --strict

# 4. Commit fixes + updated baseline
git add -A
git commit -m "Fix SwiftLint violations (reduce baseline by N)"
```

### Removing the Baseline

```bash
# When violations reach zero:
rm swiftlint_baseline.json
git rm swiftlint_baseline.json
git commit -m "Remove SwiftLint baseline (zero violations)"

# Update CI to remove --baseline flag
# CI now runs: swiftlint lint --strict
```

### Baseline File Format

```json
{
  "MyMacOSApp/Document.swift": {
    "line_length": 3,
    "function_body_length": 1
  },
  "MyMacOSApp/ViewController.swift": {
    "unused_import": 2
  }
}
```

The baseline maps file paths to rule IDs to violation counts. It does not track line numbers, so it survives code movement. If a file's violations decrease, the baseline still passes (it's a maximum, not an exact match). If violations increase, the lint fails.

### Baseline Gotchas

- **Same config required:** Generate and consume the baseline with identical `.swiftlint.yml`. Different configs produce mismatched results.
- **No strict + baseline:** `--strict` with `--baseline` is contradictory. The baseline suppresses violations that `--strict` would elevate. Use `--baseline` without `--strict` during transition, then remove baseline.
- **Regenerate after rule changes:** Adding/removing rules invalidates the baseline. Regenerate after any config change.
