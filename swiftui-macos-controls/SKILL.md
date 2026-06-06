---
name: swiftui-macos-controls
description: "macOS-specific SwiftUI control behaviors — Button styles (bordered, borderedProminent, plain, link, accessoryBar), Picker styles (segmented, menu, radioGroup, inline), Toggle styles, Slider/Stepper/ColorPicker/FontPicker, PathControl, LevelIndicator, ControlGroup, controlSize (.mini/.small/.regular/.large). Use when implementing macOS-specific control UI in SwiftUI."
sources: https://developer.apple.com/documentation/swiftui/button, https://developer.apple.com/documentation/swiftui/picker, https://developer.apple.com/documentation/swiftui/controlgroup, https://developer.apple.com/documentation/swiftui/controlsize(_:), https://developer.apple.com/documentation/appkit/nspathcontrol, https://developer.apple.com/documentation/appkit/nslevelindicator, https://serialcoder.dev/text-tutorials/macos-tutorials/flavors-of-swiftui-picker-on-macos/, https://exploreswiftui.com/library/picker/radiogrouppickerstyle
researched: 2026-06-06
platform: macOS 26+
---

# SwiftUI macOS Controls

macOS-specific SwiftUI control behaviors, style mappings, and AppKit wrapping patterns. Covers every native SwiftUI control that renders differently on macOS versus iOS, plus AppKit-only surfaces that require `NSViewRepresentable`. Targets macOS 26+ with Swift 6.3. See [references/control-patterns.md](references/control-patterns.md) for deep-dive recipes with full code.

## Contents

