# FormatStyle Patterns — Deep Dive

Concrete, copy-paste-ready patterns for macOS UI formatting with `FormatStyle`. Each section is a self-contained recipe.

## Custom FormatStyle for File Size Display

```swift
import Foundation

/// Formats byte counts as human-readable file sizes (B/KB/MB/GB/TB).
/// Binary-based (1 KB = 1024 B), matching macOS Finder behavior.
struct FileSizeFormatStyle: FormatStyle {
    enum Thresholds {
        static let kb: Int64 = 1024
        static let mb: Int64 = 1024 * 1024
        static let gb: Int64 = 1024 * 1024 * 1024
        static let tb: Int64 = 1024 * 1024 * 1024 * 1024
    }

    var locale: Locale = .current
    var maximumFractionDigits: Int = 1
    var minimumIntegerDigits: Int = 1
    var unitStyle: UnitStyle = .abbreviated

    enum UnitStyle { case wide, abbreviated, narrow }

    typealias FormatInput = Int64
    typealias FormatOutput = String

    func format(_ value: Int64) -> String {
        guard value >= 0 else { return "-" + format(-value) }

        let unit: String
        let divisor: Double

        switch value {
        case 0..<Thresholds.kb:
            return "\(value) B"
        case Thresholds.kb..<Thresholds.mb:
            unit = unitString(for: "KB", style: unitStyle)
            divisor = Double(Thresholds.kb)
        case Thresholds.mb..<Thresholds.gb:
            unit = unitString(for: "MB", style: unitStyle)
            divisor = Double(Thresholds.mb)
        case Thresholds.gb..<Thresholds.tb:
            unit = unitString(for: "GB", style: unitStyle)
            divisor = Double(Thresholds.gb)
        default:
            unit = unitString(for: "TB", style: unitStyle)
            divisor = Double(Thresholds.tb)
        }

        let result = Double(value) / divisor

        // Show as integer if it's a whole number
        if result.truncatingRemainder(dividingBy: 1) == 0 && result < 10 {
            return "\(Int(result)) \(unit)"
        }

        let formatter = NumberFormatter()
        formatter.locale = locale
        formatter.numberStyle = .decimal
        formatter.minimumFractionDigits = 0
        formatter.maximumFractionDigits = maximumFractionDigits
        formatter.minimumIntegerDigits = minimumIntegerDigits

        let numberString = formatter.string(from: NSNumber(value: result)) ?? "\(result)"
        return "\(numberString) \(unit)"
    }

    private func unitString(for unit: String, style: UnitStyle) -> String {
        switch style {
        case .wide:
            return unit // "KB", "MB", etc.
        case .abbreviated:
            return unit
        case .narrow:
            return String(unit.prefix(1)) // "K", "M", "G", "T"
        }
    }

    // Chainable modifiers:

    func maximumFractionDigits(_ digits: Int) -> Self {
        var copy = self
        copy.maximumFractionDigits = digits
        return copy
    }

    func unitStyle(_ style: UnitStyle) -> Self {
        var copy = self
        copy.unitStyle = style
        return copy
    }
}

// Provide a convenient .formatted() overload:
extension Int64 {
    func formatted(_ style: FileSizeFormatStyle) -> String {
        style.format(self)
    }
}

extension UInt64 {
    func formatted(_ style: FileSizeFormatStyle) -> String {
        style.format(Int64(self))
    }
}

// Usage examples:
let bytes: Int64 = 0
bytes.formatted(FileSizeFormatStyle()) // "0 B"

(512 as Int64).formatted(FileSizeFormatStyle()) // "512 B"

(1_536_000 as Int64).formatted(FileSizeFormatStyle()) // "1.5 MB"

(1_536_000 as Int64).formatted(FileSizeFormatStyle().maximumFractionDigits(2)) // "1.47 MB"

(1_073_741_824 as Int64).formatted(FileSizeFormatStyle()) // "1.0 GB"

(1_073_741_824 as Int64).formatted(FileSizeFormatStyle().unitStyle(.narrow)) // "1.0 G"

// In a SwiftUI view:
struct FileRowView: View {
    let file: FileItem
    var body: some View {
        HStack {
            Text(file.name)
            Spacer()
            Text(file.size.formatted(FileSizeFormatStyle()))
                .monospacedDigit()
                .foregroundStyle(.secondary)
        }
    }
}
```

