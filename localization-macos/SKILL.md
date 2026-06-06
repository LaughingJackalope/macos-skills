---
name: localization-macos
description: "Localization on macOS — String Catalogs, LocalizedStringKey, String(localized:), FormatStyle, plural rules (.stringsdict), RTL layout, NSLocalizedString migration, Xcode localization export/import, menu item localization, accessibility label localization, locale-aware formatting. Use when localizing macOS apps for multiple languages."
sources: https://developer.apple.com/documentation/xcode/localization, https://developer.apple.com/documentation/foundation/formatstyle
researched: 2026-06-06
platform: macOS 26+
---

# Localization on macOS

Localize macOS 26+ apps using String Catalogs, Foundation formatters, and SwiftUI's localization primitives. Covers `.xcstrings` catalogs, `LocalizedStringKey`, `String(localized:)`, plural rules via `.stringsdict`, `FormatStyle` for locale-aware formatting, RTL layout, menu/accessibility label localization, and migrating legacy `NSLocalizedString` code. For menu bar structure see `macos-menus-commands`; for string formatting details see `swift-formatstyle-macos`.

See [references/localization-patterns.md](references/localization-patterns.md) for in-depth patterns: String Catalog with plurals/substitutions, FormatStyle deep-dive, RTL testing setup, `.stringsdict` menu localization, and NSLocalizedString migration workflow.

## Contents