- [Button Styles](#button-styles)
- [Picker Styles](#picker-styles)
- [Toggle Styles](#toggle-styles)
- [Slider & Stepper](#slider--stepper)
- [ColorPicker & FontPicker](#colorpicker--fontpicker)
- [PathControl & LevelIndicator](#pathcontrol--levelindicator)
- [ControlGroup](#controlgroup)
- [Control Sizing](#control-sizing)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Button Styles

SwiftUI buttons adapt to macOS with a distinct set of styles. The default `.buttonStyle(.bordered)` renders as a standard push button with a bezel. Use `.buttonStyle(.borderedProminent)` for the default action — it renders as a filled blue button that pulses gently to draw attention. The `.plain` style renders as a borderless text button, useful in table rows and dense layouts. The `.link` style renders as an inline hyperlink; it does not look like a button at all.

```swift
VStack(spacing: 12) {
    Button("Save") { save() }
        .buttonStyle(.borderedProminent)
    Button("Cancel") { cancel() }
        .buttonStyle(.bordered)
    Button("Learn more") { showInfo() }
        .buttonStyle(.link)
    Button("Delete") { delete() }
        .buttonStyle(.bordered)
        .tint(.red)
}
```

### ButtonRole

Set `Button.role(.destructive)` or `Button.role(.cancel)` to signal intent to AppKit. `.destructive` buttons render with reduced prominence and are positioned after affirmative buttons in alerts and confirmation dialogs. `.cancel` buttons receive standard cancellation styling in sheets and dialogs. The role does not change the visual style of standalone buttons in forms — it only affects system-presented contexts like confirmation dialogs and alerts.

```swift
Button("Delete…", role: .destructive) { confirmDelete() }
Button("Cancel", role: .cancel) { dismiss() }
```

### Custom ButtonStyle

For reusable button appearances, conform to `ButtonStyle`. The protocol requires a single method `makeBody(configuration:)` that receives a `Configuration` with `label` (the button content), `isPressed`, and the environment:

```swift
struct AccessoryBarButtonStyle: ButtonStyle {
    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .font(.caption2)
            .padding(.horizontal, 8)
            .padding(.vertical, 3)
            .background(
                RoundedRectangle(cornerRadius: 4)
                    .fill(configuration.isPressed
                          ? Color(nsColor: .selectedControlColor)
                          : Color(nsColor: .controlColor))
            )
            .foregroundStyle(.secondary)
    }
}
```

See [references/control-patterns.md](references/control-patterns.md) for a complete `accessoryBar` button style recipe.

## Picker Styles

SwiftUI `Picker` on macOS supports several render modes, each suited to different contexts.

### segmented

Renders as an `NSSegmentedControl` — a horizontal row of tabs. Best for 2–5 mutually exclusive options. Segmented pickers do not truncate labels; they widen to fit content, so keep labels short.

```swift
Picker("View Mode", selection: $viewMode) {
    Text("List").tag(ViewMode.list)
    Text("Grid").tag(ViewMode.grid)
    Text("Gallery").tag(ViewMode.gallery)
}
.pickerStyle(.segmented)
```

### menu

Renders as a dropdown `NSPopUpButton`. Best for longer option lists where space is constrained. The selected item is always visible in the collapsed state.

```swift
Picker("Sort By", selection: $sortOrder) {
    Text("Name").tag(SortOrder.name)
    Text("Date Modified").tag(SortOrder.dateModified)
    Text("Size").tag(SortOrder.size)
    Text("Kind").tag(SortOrder.kind)
}
.pickerStyle(.menu)
```

### radioGroup

Renders as a vertical stack of `NSRadioButton` items. Unlike segmented (which is inline), radio groups take vertical space. Available on macOS 10.15+. Use in form contexts and sidebar settings panels.

```swift
Picker("Theme", selection: $theme) {
    Text("System").tag(Theme.system)
    Text("Light").tag(Theme.light)
    Text("Dark").tag(Theme.dark)
}
.pickerStyle(.radioGroup)
```

**Horizontal layout:** Apply `.horizontalRadioGroupLayout()` to render radio buttons horizontally instead of vertically. Useful in toolbars and compact forms. Set an explicit width to avoid truncation:

```swift
Picker("Alignment", selection: $alignment) {
    Text("Left").tag(Alignment.left)
    Text("Center").tag(Alignment.center)
    Text("Right").tag(Alignment.right)
}
.pickerStyle(.radioGroup)
.horizontalRadioGroupLayout()
.frame(width: 200)
```
### inline

Renders each option as a capsule-shaped toggle pill. Selected option shows with accent color fill. Works well when options are short strings and horizontal space permits.

```swift
Picker("Priority", selection: $priority) {
    Text("Low").tag(Priority.low)
    Text("Normal").tag(Priority.normal)
    Text("High").tag(Priority.high)
}
.pickerStyle(.inline)
```

### wheel

**Not available on macOS.** `.wheel` picker style is iOS/iPadOS only. On macOS it falls back to `.menu`. Never rely on wheel behavior for macOS targets — use `.menu` or `.radioGroup` instead.

## Toggle Styles

### .switch

The default on macOS. Renders as `NSSwitch` — a small rectangular checkbox in macOS 13+ (previously the round iOS-style switch). Use for boolean settings in preferences and configuration panels.

```swift
Toggle("Enable notifications", isOn: $notificationsEnabled)
    .toggleStyle(.switch)
```

### .checkbox

Renders as a standard macOS checkbox with a checkmark. Semantically identical to `.switch` on macOS — both bind to a `Bool`. The visual distinction is subtle: checkbox is a square with a checkmark, switch is a rounded pill with a knob.

```swift
Toggle("Include attachments", isOn: $includeAttachments)
    .toggleStyle(.checkbox)
```

Both styles are functionally interchangeable on macOS; choose based on visual consistency with surrounding AppKit surfaces.

## Slider & Stepper

### Slider

`Slider` on macOS renders as `NSSlider` — a thin horizontal track with a circular thumb. Always provide a label via the `value` + `label` initializer. The label is required for accessibility and determines what VoiceOver reads.

```swift
Slider(value: $volume, in: 0...1) {
    Text("Volume")
} minimumValueLabel: {
    Image(systemName: "speaker")
} maximumValueLabel: {
    Image(systemName: "speaker.wave.3")
}
.sliderStyle(.linear) // Default on macOS
```

### Stepper

Renders as `NSStepper` with a text field and increment/decrement buttons. Always set bounds to prevent unbounded input.

```swift
Stepper("Count: \(count)", value: $count, in: 0...99)
    .stepperStyle(.stepper) // Default, required to override if previously changed
```

### controlSize

Apply `.controlSize()` to both `Slider` and `Stepper` to scale them:

```swift
Slider(value: $value, in: 0...100) { Text("Value") }
    .controlSize(.mini)   // 13px height
Stepper("Count: \(count)", value: $count, in: 0...99)
    .controlSize(.small)  // 16px height
```

Sizes: `.mini`, `.small`, `.regular` (default, 20px), `.large` (24px).

## ColorPicker & FontPicker

### ColorPicker

Renders as a color swatch button that opens `NSColorPicker` on click. macOS supports all three style modifiers:

```swift
ColorPicker("Accent Color", selection: $accentColor)
    .labelsHidden()

// With opacity channel:
ColorPicker("Background", selection: $bgColor, supportsOpacity: true)
```

The color swatch adapts to the current `controlSize`. At `.mini` it renders a tiny square; at `.large` it shows a wider rectangle with the color name.

### FontPicker

**macOS-only.** `FontPicker` has no iOS equivalent. It renders a button that opens the system font panel (`NSFontPanel`). Bind to a `Font` or `CGFloat` (for size-only pickers):

```swift
FontPicker("Body Font", selection: $bodyFont)
```

See [references/control-patterns.md](references/control-patterns.md) for a full font binding recipe with live preview.

## PathControl & LevelIndicator

### PathControl

`NSPathControl` has no SwiftUI equivalent. Wrap it in `NSViewRepresentable` for a clickable file path display — clicking a path component navigates to that directory or opens it in Finder. Supports drag-and-drop natively.

Three path styles: `.standard` (full clickable path), `.popUp` (dropdown selection), `.bar` (compact single-component bar).

### LevelIndicator

`NSLevelIndicator` also requires wrapping. Use it for star ratings, storage gauges, or capacity meters. Two modes:

- **Discrete**: segmented fill (1–5 stars rating)
- **Continuous**: bar fill (disk space, battery level)

See [references/control-patterns.md](references/control-patterns.md) for complete `NSViewRepresentable` wrappers for both.

## ControlGroup

`ControlGroup` groups multiple controls into a single visual unit. On macOS it is essential for toolbar construction.

### .menu style

Renders as a single button that opens a popover menu. All child controls are invisible when collapsed. Best for toolbar overflow or secondary actions.

```swift
ControlGroup {
    Button("New", systemImage: "plus") { newItem() }
    Button("Delete", systemImage: "trash") { deleteItem() }
    Divider()
    Button("Share", systemImage: "square.and.arrow.up") { share() }
}
.label("Actions")
.controlGroupStyle(.menu)
```

### .palette style

Renders each control as an individually visible element in a horizontal row, similar to a segmented toolbar group. This is the primary style for custom toolbars.

```swift
ControlGroup {
    Button { bold() }
        .label(Text("Bold"))
    Button { italic() }
        .label(Text("Italic"))
    Button { underline() }
        .label(Text("Underline"))
}
.label("Text Styles")
.controlGroupStyle(.palette)
```

On macOS, `ControlGroup` is most commonly used with `.controlGroupStyle(.palette)` for toolbar button groups and `.controlGroupStyle(.menu)` for overflow menus. See [references/control-patterns.md](references/control-patterns.md) for a complete toolbar palette recipe.

## Control Sizing

The `.controlSize()` modifier scales controls uniformly across all supported types:

| Size    | Control height | Use case                          |
|---------|---------------|-----------------------------------|
| `.mini`    | ~13px         | Compact toolbars, status bars     |
| `.small`   | ~16px         | Sidebar settings, secondary UI    |
| `.regular` | ~20px         | Default, forms, standard layout   |
| `.large`   | ~24px         | Touch Bar, prominent settings     |

Applied consistently:

```swift
VStack {
    HStack(spacing: 4) {
        Button("Add") { add() }.controlSize(.mini)
        Button("Add") { add() }.controlSize(.small)
        Button("Add") { add() } // .regular default
        Button("Add") { add() }.controlSize(.large)
    }
    Slider(value: $val, in: 0...1) { Text("Val") }
        .controlSize(.small)
    Picker("", selection: $opt) {
        Text("A").tag(1); Text("B").tag(2)
    }
    .pickerStyle(.segmented)
    .controlSize(.small)
}
```

macOS automatically respects the system's accessibility settings. If the user increases text size in System Settings, controls scale accordingly regardless of the `.controlSize` modifier. Always test with increased text sizes.

## Constraints & Gotchas

- **Wheel picker absent.** `.wheel` is iOS/iPadOS only. On macOS it silently falls back to `.menu`. Never write `..pickerStyle(.wheel)` for macOS targets.
- **FontPicker is macOS-only.** There is no iOS equivalent. If your project is multiplatform, wrap `FontPicker` in `#if os(macOS)`.
- **NSViewRepresentable sizing.** Wrapped AppKit views do not automatically size to their SwiftUI content. Set `intrinsicContentSize` on the AppKit view or add explicit `frame(height:)` constraints in SwiftUI. Without this, the view may collapse to zero height in a `VStack` or `List`.
- **Segmented picker truncation.** `.segmented` pickers do not truncate labels — they widen to fit. Long labels push adjacent views off-screen. Keep segment labels to 1–2 words maximum.
- **RadioGroup vertical space.** `..pickerStyle(.radioGroup)` always renders vertically. In a `Form`, this adds significant height. Use `.segmented` or `.menu` for space-constrained contexts.
- **ControlGroup nesting.** Nesting `ControlGroup` inside another `ControlGroup` is undefined behavior on macOS — inner groups may not render. Use flat structures with `Divider()` for visual separation.
- **ColorPicker opacity channel.** `supportsOpacity: true` adds an alpha slider in the color panel. If your color model doesn't carry alpha, strip it on read with `.opacity(1)`.
- **Toggle style identity.** `.switch` and `.checkbox` are identical on SwiftUI — both bind to `Bool`. The visual difference is cosmetic. Don't mix both styles form-wide; pick one for consistency.

## Common Mistakes

1. **Using `.automatic` picker style expecting segmented.** `.automatic` resolves differently across contexts — it may be `.menu` in a sidebar or `.inline` in a toolbar. Be explicit about the style you need.

2. **Omitting Slider labels.** `Slider(value:in:)` without a label parameter is deprecated. Always use the labeled form.

3. **Forgetting FontPicker platform guard.** `FontPicker` compiles on macOS only. Multiplatform targets crash at build time without `#if os(macOS)`.

4. **Setting frame on NSViewRepresentable without considering Auto Layout.** If the wrapped view uses Auto Layout (e.g., `NSPathControl`), SwiftUI's `frame(height:)` may conflict. Set `translatesAutoresizingMaskIntoConstraints = false` on the AppKit view and provide explicit sizing constraints.

5. **Over-nesting ControlGroup.** Each `ControlGroup` introduces layout overhead. Prefer a flat `HStack` with `Divider()` for simple separator lines between control clusters.

6. **Using `.link` button style for primary actions.** `.link` renders as inline hyperlink text — users won't recognize it as a button. Reserve for "learn more" or external navigation.

7. **Not testing `.large` control size on non-retina displays.** `.large` controls (24px) render at 48pt on 2x displays. This is fine for Touch Bar and prominent settings but may feel oversized in compact lists.

## Review Checklist

- [ ] All pickers use an explicit `.pickerStyle()` — no `.automatic` ambiguity
- [ ] FontPicker wrapped in `#if os(macOS)` for multiplatform targets
- [ ] No `.wheel` picker style reference anywhere
- [ ] Slider labels always provided via the labeled initializer
- [ ] NSViewRepresentable wrappers tested for zero-height collapse
- [ ] ControlGroup styles chosen for context (`.palette` for toolbar, `.menu` for overflow)
- [ ] `.controlSize()` applied consistently when sizing variants are used
- [ ] Button roles (`.destructive`, `.cancel`) set where system alerts may present them
## References

- [Button — Apple Documentation](https://developer.apple.com/documentation/swiftui/button)
- [Picker — Apple Documentation](https://developer.apple.com/documentation/swiftui/picker)
- [ControlGroup — Apple Documentation](https://developer.apple.com/documentation/swiftui/controlgroup)
- [controlSize — Apple Documentation](https://developer.apple.com/documentation/swiftui/view/controlsize(_:))
- [NSPathControl — Apple Documentation](https://developer.apple.com/documentation/appkit/nspathcontrol)
- [NSLevelIndicator — Apple Documentation](https://developer.apple.com/documentation/appkit/nslevelindicator)
- [RadioGroupPickerStyle — Explore SwiftUI](https://exploreswiftui.com/library/picker/radiogrouppickerstyle)
- [Flavors of SwiftUI Picker on macOS — SerialCoder.dev](https://serialcoder.dev/text-tutorials/macos-tutorials/flavors-of-swiftui-picker-on-macos/)
- [NSViewRepresentable Recipes — nsviewrepresentable-recipes.md](references/nsviewrepresentable-recipes.md)
- [Control Patterns Deep-Dive — control-patterns.md](references/control-patterns.md)
- [NSViewRepresentable Recipes — nsviewrepresentable-recipes.md](references/nsviewrepresentable-recipes.md)
- [Control Patterns Deep-Dive — control-patterns.md](references/control-patterns.md)
