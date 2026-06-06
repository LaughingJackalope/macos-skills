# Localization Patterns

Deep-dive patterns for macOS localization: String Catalog with plurals and substitutions, FormatStyle for locale-aware display, RTL layout testing, menu item localization with `.stringsdict`, and NSLocalizedString migration workflow.

## String Catalog with Plurals and Substitutions

### Basic Plural Variation in String Catalog

In Xcode's String Catalog editor, select a key and click "Vary" → "Plural Rules." The editor generates entries for each plural category. Here's the equivalent JSON structure inside the `.xcstrings` file:

```json
{
  "sourceLanguage" : "en",
  "strings" : {
    "item_count" : {
      "localizations" : {
        "en" : {
          "variations" : {
            "plural" : {
              "zero" : {
                "stringUnit" : {
                  "state" : "translated",
                  "value" : "No items"
                }
              },
              "one" : {
                "stringUnit" : {
                  "state" : "translated",
                  "value" : "%d item"
                }
              },
              "other" : {
                "stringUnit" : {
                  "state" : "translated",
                  "value" : "%d items"
                }
              }
            }
          }
        },
        "ar" : {
          "variations" : {
            "plural" : {
              "zero" : { "stringUnit" : { "state" : "translated", "value" : "لا عناصر" } },
              "one" : { "stringUnit" : { "state" : "translated", "value" : "عنصر واحد" } },
              "two" : { "stringUnit" : { "state" : "translated", "value" : "عنصران" } },
              "few" : { "stringUnit" : { "state" : "translated", "value" : "%d عناصر" } },
              "many" : { "stringUnit" : { "state" : "translated", "value" : "%d عنصرًا" } },
              "other" : { "stringUnit" : { "state" : "translated", "value" : "%d عنصر" } }
            }
          }
        }
      }
    }
  }
}
```

**Usage in SwiftUI:**

```swift
// The catalog handles plural selection automatically based on the count
Text("item_count \(count)")
// count = 0 → "No items" (en) / "لا عناصر" (ar)
// count = 1 → "1 item" (en) / "عنصر واحد" (ar)
// count = 5 → "5 items" (en) / "5 عناصر" (ar)
```

### Substitutions with Named Placeholders

String Catalogs support named substitutions that are more readable for translators:

```swift
// Source code
Text("welcome_message \(userName)")
// Catalog key: "welcome_message" with value "Welcome, %@!"

// For multiple substitutions
Text("progress_of_total \(completed) \(total)")
// Catalog value: "%lld of %lld tasks completed"
```

**Type-aware substitutions:**

| Swift type | Format specifier | Example |
|---|---|---|
| `String` | `%@` | `Text("hello \(name)")` |
| `Int` | `%lld` | `Text("count \(n)")` |
| `Double` | `%f` | `Text("price \(price)")` |
| `Date` (via FormatStyle) | Use `Text(date, format:)` | `Text(date, format: .dateTime.day().month())` |

### Developer Comments

Always add comments. In the catalog JSON, comments appear in the `comment` field:

```swift
// Good: translator sees "Button label that saves the current document to disk"
Text("save_button", comment: "Button label that saves the current document to disk")

// Bad: translator sees only "save_button" — is this a verb? a noun? what context?
Text("save_button")
```

Comments are exported in `.xliff` files and appear in translation tools like Lokalise, Crowdin, and Localizely.

## FormatStyle for Locale-Aware Date/Number Display

### Date FormatStyle Patterns

```swift
import Foundation

// Preset styles (respects user's locale)
Date.now.formatted(date: .abbreviated, time: .shortened)
// en_US: "Jun 6, 2026 at 3:30 PM"
// fr_FR: "6 juin 2026 à 15:30"
// ja_JP: "2026/06/06 15:30"

// Custom format
Date.now.formatted(.dateTime
    .year()
    .month(.wide)
    .day()
    .hour(.defaultDigits(amPM: .wide))
    .minute(.twoDigits)
)
// en_US: "June 6, 2026 at 03:30 PM"

// Relative formatting
Date.now.addingTimeInterval(3600).formatted(.relative(presentation: .named))
// "in 1 hour" (locale-aware)

// ISO 8601 (fixed format, not locale-dependent)
Date.now.formatted(.iso8601.year().month().day())
// "2026-06-06"
```

