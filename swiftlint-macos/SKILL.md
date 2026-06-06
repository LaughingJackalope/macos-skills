---
name: swiftlint-macos
description: "SwiftLint for macOS projects — .swiftlint.yml configuration, SPM/Xcode build tool plugin integration, macOS-relevant rules (AppKit naming, NSObject patterns), custom rules for macOS anti-patterns, CI integration (xcodebuild + SwiftLint), --strict mode, baseline files, swiftlint analyze. Use when setting up or configuring SwiftLint for a macOS project."
sources: https://github.com/realm/SwiftLint, https://github.com/realm/SwiftLint/blob/main/Documentation/Configuration.md, https://github.com/realm/SwiftLint/releases/tag/0.63.3, https://github.com/realm/SwiftLint/issues/4015, https://github.com/realm/SwiftLint/issues/4786, https://blog.stackademic.com/writing-custom-swiftlint-rule-with-swiftsyntax-9cc93ae6dfe2, https://swiftpackageindex.com/GayleDunham/SwiftLintPlugin
researched: 2026-06-06
platform: macOS 26+
>
> **Updated for 2026:** SwiftLint 0.63.3 (May 2026) adds high-speed extraction, simplified binary artifact paths, and Bazel integration improvements. The `SwiftLintBuildToolPlugin` now respects `BUILD_WORKSPACE_DIRECTORY`. A separate community plugin, `SwiftLintPlugin` (GayleDunham/SwiftLintPlugin), is also maintained with releases up to v1.3 in 2026.
---

# SwiftLint for macOS

SwiftLint configuration and integration patterns for macOS 26+ projects. Covers `.swiftlint.yml` setup, SPM build tool plugin and Xcode build phase integration, macOS-specific custom rules (AppKit naming conventions, NSObject anti-patterns), CI pipelines with `xcodebuild`, `--strict` mode adoption, baseline management, and `swiftlint analyze` for compiler-log-based static analysis.

See [references/swiftlint-patterns.md](references/swiftlint-patterns.md) for a complete `.swiftlint.yml` example, detailed custom rule definitions for AppKit anti-patterns, SPM plugin configuration, GitHub Actions CI workflow, and baseline management commands.

## Contents

