---
name: swiftui-macos-accessibility
description: "Accessibility on macOS — VoiceOver (keyboard-driven, rotor differences from iOS), accessibility modifiers (.accessibilityLabel, .accessibilityValue, .accessibilityAction, .accessibilityChildren), AXCustomContent, full keyboard access, accessibilityFocused, Accessibility Inspector, Dynamic Type. Use when implementing or auditing accessibility for macOS SwiftUI apps."
sources: https://developer.apple.com/documentation/appkit/nsaccessibility, https://developer.apple.com/accessibility/macos/, https://developer.apple.com/videos/play/wwdc2025/229/, https://developer.apple.com/documentation/swiftui/view/accessibilityrotor(_:entries:entrylabel:)-2jbai, https://developer.apple.com/documentation/swiftui/view-accessibility, https://developer.apple.com/documentation/accessibility/creating_accessible_views, https://developer.apple.com/documentation/SwiftUI/Focus-Cookbook-sample, https://developer.apple.com/videos/play/wwdc2021/10119/, https://developer.apple.com/videos/play/wwdc2024/10073/, https://developer.apple.com/videos/play/wwdc2023/10036/, https://developer.apple.com/videos/play/wwdc2021/10023/, https://wesleydegroot.nl/blog/keyboard-navigation, https://wesleydegroot.nl/blog/voiceover
researched: 2026-06-06
platform: macOS 26+
---

# SwiftUI Accessibility on macOS

Accessibility on macOS centers on VoiceOver keyboard navigation, AXCustomContent for supplementary data, and full keyboard access patterns. Unlike iOS (touch-driven), macOS VoiceOver is entirely keyboard-driven — users navigate with arrow keys, Tab, and the rotor. This guide covers SwiftUI accessibility modifiers, custom rotor content, focus management, Accessibility Inspector auditing, and Dynamic Type, all with macOS 26+ Swift 6.3 patterns. For AppKit accessibility (`NSAccessibility` protocol), see `appkit-foundations`. For keyboard input and responder chain, see `macos-text-and-input`.

## Contents