- [String Catalogs](#string-catalogs)
- [SwiftUI Text](#swiftui-text)
- [String(localized:) Foundation](#stringlocalized-foundation)
- [Plural Rules](#plural-rules)
- [FormatStyle](#formatstyle)
- [RTL Layout](#rtl-layout)
- [NSLocalizedString Migration](#nslocalizedstring-migration)
- [Menu Item Localization](#menu-item-localization)
- [Accessibility Label Localization](#accessibility-label-localization)
- [Xcode Workflow](#xcode-workflow)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## String Catalogs

String Catalogs (`.xcstrings`) are the modern localization format for Apple platforms, replacing `.strings` and `.stringsdict` files in new projects. A single `.xcstrings` file stores all localizations for a target.

**Xcode editor:** Double-click the catalog in Xcode to open the spreadsheet view. Each row has a key, source language value, and one column per language. Xcode auto-extracts strings from `Text("key")`, `LocalizedStringKey`, `String(localized:)`, and `NSLocalizedString` at build time.

**Plural variations:** For a key like `item_count`, click the "Vary" button in the Xcode editor to add plural rules. The editor presents checkboxes for `zero`, `one`, `few`, `many`, `other` — only `other` is required. Each language's plural rules are handled by CLDR (Unicode Common Locale Data Repository), so Arabic gets all five categories while Japanese only needs `other`.

**Substitutes:** Use `%@` for string substitution, `%lld` for integers, and `%f` for doubles. Catalogs support named substitutions like `%name@` — the translator sees "name" as a placeholder label instead of a raw format specifier.

```swift
// Source: "Welcome, %@!"
// Catalog key: "welcome_message"
Text("welcome_message")
// Resolution at runtime: "Welcome, Alice!"
```

**Comments for translators:** Always add developer comments via `Text("key", comment: "Context: greeting shown on first launch"). Comments appear in the Xcode editor and are exported alongside the key for translators. Without comments, translators have no context for ambiguous strings (e.g., "Run" could be a verb or a noun).

**State management:** Each entry tracks translation state: `new`, `needs review`, `translated`, or `stale` (source value changed since last translation). Use state filters in the Xcode editor to find unreviewed strings before shipping.

## SwiftUI Text

`Text` has three localization modes depending on the initializer:

1. **Static string literal:** `Text("Hello")` — looks up "Hello" in the string catalog at runtime. The string literal is both the key and the default value. Simple but no control over default value vs key.

2. **Separate key and comment:** `Text("greeting_title", comment: "Title on welcome screen")` — uses "greeting_title" as the catalog key. The `comment` string appears in the catalog for translators but is never shown to users.

3. **Explicit `LocalizedStringKey`:** `Text(LocalizedStringKey(someVariable))` — dynamic key lookup. The variable is used as the key. This escapes string interpolation: `Text(LocalizedStringKey("item.\(type)"))` looks up the interpolated result as a key.

**Interpolations:** Inside a localized `Text`, interpolations carry type information to the formatter:

```swift
// Catalog value: "You have %lld items"
Text("item_count")  // ← This is wrong, use format specifier

// Correct: specify the format in the catalog, pass the value to Text
Text("You have \(count) items")
// Catalog key: "You have %lld items" (or use "item_count" with variation)
```

The type of the interpolated value determines the format: `Int` → `%lld`, `Double` → `%f`, `String` → `%@`.

**String literal vs. `LocalizedStringKey`:** A SwiftUI `Text("foo")` initializer takes a `String`, but `Text` overloads handle `LocalizedStringKey` differently. When the initializer sees a `String`, it may use it as a literal display value (no localization) or as a lookup key depending on the overload. The compiler resolves `"Hello"` as `LocalizedStringKey.init(_:)` when the `Text` initializer accepts `LocalizedStringKey`. To avoid ambiguity, be explicit: `Text("key", comment: "")` always localizes.

**`verbatim:` parameter:** `Text(verbatim: "v1.2.3")` bypasses the string catalog entirely. Use for version numbers, bundle IDs, debug strings, or anything that should never be translated. This is the correct escape hatch instead of omitting entries from the catalog.

## String(localized:) Foundation

`String(localized:table:bundle:comment:)` loads a localized string explicitly, outside of SwiftUI's `Text`:

```swift
let greeting = String(
    localized: "welcome.title",
    table: "MainWindow",        // .strings file name (nil = Localizable.strings)
    bundle: .main,
    comment: "Title on the welcome window"
)
```

**Parameters:**

| Parameter | Default | Purpose |
|---|---|---|
| `key` | required | Catalog key to look up |
| `table` | `nil` (Localizable) | For legacy `.strings` files: the filename without extension. Ignored for String Catalogs |
| `bundle` | `.main` | Bundle containing the catalog. Use `Bundle(for: type(of: self))` for framework strings |
| `specifier` | `nil` | Format specifier hint for the localization system |
| `comment` | `nil` | Translator context |

**When to use:** `String(localized:)` is for AppKit code, model-layer strings, notifications, or anywhere SwiftUI's `Text` isn't available. In SwiftUI views, prefer `Text` since it handles string extraction automatically.

**Table/bundle priority:** When a key isn't in the string catalog, the system falls back to `.strings` files in the specified table. Bundle priority order: app bundle → language `.lproj` directories → development language.

## Plural Rules

`.stringsdict` files handle complex pluralization that can't be expressed with String Catalogs alone. You can also use them alongside catalogs — Xcode reads both.

**File structure:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>items_found</key>
    <dict>
        <key>NSStringLocalizedFormatKey</key>
        <string>%#@items@</string>
        <key>items</key>
        <dict>
            <key>NSStringFormatSpecTypeKey</key>
            <string>NSStringPluralRuleType</string>
            <key>NSStringFormatValueTypeKey</key>
            <string>d</string>
            <key>zero</key>
            <string>No items found</string>
            <key>one</key>
            <string>1 item found</string>
            <key>other</key>
            <string>%d items found</string>
        </dict>
    </dict>
</dict>
</plist>
```

**Key rules:**
- `NSStringLocalizedFormatKey` references the variable with `%#@variable@` syntax.
- Each variable dictionary has `NSStringFormatSpecTypeKey` = `NSStringPluralRuleType`.
- `NSStringFormatValueTypeKey` is the format type: `d` (integer), `ld` (long), `f` (float).
- Required category: `other`. All others (`zero`, `one`, `few`, `many`) are optional and language-dependent.
- Languages differ: Arabic has 6 categories, Japanese only needs `other`, Russian needs `one`/`few`/`many`.

**Multiple variables:** Nesting plural dictionaries supports compound messages:

```xml
<key>compound_time</key>
<dict>
    <key>NSStringLocalizedFormatKey</key>
    <string>%#@hours@ and %#@minutes@</string>
    <key>hours</key>
    <dict>
        <key>NSStringFormatSpecTypeKey</key>
        <string>NSStringPluralRuleType</string>
        <key>one</key><string>1 hour</string>
        <key>other</key><string>%d hours</string>
    </dict>
    <key>minutes</key>
    <dict>
        <key>NSStringFormatSpecTypeKey</key>
        <string>NSStringPluralRuleType</string>
        <key>one</key><string>1 minute</string>
        <key>other</key><string>%d minutes</string>
    </dict>
</dict>
```

## FormatStyle

`FormatStyle` replaces `DateFormatter` and `NumberFormatter` with type-safe, locale-aware formatting. Foundation provides built-in styles for dates, numbers, currencies, lists, and more.

**Date formatting:**

```swift
import Foundation

// Current locale (default)
let formatted = Date.now.formatted(.dateTime.day().month().year())

// Explicit locale
let frenchDate = Date.now.formatted(
    .dateTime.day().month().year()
    .locale(Locale(identifier: "fr_FR"))
)

// FormatStyle preset
let brief = Date.now.formatted(date: .abbreviated, time: .shortened)
```

**Number formatting:**

```swift
let price = 49.99
let formattedPrice = price.formatted(
    .currency(code: "EUR")
    .locale(Locale(identifier: "de_DE"))
)
// "49,99 €"

let percent = 0.853
let formattedPercent = percent.formatted(.percent.precision(.fractionLength(1)))
// "85.3%"

let bytes = 1_536_000
let size = bytes.formatted(.byteCount(style: .file))
// "1.5 MB"
```

**Locale override:** Always pass `.locale()` to override the user's current locale. The default `.formatted()` call uses `Locale.current`, which respects the user's system preference. Do not create `DateFormatter` or `NumberFormatter` instances unless you need specific behavior not covered by `FormatStyle`.

**Custom `FormatStyle`:** Conform to `FormatStyle` for domain-specific types:

```swift
struct DistanceFormatStyle: FormatStyle {
    func format(_ value: Double) -> String {
        let measurement = Measurement(value: value, unit: UnitLength.meters)
        return measurement.formatted(
            .measurement(width: .abbreviated, usage: .road)
        )
    }
}

extension FormatStyle where Self == DistanceFormatStyle {
    static var distance: DistanceFormatStyle { DistanceFormatStyle() }
}

let d = 1500.0.formatted(.distance)
// "1.5 km" (or locale-appropriate unit)
```

## RTL Layout

Right-to-left layout (Arabic, Hebrew, Persian, etc.) requires correct use of `leading`/`trailing` alignment and `layoutDirection` environment.

**Leading > trailing:** SwiftUI automates this. Use `.leading` and `.trailing` alignment instead of `.left` and `.right`. Stacks, paddings, and frames with leading/trailing flip automatically.

```swift
HStack {
    Image(systemName: "chevron.right")  // ← Wrong: fixed direction
    Text("Next")
}

HStack {
    Image(systemName: "chevron.right")
        .flipsForRightToLeftLayoutDirection(true)  // Mirror the chevron in RTL
    Text("Next")
}
```

**`flipsForRightToLeftLayoutDirection(_:)`:** Explicitly mirror images or custom views in RTL. Default is `false` — most SF Symbols and SwiftUI views auto-adjust. Set to `true` for custom-drawn views or images that need horizontal mirroring.

**`layoutDirection` environment:**

```swift
// Read current direction
@Environment(\.layoutDirection) var layoutDirection

var isRTL: Bool { layoutDirection == .rightToLeft }

// Override for a subtree
MyView()
    .environment(\.layoutDirection, .rightToLeft)
```

**Known issue on macOS:** Setting `.environment(\.locale, .init(identifier: "ar"))` does NOT automatically flip layoutDirection on macOS. You must also set `.environment(\.layoutDirection, .rightToLeft)` explicitly. This is an macOS-specific behavior — iOS handles it automatically via `.locale`.

**Testing RTL in Xcode:**
1. Edit scheme → Run → Options → Application Language → "Right-to-Left Pseudolanguage"
2. Use `NSLocalizedString` pseudolanguage to double all strings (tests layout overflow)
3. Use `Double-Length Pseudolanguage` to replace all strings with doubled text

**Image direction:** Use `Image(systemName:)` with directional SF Symbols — they flip automatically. For custom assets, provide left/right variants or use `imageFlippedForRightToLeftLayoutDirection()` on `NSImage` in AppKit code.

## NSLocalizedString Migration

Migrate from `NSLocalizedString` to String Catalogs:

**Extraction:** Xcode automatically extracts `NSLocalizedString` calls into the string catalog at build time. Add the catalog to your target, build, and Xcode populates it. No manual migration needed for the basic case.

**`NSLocalizedString` compatibility:** `NSLocalizedString(key:comment:)` continues to work alongside String Catalogs. The catalog takes precedence; unmatched keys fall back to `.strings` files. This means you can migrate incrementally — the catalog doesn't replace the old system, it supplements it.

```swift
// Old (still works)
NSLocalizedString("save_button", comment: "Save button label")

// New (extracted to catalog automatically)
String(localized: "save_button", comment: "Save button label")

// SwiftUI (also extracted)
Text("save_button", comment: "Save button label")
```

**Migration workflow:**
1. Add a String Catalog to the target (File → New → String Catalog).
2. Build the project. Xcode auto-extracts `NSLocalizedString`, `String(localized:)`, and `Text()` strings.
3. Review the catalog in Xcode. Fix keys that were auto-generated (sometimes Xcode creates ugly keys from variable names).
4. Replace `NSLocalizedString` calls with `String(localized:)` or `Text()` over time. No need to remove all `NSLocalizedString` at once.
5. Remove old `.strings` files once all entries are in the catalog and translations are verified.

**Caveat:** Auto-extracted keys from `NSLocalizedString` are the raw string unless you use `genstrings` or provide explicit keys. If your code has `NSLocalizedString("Save", comment: "")`, the key is `"Save"` — which may not be ideal for translation context. Prefer explicit key/comment pairs.

## Menu Item Localization

Menu items in SwiftUI commands are localized through the string catalog like any other `Text` or `Button` label:

```swift
.commands {
    CommandMenu("File") {
        Button(action: save) {
            Text("Save", comment: "Save menu item")
        }
        .keyboardShortcut("s")
    }
}
```

The string "Save" is added to the string catalog automatically.

**For AppKit `NSMenu`:** Legacy `NSMenu` items defined in code (not storyboards) require either `.strings` files or manual localization:

```swift
let menuItem = NSMenuItem(
    title: NSLocalizedString("menu_export", comment: "Export menu item title"),
    action: #selector(exportFile),
    keyEquivalent: "e"
)
```

**Storyboard/XIB localization:** Xcode generates `.strings` files from storyboard localizations (e.g., `MainMenu.strings`). These coexist with String Catalogs. The storyboard `.strings` file takes precedence for items defined in the storyboard.

**`CFBundleName`:** For the app name in the menu bar (the first menu item), set `CFBundleName` in `Info.plist`. Localize it by adding `CFBundleName` to `InfoPlist.strings` in each `.lproj` directory:

```
// en.lproj/InfoPlist.strings
CFBundleName = "MyApp";

// fr.lproj/InfoPlist.strings
CFBundleName = "MonApp";
```

## Accessibility Label Localization

Accessibility labels follow the same patterns as visible text — they're just attached to the accessibility layer:

```swift
// SwiftUI
Button(action: save) {
    Image(systemName: "square.and.arrow.down")
}
.accessibilityLabel(Text("save_document", comment: "Accessibility label: save button"))
.accessibilityHint(Text("save_document_hint", comment: "Accessibility hint: saves the current file"))
```

**Dynamic announcement switching:** For status changes that VoiceOver should announce, post accessibility notifications:

```swift
// Announce a status change
NSAccessibilityPostNotification(
    NSApp.accessibilityFocusedUIElement,
    .init(rawValue: kUIAccessibilityAnnouncementRequestedNotification)
)

// In SwiftUI, use accessibility announcement
import SwiftUI

// For one-shot announcements:
Text("")
    .accessibilityAnnouncement("Document saved")

// Or post notification manually:
NSAccessibility.post(
    notification: .announcementRequested,
    userInfo: [
        .announcement: String(localized: "save_success"),
        .priority: NSAccessibilityPriorityLevel.high.rawValue
    ]
)
```

**Key principle:** Never hardcode accessibility strings. Every label and hint must be localized through the catalog, just like visible text. VoiceOver users in RTL locales will hear RTL string order, so plural rules and substitutions must work correctly in accessibility strings too.

## Xcode Workflow

**Export localization:** Xcode → Product → Export Localizations. Choose languages to export. Xcode creates an `.xcloc` directory per language containing `.xliff` files (standard XML localization interchange format).

**Import localization:** Xcode → Product → Import Localizations. Select the `.xcloc` or `.xliff` file. Xcode merges translations into the string catalog. New translations are marked `translated`; changed source strings mark existing translations as `stale`.

**Pseudolanguage testing (no translations needed):**
- **Double-Length Pseudolanguage:** Replaces every string with its double. Tests whether your layout handles overflow (German, Finnish are typically 30-40% longer than English).
- **Right-to-Left Pseudolanguage:** Flips layout direction without translating content. Tests RTL layout issues with readable text.
- **Accented Pseudolanguage:** Adds accents to all characters. Verifies font rendering.
- Set via: Edit Scheme → Run → Options → Application Language.

**Automanaged extraction:** Xcode's build-time string extraction means you never manually edit the `.xcstrings` JSON. The catalog is generated from source code. If you need to manually add/edit entries, use the Xcode editor — do not edit the JSON file.

## Constraints & Gotchas

- **String catalog merge conflicts:** The `.xcstrings` file is JSON. When two branches add entries to the same catalog, the merge conflict is in JSON — not human-readable. Xcode can resolve it visually (File → Source Control → Resolve Catalog Conflicts), but you cannot merge the raw JSON safely. Minimize parallel additions to the same catalog.

- **Bundle override priority:** String Catalog > `.strings` file > development language fallback. If you have both a catalog and a `.strings` file with the same key, the catalog wins. This can cause confusion during migration: old translations in `.strings` files are silently ignored once the key exists in the catalog.

- **RTL in SwiftUI — macOS-specific:** Setting `.environment(.locale, "ar")` does NOT auto-flip layoutDirection on macOS. Always set `.environment(.layoutDirection, .rightToLeft)` alongside the locale. iOS handles this automatically; macOS does not.

- **`Text("foo")` ambiguity:** Swift `Text` has overloads accepting `String`, `LocalizedStringKey`, and `AttributedString`. The compiler resolves `"Hello"` as `LocalizedStringKey` in most cases, but ambiguous contexts (e.g., passing a string variable) may not localize. Use `Text(LocalizedStringKey(explicitKey))` for dynamic keys, or `Text("key", comment: "")` for static keys.

- **Plural category availability:** Not all languages support all plural categories. Arabic supports `zero`, `one`, `two`, `few`, `many`, `other`. Japanese only uses `other`. If you define `one` for a Japanese localization, it's silently ignored. Define only the categories the target language needs.

- **FormatStyle thread safety:** `FormatStyle` instances are value types and thread-safe. Unlike `DateFormatter` (which is not thread-safe), you can freely share `FormatStyle` instances across threads.

- **Menu item `.strings` files:** Storyboard-generated menu items use `.strings` files, not the string catalog. If you add a menu item in code with `NSLocalizedString`, it uses `.strings` files. Only SwiftUI `Text` and `String(localized:)` use the catalog. This means a localized app may have both `.strings` files and a `.xcstrings` catalog.

## Common Mistakes

- **Hardcoding English strings as keys:** Using `Text("Save file")` makes "Save file" the key. If the English value changes, the key changes, breaking all translations. Use semantic keys: `Text("save_file_action", comment: "Save button label")`.

- **Using `.left`/`.right` instead of `.leading`/`.trailing`:** Fixed-direction alignment breaks RTL. Always use leading/trailing in SwiftUI. In AppKit, use `NSLayoutConstraint.Attribute.leading`/`.trailing`.

- **Forgetting `comment:` parameter:** Translators see the key and the source string but no context. A key like `"run"` could be a verb ("Run the app") or a noun ("a ski run"). Always add comments.

- **Testing only in English:** German strings are ~30% longer. Finnish, Russian, and Arabic have different plural rules. Test with pseudolanguages before shipping.

- **Editing `.xcstrings` JSON manually:** The catalog is auto-generated from source. Manual edits are overwritten by the next build. Use the Xcode editor or change the source code.

- **Mixing `NSLocalizedString` and `String(localized:)` for the same key:** Both work, but `NSLocalizedString` reads from `.strings` files while `String(localized:)` reads from the catalog. If the key exists in both, the catalog wins — but this creates confusion about which file is the source of truth.

- **Not localizing accessibility labels:** Every `.accessibilityLabel()` and `.accessibilityHint()` must go through the catalog. VoiceOver users need localized announcements.

- **Assuming `DateFormatter` is needed:** `DateFormatter` is legacy. Use `Date.FormatStyle` for new code. It's locale-aware, type-safe, and thread-safe.

## Review Checklist

- [ ] All user-facing strings use `Text("key", comment:)` or `String(localized:)` — no hardcoded display strings
- [ ] Every string has a translator comment explaining context
- [ ] Plural rules use `.stringsdict` or catalog variations — no string concatenation for plurals
- [ ] Layout uses `.leading`/`.trailing` — no `.left`/`.right` alignment
- [ ] RTL tested with pseudolanguage scheme option
- [ ] Accessibility labels and hints are localized
- [ ] `FormatStyle` used instead of `DateFormatter`/`NumberFormatter`
- [ ] `Text(verbatim:)` used for non-translatable strings (version numbers, identifiers)
- [ ] No manual edits to `.xcstrings` JSON
- [ ] Pseudolanguage testing done (double-length + RTL)
- [ ] Menu items localized (both SwiftUI commands and AppKit `NSMenu`)
- [ ] `CFBundleName` localized in `InfoPlist.strings` for app name

## References

- [references/localization-patterns.md](references/localization-patterns.md) — String Catalog with plurals/substitutions, FormatStyle deep-dive, RTL testing setup, `.stringsdict` menu localization, NSLocalizedString migration workflow
- [Apple: String Catalogs](https://developer.apple.com/documentation/xcode/string-catalogs)
- [Apple: Localizing strings that contain plurals](https://developer.apple.com/documentation/xcode/localizing-strings-that-contain-plurals)
- [Apple: FormatStyle](https://developer.apple.com/documentation/foundation/formatstyle)
- [Apple: LayoutDirection](https://developer.apple.com/documentation/swiftui/layoutdirection)
- [Apple: flipsForRightToLeftLayoutDirection](https://developer.apple.com/documentation/swiftui/view/flipsforrighttoleftlayoutdirection(_:))
- [WWDC22: Get it right (to left)](https://developer.apple.com/videos/play/wwdc2022/10107/)

## Research Addendum — 2026 Updates

### String Catalogs (Xcode 26)

- **Generated symbols** — Xcode 26 adds type-safe generated symbols from String Catalog entries. Instead of raw string keys, you get Swift symbols for each catalog entry, providing compile-time safety.
- **Xcode 26 namespace feature** — Provides a namespace separator for string catalog keys, enabling better organization in large catalogs.
- **Migration Assistant** — Xcode's built-in Migration Assistant helps migrate existing `.strings`/`.stringsdict` files to `.xcstrings` catalogs. Preserves coexistence during incremental migration.
- **Multiple catalogs** — You can have multiple `.xcstrings` files in one project. Specify which catalog to use via `tableName`/`table` parameters.
- **Vary by Device** — In the String Catalog editor, select a key and use "Vary by Device" to create macOS-specific strings (e.g., "Click to continue" on macOS vs. "Tap to continue" on iOS).

### SwiftUI RTL on macOS (2026)

- **Critical**: Setting `.environment(\.locale, Locale(identifier: "ar"))` does NOT auto-flip `layoutDirection` on macOS. You must also explicitly set `.environment(\.layoutDirection, .rightToLeft)`. This is an iOS/macOS divergence that persists.
- **Testing**: Use "Right-to-Left Pseudolanguage" in Xcode scheme options. Unlike iOS, this doesn't fully test macOS RTL behavior — you also need `-AppleLocale ar` launch argument or System Settings language switch.
- **`flipsForRightToLeftLayoutDirection(true)`** — SF Symbols auto-mirror, but custom images and programmatic drawings need explicit mirroring.
- **NSView-level RTL**: Use `NSApp.userInterfaceLayoutDirection` in AppKit code to detect RTL and adjust layout manually for custom views.

### NSLocalizedString → String Catalogs Migration (2026)

- **Auto-extraction** builds the catalog from source code. Don't manually edit the `.xcstrings` JSON — the rebuild will overwrite it.
- **Migration order**: Remove old `.strings`/`.stringsdict` files from `Base.lproj` (they should live in language-specific `.lproj` folders before migration). Then use Xcode's "Migrate to String Catalog" action.
- **State tracking**: The catalog tracks `new`, `needs review`, `translated`, and `stale` states. Use state filters in Xcode to find unreviewed strings before shipping.
- **Storyboard `.strings` files** coexist with String Catalogs. Storyboard strings take precedence for items defined in the storyboard. Code-defined strings use the catalog.

### NSMenu Localization (2026)

- **SwiftUI commands** localize via the string catalog automatically. `Text("Save", comment: "...")` is auto-extracted.
- **AppKit `NSMenu`** items defined in code use `NSLocalizedString` → `.strings` files. These are separate from the string catalog.
- **Storyboard-based menus** use generated `.strings` files (e.g., `MainMenu.strings`). These coexist with String Catalogs.
- **Best practice**: Migrate new menu items to SwiftUI commands where possible to unify on the string catalog. Keep legacy `NSMenu` items with `NSLocalizedString` and `.strings` files until you can migrate.