- [.swiftlint.yml Configuration](#swiftlintyml-configuration)
- [Build Tool Plugin](#build-tool-plugin)
- [macOS-Relevant Rules](#macos-relevant-rules)
- [Custom Rules](#custom-rules)
- [CI Integration](#ci-integration)
- [--strict Mode](#--strict-mode)
- [Baseline Files](#baseline-files)
- [swiftlint analyze](#swiftlint-analyze)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## .swiftlint.yml Configuration

SwiftLint is configured via `.swiftlint.yml` at the project root. Paths are case-sensitive. Wildcards (`*`, `**`) are supported in `included` and `excluded`. Excluded overrides included — a path in both lists is excluded.

**Core structure:**

```yaml
included:
  - Sources
  - App

 excluded:
  - Sources/Generated
  - "**/Package.swift"
  - "**/.build"

disabled_rules:
  - trailing_comma
  - opening_brace

opt_in_rules:
  - array_init
  - discarded_observation
  - discouraged_optional_collection

analyzer_rules:
  - unused_declaration
  - unused_import

# Rule-specific configuration
line_length:
  warning: 140
  error: 200

type_body_length:
  warning: 300
  error: 500

function_body_length:
  warning: 60
  error: 120

cyclomatic_complexity:
  warning: 10
  error: 20

nesting:
  type_level:
    warning: 2
    error: 4

large_tuple:
  warning: 3

custom_rules:
  no_nslocalizedstring:
    name: "Use LocalizedStringResource"
    regex: 'NSLocalizedString\('
    message: "Use LocalizedStringResource or #localized instead of NSLocalizedString"
    severity: warning
  # ... (see references/swiftlint-patterns.md for full set)
```

**`disabled_rules`:** Removes built-in rules from the default set. Use sparingly — prefer tightening rule parameters over disabling them entirely.

**`opt_in_rules`:** Enables rules not in the default set. These are often the most valuable rules for quality. Recommended for macOS: `discouraged_optional_collection`, `discouraged_none_name`, `discouraged_object_literal`, `explicit_self`, `unused_declaration` (analyzer), `unused_import` (analyzer), `capture_variable`, `strict_fileprivate`, `unused_capture_list`, `typesafe_array_init`, `legacy_objc_type`.

**`only_rules`:** When set, only the listed rules run. Mutually exclusive with `disabled_rules` and `opt_in_rules`. Useful for incremental adoption — start with a small allowlist and expand.

**`analyzer_rules`:** Subset of rules powered by the Swift AST via SourceKitten. Only available in `swiftlint analyze` mode. All analyzer rules are opt-in. Key ones: `unused_declaration`, `unused_import`, `unused_capture_list`.

**Severity levels:** Each rule can set `warning` and/or `error` thresholds. A `warning` always implies an `error` at the same threshold unless `error` is explicitly set higher. When only `warning` is set, the violation is a warning. When only `error` is set, the violation is an error. When both are set, violations below `warning` pass, between `warning` and `error` are warnings, and above `error` are errors.

**Nested configurations:** You can place `.swiftlint.yml` files in subdirectories. SwiftLint walks from the file's directory up to root, finding the first config. Nested configs merge with the parent: `merged.included = (parent.included - child.excluded) + child.included`. If you pass `--config`, nesting is disabled — the specified file is used exclusively.

**Cache:** Set `cache_path` to speed up repeated lints in CI:

```yaml
cache_path: .swiftlint_cache
```

Cache key is config + SwiftLint version. Invalidate when either changes.

## Build Tool Plugin

### SPM Build Tool Plugin

Add SwiftLint as a package dependency and apply the build tool plugin to targets:

```swift
// Package.swift
// swift-tools-version: 5.9
import PackageDescription

let package = Package(
    name: "MyMacOSApp",
    platforms: [.macOS(.v14)],
    dependencies: [
        .package(url: "https://github.com/realm/SwiftLint", from: "0.57.0"),
    ],
    targets: [
        .target(
            name: "MyMacOSApp",
            plugins: [
                .plugin(name: "SwiftLintBuildToolPlugin", package: "SwiftLint")
            ]
        ),
    ]
)
```

The plugin runs SwiftLint before each target build, using the `.swiftlint.yml` in the package root. Violations are reported as build warnings/errors in Xcode.

**Plugin commands (manual invocation):**

```bash
swift package swiftlint-lint    # Lint without building
swift package swiftlint-fix     # Auto-fix issues
swift package swiftlint-rules   # List active rules
swift package swiftlint-version # Show SwiftLint version
```

### Xcode Build Phase (Alternative)

When the plugin doesn't fit (multi-target workspaces, custom config paths), add a Run Script build phase **before Compile Sources**:

```bash
if [[ "$(uname -m)" == arm64 ]]; then
    export PATH="/opt/homebin:$PATH"
else
    export PATH="/usr/local/bin:$PATH"
fi

if which swiftlint > /dev/null; then
    swiftlint lint --config "${SRCROOT}/.swiftlint.yml"
else
    echo "SwiftLint not installed. Install via Homebrew: brew install swiftlint"
    # Do NOT fail the build here unless strict mode is desired
fi
```

**Critical:** Add `--config` explicitly when using a build phase, because Xcode may invoke the script from a different working directory than the project root.

### Plugin vs Build Phase Tradeoffs

| Factor | SPM Plugin | Xcode Build Phase |
|--------|-----------|-------------------|
| Requires SwiftLint install | No (bundled) | Yes (Homebrew/Mint) |
| Custom config path | Root only | Any path |
| --strict mode | Via Xcode build settings | Via --strict flag |
| Works in Xcode Cloud | With -skipPackagePluginValidation | Yes |
| Works for all targets | Per-target opt-in | Single invocation |
| Fix in Xcode | Via command | Via --fix flag |

**Prefer the SPM plugin** when all targets are SPM targets and you can use the root config. **Use a build phase** for mixed SPM/Xcode projects or when you need multiple config files.
>
> ### Bazel Integration (2026)
>
> SwiftLint 0.63.3 supports Bazel via `bazel_dep` with bzlmod or `http_archive`:
>
> ```python
> # MODULE.bazel (bzlmod)
> bazel_dep(name = "swiftlint", version = "0.63.3", repo_name = "SwiftLint")
>
> # Or in WORKSPACE (without bzlmod):
> http_archive(
>     name = "SwiftLint",
>     sha256 = "<sha256>",
>     url = "https://github.com/realm/SwiftLint/releases/download/0.63.3/bazel.tar.gz",
> )
> ```
>
> Run from the workspace root: `bazel run @SwiftLint//:swiftlint -- lint --config .swiftlint.yml`

## macOS-Relevant Rules

macOS projects using Objective-C interop, AppKit patterns, or import into C/enforce naming conventions that differ from pure Swift. Configure these rules for macOS:

**`attributes`:** Require `@objc` and other attributes on separate lines. macOS code with heavy Objective-C interop has many attributes:

```yaml
attributes:
  always_on_same_line: []
  always_on_own_line: ["@objc", "@IBOutlet", "@IBSegueAction", "@MainActor"]
```

**`identifier_name`:** Relax for Objective-C style selectors and common AppKit patterns:

```yaml
identifier_name:
  min_length:
    warning: 2
    error: 1
  excluded:
    - id
    - x
    - y
    - vc
    - wc
  allowed_symbols: ["_"]
```

**`type_name`:** Allow `NS` prefixes only on types that wrap AppKit/UIKit classes (a legitimate macOS pattern):

```yaml
type_name:
  min_length: 3
  max_length: 50
  excluded: []
```

**`legacy_objc_type`:** Disable for macOS projects that use `NSObject` subclasses with Core Data or Objective-C interop. Enable for pure Swift targets:

```yaml
# In a macOS project's root config:
legacy_objc_type: disabled

# In a pure Swift target's child config:
legacy_objc_type:
  severity: warning
```

**`explicit_self`:** For macOS projects with many closure-based AppKit callbacks, consider setting to `warning` first:

```yaml
explicit_self:
  severity: warning
```

**`prohibited_interface_builder`:** Enable for projects that forbid `.xib`/`.storyboard` files entirely (code-only AppKit UIs):

```yaml
prohibited_interface_builder:
  severity: error
```

## Custom Rules

Define regex-based custom rules in `custom_rules` section of `.swiftlint.yml`. Each rule has an identifier, regex pattern, optional `match_kinds`, `message`, and `severity`.

**AppKit naming convention — enforce `NS` prefix only for wrapping AppKit types:**

```yaml
custom_rules:
  ns_prefix_on_non_wrapper:
    name: "NS Prefix Misuse"
    regex: '(?:class|struct|enum)\s+NS(?!View|Window|Controller|Document|Application|Responder|Array|Dictionary|Set|String|Data|Number|Date|URL|Request|Response|Error|Notification|Timer|Operation|Lock|Thread|Port|Socket|Pipe|Handle|Image|Color|Font|BezierPath|Gradient|Shadow|Appearance|Menu|MenuItem|TableView|TableViewCell|CollectionView|CollectionViewItem|ScrollView|SplitView|StackView|VisualEffectView|TouchBar|GestureRecognizer|PanGestureRecognizer|TapGestureRecognizer|PressGestureRecognizer|RotationGestureRecognizer|MagnificationGestureRecognizer|AccessibilityElement|AccessibilityOrderedSet|AccessibilityCustomAction|AccessibilityCustomRotor|AccessibilityCustomRotorItemResult|AccessibilityCustomRotorSearchParameters|Action|Synthesizer|SpeechRecognizer|Audio|Video|Capture|Device|Player|Output|Input|Session|Sample|Buffer|Format|Description|Queue|Time|Value|Range|Index|Path|Predicate|Expression|Sort|Descriptor)\w+'
    message: "Only use NS prefix when wrapping or extending an AppKit type"
    severity: warning
    match_kinds:
      - typeidentifier
```
>
> ### SwiftSyntax-Based Custom Rules
>
> For complex rules that regex cannot express, SwiftLint supports rules written in Swift using SwiftSyntax for fast, accurate AST traversal. These require building a custom SwiftLint binary or using Bazel:
>
> ```swift
> import SwiftLintCore
> import SwiftSyntax
>
> @SwiftSyntaxRule
> struct NoForceUnwrapInAppKitRule: OptInRule {
>     var configuration = SeverityConfiguration(.error)
>
>     static let description = RuleDescription(
>         identifier: "no_force_unwrap_appkit",
>         name: "No Force Unwrap in AppKit",
>         description: "Avoid force unwraps in AppKit lifecycle methods",
>         kind: .lint
>     )
>
>     func makeVisitor(file: SwiftLintFile) -> SyntaxVisitor {
>         MethodCallVisitor(viewMode: .sourceAccurate)
>     }
> }
> ```
>
> **Note:** Swift-based rules require building SwiftLint from source or using Bazel. They cannot be used with the standard Homebrew binary. For most projects, regex-based custom rules with `match_kinds` are simpler and faster to iterate on.
>
> A declarative DSL for SwiftSyntax rules is available via PR #4023 in the SwiftLint repo, providing a SwiftUI-like matcher system for simpler rule definitions.

**Disallow `NSLog` — enforce `os_log` or `Logger`:**

```yaml
custom_rules:
  no_nslog:
    name: "Use os_log or Logger"
    regex: 'NSLog\('
    message: "Use Logger from os.log instead of NSLog for structured logging"
    severity: error
```

**Enforce completion handler naming convention:**

```yaml
custom_rules:
  completion_handler_name:
    name: "Completion Handler Naming"
    regex: 'completion(?:Handler)?:\s*\((?:[^)]*)\)\s*->\s*Void'
    excluded: ['subscript']
    message: "Completion handlers should be named 'completion:' or 'onCompletion:'"
    severity: warning
    match_kinds:
      - argument
```

**Detect unwrapped `IBOutlet` references:**

```yaml
custom_rules:
  unwrapped_iboutlet:
    name: "Unwrapped IBOutlet"
    regex: '@IBOutlet\s+(?:weak\s+)?var\s+\w+!\s*:\s*(?:NS|UI)'
    message: "IBOutlets should use Optional (?) or ImplicitlyUnwrappedOptional only when necessary. Prefer Optional with guard-let in viewDidLoad."
    severity: error
```

**Match kinds:** Control what syntax kinds the regex matches against, reducing false positives. Available kinds: `identifier`, `typeidentifier`, `keyword`, `string`, `comment`, `argument`, `attribute`, `number`, `directive`. For rules targeting type declarations, use `match_kinds: [typeidentifier]`. For rules targeting function parameters, use `match_kinds: [argument]`.

**Capture groups:** Use `capture_group` to highlight a specific part of the match:

```yaml
custom_rules:
  no_fatal_error:
    name: "No fatalError"
    regex: 'fatalError\(([^)]*)\)'
    capture_group: 1
    message: "fatalError crashes in production. Use graceful handling."
    severity: error
```

## CI Integration

### GitHub Actions

Use `step-security/action-swiftlint` or run SwiftLint directly:

```yaml
name: SwiftLint

on:
  pull_request:
    paths:
      - '**.swift'

jobs:
  swiftlint:
    runs-on: macos-15
    steps:
      - uses: actions/checkout@v4

      - name: Install SwiftLint
        run: brew install swiftlint

      - name: Lint (Strict)
        run: swiftlint lint --strict --config .swiftlint.yml --reporter github-actions-logging
```

### With xcodebuild (full pipeline)

When running `swiftlint analyze` with compiler logs:

```yaml
  analyze:
    runs-on: macos-15
    needs: swiftlint
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.2.app

      - name: Clean Build Log
        run: |
          rm -rf /tmp/DerivedData
          mkdir -p /tmp/logs

      - name: Build (produce compiler log)
        run: |
          xcodebuild clean build \
            -scheme MyMacOSApp \
            -destination 'platform=macOS' \
            -derivedDataPath /tmp/DerivedData \
            | tee /tmp/logs/xcodebuild.log \
            -skipPackagePluginValidation

      - name: Analyze with SwiftLint
        run: |
          swiftlint analyze \
            --compiler-log-path /tmp/logs/xcodebuild.log \
            --strict \
            --config .swiftlint.yml \
            --reporter json > /tmp/swiftlint-report.json

      - name: Upload report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          path: /tmp/swiftlint-report.json
```

**`-skipPackagePluginValidation`:** Required in CI to avoid Xcode prompting for plugin trust. Pass this on the `xcodebuild` command line.

**Failing the build:** Use `--strict` to treat warnings as errors (exit code ≠ 0). Without `--strict`, only `error`-severity violations fail the build. For CI, always use `--strict` to enforce zero warnings.

**Reporters for CI:** Use `--reporter github-actions-logging` for inline PR annotations, `--reporter json` for machine-readable output, or `--reporter junit` for test-result ingestion.

## --strict Mode

`--strict` elevates all warnings to errors. SwiftLint exits with a non-zero code if any violation exists (warning or error).

**Incremental adoption strategy:**

1. **Phase 1 — Baseline:** Generate a baseline of current violations: `swiftlint --write-baseline swiftlint_baseline.json`
2. **Phase 2 — Config-level strictness:** Set `strict: true` in `.swiftlint.yml` instead of passing `--strict` on CLI — this makes all runs strict by default:
   ```yaml
   strict: true
   ```
3. **Phase 3 — Fix violations:** Run `swiftlint --fix` after each rule change. Update baseline after fixes.
4. **Phase 4 — Enforce in CI:** Add `--strict` to CI step. Remove baseline once zero violations.

**Baseline + strict conflict:** You cannot use `--strict` and `--baseline` together effectively — the baseline suppresses violations that `--strict` would elevate. Use baselines only during transition periods, not as a permanent strict-mode mechanism.

## Baseline Files

Baselines capture existing violations so only new violations fail the lint. Useful for adopting SwiftLint in an existing codebase.

**Generate a baseline:**

```bash
swiftlint --write-baseline swiftlint_baseline.json
```

**Use the baseline in subsequent runs:**

```bash
swiftlint lint --baseline swiftlint_baseline.json
```

**In CI:**

```bash
# Only fail on violations NOT in the baseline
swiftlint lint --baseline swiftlint_baseline.json --strict
```

**Rules:**
- Generate the baseline with the **same `.swiftlint.yml`** used for linting. Different configs produce unreliable baselines.
- The baseline file maps file paths + rule IDs to violation counts. It does not suppress by line number, so it survives code movement.
- Regenerate the baseline after fixing a batch of violations. Commit the updated baseline.
- **Remove the baseline** once violations reach zero. Don't commit a permanent baseline — it masks drift.

**Configuration reference:**

```yaml
# Not a separate key — baseline is CLI-only
# Use --write-baseline and --baseline flags
```

## swiftlint analyze

`swiftlint analyze` performs type-aware static analysis using the Swift compiler's AST via SourceKitten. It requires a **compiler build log** from a clean build.

**Generate the build log:**

```bash
# Clean first — incremental logs are unreliable
rm -rf ~/Library/Developer/Xcode/DerivedData/MyApp-*

xcodebuild clean build \
  -scheme MyMacOSApp \
  -destination 'platform=macOS' \
  -derivedDataPath /tmp/dd \
  OTHER_SWIFT_FLAGS='-Xfrontend -emit-parseable-module-interface' \
  | tee /tmp/build.log
```

**Run analysis:**

```bash
swiftlint analyze --compiler-log-path /tmp/build.log --strict
```

**What it catches:** Analyzer rules detect issues regular rules cannot — unused imports, unused declarations, unused capture variables, redundant type annotations. These require type information only available from the compiler.

**Key analyzer rules:**
- `unused_import`: Imported modules not referenced in the file
- `unused_declaration`: Types, functions, properties never used
- `unused_capture_list`: `[weak self]` when `self` isn't captured
- `redundant_type_annotation`: Types that can be inferred
- `explicit_self`: Explicit `self.` where inference suffices
- `capture_variable`: Variables captured before mutation

**Performance:** Analysis is significantly slower than regular lint (uses SourceKit). Run it as a separate CI job, not in the SPM build plugin. Expect minutes for large projects.

**Compiler log compatibility:** SwiftLint's SourceKit integration can break with new Xcode versions. If `analyze` returns 0 violations unexpectedly, check `xcodebuild.log` for SourceKit errors. Use the latest SwiftLint release for Xcode compatibility.

## Constraints & Gotchas

- **SPM plugin vs build phase timing:** The SPM build tool plugin runs per-target, before each target compiles. The Xcode build phase runs once per build phase invocation. For multi-target projects, the plugin lints each target independently (good), but the build phase can lint everything in one pass (better for cross-target duplicate detection).

- **`--config` disables nesting:** If you pass `--config /path/.swiftlint.yml`, nested configurations are ignored. The specified file is the only config. Prefer nesting over `--config` when possible.

- **Rule conflicts:** `only_rules` + `disabled_rules`/`opt_in_rules` is a runtime error. `analyzer_rules` + regular `opt_in_rules` is fine — analyzer rules are a separate category.

- **Cache invalidation:** SwiftLint's cache key includes the config file. Changing `.swiftlint.yml` auto-invalidates. Changing SwiftLint version does not auto-invalidate — manually clear `.swiftlint_cache` on version upgrades.

- **Xcode's Run Script build phase working directory:** Xcode sets `SRCROOT` as the working directory, but the build phase's `pwd` may differ. Always use `"${SRCROOT}/.swiftlint.yml"` or use `--config` with absolute paths.

- **`--fix` in build phases:** Never auto-fix (`--fix`) in a build phase. It modifies source files during the build, invalidating Xcode's dependency graph. Run `--fix` manually or in a pre-commit hook.

- **Pre-commit hook timing:** When using a pre-commit hook with `--fix`, the fixed changes are staged but not included in the current commit. Re-stage the fixed files.

- **`write_baseline` requires the same config:** If you generate a baseline with one `.swiftlint.yml` and lint with another, violations may not match. Always use identical configs for baseline write and read.

## Common Mistakes
1. **Xcode 14+ script phase "will run every build" warning.** If using a Run Script build phase, Xcode 14 warns when inputs/outputs are unspecified. Either uncheck "Based on dependency analysis" in the build phase config, or add `$(DERIVED_FILE_DIR)/swiftlint_output` as an output file reference.
2. **`--strict` not supported in Xcode Build Plugin (as of 0.63.3).** The `SwiftLintBuildToolPlugin` does not pass `--strict` to the underlying `swiftlint` invocation. To enable strict mode from the plugin, set `strict: true` in `.swiftlint.yml`. The `--strict` CLI flag only works with the standalone `swiftlint lint` command or the Xcode Run Script phase.
3. **Forgetting `--strict` in CI:** Without `--strict`, warnings are reported but the build passes. CI should use `--strict` to enforce zero violations.

4. **Using `disabled_rules` instead of tuning:** Disabling `line_length` instead of setting a reasonable `warning: 140, error: 200` removes a useful check. Always tune before disabling.

5. **Stale baseline files:** Committing a baseline and never updating it indefinitely hides new violations. Regenerate after each cleanup sprint.

6. **Custom rules without `match_kinds`:** A regex like `NSLog\(` without `match_kinds` matches in comments, strings, and code, creating false positives. Add `match_kinds: [identifier]` to constrain.

7. **Running `swiftlint analyze` without a clean build log:** Incremental build logs miss files, producing incomplete analysis. Always `clean build` before capturing the log.

8. **Build phase before/after Compile Sources ordering:** Place SwiftLint's build phase before Compile Sources so violations prevent compilation. Placement after means the build may succeed before lint runs.

9. **Not handling `requiresApproval` in SPM plugin CI:** Xcode Cloud and CI environments require `-skipPackagePluginValidation`. Without it, the build hangs waiting for interactive trust.

## Review Checklist

- [ ] `.swiftlint.yml` exists at project root with `included`/`excluded` paths
- [ ] `opt_in_rules` include macOS-relevant rules (`discouraged_optional_collection`, `explicit_self`, `unused_import`)
- [ ] Custom rules use `match_kinds` to avoid false positives in comments/strings
- [ ] `NSLog` is replaced with `Logger` (custom rule enforces this)
- [ ] `IBOutlets` are not force-unwrapped without justification
- [ ] SPM plugin or Xcode build phase is configured (not both, to avoid double-linting)
- [ ] CI runs `swiftlint lint --strict` (build fails on any violation)
- [ ] CI uses `-skipPackagePluginValidation` for `xcodebuild`
- [ ] Baseline file exists only during active cleanup; removed when violations are zero
- [ ] `swiftlint analyze` runs in a separate CI step with a clean build log
- [ ] `cache_path` configured for faster CI runs
- [ ] `strict: true` in config (or `--strict` on CLI) to fail on warnings

## References

- [references/swiftlint-patterns.md](references/swiftlint-patterns.md) — Complete `.swiftlint.yml`, custom rule definitions, SPM config, CI workflow, baseline commands
- [SwiftLint GitHub](https://github.com/realm/SwiftLint)
- [SwiftLint Configuration Documentation](https://github.com/realm/SwiftLint/blob/main/Documentation/Configuration.md)
- [SwiftLint Rules Reference](https://realm.github.io/SwiftLint/rule-directory.html)
- [step-security/action-swiftlint](https://github.com/step-security/action-swiftlint) — GitHub Action for SwiftLint CI
- [SwiftLint 0.63.3 Release — High-Speed Extraction](https://github.com/realm/SwiftLint/releases/tag/0.63.3)
- [SwiftLint Bazel Integration](https://github.com/realm/SwiftLint/releases) — Bazel setup instructions and bzlmod config
- [Xcode 14 build phase warning — SwiftLint Issue #4015](https://github.com/realm/SwiftLint/issues/4015)
- [--strict not supported in Xcode Build Plugin — Issue #4786](https://github.com/realm/SwiftLint/issues/4786)
- [Writing Custom SwiftLint Rules with SwiftSyntax](https://blog.stackademic.com/writing-custom-swiftlint-rule-with-swiftsyntax-9cc93ae6dfe2)
- [SwiftLintPlugin (Community)](https://swiftpackageindex.com/GayleDunham/SwiftLintPlugin) — Community-maintained SPM plugin
- [Declarative DSL for SwiftSyntax Rules — PR #4023](https://github.com/realm/SwiftLint/pull/4023)