### File size with progress/comparison

```swift
/// Shows "used of total" with automatic unit selection
struct StorageRatioFormat: FormatStyle {
    var locale: Locale = .current

    typealias FormatInput = (used: Int64, total: Int64)
    typealias FormatOutput = String

    func format(_ value: (used: Int64, total: Int64)) -> String {
        let totalFormatted = value.total.formatted(FileSizeFormatStyle())
        let usedFormatted = value.used.formatted(
            FileSizeFormatStyle().maximumFractionDigits(2)
        )
        return "\(usedFormatted) of \(totalFormatted)"
    }
}

// Usage:
let usage = (used: 234_000_000_000 as Int64, total: 1_000_000_000_000 as Int64)
usage.formatted(StorageRatioFormat())
// "234.00 GB of 1.0 TB"
```

## Date Formatting Patterns for macOS UI Contexts

### Relative dates for list rows

```swift
/// Formats a date relative to now, with fallback to absolute date.
/// Great for file lists, message queues, and activity feeds.
struct SmartDateFormat: FormatStyle {
    var locale: Locale = .current
    var fallback: Date.FormatStyle = .dateTime.day().month(.abbreviated).year()

    typealias FormatInput = Date
    typealias FormatOutput = String

    func format(_ value: Date) -> String {
        let calendar = Calendar.current
        let now = Date()

        if calendar.isDateInToday(value) {
            return value.formatted(date: .omitted, time: .shortened)
            // "12:30 PM"
        }

        if calendar.isDateInYesterday(value) {
            return "Yesterday, " + value.formatted(date: .omitted, time: .shortened)
            // "Yesterday, 12:30 PM"
        }

        let daysDifference = calendar.dateComponents([.day], from: value, to: now).day ?? 0
        if daysDifference < 7 {
            return value.formatted(.dateTime.weekday(.wide).hour().minute())
            // "Friday, 12:30 PM"
        }

        return value.formatted(fallback)
            // "Jun 1, 2026"
    }
}

// Usage:
let dates = [
    Date(),
    Date(timeIntervalSinceNow: -3600),
    Date(timeIntervalSinceNow: -86400),
    Date(timeIntervalSinceNow: -259_200), // 3 days ago
    Date(timeIntervalSinceNow: -604_800), // 7 days ago
]

for date in dates {
    print(date.formatted(SmartDateFormat()))
}
// "12:30 PM"
// "11:30 AM"
// "Yesterday, 12:30 PM"
// "Wednesday, "12:30 PM"
// "May 30, 2026"
```

### Compact dates for menu bar / status item

```swift
/// Ultra-compact date for status bar display (max ~12 characters).
struct StatusBarDateFormat: FormatStyle {
    var locale: Locale = .current
    typealias FormatInput = Date
    typealias FormatOutput = String

    func format(_ value: Date) -> String {
        let calendar = Calendar.current
        let now = Date()

        if calendar.isDate(value, inSameDayAs: now) {
            return value.formatted(date: .omitted, time: .shortened) // "12:30 PM"
        }

        let formatter = DateFormatter()
        formatter.locale = locale
        formatter.dateFormat = "MMM d" // "Jun 6"
        return formatter.string(from: value)
    }
}
```

### Full dates for inspector / detail panel

```swift
/// Detailed date for inspector panels and preferences.
struct InspectorDateFormat: FormatStyle {
    var locale: Locale = .current
    var showSeconds: Bool = true
    var showZone: Bool = false

    typealias FormatInput = Date
    typealias FormatOutput = String

    func format(_ value: Date) -> String {
        var style = Date.FormatStyle()
            .weekday(.wide)
            .month(.wide)
            .day()
            .year()
            .hour(.defaultDigits(amPM: .wide))
            .minute(.twoDigits)

        if showSeconds {
            style = style.second(.twoDigits)
        }

        if showZone {
            style = style.timeZone(.localizedGMT)
        }

        return value.formatted(style.locale(locale))
        // "Friday, June 6, 2026 at 12:30:00 PM GMT-07:00"
    }
}

// In an inspector view:
struct FileMetadataInspector: View {
    let file: FileItem
    var body: some View {
        Form {
            LabeledContent("Created") {
                Text(file.created.formatted(InspectorDateFormat()))
                    .monospacedDigit()
            }
            LabeledContent("Modified") {
                Text(file.modified.formatted(InspectorDateFormat()))
                    .monospacedDigit()
            }
        }
        .formStyle(.grouped)
    }
}
```

