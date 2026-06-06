# Accessibility Patterns

Deep-dive patterns for macOS SwiftUI accessibility. Each pattern is self-contained with full code, explanation, and gotchas.

---

## Contents

- [Custom Accessibility Element with Actions](#custom-accessibility-element-with-actions)
- [AXCustomContentProvider Implementation](#axcustomcontentprovider-implementation)
- [Full Keyboard Access Wiring](#full-keyboard-access-wiring)
- [VoiceOver-Optimized Table with Custom Rotor](#voiceover-optimized-table-with-custom-rotor)
- [Accessibility Audit Checklist](#accessibility-audit-checklist)

---

## Custom Accessibility Element with Actions

When building a custom control (e.g., a card, a canvas, or a composite widget), you need to override the default accessibility tree and provide explicit actions. The pattern: merge children into one element, add a label/value, and register custom actions.

```swift
import SwiftUI

struct SelectableCard: View {
    let title: String
    let subtitle: String
    @Binding var isSelected: Bool
    let onActivate: () -> Void
    let onPreview: () -> Void

    var body: some View {
        VStack(alignment: .leading, spacing: 4) {
            HStack {
                Text(title)
                    .font(.headline)
                Spacer()
                if isSelected {
                    Image(systemName: "checkmark.circle.fill")
                        .foregroundStyle(.accent)
                }
            }
            Text(subtitle)
                .font(.subheadline)
                .foregroundStyle(.secondary)
        }
        .padding(12)
        .background(
            RoundedRectangle(cornerRadius: 10)
                .fill(isSelected ? Color.accentColor.opacity(0.1) : Color.clear)
                .stroke(isSelected ? Color.accentColor : Color.gray.opacity(0.3), lineWidth: 1)
        )
        .accessibilityElement(children: .ignore)
        .accessibilityLabel(title)
        .accessibilityValue("\(subtitle), \(isSelected ? "selected" : "not selected")")
        .accessibilityAddTraits(isSelected ? [.isButton, .isSelected] : .isButton)
        .accessibilityAction(.default) {
            onActivate()
        }
        .accessibilityAction(named: "Preview") {
            onPreview()
        }
        .onTapGesture {
            onActivate()
        }
    }
}
```

**Key points:**

- `.accessibilityElement(children: .ignore)` suppresses the default child tree (VStack, Text, Image all become invisible to VoiceOver). You supply a single label and value.
- `.accessibilityAddTraits(.isButton)` tells VoiceOver this is pressable. Without it, VO announces it as static text.
- `.accessibilityAction(.default)` handles the VO+Space primary activation.
- `.accessibilityAction(named:)` adds a secondary action discoverable via the VO actions menu.
- The `onTapGesture` handles mouse users; the `.accessibilityAction(.default)` handles VoiceOver users.

**SwiftUI usage:**

```swift
struct CardList: View {
    @State private var selectedID: UUID?

    var body: some View {
        ScrollView {
            LazyVStack(spacing: 8) {
                ForEach(documents) { doc in
                    SelectableCard(
                        title: doc.name,
                        subtitle: doc.kind,
                        isSelected: .constant(selectedID == doc.id),
                        onActivate: { selectedID = doc.id },
                        onPreview: { showPreview(doc) }
                    )
                }
            }
            .padding()
        }
    }
}
```

---

## AXCustomContentProvider Implementation

`AXCustomContentProvider` uses the `.accessibilityCustomContent` modifier to attach supplementary information to a VoiceOver element. On macOS, users access this through the AXCustom Content rotor (VO+U → Custom Content).

```swift
struct FileRow: View {
    let file: FileItem

    var body: some View {
        HStack(spacing: 12) {
            Image(systemName: file.icon)
                .frame(width: 24)

            VStack(alignment: .leading, spacing: 2) {
                Text(file.name)
                    .font(.body)
                HStack(spacing: 8) {
                    Text(file.modifiedAt, style: .relative)
                    Text("•")
                    Text(file.kind)
                }
                .font(.caption)
                .foregroundStyle(.secondary)
            }

            Spacer()

            Text(file.formattedSize)
                .font(.caption.monospacedDigit())
                .foregroundStyle(.secondary)
        }
        .padding(.vertical, 6)
        // Primary accessibility data
        .accessibilityElement(children: .ignore)
        .accessibilityLabel(file.name)
        .accessibilityValue("\(file.modifiedAt.formatted(date: .abbreviated, time: .omitted)), \(file.kind)")
        // Supplementary data via AXCustomContent
        .accessibilityCustomContent("Size", file.formattedSize)
        .accessibilityCustomContent(
            "Created",
            file.createdAt.formatted(date: .long, time: .omitted),
            importance: .high
        )
        .accessibilityCustomContent("Last opened", file.openedAt?.formatted() ?? "Never")
        .accessibilityCustomContent("Tags", file.tags.joined(separator: ", "))
        .accessibilityAction(.default) {
            openFile(file)
        }
        .accessibilityAction(named: "Get Info") {
            showFileInfo(file)
        }
    }
}
```

**Importance levels:**

- `.high` — announced during normal navigation."Created: January 5, 2025" gets read when VoiceOver moves to the row.
- `.default` — available in the rotor but not auto-announced. Users drill into it via the AXCustom Content rotor.

**When to use `.high` importance:** Use sparingly. Only the most critical metadata should auto-announce. Every `.high` item adds verbosity to navigation. "Created date" for a document might warrant high importance; "Size" usually does not.

**Rotor navigation flow:** User opens rotor with VO+U → navigates to AXCustom Content → VO reads each custom content item in label: value order. High-importance items are listed first.

---

## Full Keyboard Access Wiring

Full Keyboard Access lets users tab through every interactive control. This pattern shows how to make a custom view hierarchy work correctly with Tab, Shift+Tab, and Enter/Space activation, plus VoiceOver.

```swift
struct SettingsPanel: View {
    @State private var isEnabled = false
    @State private var selectedOption = "Daily"
    @State private var threshold: Double = 50.0
    @FocusState private var focusedField: SettingsField?

    enum SettingsField: Hashable {
        case toggle, picker, slider, saveButton, resetButton
    }

    var body: some View {
        VStack(alignment: .leading, spacing: 16) {
            // Toggle: already focusable by default
            Toggle("Enable notifications", isOn: $isEnabled)
                .focused($focusedField, equals: .toggle)

            // Picker: already focusable by default
            Picker("Frequency", selection: $selectedOption) {
                ForEach(["Daily", "Weekly", "Monthly"], id: \.self) { option in
                    Text(option).tag(option)
                }
            }
            .focused($focusedField, equals: .picker)

            // Slider with custom focus
            VStack(alignment: .leading) {
                Label("Threshold: \(Int(threshold))", systemImage: "gauge")
                    .labelStyle(.titleAndIcon)
                Slider(value: $threshold, in: 0...100)
                    .focused($focusedField, equals: .slider)
            }

            // Custom button row
            HStack {
                Button("Save", action: save)
                    .buttonStyle(.borderedProminent)
                    .focused($focusedField, equals: .saveButton)

                Button("Reset", action: reset)
                    .buttonStyle(.bordered)
                    .focused($focusedField, equals: .resetButton)
            }
        }
        .padding()
        .frame(maxWidth: 400)
        // Default focus when panel appears
        .onAppear {
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                focusedField = .toggle
            }
        }
        // Handle Enter/Space for custom focus
        .onKeyPress(.return) {
            handleActivation(for: focusedField)
            return .handled
        }
    }

    private func handleActivation(for field: SettingsField?) {
        switch field {
        case .toggle: isEnabled.toggle()
        case .resetButton: reset()
        case .saveButton: save()
        default: break
        }
    }
}
```

**Key patterns:**

1. **`@FocusState` enum** — Model each focusable element with a hashable case. Works for any mix of focusable and non-focusable views.
2. **Default focus** — Set in `.onAppear` with a short delay so the animation completes.
3. **Non-text-field controls** (`Button`, `Toggle`, `Picker`, `Slider`) are focusable by default — no `.focusable(true)` needed.
4. **Custom-drawn views** — If you use `Canvas` or a custom gesture handler, add `.focusable(true)` and `.accessibilityAddTraits(.isButton)`.
5. **Arrow keys in grouped controls** — `Picker` with segmented style navigates with arrow keys when focused. `Slider` responds to left/right arrows.

**Verifying in your app:**

1. System Settings → Keyboard → turn on Full Keyboard Access
2. Tab through your UI — every interactive element should receive focus ring
3. Shift+Tab to go backward
4. Space/Enter should activate the focused control

---

## VoiceOver-Optimized Table with Custom Rotor

Tables are the most complex accessibility challenge in SwiftUI on macOS. A `List` with custom rows works for simple cases, but for data-dense tables (spreadsheets, file browsers, log viewers), you need structured custom content and careful child management.

```swift
struct DataTable: View {
    @State private var sortColumn: String = "name"
    @State private var sortAscending = true
    @State private var selection: Set<UUID> = []

    var body: some View {
        Table(of: TableRow.self, selection: $selection) {
            TableColumn("Name") { row in
                Text(row.name)
                    .accessibilityCustomContent("Name", row.name, importance: .high)
            }
            .width(min: 150, ideal: 200)

            TableColumn("Date") { row in
                Text(row.date.formatted(date: .abbreviated, time: .omitted))
                    .accessibilityCustomContent("Modified", row.date.formatted(.dateTime))
            }
            .width(120)

            TableColumn("Size") { row in
                Text(row.formattedSize)
                    .accessibilityCustomContent("Size", row.formattedSize)
            }
            .width(80)

            TableColumn("Kind") { row in
                Text(row.kind)
                    .accessibilityCustomContent("Kind", row.kind, importance: .high)
            }
            .width(100)

            TableColumn("Status") { row in
                StatusBadge(status: row.status)
                    .accessibilityCustomContent("Status", row.status.rawValue, importance: .high)
            }
            .width(100)

            // Each row provides rotor-accessible supplementary data
            TableColumn("") { row in
                Menu {
                    Button("Open") { open(row) }
                    Button("Reveal in Finder") { reveal(row) }
                    Divider()
                    Button("Delete", role: .destructive) { delete(row) }
                } label: {
                    Image(systemName: "ellipsis.circle")
                }
                .buttonStyle(.plain)
                .accessibilityAction(named: "Open") { open(row) }
                .accessibilityAction(named: "Reveal in Finder") { reveal(row) }
                .accessibilityAction(named: "Delete") { delete(row) }
            }
            .width(30)
        } rows: {
            ForEach(sortedRows) { row in
                TableRow(row)
                    .contextMenu {
                        Button("Open") { open(row) }
                        Button("Reveal in Finder") { reveal(row) }
                    }
            }
        }
        // Table-level accessibility
        .accessibilityLabel("File browser")
        .accessibilityHint("Navigate rows with arrow keys. Open rotor with VO+U for column details.")
        .accessibilityAction(.default) {
            if let first = selection.first {
                open(rows[first])
            }
        }
    }
}
```

**Column-level `.accessibilityCustomContent`:** Attaching custom content to each cell within a `TableColumn` means VoiceOver users can navigate the rotor and hear column data without the entire row being read as one giant blob.

**Quick Nav for tables:** With Quick Nav on, `T` navigates tables, then arrow keys move cell-by-cell. Ensure table cells have labels so Quick Nav isn't navigating silent cells.

**NSViewRepresentable alternative:** For tables with 1000+ rows, `NSTableView` wrapped in `NSViewRepresentable` with explicit `accessibilityChildren` is more performant. See the `swiftui-appkit-interop` skill for the wrapping pattern. The `NSTableView` accessibility API is `NSTableView.accessibilityRows` + `NSTableView.accessibilityColumns`.

---

## Accessibility Audit Checklist

Run this checklist after implementing accessibility features. Complete it with Accessibility Inspector + live VoiceOver testing.

### Structural Audit (Accessibility Inspector)

1. **Run Audit** — Inspector → Audit → Run. Fix every warning.
2. **Tree walk** — Inspector → explore the tree. Every visible interactive element should appear.
3. **Verify labels** — Crosshair on each element. Confirm label is concise and meaningful.
4. **Verify traits** — Buttons have "button" trait, toggles have "toggle," etc.
5. **Verify actions** — Interactive elements have at least a "Press" action.
6. **Check custom content** — Complex rows show AXCustomContent entries in the inspector.

### VoiceOver Test Script (press Cmd+F5)

1. **Navigate entire window** — Arrow keys (VO+Right/Left). Every interactive element is reachable?
2. **Rotor test** — Press VO+U. Navigate Headings, Links, Form Controls. Does the rotor show expected items?
3. **Custom content rotor** — VO+U → "Custom Content." Are supplementary values present?
4. **Actions test** — On a multi-action element: VO+Shift+Down to interact, navigate actions, activate each one.
5. **Announcements** — Trigger a status change (save, error). Does VoiceOver announce it?
6. **Activation** — On every button/link: VO+Space activates correctly?
7. **Forms** — In text fields: typed input is announced (if "speak typed characters" is on in VoiceOver Utility).

### Keyboard-Only Test Script

1. **Enable Full Keyboard Access** — System Settings → Keyboard → Full Keyboard Access.
2. **Tab through all controls** — Every button, toggle, picker, slider, and custom widget gets focus.
3. **Shift+Tab backward** — Focus moves in reverse correctly.
4. **Enter/Space** — Activates the focused element.
5. **Escape** — Dismisses popovers/sheets.
6. **Arrow keys** — Move within grouped controls (slider, segmented picker, table).

### Dynamic Type Test Script

1. **System Settings → Accessibility → Display → Text Size** — drag slider from smallest to largest.
2. Text scales at every position. No clipping, wrapping issues, or truncation.
3. Layout adapts (columns don't overlap, buttons don't clip).
4. **Semantic fonts only** — search codebase for hardcoded point sizes in informational text.

### Edge Cases

1. **Empty state** — When a list is empty, what does VoiceOver announce? An empty table with 0 rows should say so.
2. **Loading state** — If data loads asynchronously, `accessibilityLabel` should reflect "Loading" vs "3 items."
3. **Error state** — Use `AccessibilityNotification.Announcement` to announce errors without moving focus.
4. **Modal dialogs** — VoiceOver focus should be trapped in the sheet/alert. Arrow keys shouldn't reach the background content.
5. **Drag & drop** — VoiceOver users can't drag with a mouse. Provide an alternative action (e.g., "Move to…" menu item).

---
