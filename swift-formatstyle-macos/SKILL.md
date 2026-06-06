---
name: swift-formatstyle-macos
description: "FormatStyle on macOS — Date.FormatStyle, Number.FormatStyle, Measurement.FormatStyle, List.FormatStyle, Duration.TimeFormatStyle, custom FormatStyle conformances, formatting for menu items/status bar/inspector/large-number display in Table columns. Use when formatting values for display in macOS UI."
sources: https://developer.apple.com/documentation/foundation/formatstyle, https://developer.apple.com/documentation/foundation/date/formatstyle
researched: 2026-06-06
platform: macOS 26+
---

# FormatStyle on macOS

Foundation's `FormatStyle` protocol and its conformances for Date, Number, Measurement, List, and Duration — the unified formatting API that replaces `DateFormatter`, `NumberFormatter`, `MeasurementFormatter`, and `ListFormatter` with a type-safe, chainable, composable interface. macOS 26+ with Swift 6.3. Covers all built-in format styles, custom `FormatStyle` conformance, locale overrides, and macOS-specific display patterns (compact menu labels, status bar, inspector detail, aligned Table columns). For SwiftUI text formatting, use this skill alongside `swiftui-macos-patterns`. For locale and localization, see `macos-text-and-input`.

See [references/formatstyle-patterns.md](references/formatstyle-patterns.md) for deep-dive patterns: custom FormatStyle for file size display, date formatting patterns for macOS UI contexts, number formatting for Table columns, duration formatting for timers, and locale-aware measurement display.

## Contents