### ISO 8601 for filenames and API payloads

```swift
/// Date formatted for use in filenames (no colons, no spaces).
struct FilenameDateFormat: FormatStyle {
    typealias FormatInput = Date
    typealias FormatOutput = String

    func format(_ value: Date) -> String {
        value.formatted(.iso8601.year().month().day().dateSeparator(.dash))
            + "T" + value.formatted(.iso8601.hour().minute().second().timeSeparator(.omitted))
    }
}

Date().formatted(FilenameDateFormat()) // "2026-06-06T123615"

// For user-visible filenames:
let filename = "Export_\(Date().formatted(FilenameDateFormat())).csv"
// "Export_"2026-06-06T123615".csv"
```

## Number Formatting for Table Columns

### Aligned numbers with color coding

```swift
import SwiftUI

/// Formats numbers for Table columns with consistent alignment.
struct TableNumberFormat {
    /// Default: right-aligned, monospaced digits, appropriate precision
    static let sharePercentage: Number.FormatStyle = .percent
        .precision(.fractionLength(1))
        .sign(strategy: .always())

    static let currencyUSD: Number.FormatStyle = .currency(code: "USD")
        .precision(.fractionLength(2))

    static let compactCount: Number.FormatStyle = .number
        .notation(.compactName)
        .precision(.fractionLength(1))

    static let exactCount: Number.FormatStyle = .number
        .precision(.fractionLength(0))

    /// For change/delta columns: always show sign, color in SwiftUI
    static let delta: Number.FormatStyle = .number
        .sign(strategy: .always())
        .precision(.fractionLength(2))
}

// Table with aligned columns:
struct TransactionTable: View {
    let transactions: [Transaction]

    var body: some View {
        Table(transactions) {
            TableColumn("Date") { tx in
                Text(tx.date.formatted(date: .abbreviated, time: .omitted))
                    .monospacedDigit()
            }
            .width(min: 100, ideal: 120)

            TableColumn("Description") { tx in
                Text(tx.description)
            }

            TableColumn("Amount") { tx in
                Text(tx.amount.formatted(TableNumberFormat.currencyUSD))
                    .monospacedDigit()
                    .frame(maxWidth: .infinity, alignment: .trailing)
                    .foregroundStyle(tx.amount < 0 ? .red : .primary)
            }
            .width(min: 100, ideal: 120)

            TableColumn("Change") { tx in
                Text(tx.change.formatted(TableNumberFormat.delta))
                    .monospacedDigit()
                    .frame(maxWidth: .infinity, alignment: .trailing)
                    .foregroundStyle(tx.change > 0 ? .green : tx.change < 0 ? .red : .secondary)
            }
            .width(min: 80, ideal: 100)
        }
    }
}
```

### Notation styles for dense financial tables

```swift
let values: [Double] = [1_234, 12_345_678, 1_500_000_000, 42_000]

// Compact notation for wide tables:
for v in values {
    print(v.formatted(.number.notation(.compactName).precision(.fractionLength(1))))
}
// "1.2K"
// "12.3M"
// "1.5B"
// "42K"

// Scientific notation for very large/small:
let avogadro: Double = 6.022e23
avogadro.formatted(.number.notation(.scientific).precision(.fractionLength(3)))
// "6.022E23"

// Automatic (picks based on magnitude):
(1_500_000.0).formatted(.number) // "1,500,000"
(1_500_000.0).formatted(.number.notation(.compactName)) // "1.5M"
```

### Colored number display with AttributedFormatStyle