### Number FormatStyle Patterns

```swift
// Currency (locale-aware symbol placement)
let price = 1234.56
price.formatted(.currency(code: "USD").locale(Locale(identifier: "en_US")))
// "$1,234.56"
price.formatted(.currency(code: "EUR").locale(Locale(identifier: "de_DE")))
// "1.234,56 €"
price.formatted(.currency(code: "JPY").locale(Locale(identifier: "ja_JP")))
// "¥1,235" (no decimal for JPY)

// Percentage
0.853.formatted(.percent.precision(.fractionLength(1)))
// "85.3%" (en_US) / "85,3 %" (fr_FR)

// Byte count
2_147_483_648.formatted(.byteCount(style: .file))
// "2.1 GB" (en_US) / "2,1 Go" (fr_FR)

// Scientific notation
6.022e23.formatted(.number.notation(.scientific))
// "6.022E23"

// Significant digits
3.14159.formatted(.number.precision(.significantDigits(3)))
// "3.14"
```

### Formatting Dates in SwiftUI Text

SwiftUI's `Text` has a special initializer for formatted dates:

```swift
// Uses FormatStyle internally
Text("Last saved: \(lastSavedDate, format: .dateTime.day().month().year())")

// Relative
Text("Updated \(lastModified, format: .relative(presentation: .named))")

// Timer style
Text(eventDate, format: .timer(countingDownIn: .init(second: 0)))
// "2:35" (countdown)
```

## RTL Layout Testing Setup

### Scheme Configuration

1. In Xcode, select the scheme → Edit Scheme → Run → Options.
2. Set **Application Language** to "Right-to-Left Pseudolanguage."
3. Set **Application Region** to a region that uses an RTL language (e.g., Saudi Arabia).
4. Run the app. All horizontal layouts flip, but text remains readable English.

### Programmatic RTL Override

For testing in code or SwiftUI previews:

```swift
struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
            .environment(\.layoutDirection, .rightToLeft)
            .environment(\.locale, Locale(identifier: "ar"))
            .previewDisplayName("RTL Arabic")
    }
}
```

**Important:** On macOS, setting `.locale` alone does NOT flip layout direction. You must set both `.layoutDirection` and `.locale`. This differs from iOS where `.locale` alone is sufficient.

### Checking RTL in Code

```swift
struct AdaptiveView: View {
    @Environment(\.layoutDirection) var direction

    var body: some View {
        HStack {
            Image(systemName: "arrow.right")
                .flipsForRightToLeftLayoutDirection(true)

            Text("Next")

            Spacer()

            if direction == .rightToLeft {
                Text("RTL active")
                    .foregroundColor(.secondary)
            }
        }
    }
}
```

### Common RTL Issues on macOS

| Issue | Symptom | Fix |
|---|---|---|
| Fixed `.left`/`.right` alignment | Content stays left-aligned in RTL | Replace with `.leading`/`.trailing` |
| Custom images don't flip | Arrows point wrong direction | Add `.flipsForRightToLeftLayoutDirection(true)` |
| `NSImage` in AppKit | Image not mirrored | Use `imageFlippedForRightToLeftLayoutDirection()` |
| Padding with fixed insets | Asymmetric layout in RTL | Use `.padding(.leading, 10)` not `.padding(.leading, 10).padding(.trailing, 20)` asymmetrically |
| Text truncation | Long translated strings truncated | Test with Double-Length Pseudolanguage |

## Menu Item Localization with .stringsdict

### Localizing NSMenu Titles in AppKit

For AppKit apps using `NSMenu` programmatically (not via storyboard), use `NSLocalizedString` with `.stringsdict` for plural-aware menu items:

