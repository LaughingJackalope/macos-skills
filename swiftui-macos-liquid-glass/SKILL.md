---
name: swiftui-macos-liquid-glass
description: "Liquid Glass on macOS 26 — glassEffect, GlassEffectContainer, glass toolbar/sidebar/panel patterns, NSVisualEffectView vibrancy materials vs SwiftUI .glassBackgroundDisplayMode, windowBackgroundDragBehavior, translucent sidebar, title bar blending. Use when implementing Liquid Glass visual effects in macOS 26 SwiftUI apps."
sources: https://developer.apple.com/documentation/SwiftUI/Applying-Liquid-Glass-to-custom-views, https://developer.apple.com/documentation/TechnologyOverviews/adopting-liquid-glass, https://developer.apple.com/videos/play/wwdc2025/323/, https://developer.apple.com/documentation/swiftui/glassbackgrounddisplaymode, https://developer.apple.com/documentation/appkit/nsvisualeffectview, https://developer.apple.com/documentation/swiftui/customizing-window-styles-and-state-restoration-behavior-in-macos, https://www.atelier-socle.com/en/articles/swiftui-liquid-glass-guide
researched: 2026-06-06
platform: macOS 26+
---

# Liquid Glass on macOS 26

Liquid Glass is macOS 26's system-wide material language. It replaces the frosted `NSVisualEffectView` aesthetic with adaptive, physically-based glass that refracts and tints content beneath it. In SwiftUI, glass is applied via `.glassEffect()`, grouped in `GlassEffectContainer`, and integrated into toolbars, sidebars, panels, and title bars. This skill covers the full glass surface API, vibrancy bridging with AppKit, and the gotchas that separate a native macOS 26 look from a broken one.

For `NSViewRepresentable` wrapping of `NSVisualEffectView` for advanced vibrancy, see the `swiftui-appkit-interop` skill. For toolbar structure, see `swiftui-macos-toolbar-sidebar`. For sidebar navigation, see `swiftui-macos-navigation`.

## Contents