```swift
/// Formats a value with color indicators for positive/negative/zero.
struct FinancialFormat: FormatStyle {
    typealias FormatInput = Double
    typealias FormatOutput = (text: String, color: Color)

    var locale: Locale = .current

    func format(_ value: Double) -> (text: String, color: Color) {
        let formatted = value.formatted(.currency(code: "USD").sign(strategy: .always()))
        let color: Color = value > 0 ? .green : value < 0 ? .red : .secondary
        return (formatted, color)
    }
}

// In SwiftUI:
struct ChangeCell: View {
    let value: Double
    var body: some View {
        let result = value.formatted(FinancialFormat())
        Text(result.text)
            .foregroundStyle(result.color)
            .monospacedDigit()
            .frame(maxWidth: .infinity, alignment: .trailing)
    }
}
```

## Duration Formatting for Timer Displays

### Stopwatch / elapsed time

```swift
/// Formats a TimeInterval as a stopwatch display.
/// Shows hours only when > 0, always shows minutes and seconds.
struct StopwatchFormat: FormatStyle {
    typealias FormatInput = TimeInterval
    typealias FormatOutput = String

    var locale: Locale = .current
    var showsHours: Bool = true
    var showsMilliseconds: Bool = false

    func format(_ value: TimeInterval) -> String {
        guard value >= 0 else { return "-" + format(-value) }

        let totalMs = Int(value * 1000)
        let hours = totalMs / 3_600_000
        let minutes = (totalMs % 3_600_000) / 60_000
        let seconds = (totalMs % 60_000) / 1_000
        let ms = totalMs % 1_000

        if hours > 0 && showsHours {
            if showsMilliseconds {
                return String(format: "d:%02d:%02d.%03d", hours, minutes, seconds, ms)
            }
            return String(format: "d:%02d:%02d", hours, minutes, seconds)
        }

        if showsMilliseconds {
            return String(format: "%d:%02d.%03d", minutes, seconds, ms)
        }
        return String(format: "%d:%02d", minutes, seconds)
    }
}

// Usage:
(0 as TimeInterval).formatted(StopwatchFormat()) // "0:00"
(90.5 as TimeInterval).formatted(StopwatchFormat()) // "1:30"
(3661.0 as TimeInterval).formatted(StopwatchFormat()) // "1:01:01"

// With milliseconds — great for audio/video editing:
(90.567 as TimeInterval).formatted(StopwatchFormat(showsMilliseconds: true))
// "1:30.567"

// Compact mode (no hours unless needed):
(45 as TimeInterval).formatted(StopwatchFormat()) // "0:45"
```

### Countdown timer

```swift
/// Formats a remaining time as a countdown, with natural language fallback.
struct CountdownFormat: FormatStyle {
    typealias FormatInput = TimeInterval
    typealias FormatOutput = String

    var locale: Locale = .current
    var urgencyThreshold: TimeInterval = 60 // seconds

    func format(_ value: TimeInterval) -> String {
        guard value > 0 else { return "0:00" }

        let hours = Int(value) / 3600
        let minutes = (Int(value) % 3600) / 60
        let seconds = Int(value) % 60

        if hours > 0 {
            return String(format: "d:%02d:%02d", hours, minutes, seconds)
        }

        // Under urgency threshold, show seconds prominently:
        if value < urgencyThreshold {
            return String(format: "0:%02d", seconds)
        }

        return String(format: "d:%02d", minutes, seconds)
    }

    func style(for value: TimeInterval) -> (text: String, color: Color) {
        let text = format(value)
        let color: Color = value < urgencyThreshold ? .red : value < 300 ? .orange : .primary
        return (text, color)
    }
}

// In a timer view with color urgency:
struct CountdownView: View {
    let remaining: TimeInterval
    @State private var updateTrigger = false

    var body: some View {
        let result = CountdownFormat().style(for: remaining)
        Text(result.text)
            .monospacedDigit(.body)()
            .foregroundStyle(result.color)
            .animation(.easeInOut(duration: 0.3), value: result.color)
    }
}
```

### Duration components for detailed display