```swift
// Localizable.strings
"menu_items_selected" = "%d items selected";

// Localizable.stringsdict
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>menu_items_selected</key>
    <dict>
        <key>NSStringLocalizedFormatKey</key>
        <string>%#@count@</string>
        <key>count</key>
        <dict>
            <key>NSStringFormatSpecTypeKey</key>
            <string>NSStringPluralRuleType</string>
            <key>NSStringFormatValueTypeKey</key>
            <string>d</string>
            <key>one</key>
            <string>%d item selected</string>
            <key>other</key>
            <string>%d items selected</string>
        </dict>
    </dict>
</dict>
</plist>
```

```swift
// In the menu update code:
func updateSelectionMenu() {
    let count = selectedItems.count
    let title = String.localizedStringWithFormat(
        NSLocalizedString("menu_items_selected",
                          comment: "Menu title showing selection count"),
        count
    )
    selectionMenuItem.title = title
}
```

### SwiftUI Command Menu Localization

SwiftUI command menus use the string catalog automatically:

```swift
.commands {
    CommandMenu("Selection") {
        Button {
            deleteSelected()
        } label: {
            // This string is auto-extracted to the catalog
            Text("delete_selected_items \(selectedCount)",
                 comment: "Menu item to delete selected items")
        }
        .keyboardShortcut(.delete, modifiers: .command)
        .disabled(selectedCount == 0)
    }
}
```

### InfoPlist.strings for App Name

The app name in the menu bar comes from `CFBundleName`. Localize it:

```
// en.lproj/InfoPlist.strings
"CFBundleName" = "MyApp";

// fr.lproj/InfoPlist.strings
"CFBundleName" = "MonApp";

// ar.lproj/InfoPlist.strings
"CFBundleName" = "تطبيقي";
```

## NSLocalizedString Migration Workflow

### Phase 1: Add String Catalog

1. In Xcode, File → New → File → String Catalog. Name it `Localizable.xcstrings` (default).
2. Ensure it's added to the app target (and any framework targets that have their own strings).
3. Build the project. Xcode scans source code and populates the catalog.

### Phase 2: Verify Extraction

After building, open the catalog in Xcode. Verify:
- All `NSLocalizedString` calls appear as entries.
- Keys are meaningful (not raw English strings unless intentional).
- Comments are present for ambiguous keys.
- No duplicate keys from different source files.

### Phase 3: Add Translations

1. Select the catalog → click "+" at the bottom → add a language.
2. For each entry, fill in the translation or use Xcode's machine translation (Xcode 15+).
3. Mark entries as `translated` when reviewed.

### Phase 4: Replace NSLocalizedString Calls (Incremental)

Replace calls over time — no need for a big-bang migration:

```swift
// Before:
let title = NSLocalizedString("doc_title", comment: "Document window title")

// After (Foundation):
let title = String(localized: "doc_title", comment: "Document window title")

// After (SwiftUI):
Text("doc_title", comment: "Document window title")
```

### Phase 5: Remove Legacy .strings Files

Once all entries are in the catalog and verified:
1. Delete `Localizable.strings` and `Localizable.stringsdict` from the project.
2. Delete language-specific `.lproj/Localizable.strings` directories.
3. Build and test — the catalog is the sole source of truth.

### Migration Gotchas

- **Keys with format specifiers:** `NSLocalizedString("items_count", value: "%d items", comment: "")` — the `value` parameter is the default. In the catalog, the source language value becomes the default. Ensure the format specifiers match.
- **Table parameter:** `NSLocalizedString("key", tableName: "MainWindow", ...)` reads from `MainWindow.strings`. The String Catalog ignores the table parameter — all entries go into one catalog. If you need per-module separation, use multiple catalogs (one per module/framework).
- **Bundle parameter:** Framework strings need their own catalog in the framework target. `String(localized:)` with `bundle: .module` reads from the framework's catalog.
- **Stale entries:** After migration, old `.strings` files may still contain entries not in the catalog. Xcode marks these as "not in catalog" — review and either add them to the catalog or remove them.