- [Liquid Glass Overview](#liquid-glass-overview)
- [glassEffect()](#glasseffect)
- [GlassEffectContainer](#glasseffectcontainer)
- [Toolbar Glass](#toolbar-glass)
- [Sidebar Glass](#sidebar-glass)
- [Panel Glass](#panel-glass)
- [NSVisualEffectView Bridge](#nsvisualeffectview-bridge)
- [Vibrancy Materials](#vibrancy-materials)
- [Dark Mode Considerations](#dark-mode-considerations)
- [windowBackgroundDragBehavior](#windowbackgrounddragbehavior)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Liquid Glass Overview

Liquid Glass is a translucent, refractive material that adapts to content beneath it. Unlike the flat blur of `NSVisualEffectView`, glass:

- **Refracts** — content behind glass appears subtly distorted, simulating physical glass.
- **Tints** — the glass picks up color from its own tint and from the background.
- **Adapts** — appearance shifts between light and dark mode, and responds to the vibrancy of the underlying content.
- **Layers** — multiple glass surfaces stack, with each layer refracting through the ones above.

**Design principles:**
- Glass is for **chrome**, not content. Apply it to toolbars, sidebars, floating panels, and overlays — not to the main document area.
- Use glass to **reduce visual weight**. A glass toolbar feels lighter than an opaque one, letting the content breathe.
- Glass **blurs and tints** what's behind it. Ensure text and controls on glass remain legible against varying backgrounds.
- The system manages glass quality and performance. Don't try to replicate the effect with custom blur modifiers.

**System materials:** macOS 26 defines glass as the primary material for window chrome. It coexists with `NSVisualEffectView` vibrancy for AppKit interop and legacy surfaces. SwiftUI's `.glassBackgroundDisplayMode` bridges the two.

## glassEffect()

Apply `.glassEffect()` to any SwiftUI view to render it with the Liquid Glass material. Available on macOS 26+.

```swift
// Basic glass on a button — default .regular glass in a Capsule shape
Button("Action") { /* ... */ }
    .glassEffect()

// Custom shape — rectangular glass
Text("Status")
    .padding(.horizontal, 12)
    .padding(.vertical, 6)
    .glassEffect(in: .rect(cornerRadius: 16))

// Custom glass variant with tint
Text("Active")
    .padding()
    .glassEffect(.regular.tint(.blue.opacity(0.2)))
```

### Glass Type Parameter

`.glassEffect()` accepts a `Glass` parameter (default `.regular`):

| Glass Variant | Behavior |
|---|---|
| `.regular` | Standard Liquid Glass — full refraction and vibrancy. |
| `.clear` | Minimal glass — reduced opacity, more content shows through. |

The shape parameter (second argument, default `Capsule`) determines the glass silhouette:

```swift
.glassEffect(.regular, in: .rect(cornerRadius: 12))  // Rounded rectangle
.glassEffect(.clear, in: .capsule)                      // Capsule (default shape)
.glassEffect(.regular, in: .circle)                     // Circular glass
```

### Tint

Chain `.tint(_:)` on the `Glass` type or use the `tint:` parameter:

```swift
// Subtle warm tint via Glass chain
.glassEffect(.regular.tint(.orange.opacity(0.15)))

// Via parameter
.glassEffect(tint: .orange.opacity(0.15))
```

Keep tint opacity below 0.3 — heavy tints obscure the refractive quality.

### interactive()

Enable pointer-responsive behavior (scaling, bouncing) on glass controls:

```swift
Button("Action") { /* ... */ }
    .glassEffect(.regular.interactive())
```


### glassEffectID
Assign a `.glassEffectID()` to enable **morphing** between glass elements. When two views share the same glass effect ID and are in the same `GlassEffectContainer`, the system animates the glass shape as one transforms into the other.

```swift
@State private var expanded = false

var body: some View {
    GlassEffectContainer {
        if expanded {
            DetailView()
                .glassEffectID("card")
        } else {
            SummaryView()
                .glassEffectID("card")
        }
    }
}
```

The morphing animation is automatic — the system interpolates the glass boundary between the two views' frames. Both views must be in the same container and have the same ID.

## GlassEffectContainer

`GlassEffectContainer` groups multiple glass elements so the system treats them as a single cohesive glass surface. Without a container, each `.glassEffect()` view renders as an independent glass tile.

### Container Shape

```swift
GlassEffectContainer {
    HStack(spacing: 8) {
        Button("New") { /* ... */ }
            .glassEffect()
        Button("Open") { /* ... */ }
            .glassEffect()
        Button("Save") { /* ... */ }
            .glassEffect()
    }
    .padding(8)
}
```

The container's shape is derived from its content layout. The glass surface fills the bounding box of the container's subviews, creating a unified panel rather than individual glass buttons.

### glassEffectUnion

Use `.glassEffectUnion(id:combined:)` to control which glass elements within a container share the same glass surface:

```swift
GlassEffectContainer {
    VStack {
        // These two share one glass surface
        HStack {
            Button("Bold") { /* ... */ }
                .glassEffect()
                .glassEffectUnion(id: "textStyle", union: true)
            Button("Italic") { /* ... */ }
                .glassEffect()
                .glassEffectUnion(id: "textStyle", union: true)
        }

        // This is a separate glass surface
        Button("Export") { /* ... */ }
            .glassEffect()
            .glassEffectUnion(id: "export", union: true)
    }
}
```

Elements with the same `id` and `union: true` merge into one glass surface. Elements with different IDs or `union: false` render independently. This is useful for grouping related controls (formatting, navigation, actions) into visually distinct glass panels within the same container.

### Multiple Containers

You can nest or stack `GlassEffectContainer` instances. Each container produces its own glass surface:

```swift
VStack {
    GlassEffectContainer {
        // Toolbar glass
        HStack { /* toolbar items */ }
    }

    GlassEffectContainer {
        // Sidebar glass
        VStack { /* sidebar items */ }
    }
}
```

## Toolbar Glass

The standard macOS 26 toolbar uses Liquid Glass by default when you apply `.glassEffect()` to toolbar content. The glass blends with the title bar for a seamless chrome appearance.

```swift
struct ContentView: View {
    var body: some View {
        NavigationSplitView {
            SidebarView()
        } detail: {
            DetailView()
        }
        .toolbar {
            ToolbarItemGroup(placement: .primaryAction) {
                Button("Add") { /* ... */ }
                    .glassEffect()

                Button("Share") { /* ... */ }
                    .glassEffect()
            }
        }
        .windowToolbarStyle(.unified)
    }
}
```

### Title Bar Blending

To make the toolbar glass blend seamlessly with the title bar:

1. Use `.windowToolbarStyle(.unified)` — this is the default on macOS 26.
2. Apply `.glassEffect()` to toolbar items.
3. The system automatically extends the glass material into the title bar area.

```swift
// Full title bar blending
.windowToolbarStyle(.unified)
.toolbar {
    ToolbarItem(placement: .principal) {
        Text("Document Title")
            .font(.headline)
    }
    ToolbarItem(placement: .primaryAction) {
        Button("Done") { /* ... */ }
            .glassEffect()
    }
}
```

**Avoid** `.windowToolbarStyle(.expanded)` if you want glass blending — the expanded style adds height and separates the toolbar from the title bar visually.

## Sidebar Glass

Apply glass to a sidebar for a translucent navigation panel that reveals the window background beneath.

```swift
NavigationSplitView {
    List(selection: $selectedItem) {
        ForEach(items) { item in
            Label(item.name, systemImage: item.icon)
                .tag(item.id)
        }
    }
    .scrollContentBackground(.hidden)
    .glassBackgroundEffect(displayMode: .implicit)
    .navigationSplitViewColumnWidth(min: 180, ideal: 220)
} detail: {
    DetailView()
}
```

| Display Mode | Use Case |
|---|---|
| `.always` | Always show glass. Use when you want glass even inside another glass container. |
| `.implicit` | Glass renders only when the view is not already inside a glass container. Default behavior that avoids duplicate backgrounds. |
| `.never` | Glass is disabled; use to suppress glass in a container context. |

```swift
// Default — glass renders unless already inside a glass container
List {
    ForEach(items) { item in
        Label(item.name, systemImage: item.icon)
    }
}
.glassBackgroundEffect(displayMode: .implicit)

// Force glass even when nested
SidebarContent()
    .glassBackgroundEffect(displayMode: .always)

// Suppress glass in a container context
ItemView()
    .glassBackgroundEffect(displayMode: .never)
```

**Key difference from `NSVisualEffectView`:** `.glassBackgroundEffect` is a SwiftUI-native property that works with the system's glass rendering pipeline. `NSVisualEffectView` materials are AppKit-level and don't participate in the glass refraction system. Use `.glassBackgroundEffect` for new code; bridge to `NSVisualEffectView` only when you need specific AppKit vibrancy materials not exposed in SwiftUI.

**Gotcha:** `.implicit` mode does not suppress duplicates when a subview is offset in the z-axis. If two views have z-axis displacement, both render their glass backgrounds even if they overlap.

### Vibrancy in Sidebar

For sidebar rows that need to adapt their appearance to the glass background:

```swift
List(items, selection: $selectedItem) { item in
    Label(item.name, systemImage: item.icon)
        .foregroundStyle(.primary) // Adapts to vibrancy automatically
        .tag(item.id)
}
.glassBackgroundEffect(displayMode: .implicit)
```

Text and SF Symbols automatically adapt their contrast when placed over vibrant glass. Use `.foregroundStyle(.primary)` or `.foregroundStyle(.secondary)` — avoid hardcoded colors in glass contexts.
## Panel Glass

Inspector panels, side panels, and floating tool palettes use glass to feel lightweight and non-obstructive.

```swift
struct EditorView: View {
    @State private var showInspector = true

    var body: some View {
        HStack(spacing: 0) {
            CanvasView()
                .frame(maxWidth: .infinity)

            if showInspector {
                InspectorPanel()
                    .frame(width: 280)
                    .glassEffect()
                    .inspector(isPresented: $showInspector)
            }
        }
    }
}
```

### Inspector with Glass

The `.inspector(isPresented:)` modifier on macOS 26 automatically applies a glass-like appearance. For custom glass behavior:

```swift
InspectorPanel()
    .frame(width: 280)
    .glassEffect(tint: .indigo.opacity(0.1))
    .inspector(isPresented: $showInspector)
    .inspectorColumnWidth(min: 240, ideal: 280, max: 360)
```

### Floating Panels

For non-inspector floating panels (tool palettes, color pickers):

```swift
GlassEffectContainer {
    VStack(alignment: .leading, spacing: 12) {
        Text("Tools")
            .font(.headline)

        Divider()

        ToolGrid()
    }
    .padding(16)
    .frame(width: 240)
}
.windowStyle(.floating)
```

## NSVisualEffectView Bridge

When SwiftUI's glass APIs aren't sufficient — for example, when you need a specific `NSVisualEffectView.Material` not exposed in SwiftUI — wrap `NSVisualEffectView` in an `NSViewRepresentable`.

```swift
import SwiftUI
import AppKit

struct VibrancyView: NSViewRepresentable {
    var material: NSVisualEffectView.Material
    var blendingMode: NSVisualEffectView.BlendingMode = .withinWindow

    func makeNSView(context: Context) -> NSVisualEffectView {
        let view = NSVisualEffectView()
        view.material = material
        view.blendingMode = blendingMode
        view.state = .active
        return view
    }

    func updateNSView(_ nsView: NSVisualEffectView, context: Context) {
        nsView.material = material
        nsView.blendingMode = blendingMode
    }
}
```

### Usage

```swift
// Full-screen vibrancy background behind a custom view
ZStack {
    VibrancyView(material: .underWindowBackground)
        .ignoresSafeArea()

    CustomDrawingView()
        .frame(width: 400, height: 300)
}
```

**Important:** `NSVisualEffectView` does not participate in the Liquid Glass refraction system. It renders as a traditional blur. Use it only when you need specific AppKit materials that SwiftUI doesn't expose. For new chrome surfaces, prefer `.glassEffect()` and `.glassBackgroundDisplayMode()`.

## Vibrancy Materials

macOS 26 exposes vibrancy through both SwiftUI's `.glassBackgroundDisplayMode` and AppKit's `NSVisualEffectView.Material`. Here's the full material catalog for the AppKit bridge:

| Material | Description | Use Case |
|---|---|---|
| `.underWindowBackground` | Vibrancy derived from the window background. | Full-window backgrounds. |
| `.fullScreenUI` | Strong vibrancy for full-screen UI. | Media players, presentation mode. |
| `.menu` | Menu-specific vibrancy. | Custom menu backgrounds. |
| `.popover` | Popover vibrancy. | Custom popover backgrounds. |
| `.sheet` | Sheet vibrancy. | Custom sheet backgrounds. |
| `.headerView` | Header view vibrancy. | Table/collection view headers. |
| `.selection` | Selection highlight vibrancy. | Selected rows in lists. |
| `.titlebar` | Title bar vibrancy. | Custom title bar backgrounds. |
| `.windowBackground` | Standard window background vibrancy. | General-purpose backgrounds. |
| `.hudWindow` | HUD-style dark vibrancy. | Heads-up display panels. |
. `.contentBackground` | Content area vibrancy. | Scroll view backgrounds. |
| `.underPageBackground` | Under-page vibrancy. | Behind document content. |

### Choosing Between SwiftUI and AppKit Vibrancy

| Scenario | API |
|---|---|
| Sidebar, toolbar, panel chrome | `.glassEffect()` / `.glassBackgroundDisplayMode()` |
| Custom AppKit view needing specific material | `NSVisualEffectView` bridge |
| Full-window background blur | `NSVisualEffectView` with `.underWindowBackground` |
| Popover/sheet background | `.glassBackgroundDisplayMode(.vibrant)` or `.popover` material |

## Dark Mode Considerations

Liquid Glass adapts automatically between light and dark mode, but you need to account for the shift:

- **Tint colors** that look subtle in light mode may become overpowering in dark mode. Test all tint values in both appearances.
- **Text contrast** — `.foregroundStyle(.primary)` and `.foregroundStyle(.secondary)` adapt automatically. Hardcoded colors (`.foregroundStyle(.blue)`) do not.
- **Glass intensity** — the system increases blur intensity in dark mode to maintain legibility. Don't try to compensate manually.
- **Testing** — use `NSApp.appearance` to toggle between `.aqua` and `.darkAqua` during development:

```swift
// In a debug menu or preview
Button("Toggle Appearance") {
    NSApp.appearance = NSApp.appearance == .aqua ? .darkAqua : .aqua
}
```

```swift
// In SwiftUI previews
#Preview("Light") {
    ContentView()
    .preferredColorScheme(.light)
}

#Preview("Dark") {
    ContentView()
    .preferredColorScheme(.dark)
}
```

## windowBackgroundDragBehavior

By default, clicking and dragging on a glass surface does **not** move the window. To enable window dragging on glass areas, use `.windowBackgroundDragBehavior(.enabled)`:

```swift
var body: some View {
    MainView()
        .glassBackgroundDisplayMode(.vibrant)
        .windowBackgroundDragBehavior(.enabled)
}
```

This is essential for title-bar-less or borderless windows where the user needs a drag target. Without it, users can't reposition the window by dragging glass surfaces.

**Scope:** The modifier applies to the entire view hierarchy below it. Place it at the root of your window's content view.

**Interaction with controls:** Buttons and interactive elements within the glass area still receive their events normally. The drag behavior activates only on non-interactive glass regions.

## Constraints & Gotchas

- **Performance with many glass layers.** Each glass surface requires an offscreen render pass for the blur/refraction. Stacking 5+ glass surfaces (e.g., a glass sidebar inside a glass panel inside a glass toolbar) causes measurable frame drops on integrated GPU. Limit to 2-3 glass layers visible simultaneously.
- **Accessibility.** Glass reduces contrast. Ensure text on glass meets WCAG AA contrast ratios (4.5:1 for body text). Use the Accessibility Inspector to verify. The system does not auto-correct contrast on custom-tinted glass.
- **glassEffect() on non-rectangular views.** Glass renders as a rectangular bounding box. Views with corner radius or custom shapes still produce a rectangular glass surface — the shape is not clipped to the glass.
- **Animation performance.** Animating views with `.glassEffect()` (especially frame changes) is more expensive than animating opaque views. Prefer opacity animations over frame animations for glass elements.
- **NSToolbar integration.** AppKit `NSToolbar` does not automatically participate in Liquid Glass. Only SwiftUI `.toolbar {}` content with `.glassEffect()` gets the glass treatment. For AppKit toolbars, wrap the toolbar background in an `NSVisualEffectView` with `.titlebar` material.
- **Window capture.** Screen recording and window capture tools may show glass surfaces as more opaque than they appear on screen. This is expected — the capture pipeline doesn't always preserve the live refraction effect.
- **Minimum deployment target.** All glass APIs require macOS 26. There is no backward-compatible fallback. Use `#available(macOS 26, *)` and provide opaque fallbacks for older OS versions.

## Common Mistakes

1. **Applying glass to content, not chrome.** Glass on a document editor, image canvas, or data table obscures the content the user needs to see. Glass is for surrounding chrome — toolbars, sidebars, panels, overlays.

2. **Heavy tint opacity.** `tint: .blue.opacity(0.8)` kills the refractive quality and looks like a colored overlay, not glass. Keep tint opacity below 0.3.

3. **Forgetting GlassEffectContainer.** Applying `.glassEffect()` to individual buttons without a container produces disconnected glass tiles. Wrap related controls in a `GlassEffectContainer` for a unified surface.

4. **Hardcoded colors on glass.** `.foregroundStyle(.white)` on glass becomes invisible in light mode. Always use semantic styles (`.primary`, `.secondary`, `.tertiary`) on glass surfaces.

5. **Mixing glass and non-glass in the same visual group.** A toolbar where some buttons have `.glassEffect()` and others don't creates visual inconsistency. Apply glass uniformly within a container.

6. **Ignoring dark mode.** Glass that looks elegant in light mode can become a muddy, low-contrast mess in dark mode. Always test both.

7. **Using NSVisualEffectView when SwiftUI glass suffices.** If you're building a new SwiftUI toolbar or sidebar, use `.glassEffect()` — don't reach for `NSVisualEffectView` unless you need a specific AppKit-only material.

## Review Checklist

- [ ] Glass is applied to chrome (toolbars, sidebars, panels), not to content areas.
- [ ] Related glass controls are grouped in a `GlassEffectContainer`.
- [ ] Tint opacity is ≤ 0.3 for refractive quality.
- [ ] Text on glass uses semantic foreground styles (`.primary`, `.secondary`).
- [ ] Appearance tested in both light and dark mode.
- [ ] Accessibility contrast verified with Accessibility Inspector.
- [ ] No more than 2-3 glass layers stacked simultaneously.
- [ ] `.windowBackgroundDragBehavior(.enabled)` set if the window needs drag-on-glass.
- [ ] `#available(macOS 26, *)` guards with opaque fallbacks for multi-target apps.
- [ ] `NSVisualEffectView` bridge used only for AppKit-specific materials, not as a default.

## References

- [Applying Liquid Glass to custom views — Apple Developer](https://developer.apple.com/documentation/SwiftUI/Applying-Liquid-Glass-to-custom-views)
- [Adopting Liquid Glass — Apple Developer](https://developer.apple.com/documentation/TechnologyOverviews/adopting-liquid-glass)
- [Build a SwiftUI app with the new design — WWDC25](https://developer.apple.com/videos/play/wwdc2025/323/)
- [GlassBackgroundDisplayMode — Apple Developer](https://developer.apple.com/documentation/swiftui/glassbackgrounddisplaymode)
- [NSVisualEffectView — Apple Developer](https://developer.apple.com/documentation/appkit/nsvisualeffectview)
- [Customizing window styles — Apple Developer](https://developer.apple.com/documentation/swiftui/customizing-window-styles-and-state-restoration-behavior-in-macos)
- [SwiftUI Liquid Glass Guide — Atelier Socle](https://www.atelier-socle.com/en/articles/swiftui-liquid-glass-guide)
- [references/glass-patterns.md](references/glass-patterns.md) — Deep-dive patterns: GlassEffectContainer with toolbar buttons, translucent sidebar with vibrancy, NSVisualEffectView wrapper for custom vibrancy, inspector panel with glass background.