```swift
/// Shows each non-zero component of a duration.
struct DetailedDurationFormat: FormatStyle {
    typealias FormatInput = Duration
    typealias FormatOutput = String

    var locale: Locale = .current

    func format(_ value: Duration) -> String {
        let totalSeconds = value.components.seconds
        let hours = totalSeconds / 3600
        let minutes = (totalSeconds % 3600) / 60
        let seconds = totalSeconds % 60

        var parts: [String] = []
        if hours > 0 { parts.append("\(hours)h") }
        if minutes > 0 { parts.append("\(minutes)m") }
        if seconds > 0 || parts.isEmpty { parts.append("\(seconds)s") }

        return parts.joined(separator: " ")
    }
}

// Usage:
Duration.seconds(3661).formatted(DetailedDurationFormat()) // "1h 1m 1s"
Duration.seconds(45).formatted(DetailedDurationFormat()) // "45s"
```

## Locale-Aware Measurement Display

### Adaptive temperature display

```swift
/// Temperature that adapts to locale and supports override.
struct TemperatureFormat: FormatStyle {
    enum Mode { case auto, celsius, fahrenheit }

    typealias FormatInput = Measurement<UnitTemperature>
    typealias FormatOutput = String

    var mode: Mode = .auto
    var locale: Locale = .current

    func format(_ value: Measurement<UnitTemperature>) -> String {
        switch mode {
        case .auto:
            return value.formatted(.measurement(width: .abbreviated, usage: .weather))
        case .celsius:
            return value.converted(to: .celsius)
                .formatted(.measurement(width: .abbreviated).locale(locale))
        case .fahrenheit:
            return value.converted(to: .fahrenheit)
                .formatted(.measurement(width: .abbreviated).locale(locale))
        }
    }
}

// Usage:
let temp = Measurement(value: 23, unit: UnitTemperature.celsius)

// Auto-detects from locale:
temp.formatted(TemperatureFormat().locale(Locale(identifier: "en_US")))
// "73°F"
temp.formatted(TemperatureFormat().locale(Locale(identifier: "de_DE")))
// "23°C"

// Force unit:
temp.formatted(TemperatureFormat(mode: .fahrenheit))
// "73°F"
```

### Network speed with locale

```swift
struct NetworkSpeedFormat: FormatStyle {
    typealias FormatInput = Double // bytes per second
    typealias FormatOutput = String

    var locale: Locale = .current

    func format(_ value: Double) -> String {
        let measurement = Measurement(value: value, unit: UnitInformationStorage.bytes)
            .converted(to: .megabytes)
        return measurement.formatted(.measurement(width: .narrow)) + "/s"
    }
}

(12_500_000.0).formatted(NetworkSpeedFormat()) // "12.5MB/s"
(150_000.0).formatted(NetworkSpeedFormat()) // "0.1MB/s"

// For slow speeds, switch to KB:
struct AdaptiveSpeedFormat: FormatStyle {
    typealias FormatInput = Double // bytes per second
    typealias FormatOutput = String
    var locale: Locale = .current

    func format(_ value: Double) -> String {
        if value >= 1_000_000 {
            return (value / 1_000_000).formatted(.number.precision(.fractionLength(1))) + " MB/s"
        } else if value >= 1_000 {
            return (value / 1_000).formatted(.number.precision(.fractionLength(1))) + " KB/s"
        }
        return value.formatted(.number.precision(.fractionLength(0))) + " B/s"
    }
}

(12_500_000.0).formatted(AdaptiveSpeedFormat()) // "12.5 MB/s"
(150_000.0).formatted(AdaptiveSpeedFormat()) // "150.0 KB/s"
(500.0).formatted(AdaptiveSpeedFormat()) // "500 B/s"
```

### File transfer progress with locale

```swift
struct TransferProgressFormat: FormatStyle {
    typealias FormatInput = (completed: Int64, total: Int64)
    typealias FormatOutput = String

    var locale: Locale = .current
    var showPercentage: Bool = true

    func format(_ value: (completed: Int64, total: Int64)) -> String {
        let completed = value.completed.formatted(FileSizeFormatStyle())
        let total = value.total.formatted(FileSizeFormatStyle())

        if showPercentage, value.total > 0 {
            let pct = Double(value.completed) / Double(value.total) * 100
            let pctStr = pct.formatted(.number.precision(.fractionLength(1)))
            return "\(completed) of \(total) (\(pctStr)%)"
        }
        return "\(completed) of \(total)"
    }
}

(TransferProgressFormat()).format(
    (completed: 1_536_000, total: 10_485_760)
)
// "1.5 MB of 10.0 MB (14.6%)"
