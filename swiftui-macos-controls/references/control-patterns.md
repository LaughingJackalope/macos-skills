# Control Patterns — Deep-Dive Recipes

Complete working recipes for macOS-specific SwiftUI control implementations. Each section covers a pattern with full code, usage example, and gotchas.

---

## Contents

- [1. Custom ButtonStyle for Accessory Bar Button](#1-custom-buttonstyle-for-accessory-bar-button)
- [2. Picker with .radioGroup in a Form](#2-picker-with-radiogroup-in-a-form)
- [3. NSPathControl NSViewRepresentable Wrapper](#3-nspathcontrol-nsviewrepresentable-wrapper)
- [4. FontPicker with Font Binding](#4-fontpicker-with-font-binding)
- [5. ControlGroup Palette Style for Toolbar](#5-controlgroup-palette-style-for-toolbar)

---

## 1. Custom ButtonStyle for Accessory Bar Button

Accessory bar buttons sit below content (metadata, tags, status). They need a compact, muted appearance that doesn't compete with primary controls.

```swift
import SwiftUI

struct AccessoryBarButtonStyle: ButtonStyle {
    var isToggled: Bool = false

    func makeBody(configuration: Configuration) -> some View {
        configuration.label
            .font(.caption)
            .padding(.horizontal, 8)
            .padding(.vertical, 2)
            .background {
                RoundedRectangle(cornerRadius: 4, style: .continuous)
                    .fill(background(isPressed: configuration.isPressed))
            }
            .foregroundStyle(.secondary)
    }

    private func background(isPressed: Bool) -> Color {
        if isToggled && isPressed {
            return Color(nsColor: .selectedControlColor)
        } else if isPressed {
            return Color(nsColor: .controlBackgroundColor)
        } else if isToggled {
            return Color(nsColor: .selectedContentBackgroundColor).opacity(0.15)
        } else {
            return Color(nsColor: .controlColor)
        }
    }
}

extension View {
    func accessoryBarButton(isToggled: Bool = false) -> some View {
        self.buttonStyle(AccessoryBarButtonStyle(isToggled: isToggled))
    }
}
```

### Usage

```swift
struct MetadataBar: View {
    @State private var isFavorite = false
    @State private var isShared = false

    var body: some View {
        HStack(spacing: 4) {
            Button {
                isFavorite.toggle()
            } label: {
                Label("Favorite", systemImage: isFavorite ? "heart.fill" : "heart")
            }
            .accessoryBarButton(isToggled: isFavorite)

            Button {
                isShared.toggle()
            } label: {
                Label("Shared", systemImage: isShared ? "person.2.fill" : "person.2")
            }
            .accessoryBarButton(isToggled: isShared)

            Spacer()

            Text("Modified 2h ago")
                .font(.caption2)
                .foregroundStyle(.tertiary)
        }
        .padding(.horizontal, 12)
        .padding(.vertical, 4)
        .background(Color(nsColor: .controlBackgroundColor))
    }
}
```

### Gotchas

- **Dynamic type.** `.caption` scales with accessibility text size. Test at 150% size — the padded layout should not clip.
- **Toggled state.** The `isToggled` parameter is not tracked by SwiftUI automatically — you must pass it as a binding from the parent. The `ButtonStyle` struct is a value type; it cannot observe changes on its own.
- **Hover effects.** macOS expects a subtle hover highlight on accessory buttons. Add `.onHover` to the label if you want a hover glow without waiting for press.

---

## 2. Picker with .radioGroup in a Form

Radio groups are the standard way to present mutually exclusive options in macOS preferences panels. In a SwiftUI `Form`, radio groups integrate with the label column automatically.

```swift
struct AppearanceSettings: View {
    @AppStorage("colorScheme") private var colorScheme: ColorSchemePreference = .system

    var body: some View {
        Form {
            Section("Appearance") {
                Picker("Color Scheme", selection: $colorScheme) {
                    Text("System").tag(ColorSchemePreference.system)
                    Text("Light").tag(ColorSchemePreference.light)
                    Text("Dark").tag(ColorSchemePreference.dark)
                }
                .pickerStyle(.radioGroup)
            }
        }
        .formStyle(.grouped) // Standard macOS preferences style
        .frame(width: 360)
    }
}

enum ColorSchemePreference: String, CaseIterable {
    case system, light, dark
}
```

### Usage in a TabView

```swift
struct PreferencesView: View {
    var body: some View {
        TabView {
            AppearanceSettings()
                .tabItem { Label("Appearance", systemImage: "paintbrush") }

            GeneralSettings()
                .tabItem { Label("General", systemImage: "gear") }
        }
        .frame(width: 400, height: 300)
    }
}
```

### Gotchas

- **Vertical space.** Each radio option adds ~24pt of height. Three options = ~72pt. In a `List`, this may scroll awkwardly. Use `.segmented` for ≤4 options where horizontal space permits.
- **Form integration.** `..pickerStyle(.radioGroup)` inside a `Form` does not use the form's label column — it takes the full width. The first argument to `Picker` becomes the section header, not a trailing label.
- **`.formStyle(.grouped)`** is the standard macOS preferences style. `.automatic` may resolve to a grouped style on macOS 13+, but being explicit avoids ambiguity.

---

## 3. NSPathControl NSViewRepresentable Wrapper

`NSPathControl` has no SwiftUI equivalent. Wrap it for a clickable breadcrumb-style file path with drag-and-drop support.

```swift
import SwiftUI
import AppKit

struct PathControl: NSViewRepresentable {
    var url: URL?
    var onPathChange: ((URL) -> Void)?
    var allowedContentTypes: [UTType] = [.folder, .item]

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    func makeNSView(context: Context) -> NSPathControl {
        let control = NSPathControl()
        control.delegate = context.coordinator
        control.isEditable = false
        control.pathStyle = .standard
        control.translatesAutoresizingMaskIntoConstraints = false
        // Intrinsic height: 22pt for regular, 17pt for mini
        control.setContentHuggingPriority(.defaultHigh, for: .vertical)
        return control
    }

    func updateNSView(_ nsView: NSPathControl, context: Context) {
        if nsView.url != url {
            nsView.url = url
        }
    }

    @MainActor
    final class Coordinator: NSObject, NSPathControlDelegate {
        var parent: PathControl

        init(_ parent: PathControl) { self.parent = parent }

        func pathControl(
            _ pathControl: NSPathControl,
            willDisplay openPanel: NSOpenPanel
        ) {
            openPanel.canChooseDirectories = true
            openPanel.canChooseFiles = false
            openPanel.allowedContentTypes = parent.allowedContentTypes
        }

        func pathControl(
            _ pathControl: NSPathControl,
            validateDrop info: any NSDraggingInfo
        ) -> NSDragOperation {
            .copy
        }

        func pathControl(
            _ pathControl: NSPathControl,
            acceptDrop info: any NSDraggingInfo
        ) -> Bool {
            guard let urls = info.draggingPasteboard.readObjects(
                forClasses: [NSURL.self]
            ) as? [URL],
            let url = urls.first else { return false }
            parent.onPathChange?(url)
            return true
        }

        // Handle clicks on path components
        @objc func pathComponentClicked(_ sender: NSPathControl) {
            if let clickedItem = sender.clickedPathComponentCell,
               let url = clickedItem.url {
                parent.onPathChange?(url)
            }
        }
    }
}
```

### Usage

```swift
struct ProjectPicker: View {
    @State private var projectURL: URL?

    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            Text("Project Location")
                .font(.headline)

            PathControl(
                url: projectURL,
                onPathChange: { url in
                    projectURL = url
                },
                allowedContentTypes: [.folder]
            )
            .frame(height: 22) // Explicit height prevents collapse

            HStack {
                Text(projectURL?.path ?? "No location selected")
                    .font(.caption)
                    .foregroundStyle(.secondary)
                    .lineLimit(1)
                Spacer()
                Button("Browse…") {
                    let panel = NSOpenPanel()
                    panel.canChooseFiles = false
                    panel.canChooseDirectories = true
                    panel.allowedContentTypes = [.folder]
                    if panel.runModal() == .OK {
                        projectURL = panel.url
                    }
                }
                .controlSize(.small)
            }
        }
        .padding()
    }
}
```

### Gotchas

- **Zero-height collapse.** Without an explicit `.frame(height:)`, the `NSPathControl` may collapse in a `VStack`. Always set a height constraint matching the `controlSize`.
- **Clicked component.** `clickedPathComponentCell` returns the specific path segment the user clicked. Call `makeFirstResponder` on the window if you want the click to navigate to that intermediate directory.
- **Path styles.** `.standard` is the most common — it shows `/Users/mp/Docs/Project` with each segment clickable. `.popUp` shows only the leaf with a dropdown to navigate parents. `.bar` shows a compact single-component bar. Choose once and don't change dynamically.
- **Double-click behavior.** Double-clicking the path control opens the file in its default app, not a navigation. Override this by subclassing if you need custom behavior.

---

## 4. FontPicker with Font Binding

`FontPicker` is the macOS-only SwiftUI control for the system font panel. It binds to a `Font` value and synchronizes with `NSFontPanel`.

```swift
import SwiftUI

struct FontSettingsPanel: View {
    @State private var bodyFont: Font = .body
    @State private var headingFont: Font = .title

    var body: some View {
        Form {
            Section("Body Text") {
                HStack {
                    FontPicker("Body Font", selection: $bodyFont)
                        .labelsHidden()
                    Spacer()
                    Text("Preview: \(fontDescription(bodyFont))")
                        .foregroundStyle(.secondary)
                }
            }

            Section("Headings") {
                HStack {
                    FontPicker("Heading Font", selection: $headingFont)
                        .labelsHidden()
                    Spacer()
                    Text("Preview: \(fontDescription(headingFont))")
                        .foregroundStyle(.secondary)
                }
            }

            Section {
                Text("Sample body text in the selected font.")
                    .font(bodyFont)
                Text("Sample heading text.")
                    .font(headingFont)
            }
        }
        .formStyle(.grouped)
        .frame(width: 400, height: 300)
    }

    private func fontDescription(_ font: Font) -> String {
        // Extract description for display purposes
        let mirror = Mirror(reflecting: font)
        return String(describing: font)
    }
}
```

### Persistence with @AppStorage

```swift
struct PersistentFontSettings: View {
    @AppStorage("bodyFontName") private var bodyFontName = "SF Pro"
    @AppStorage("bodyFontSize") private var bodyFontSize: Double = 13

    private var bodyFont: Font {
        .custom(bodyFontName, size: bodyFontSize)
    }

    var body: some View {
        HStack {
            FontPicker("Font", selection: Binding(
                get: { bodyFont },
                set: { newFont in
                    // NSFontPanel sends the new font; extract name and size
                    let nsFont = NSFont(descriptor: newFont.nsFontDescriptor, size: 0)
                    // Simplified: store current name/size from panel
                }
            ))
            .labelsHidden()

            Text("Current: \(bodyFontName) \(Int(bodyFontSize))pt")
                .font(.caption)
                .foregroundStyle(.secondary)
        }
    }
}
```

### Gotchas

- **Platform guard.** `FontPicker` is macOS-only. Wrap in `#if os(macOS)` / `#else` for multiplatform targets. On iOS, provide a custom font selection UI.
- **Font panel singleton.** `NSFontPanel` is a singleton. If multiple `FontPicker` instances exist, they all control the same panel. The last-activated picker receives changes.
- **Binding complexity.** The `Font` type is opaque — you cannot extract the font name or size from it directly. For persistence, store the name and size separately in `@AppStorage` and reconstruct the `Font` on read.
- **Live preview.** Changes in the font panel update the binding immediately. If you need debounced saves (e.g., to a server), add a `.debounce` operator or save on `onDisappear`.

---

## 5. ControlGroup Palette Style for Toolbar

`ControlGroup` with `.palette` style is the idiomatic way to build custom toolbars in SwiftUI on macOS. Each child control renders as a visible, individually clickable element.

```swift
struct EditorToolbar: View {
    @State private var isBold = false
    @State private var isItalic = false
    @State private var isUnderline = false
    @State private var textAlignment: TextAlignment = .leading

    var body: some View {
        HStack(spacing: 0) {
            // Text formatting group
            ControlGroup {
                Button { isBold.toggle() } label: {
                    Image(systemName: "bold")
                }
                .accessoryBarButton(isToggled: isBold)

                Button { isItalic.toggle() } label: {
                    Image(systemName: "italic")
                }
                .accessoryBarButton(isToggled: isItalic)

                Button { isUnderline.toggle() } label: {
                    Image(systemName: "underline")
                }
                .accessoryBarButton(isToggled: isUnderline)
            }
            .controlGroupStyle(.palette)
            .label("Text Formatting")

            Divider()
                .frame(height: 16)

            // Alignment group
            ControlGroup {
                Button { textAlignment = .leading } label: {
                    Image(systemName: "text.alignleft")
                }
                .accessoryBarButton(isToggled: textAlignment == .leading)

                Button { textAlignment = .center } label: {
                    Image(systemName: "text.aligncenter")
                }
                .accessoryBarButton(isToggled: textAlignment == .center)

                Button { textAlignment = .trailing } label: {
                    Image(systemName: "text.alignright")
                }
                .accessoryBarButton(isToggled: textAlignment == .trailing)
            }
            .controlGroupStyle(.palette)
            .label("Alignment")

            Divider()
                .frame(height: 16)

            // Actions group as menu
            ControlGroup {
                Button("Export PDF…") { exportPDF() }
                Button("Print…") { printDocument() }
                    .keyboardShortcut("p")
            }
            .controlGroupStyle(.menu)
            .label("Actions")

            Spacer()
        }
        .padding(.horizontal, 12)
        .padding(.vertical, 6)
        .background(Color(nsColor: .controlBackgroundColor))
    }

    private func exportPDF() { /* ... */ }
    private func printDocument() { /* ... */ }
}
```

### Usage in a Window

```swift
struct EditorWindow: View {
    @State private var text = "Type here…"

    var body: some View {
        VStack(spacing: 0) {
            EditorToolbar()

            Divider()

            TextEditor(text: $text)
                .font(.body)
                .padding()
        }
    }
}
```

### Gotchas

- **Palette vs menu.** `.palette` shows all controls inline — use for primary actions. `.menu` collapses into a single button — use for secondary/overflow actions. Don't mix styles within a single `ControlGroup`.
- **Divider between groups.** `Divider()` between `ControlGroup` elements renders as a vertical separator in toolbar context. Set a `frame(height:)` to constrain it.
- **Label accessibility.** The `.label()` modifier on `ControlGroup` sets the accessibility label. It is not visible in the palette style but is read by VoiceOver.
- **Keyboard shortcuts in menu style.** `.keyboardShortcut()` on buttons inside a `.menu` ControlGroup work when the menu is open. They do not fire globally — use `CommandMenu` for global shortcuts.
- **Toggled state in palette.** Unlike segmented pickers, palette buttons don't automatically show selection state. You must manually apply a visual indicator (background color, bold icon) via a custom `ButtonStyle` like `AccessoryBarButtonStyle`.