- [FormatStyle Protocol](#formatstyle-protocol)
- [Date Formatting](#date-formatting)
- [Number Formatting](#number-formatting)
- [Measurement Formatting](#measurement-formatting)
- [List Formatting](#list-formatting)
- [Duration Formatting](#duration-formatting)
- [Custom FormatStyle](#custom-formatstyle)
- [macOS Display Patterns](#macos-display-patterns)
- [Locale](#locale)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## FormatStyle Protocol

`FormatStyle` is a protocol that standardizes all formatting in Foundation. Every conformance has three components: the style type, a `format(_:)` method, and a `locale` property.

**Protocol definition:**

```swift
protocol FormatStyle {
    typealias FormatInput
    typealias FormatOutput
    func format(_ value: FormatInput) -> FormatOutput
    var locale: Locale { get set }
}
```

The key design: `FormatInput` and `FormatOutput` are associated types, so each style declares what it accepts and what it produces. `Date.FormatStyle` takes `Date` and produces `String`. `Number.FormatStyle` produces a `String`. Custom styles can produce anything — attributed strings, rich text, even view descriptions.

**Chaining with dot syntax.():** Every format style can be further configured by chaining calls. These return a _new_ value with the modifier applied; the original is unchanged (value semantics):

```swift
let dateStyle = Date.FormatStyle()
    .year()
    .month(.abbreviated)
    .day()

let formatter = Date.FormatStyle()
let compact = formatter.month(.abbreviated).day() // `formatter` unchanged
```

**.formatted() shorthand:** Any type whose `FormatStyle` conforms to `FormatStyle` where `FormatOutput == String` can use the `.formatted()` instance method. This is the idiomatic entry point:

```swift
let now = Date()
now.formatted() // default style: "6/6/2026, 12:00 PM"

now.formatted(date: .abbreviated, time: .shortened)
// "Jun 6, 2026, 12:00 PM"

now.formatted(.iso8601)
// "2026-06-06T12:00:00Z"

now.formatted(.relative(presentation: .named))
// "today" (if today)
```

**`formatted(_:)` vs creating a style instance:** For one-shot formatting, use `.formatted(...)`. For reusable formatting (e.g., in a Table column or repeatedly in a view body), create a style instance once and reuse it:

```swift
// Good: reusable style
extension Date.FormatStyle {
    static let shortDate = Date.FormatStyle(date: .abbreviated, time: .omitted)
}

// In a loop — no repeated allocation:
ForEach(dates, id: \.self) { date in
    Text(date.formatted(.shortDate))
}
```

## Date Formatting

`Date.FormatStyle` replaces `DateFormatter` with a type-safe, locale-aware API.

**Scoped static methods — .dateTime, .iso8601, .relative, .formatted(date:time:):**

```swift
import Foundation

let now = Date()

// Default: full date and time
now.formatted() // "6/6/2026, 12:00 PM"

// Scoped static methods — each returns a configured Date.FormatStyle:
now.formatted(.dateTime.day().month(.abbreviated).year())
// "Jun 6, 2026"

now.formatted(.dateTime.hour().minute())
// "12:00 PM"

// ISO 8601 — great for API payloads / filenames
now.formatted(.iso8601)
// "2026-06-06T12:00:00Z"

now.formatted(.iso8601.year().month().day().dateSeparator(.dash))
// "2026-06-06"

// Relative presentation — great for list UIs
now.formatted(.relative(presentation: .named))
// "now"

Date(timeIntervalSinceNow: -86400).formatted(.relative(presentation: .named))
// "yesterday"

// .formatted(date:time:) — the most common shorthand:
now.formatted(date: .abbreviated, time: .omitted)
// "Jun 6, 2026"

now.formatted(date: .numeric, time: .shortened)
// "(6/6/26", "12:00 PM")

now.formatted(date: .complete, time: .complete)
// "Friday", "June 6, 2026", "12:00:00 PM", "Pacific Daylight Time"

// Omited entirely:
now.formatted(date: .omitted, time: .shortened)
// "12:00 PM"
```

**Building a date style piece by piece:**

```swift
let style = Date.FormatStyle()
    .year(.defaultDigits)
    .month(.abbreviated)
    .day(.twoDigits)
    .hour(.defaultDigits(amPM: .abbreviated))
    .minute(.twoDigits)
// "Jun "06", "2026 at 12:00 PM"

// Calendar override:
let gregorianStyle = Date.FormatStyle(calendar: Calendar(identifier: .gregorian))
```

**Parsing dates from strings:**

```swift
let input = "2026-06-06T12:00:00Z"
let style = Date.FormatStyle().iso8601
let date = try? style.parse(input)
```

## Number Formatting

`Number.FormatStyle` replaces `NumberFormatter` with chainable configuration.

**Scoped static methods — .number, .percent, .currency, .precision:**

```swift
let value: Double = 1234.5678

// Default number:
value.formatted() // 1,234.5678

// Scoped static method for number:
value.formatted(.number)
// 1,234.568

// Precision — fraction length:
value.formatted(.number.precision(.fractionLength(0)))
// 1,235

value.formatted(.number.precision(.fractionLength(2)))
// 1,234.57

// Significant digits:
value.formatted(.number.precision(.significantDigits(3)))
// 1,230

// Percentage:
let rate: Double = 0.856
rate.formatted(.percent) // 85.6%

rate.formatted(.percent.precision(.fractionLength(0)))
// 86%

rate.formatted(.sign(strategy: .always()).precision(.fractionLength(1)))
// ±85.6%

// Currency:
value.formatted(.currency(code: "USD"))
// $1,234.57

value.formatted(.currency(code: "EUR"))
// €1,234.57

value.formatted(.currency(code: "JPY").precision(.fractionLength(0)))
// ¥1,235

// Grouping — no grouping for compact display:
value.formatted(.number.grouping(.never))
// 1234.57

// Scale — useful for axis labels:
(1_500_000.0).formatted(.number.scale(1_000_000).precision(.fractionLength(1)))
// 1.5
```

**Integer formatting:**

```swift
let count: Int = 42000
count.formatted(.number) // 42,000

count.formatted(.number.grouping(.never)) // 42000

// Hex (not on Number.FormatStyle — use String interpolation):
String(format: "0x%02X", 255) // "0xFF"
```

## Measurement Formatting

`Measurement.FormatStyle` replaces `MeasurementFormatter` with chainable configuration for `Measurement<UnitType>`.

**Scoped static methods — .measurement(width:usage:):**

```swift
import Foundation

let distance = Measurement(value: 100, unit: UnitLength.kilometers)

// Default — picks unit based on locale:
distance.formatted() // "62.14 mi" or "100 km" depending on locale

// Width variants:
distance.formatted(.measurement(width: .wide))
// "100 kilometers"

distance.formatted(.measurement(width: .abbreviated))
// "100 km"

distance.formatted(.measurement(width: .narrow))
// "100km"

// Usage — the formatter picks the most appropriate unit:
let personHeight = Measurement(value: 180, unit: UnitLength.centimeters)
personHeight.formatted(.measurement(width: .abbreviated, usage: .personHeight))
// "5 ft 11 in" (en_US) or "180 cm" (en_GB)

let temperature = Measurement(value: 23, unit: UnitTemperature.celsius)
temperature.formatted(.measurement(width: .abbreviated, usage: .weather))
// "23°C" or "73°F" depending on locale

let fuel = Measurement(value: 45, unit: UnitVolume.liters)
fuel.formatted(.measurement(width: .abbreviated, usage: .fuelVolume))
// "11.9 gal" (en_US) or "45 L" (en_GB)

// Locale override:
distance.formatted(.measurement(width: .wide).locale(Locale(identifier: "de_DE")))
// "100 Kilometer"

// Force a specific unit:
distance.converted(to: .miles).formatted(.measurement(width: .abbreviated))
// "62.1 mi"
```

## List Formatting

`List.FormatStyle` replaces `ListFormatter` for joining collections into natural-language lists.

**Scoped static methods — .list(type:width:), .or, .and:**

```swift
let items = ["Alice", "Bob", "Charlie"]

// Default — .and list:
items.formatted() // "Alice, Bob, and Charlie"

// Scoped static methods:
items.formatted(.list(type: .and))
// "Alice, Bob, and Charlie"

items.formatted(.list(type: .or))
// "Alice, Bob, or Charlie"

// Width variants:
items.formatted(.list(type: .and, width: .wide))
// "Alice, Bob, and Charlie"

items.formatted(.list(type: .and, width: .short))
// "Alice, Bob, & Charlie"

items.formatted(.list(type: .and, width: .narrow))
// "Alice, Bob, Charlie"

// Single item:
["Alice"].formatted(.list(type: .and))
// "Alice"

// Empty:
[String]().formatted(.list(type: .and))
// ""

// With custom element formatting:
let numbers = [1, 2, 3]
numbers.formatted(.list(type: .or) {
    $0.formatted(.number)
})
// "1, 2, or 3"
```

## Duration Formatting

`Duration.TimeFormatStyle` formats `Duration` (Swift's native duration type) and `TimeInterval`.

**Scoped static methods — .hours, .minutes(.annotated), positional/duration style:**

```swift
import Foundation

let duration: Duration = .seconds(3661) // 1h 1m 1s

// Default — picks the most appropriate unit:
duration.formatted() // "1:01:01"

// Scoped static methods:
duration.formatted(.time(pattern: .hourMinuteSecond))
// "1:01:01"

duration.formatted(.time(pattern: .hourMinute))
// "1:01"

duration.formatted(.time(pattern: .minuteSecond))
// "61:01"

// Positional style — each component shown:
duration.formatted(.time(pattern: .hourMinuteSecond(padHourToLength: 2)))
// "01:01:01"

// Duration style — natural language:
duration.formatted(.units(allowed: [.hours, .minutes, .seconds]))
// "1 hr, 1 min, 1 sec"

duration.formatted(.units(allowed: [.hours, .minutes]))
// "1 hr, 1 min"

duration.formatted(.units(allowed: [.minutes]))
// "61 min"

// Width:
duration.formatted(.units(allowed: [.hours, .minutes, .seconds], width: .wide))
// "1 hour, 1 minute, 1 second"

duration.formatted(.units(allowed: [.hours, .minutes, .seconds], width: .abbreviated))
// "1 hr, 1 min, 1 sec"

duration.formatted(.units(allowed: [.hours, .minutes, .seconds], width: .narrow))
// "1h 1m 1s"

// TimeInterval (Double) also works:
let interval: TimeInterval = 90.5
interval.formatted(.time(pattern: .minuteSecond))
// "1:30"

// Countdown style:
duration.formatted(.units(allowed: [.hours, .minutes, .seconds], width: .narrow))
// "1h 1m 1s"
```

## Custom FormatStyle

Implement the `FormatStyle` protocol to create reusable, chainable formatters for your own types.

**Minimal conformance:**

```swift
struct FileSizeFormatStyle: FormatStyle {
    typealias FormatInput = Int64
    typealias FormatOutput = String

    var locale: Locale = .current

    func format(_ value: Int64) -> String {
        let units = ["B", "KB", "MB", "GB", "TB"]
        var size = Double(value)
        var unitIndex = 0

        while size >= 1024 && unitIndex < units.count - 1 {
            size /= 1024
            unitIndex += 1
        }

        if unitIndex == 0 {
            return "\(Int(size)) \(units[unitIndex])"
        } else {
            return String(format: "%.1f %@", size, units[unitIndex])
        }
    }
}

// Usage:
let bytes: Int64 = 1_536_000
bytes.formatted(FileSizeFormatStyle())
// "1.5 MB"
```

**With chainable modifiers:**

```swift
struct FileSizeFormatStyle: FormatStyle {
    enum Precision { case automatic, bytes, kilobytes, megabytes }

    var locale: Locale = .current
    var precision: Precision = .automatic
    var includesUnit: Bool = true

    func format(_ value: Int64) -> String {
        let units = ["B", "KB", "MB", "GB", "TB"]
        var size = Double(value)
        var unitIndex = 0

        switch precision {
        case .automatic:
            while size >= 1024 && unitIndex < units.count - 1 {
                size /= 1024
                unitIndex += 1
            }
        case .bytes:
            unitIndex = 0
        case .kilobytes:
            size = Double(value) / 1024
            unitIndex = 1
        case .megabytes:
            size = Double(value) / (1024 * 1024)
            unitIndex = 2
        }

        let number = unitIndex == 0 ? "\(Int(size))" : String(format: "%.1f", size)
        return includesUnit ? "\(number) \(units[unitIndex])" : number
    }

    // Chainable modifier:
    func precision(_ precision: Precision) -> Self {
        var copy = self
        copy.precision = precision
        return copy
    }

    func unitIncluded(_ includes: Bool) -> Self {
        var copy = self
        copy.includesUnit = includes
        return copy
    }
}

// Usage:
let bytes: Int64 = 1_536_000
bytes.formatted(FileSizeFormatStyle().precision(.megabytes))
// "1.5 MB"

bytes.formatted(FileSizeFormatStyle().unitIncluded(false))
// "1.5"
```

**`FormatOutput` doesn't have to be `String`:**

```swift
struct AttributedFormatStyle: FormatStyle {
    typealias FormatInput = String
    typealias FormatOutput = AttributedString

    var locale: Locale = .current

    func format(_ value: String) -> AttributedString {
        var result = AttributedString(value)
        result.foregroundColor = .secondaryLabel
        result.font = .monospaced(.body)()
        return result
    }
}
```

## macOS Display Patterns

Different UI contexts on macOS demand different formatting densities.

**Menu items — compact dates:**

```swift
// In a menu item label, use abbreviated date + omitted time:
let lastOpened = Date(timeIntervalSinceNow: -3600)
Text(lastOpened.formatted(date: .abbreviated, time: .omitted))
// "Jun 6, 2026"

// For "recently opened" menus, use relative:
Text(lastOpened.formatted(.relative(presentation: .named)))
// "1 hour ago"
```

**Status bar — short numbers:**

```swift
// In a menu bar extra or status item, use narrow formatting:
let downloadSpeed: Double = 12_500_000 // bytes/sec
Text(downloadSpeed.formatted(.number.precision(.fractionLength(1))))
// "12.5" — pair with a separate "MB/s" label

// Or use measurement:
let speed = Measurement(value: 12.5, unit: UnitInformationStorage.megabytes)
Text(speed.formatted(.measurement(width: .narrow)))
// "12.5MB"
```

**Inspector — detailed formatting:**

```swift
// In an inspector panel, show full detail:
let fileSize: Int64 = 1_536_000
VStack(alignment: .leading) {
    Text("Size")
    Text(fileSize.formatted(FileSizeFormatStyle()))
    Text("\(fileSize.formatted(.number)) bytes")
        .foregroundStyle(.secondary)
        .font(.caption)
}
// "1.5 MB"
// "1,536,000 bytes"
```

**Table columns — aligned numbers:**

```swift
// In a SwiftUI Table, use fixed-width formatting for alignment:
Table(files) {
    TableColumn("Name") { file in
        Text(file.name)
    }
    TableColumn("Size") { file in
        Text(file.size.formatted(FileSizeFormatStyle()))
            .monospacedDigit()
            .frame(maxWidth: .infinity, alignment: .trailing)
    }
    TableColumn("Modified") { file in
        Text(file.modified.formatted(date: .abbreviated, time: .shortened))
            .monospacedDigit()
    }
}

// For NSTableView (AppKit), use the same styles in cell configuration:
let sizeCell = NSTextField(labelWithString: file.size.formatted(FileSizeFormatStyle()))
sizeCell.alignment = .right
sizeCell.font = .monospacedSystemFont(ofSize: 11, weight: .regular)
```

**Large-number display in Table columns:**

```swift
// For financial or scientific data, use grouping and fixed precision:
let price: Double = 1234567.89
price.formatted(.currency(code: "USD").precision(.fractionLength(2)))
// "$1,234,567.89"

// Compact large numbers for dense tables:
let count: Int = 1234567
count.formatted(.number.notation(.compactName))
// "1.2M"
```

## Locale

Every `FormatStyle` carries a `locale` property. The default is `Locale.current`.

**Override locale per format:**

```swift
let date = Date()

// Per-call locale override:
date.formatted(.dateTime.day().month().year().locale(Locale(identifier: "ja_JP")))
// "2026年6月6日"

date.formatted(.dateTime.day().month().year().locale(Locale(identifier: "de_DE")))
// "6. Juni 2026"

// Per-style locale:
let euroStyle = Number.FormatStyle()
    .precision(.fractionLength(2))
    .locale(Locale(identifier: "de_DE"))

(1234.56).formatted(euroStyle)
// "1.234,56" — comma as decimal separator

// Currency with locale:
let price: Double = 99.99
price.formatted(.currency(code: "EUR").locale(Locale(identifier: "fr_FR")))
// "99,99 €"

price.formatted(.currency(code: "EUR").locale(Locale(identifier: "de_DE")))
// "99,99 €"
```

**Locale affects:**
- Number grouping (`,` vs `.` vs ` `)
- Decimal separator
- Date component ordering (MDY vs DMY vs YMD)
- Month/day names
- Currency symbol position (before/after, with/without space)
- Measurement unit selection (metric vs imperial)

## Constraints & Gotchas

- **Locale vs. calendar:** Setting a locale on `Date.FormatStyle` also changes the calendar. If you need a specific calendar with a different locale, set both explicitly: `Date.FormatStyle(calendar: .gregorian).locale(Locale(identifier: "ja_JP"))`.
- **Threading:** `FormatStyle` values are value types (structs). They are safe to create on any thread. However, `Locale.current` is thread-local — its value may differ across threads. Capture the locale explicitly if you need consistency.
- **`.formatted()` is not KVO-compliant:** You cannot observe a formatted string. Format at display time, not in a stored property.
- **Measurement `.usage` is locale-dependent:** `.personHeight` returns feet/inches in `en_US` but centimeters in `en_GB`. Don't assume the unit — only assume the semantic meaning.
- **Duration vs. TimeInterval:** `Duration.TimeFormatStyle` works with `Duration` (Swift's native type) and `TimeInterval` (Double). `Duration` is preferred for new code; `TimeInterval` bridging is provided for compatibility.
- **List formatting with empty collections:** `[String]().formatted(.list(type: .and))` returns `""` (empty string). Handle this in UI — don't assume non-empty output.
- **Currency code is not validated:** `.currency(code: "XYZ")` will format with "XYZ" even if it's not a real currency. Validate codes against `ISO4217CurrencyCodes` if needed.
- **NumberFormatter bridging:** `Number.FormatStyle.FormatInput` accepts `Double`, `Int`, `Float`, and other numeric types, but not `Decimal`. For `Decimal`, use `NumberFormatter` directly or convert to `Double` (with potential precision loss).

## Common Mistakes

- **Using `DateFormatter` directly:** Prefer `Date.FormatStyle` in all new code. `DateFormatter` is not Sendable, requires manual locale/calendar management, and is more error-prone.
- **Creating format styles in tight loops:** `Date.FormatStyle()` is cheap to create, but if you're formatting thousands of values, hoist the style into a `static let` or instance property.
- **Ignoring locale in tests:** Tests that hardcode formatted strings like `"Jun 6, 2026"` will fail on non-English systems. Use `.locale(Locale(identifier: "en_US_POSIX"))` in tests for deterministic output.
- **Confusing `.precision(.fractionLength(N))` with `.precision(.significantDigits(N))`:** `fractionLength` controls digits after the decimal point. `significantDigits` controls total meaningful digits. `1234.567.formatted(.number.precision(.fractionLength(1)))` → `"1,234.6"`. `1234.567.formatted(.number.precision(.significantDigits(3)))` → `"1,230"`.
- **Forgetting `.monospacedDigit()` in Table columns:** Without it, digits have proportional widths and columns don't align visually.
- **Using `.currency(code:)` without considering locale:** The currency code determines the symbol and decimal places, but the locale determines grouping, separator, and symbol position. Both matter.

## Review Checklist

- [ ] All date formatting uses `Date.FormatStyle`, not `DateFormatter`
- [ ] All number formatting uses `Number.FormatStyle`, not `NumberFormatter`
- [ ] Reusable styles are hoisted to `static let` or instance properties, not recreated per call
- [ ] Locale is explicitly set for deterministic output (tests, APIs, filenames)
- [ ] Table column numbers use `.monospacedDigit()` for alignment
- [ ] Menu items use compact/abbreviated styles, not full formatting
- [ ] Custom `FormatStyle` conformances implement `locale` and `format(_:)` correctly
- [ ] `Measurement.FormatStyle` uses `.usage:` for locale-appropriate unit selection
- [ ] Tests use `Locale(identifier: "en_US_POSIX")` for deterministic string assertions
- [ ] Empty collections are handled when using `List.FormatStyle`

## References

- [references/formatstyle-patterns.md](references/formatstyle-patterns.md) — Custom FormatStyle for file size display, date formatting patterns for macOS UI contexts, number formatting for Table columns, duration formatting for timers, locale-aware measurement display
- [Apple: FormatStyle protocol](https://developer.apple.com/documentation/foundation/formatstyle)
- [Apple: Date.FormatStyle](https://developer.apple.com/documentation/foundation/date/formatstyle)
- [Apple: Number.FormatStyle](https://developer.apple.com/documentation/foundation/number/formatstyle)
- [Apple: Measurement.FormatStyle](https://developer.apple.com/documentation/foundation/measurement/formatstyle)
- [Apple: List.FormatStyle](https://developer.apple.com/documentation/foundation/list/formatstyle)
- [Apple: Duration.TimeFormatStyle](https://developer.apple.com/documentation/foundation/duration/timeformatstyle)

## Research Addendum — 2026 Updates

### FormatStyle Date & Number Patterns (2025-2026)

- **`Date.FormatStyle` is the default** — Apple's official stance in 2026 is that `DateFormatter` is legacy. `Date.FormatStyle` is locale-aware, type-safe, `Sendable`, and thread-safe.
- **`.locale()` affects calendar**: Setting a locale on `Date.FormatStyle` also changes the calendar (e.g., Japanese era calendar for `ja_JP`). Override with `Date.FormatStyle(calendar: .gregorian).locale(...)` if you need a specific calendar separate from locale.
- **`.formatted()` is `Sendable`-safe**: FormatStyle instances are value types and safe to share across concurrency boundaries. Unlike `DateFormatter`, which is not `Sendable`.
- **`.formatted(.relative(presentation: .named))`** — Great for menu items, recents lists: `"yesterday"`, `"2 hours ago"`, `"last week"`.
- **Number `.notation(.compactName)`** — Built-in compact notation for large numbers: `1_234_567.formatted(.number.notation(.compactName))` → `"1.2M"`. Use in Table columns where space is limited.
- **Measurement `.usage` drives unit selection**: `.personHeight`, `.weather`, `.fuelVolume`, `.road`, `.asphalt` — the formatter picks the most appropriate unit for the locale. Don't assume metric/imperial.

### Custom FormatStyle Conformance (2026)

- **Protocol caching**: Foundation caches identical customized format styles for efficiency. Creating the same style twice returns the cached instance.
- **`ParseableFormatStyle`** — Conform to this alongside `FormatStyle` to enable parsing strings back into your type. Requires a `parseStrategy` property.
  ```swift
  extension FileSizeFormatStyle: ParseableFormatStyle {
      var parseStrategy: FileSizeParseStrategy { FileSizeParseStrategy() }
  }
  ```
- **`FormatOutput = AttributedString`** — Custom styles can produce `AttributedString` for rich text output, useful in `Text` views with mixed styling.
- **Chainable modifiers pattern**: Add methods like `.precision(_:)` that return `var copy = self; copy.precision = precision; return copy`. This is the idiomatic pattern for configurable styles.
- **Testing tip**: Use `.locale(Locale(identifier: "en_US_POSIX"))` in unit tests for deterministic formatted string output regardless of test runner locale.