- [macOS VoiceOver — Key Differences from iOS](#macos-voiceover--key-differences-from-ios)
- [Accessibility Modifiers](#accessibility-modifiers)
- [Accessibility Actions](#accessibility-actions)
- [Accessibility Children](#accessibility-children)
- [AXCustomContent](#axcustomcontent)
- [Full Keyboard Access](#full-keyboard-access)
- [accessibilityFocused](#accessibilityfocused)
- [Accessibility Inspector](#accessibility-inspector)
- [Dynamic Type](#dynamic-type)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## macOS VoiceOver — Key Differences from iOS

macOS VoiceOver is **keyboard-driven**. Users press VO (VoiceOver modifier, typically `Cmd+Option`) + Arrow Keys to navigate between elements. This is a fundamental departure from iOS where swipe gestures drive navigation.

**Key differences from iOS:**

| Behavior | iOS | macOS |
|----------|-----|-------|
| Navigation | Swipe left/right | VO + Arrow Keys |
| Rotor | Touch gesture on screen | VO + U (opens rotor window) |
| Quick Nav | Not available | Arrow keys navigate by headings, links, form controls |
| Interaction | Double-tap to activate | VO + Space to interact |
| Headings rotor | Yes | Yes (Quick Nav also supports `H` to jump headings) |

**Quick Nav mode:** Available only on macOS. When enabled, standard arrow keys (without VO modifier) navigate the accessibility tree by element type — `H` for headings, `L` for links, `T` for tables, `B` for buttons. This is the fastest way for power users to traverse complex UIs. Quick Nav can be toggled in VoiceOver Utility → Web → "Enable Quick Nav".

**Enabling VoiceOver testing:**

1. System Settings → Accessibility → VoiceOver → turn on, or press `Cmd+F5`.
2. In code, check `NSWorkspace.shared.isVoiceOverEnabled` to conditionally adapt behavior.
3. For automated tests, use `XCUISetters` or accessibility audit APIs.

**Testing workflow:**

```swift
// Detect VoiceOver at runtime
import AppKit

if NSWorkspace.shared.isVoiceOverEnabled {
    // Provide additional hints, simplify navigation, or adjust timers
    print("VoiceOver is active")
}
```

VoiceOver reads the accessibility tree it receives from `NSAccessibility`. SwiftUI auto-generates most of the tree, but custom elements need explicit labels, hints, and actions (covered below).

## Accessibility Modifiers

SwiftUI provides a family of modifiers that map to `NSAccessibilityAttribute` values. Always provide at minimum `.accessibilityLabel` for custom controls — VoiceOver reads nothing for unlabeled views.

**`.accessibilityLabel()`** — The primary description of the element. Keep it concise (1–3 words for buttons, short phrases for complex controls).

```swift
Button(action: export) {
    Image(systemName: "square.and.arrow.up")
}
.accessibilityLabel("Export document")
```

**`.accessibilityLabel()` with `Text` (2025):** When a `Button` uses a `Text` label, the text is automatically the accessibility label. Adding `.accessibilityLabel` overrides (not appends to) the text label — use this to provide a *different* label (e.g., shorter for screen readers). Do not set both visible text and a redundant label; pick one.
**`.accessibilityInputLabel()` (macOS 15+):** Provides a shorter label for input contexts (e.g., menus) while the full label is used in navigation. Use when the descriptive label is too long for repeated menu reading.
```swift
Toggle("Enable automatic backup of all documents", isOn: $enabled)
    .accessibilityInputLabel("Auto-backup")
```
**`.accessibilityValue()`** — The current value of the element (e.g., "50%" for a slider, "3 of 10" for a page indicator).

```swift
@State private var progress: Double = 0.5

HStack {
    ProgressView(value: progress)
    Text("\(Int(progress * 100))%")
        .accessibilityHidden(true) // label already covers this
}
.accessibilityElement(children: .ignore)
.accessibilityLabel("Upload progress")
.accessibilityValue("\(Int(progress * 100)) percent")
```

**`.accessibilityHint()`** — Additional context read after the label and value. Use sparingly — hints add verbosity. Prefer letting the action speak for itself.

```swift
.accessibilityLabel("Composer")
.accessibilityHint("Opens a new email composition window")
```

**`.accessibilityIdentifier()`** — Not read by VoiceOver but used by UI tests (`XCUIApplication`) to locate elements. Does not affect spoken output.

```swift
.accessibilityIdentifier("mainSendButton")
```

**`.accessibilityHidden()`** — Removes an element from the accessibility tree entirely. Use to hide decorative images, redundant labels (where the parent already provides context), and layout-only containers.

```swift
Image(systemName: "hexagon.fill")
    .accessibilityHidden(true) // decorative icon, parent has label
```

**Combining modifiers:** Attributes compose in this reading order: `label → value → hint → trait`. The system concatenates them. Don't duplicate information across label/value/hint.

## Accessibility Actions

VoiceOver calls "_actionable_" elements — those with associated actions. SwiftUI exposes actions through modifiers that map to `NSAccessibilityAction`.

**`.accessibilityAction(named:)`** — Adds a discoverable action. VO+Shift+Down interacts, VO+Shift+Up to exit. Users see the action name in the actions menu.

```swift
.accessibilityAction(named: "Preview") {
    showPreview = true
}
.accessibilityAction(named: "Share") {
    shareItem()
}
```

**`.accessibilityAction(_:handler:)`** — The newer API using `AccessibilityActionKind` for standard semantic actions (`.default`, `.delete`, `.escape`):

```swift
.accessibilityAction(.default) {
    // Equivalent to primary activation (VO+Space)
    activateItem()
}
.accessibilityAction(.delete) {
    removeItem()
}
```

**`.accessibilityCustomAction`** — Maps to `NSAccessibilityCustomAction` for custom rotor actions on macOS. These appear in the rotor and VO actions menu:

```swift
.accessibilityCustomAction(
    AccessibilityCustomAction(
        name: "Mark as read",
        perform: { markRead(); return true }
    )
)
.accessibilityActivationPoint()  -- Sets the point (in screen coords) where
                                   activating the action targets. Useful for
                                   large elements where VO should focus on
                                   a specific sub-region.
```

```swift
// Override activation point to target center of a custom view
.accessibilityActivationPoint(.init(x: 0.5, y: 0.5))
```

**Standard traits with built-in actions:** Buttons automatically get a "press" action. Toggles get "activate". You don't need to add `.accessibilityAction(.default)` for standard SwiftUI `Button` or `Toggle` — they already expose activation.

## Accessibility Children

By default, SwiftUI views expose their children as separate accessibility elements to VoiceOver. For composite controls (a custom card with internal labels), this leaks implementation.

**`.accessibilityElement(children:)`** controls child exposure:

```swift
// Flat: all recursive children become ONE element
HStack {
    Image(systemName: "envelope")
        .accessibilityHidden(true)
    Text("Inbox")
    Text("\(unreadCount)")
        .accessibilityLabel("\(unreadCount) unread")
}
.accessibilityElement(children: .combine)
.accessibilityLabel("Inbox, \(unreadCount) unread")
```

| Mode | Behavior |
|------|----------|
| `.combine` | Merges children into one element. Reads concatenated labels. |
| `.ignore` | Children disappear; the aggregate element provides its own label/value. |
| `.contain` | Children remain accessible but grouped under the parent (semantic group). |

**`.accessibilityChildren()`** (macOS 14+) — Replaces the child tree entirely. Use for complex custom controls where you need to inject or reorder child elements:

```swift
.accessibilityChildren {
    AccessibilityChild {
        Text("First name").accessibilityLabel("First name")
    }
    AccessibilityChild {
        Text("Last name").accessibilityLabel("Last name")
    }
}
```
**`onMoveCommand` for keyboard navigation (2025):** Handle arrow key navigation in custom layouts:
```swift
.onMoveCommand { direction in
    switch direction {
    case .up:    moveSelection(up: true)
    case .down:  moveSelection(up: false)
    case .left:  moveColumn(offset: -1)
    case .right: moveColumn(offset: 1)
    @unknown default: break
    }
}
```
Respect `layoutDirection` for RTL languages. Combine with `onKeyPress(.return)` for activation.

## AXCustomContent

`AXCustomCustomContent` (exposed via `AXCustomContentProvider`) is a macOS-specific mechanism for providing supplementary information that doesn't belong in the label/value flow. Users access it via the AXCustom Content rotor item — this is the macOS equivalent of `accessibilityCustomContent` on iOS.

**Implementation in SwiftUI:**

```swift
import SwiftUI

struct DocumentRow: View {
    let document: Document

    var body: some View {
        HStack {
            Image(systemName: document.icon)
            VStack(alignment: .leading) {
                Text(document.name)
                Text(document.dateModified, style: .relative)
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }
        }
        .accessibilityLabel(document.name)
        .accessibilityValue(document.dateModified.formatted())
        .accessibilityCustomContent("Size", document.fileSize)
        .accessibilityCustomContent("Kind", document.kind)
        .accessibilityCustomContent(
            "Created",
            document.dateCreated,
            importance: .high  // surfaces higher in rotor
        )
    }
}
```

**`accessibilityCustomContent` parameters:**

| Parameter | Purpose |
|-----------|---------|
| `label` | The content key (e.g., "Size", "Author") |
| `value` | The content value (any `String`, `Date`, `LocalizedStringKey`, etc.) |
| `importance` | `.default` or `.high`. High-importance items appear first in the rotor and are announced during navigation. |

**When to use AXCustomContent vs label/value:** Label and value are for the "headline" information. Use AXCustomContent for secondary metadata (file size, author, creation date) that a VoiceOver user would occasionally want but shouldn't hear on every navigation pass. High-importance content IS announced during normal navigation, making it ideal for critical metadata.

**Pattern:** For tables or lists where each row has multiple data columns, provide columns as custom content so rotor users can navigate the data meaningfully.

## Full Keyboard Access

Full Keyboard Access (System Settings → Keyboard → Full Keyboard Access) allows users to tab through all controls, not just text fields and lists. SwiftUI participates via the focus system.

**Making views keyboard-focusable:**

```swift
HStack {
    Button("OK") { dismiss() }
    Button("Cancel") { cancel() }
    Button("Apply") { apply() }
}
.focusable(true) // all three buttons participate in tab order
```

By default, SwiftUI `Button`, `Toggle`, `Picker`, `Slider`, and `TextField` are already focusable. Custom views (e.g., clickable `HStack`) need `.focusable(true)` added.
**Focus ring and `contentShape` (WWDC23, macOS):** Tie the focus ring to the view's shape via `.contentShape`:
```swift
Button(action: {}) {
    Label("Download", systemName: "arrow.down.circle")
}
.buttonStyle(.borderedProminent)
.contentShape(Capsule()) // Focus ring follows capsule shape
```
**`focusEffectDisabled` + `SelectionShapeStyle` (WWDC23):** For containers of selectable content, disable the default focus ring and use a selection-adaptive border:
```swift
VStack { /* selectable items */ }
.focusEffectDisabled()
.border(isSelected ? Color.accentColor : Color.clear)
```
**`FocusedValues` API (2025):** Expose selection bindings to main menu commands:
```swift
struct SelectionKey: FocusedValueKey {
    typealias Value = Binding<Set<Item.ID>>
}
// Child view sets the focused value
.focusedValue(\.selection, $selectedIDs)
// Menu command reads it
Command("Favorite") {
    if let selection = FocusedValues.shared[SelectionKey.self] {
        selection.wrappedValue.forEach { favorite($0) }
    }
}
```

**Custom tab order** — Use `.focusOrder` when default layout order doesn't match logical flow:

```swift
@FocusState private var focusedField: FormField?

VStack {
    TextField("Name", text: $name)
        .focused($focusedField, equals: .name)

    TextField("Email", text: $email)
        .focused($focusedField, equals: .email)
}
.focusOrder([.email, .name]) // tab goes email first, then name
```

**`.accessibilityRespondsToUserInteraction()`** — Explicitly marks a view as interactive in the accessibility tree. Important for custom elements where SwiftUI might not infer interactivity:

```swift
Canvas { context, size in
    // custom canvas drawing
}
.frame(width: 400, height: 300)
.accessibilityElement(children: .ignore)
.accessibilityLabel("Chart of weekly sales")
.accessibilityRespondsToUserInteraction(true)
.accessibilityAction(.default) {
    showChartDetail()
}
```

## accessibilityFocused

`accessibilityFocused` (available macOS 14+) creates a binding that tracks VoiceOver focus. Use it to observe or programmatically direct VoiceOver to specific elements — critical for announcing dynamic content changes.

```swift
struct AlertView: View {
    @AccessibilityFocusState private var isAlertFocused: Bool
    @State private var showAlert = false

    var body: some View {
        VStack {
            // ... main content

            if showAlert {
                Text("File saved successfully")
                    .padding()
                    .background(Color.green.opacity(0.2))
                    .cornerRadius(8)
                    .accessibilityFocused($isAlertFocused)
                    .accessibilityLabel("File saved successfully")
                    .accessibilityAddHint(.announcement)
            }
        }
        .onChange(of: showAlert) { _, showing in
            if showing {
                // Move VoiceOver focus to the announcement
                DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
                    isAlertFocused = true
                }
            }
        }
    }
}
```

**`@AccessibilityFocusState`** supports equality-based binding for multiple focus targets:

```swift
enum FocusTarget: Hashable {
    case playButton, stopButton, slider
}

@AccessibilityFocusState private var focused: FocusTarget?

// Direct VoiceOver to the slider
focused = .slider
```

**Announce without moving focus** — Use `AccessibilityNotification` for transient announcements:

```swift
// Announce a countdown or error without stealing focus
AccessibilityNotification.Announcement("Upload complete")
    .post( priority: .high )
```

## Accessibility Inspector

Xcode's Accessibility Inspector is the primary tool for auditing both AppKit and SwiftUI accessibility on macOS.

**Opening it:** Xcode → Open Developer Tool → Accessibility Inspector, or via Spotlight.

**Audit mode** — Click the Audit tab (stopwatch icon), then click "Run Audit" on the target app window. The inspector scans the full accessibility tree and reports:
- Missing labels
- Non-standard traits (e.g., interactive views without button trait)
- Insufficient contrast (when combined with the Display tab)
- Orphaned elements

**Attribute inspection** — Click the crosshair icon, then hover over any UI element. The inspector shows:
- Title (accessibilityLabel)
- Value (accessibilityValue)
- Traits (button, image, adjustable, etc.)
- Actions (all `NSAccessibilityAction` entries)
- Custom content (AXCustomContent entries)
- Frame and position

**macOS-specific inspector features:**
- Run audit on a remote simulator or local app simultaneously.
- Simulate VoiceOver navigation in the inspector to test custom rotor content.
- Inspector shows the live `NSAccessibility` tree — every SwiftUI element maps to an `NSAccessibilityElement` or `NSView/NSAccessibilityRepresentable`.

**Workflow:** Inspect → identify missing labels → add modifiers → re-audit → test with actual VoiceOver (`Cmd+F5`). The audit catches structural issues; real VoiceOver testing catches flow and navigation issues.

## Dynamic Type

macOS supports Dynamic Type (called "Display & Text Size" in System Settings). While macOS apps traditionally have smaller, denser layouts than iOS, supporting Dynamic Type respects user preferences.

**Text styles:**

```swift
// These scale with the user's "Text Size" setting in System Settings
Text("Body text").font(.body)
Text("Title").font(.title)
Text("Caption").font(.caption)
```

**`.dynamicTypeSize()`** — Sets a cap on Dynamic Type scaling:

```swift
// Allow scaling from small up to accessibility size 3
.dynamicTypeSize(...DynamicTypeSize.accessibility3)

// Or lock entirely
.dynamicTypeSize(.large) // always renders at "Large" size
```

**macOS delta from iOS:** Dynamic Type on macOS has a smaller range. The "Largest Accessibility Size" on macOS scales less aggressively than iOS. The system text size slider in System Settings → Display controls the base size, while accessibility sizes are in System Settings → Accessibility → Display.

**Testing:** System Settings → Accessibility → Display → Text Size, then drag the slider. The app should update live if using `.font(.body)` or similar semantic fonts.

**Do not hardcode point sizes for text that conveys information.** Use semantic styles (`.body`, `.headline`, `.caption`). Monospaced code can use fixed sizes (`.monospaced(.body)()` scales; `.monospacedSystemFont(ofSize: 13)` does not).

## Constraints & Gotchas

- **Rotor behavior differs from iOS.** The rotor on macOS opens as a popup window (VO+U), not an on-screen wheel. Custom rotor items come from `accessibilityCustomAction` and `AXCustomContent`, not from `accessibilityRotor` (which is an iOS-only API).
- **No `accessibilityScrollAction`** on macOS. Scrolling via VoiceOver uses standard scroll keys, not the iOS scroll action API.
- **No `accessibilityRepresentative`** on macOS — iOS's slot-fill approach for lists doesn't apply.
- **`NSAccessibility` protocol ≠ SwiftUI modifiers.** When wrapping `NSViewRepresentable`, AppKit's `NSAccessibility` protocol takes precedence over SwiftUI modifiers. Coordinators must set `accessibilityLabel`, `accessibilityRole`, etc. on the `NSView` directly for interop scenarios.
- **`accessibilityHidden` removes from tree visually** — double-check that hidden elements aren't actually needed. Use `.accessibilityElement(children: .combine)` instead of hiding children when they contribute to meaning.
- **VoiceOver on macOS reads `TextField` placeholders as the value** — not as a hint. Always set a proper label; placeholders vanish when the field has content.
- **Full Keyboard Access must be enabled separately** — not all users have it on. Keyboard-only users with Tab navigation need `.focusable()`.

## Common Mistakes

- **No label on icon buttons.** `Button { } label: { Image(systemName: "trash") }` is read as "button" by VoiceOver — users can't tell what it does. Always add `.accessibilityLabel("Delete")`.
- **Redundant label + visible text.** Don't set `.accessibilityLabel("Cancel")` on `Button("Cancel")` — the button text is already used as the label (unless you need to override it).
- **Custom toggle without `.accessibilityAddTraits(.isToggle)`** — VoiceOver won't announce the checked/unchecked state. Use `Toggle` when possible.
- **Using `.accessibilityLabel` instead of `.accessibilityIdentifier`** for test targeting — the identifier doesn't affect VoiceOver; the label does. Serve both audiences.
- **Overusing `.accessibilityHidden(true)`** — hiding too many elements makes navigation incomplete. Prefer `.combine` or `.contain` to restructure rather than remove.
- **Not providing AXCustomContent for complex rows** — table rows with multiple columns of data need custom content or rotor actions to be usable by VoiceOver.
- **Forgetting `.accessibilityRespondsToUserInteraction(true)`** on custom drawn elements — Canvas, Metal views, and custom gesture recognizers are invisible to VoiceOver without it.

## Review Checklist

- [ ] Every interactive element has a meaningful `.accessibilityLabel`
- [ ] Decorative images use `.accessibilityHidden(true)`
- [ ] Custom composite controls use `.accessibilityElement(children: .combine)` or `.accessibilityChildren`
- [ ] Dynamic text uses semantic fonts (`.body`, `.headheadline`, not hardcoded point sizes)
- [ ] App tested with VoiceOver (`Cmd+F5`) — navigation order is logical
- [ ] App tested with Full Keyboard Access — tab order covers all interactive elements
- [ ] Accessibility Inspector audit passes with zero warnings
- `accessibilityFocused` used for announcing state changes or errors to VoiceOver
- [ ] Complex row data provides `AXCustomContent` (important metadata with `.high` importance)
- [ ] `NSViewRepresentable` wrappers set `accessibilityLabel`/`accessibilityRole` on the AppKit view directly
**Custom rotors with `accessibilityRotor` (WWDC25, macOS 15+):** SwiftUI now supports custom rotors via `.accessibilityRotor(_:entries:entryLabel:)`. Create named rotors (e.g., "Bookmarks") that populate with `AccessibilityRotorEntry` items. Users access via VO+U. Entries' identifiers must align with items in your `ScrollView`/`ForEach`:
```swift
.accessibilityRotor("Bookmarks") {
    ForEach(pages) { page in
        if page.isBookmarked {
            AccessibilityRotorEntry(page.title, id: page.id)
        }
    }
}
```
**`accessibilityDefaultFocus` (WWDC25):** Set initial VoiceOver focus when a new scene appears:
```swift
.accessibilityDefaultFocus($focusedForVoiceOver, true)
```
Use this to direct focus to the most relevant element (e.g., first search result, primary content area) instead of the default top-left.
**`accessibilitySortPriority` (2025):** Adjust the navigation order independently of layout order. Higher values get visited first:
```swift
Header().accessibilitySortPriority(10)
Content().accessibilitySortPriority(5)
Sidebar().accessibilitySortPriority(1)
```

## References
- [references/accessibility-patterns.md](references/accessibility-patterns.md) — Deep-dive patterns
- [Make your Mac app more accessible - WWDC25](https://developer.apple.com/videos/play/wwdc2025/229/) — Apple WWDC (custom rotors, default focus, sort priority)
- [SwiftUI Accessibility: Beyond the basics - WWDC21](https://developer.apple.com/videos/play/wwdc2021/10119/) — Apple WWDC
- [Catch up on accessibility in SwiftUI - WWDC24](https://developer.apple.com/videos/play/wwdc2024/10073/) — Apple WWDC
- [Build accessible apps with SwiftUI - WWDC23](https://developer.apple.com/videos/play/wwdc2023/10036/) — Apple WWDC
- [Accessibility modifiers](https://developer.apple.com/documentation/swiftui/view-accessibility) — Apple Docs
- [Creating Accessible Views](https://developer.apple.com/documentation/accessibility/creating_accessible_views) — Apple Docs
- [Keyboard Navigation - Wesley de Groot](https://wesleydegroot.nl/blog/keyboard-navigation) — 2026 Guide
- [VoiceOver - Wesley de Groot](https://wesleydegroot.nl/blog/voiceover) — 2026 Guide
- [Focus Cookbook](https://developer.apple.com/documentation/SwiftUI/Focus-Cookbook-sample) — Apple Docs
- [Direct and reflect focus in SwiftUI - WWDC21](https://developer.apple.com/videos/play/wwdc2021/10023/) — Apple WWDC
- [accessibilityRotor](https://developer.apple.com/documentation/swiftui/view/accessibilityrotor(_:entries:entrylabel:)-2jbai) — Apple Docs
---
