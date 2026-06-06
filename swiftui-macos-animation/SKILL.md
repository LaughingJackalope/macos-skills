---
name: swiftui-macos-animation
description: "SwiftUI animations on macOS — withAnimation, Animation types, matchedGeometryEffect, PhaseAnimator, KeyframeAnimator, transitions, spring/interactiveSpring, reduced motion (accessibilityReduceMotion), window open/close transitions, custom sheet/popover transitions. Use when adding animations to macOS SwiftUI views or windows."
sources: https://developer.apple.com/documentation/swiftui/animation, https://developer.apple.com/documentation/swiftui/withanimation(), https://developer.apple.com/documentation/swiftui/view/matchedgeometryeffect(id:in:properties:anchor:issource:), https://developer.apple.com/documentation/swiftui/phaseanimator, https://developer.apple.com/documentation/swiftui/keyframeanimator, https://developer.apple.com/documentation/swiftui/EnvironmentValues/accessibilityreducemotion, https://developer.apple.com/SwiftUI/Controlling-the-timing-and-movements-of-your-animations, https://www.hackingwithswift.com/articles/278/whats-new-in-swiftui-for-ios-26, https://developer.apple.com/videos/play/wwdc2025/256/
researched: 2026-06-06
platform: macOS 26+
---

# SwiftUI Animation on macOS

Animating views and windows in SwiftUI on macOS. Covers `withAnimation`, `Animation` types, `matchedGeometryEffect`, `PhaseAnimator`, `KeyframeAnimator`, transitions, spring tuning, reduced motion, and window/sheet presentation. Targets macOS 26+ with Swift 6.3; version notes where APIs require macOS 13+ or 14+.

For AppKit-driven animations or `NSViewRepresentable`-hosted views where AppKit animation is needed, see `swiftui-appkit-interop`. For window management and scene types, see `macos-windows-scenes`.

See [references/animation-patterns.md](references/animation-patterns.md) for complete patterns including sidebar-to-detail hero animation, loading state phases, complex keyframe choreography, reduced motion fallbacks, and spring tuning.

## Contents

- [withAnimation & Animation Types](#withanimation--animation-types)
- [matchedGeometryEffect](#matchedgeometryeffect)
- [PhaseAnimator](#phaseanimator)
- [KeyframeAnimator](#keyframeanimator)
- [Transitions](#transitions)
- [macOS Window Transitions](#macos-window-transitions)
- [Sheet & Popover Transitions](#sheet--popover-transitions)
- [Reduced Motion](#reduced-motion)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

---

## withAnimation & Animation Types

### Implicit Animations

Apply `.animation(_:value:)` to a view. SwiftUI automatically animates changes to animatable properties when the watched value changes. Only use for simple, single-property animations — implicit animations compose poorly and can trigger unexpectedly when the view hierarchy changes.

```swift
struct ToggleView: View {
    @State private var isExpanded = false

    var body: some View {
        VStack {
            Button(isExpanded ? "Collapse" : "Expand") {
                isExpanded.toggle()
            }

            if isExpanded {
                DetailPanel()
                    .animation(.easeInOut(duration: 0.3), value: isExpanded)
            }
        }
    }
}
```

**Gotcha:** Implicit `.animation()` applies to all animatable changes on the view and its subviews. If multiple values change at once (e.g., frame + opacity), they all animate. For precise control, prefer explicit `withAnimation`.

### Explicit withAnimation

`withAnimation` wraps state changes so SwiftUI animates the resulting view updates. This is the primary tool for controlled animations:

```swift
Button("Animate") {
    withAnimation(.spring(duration: 0.6, bounce: 0.3)) {
        isVisible.toggle()
        offset = isVisible ? 0 : 100
    }
}
```

Wrap only the state mutations you want animatable. Multiple state changes inside a single `withAnimation` block are batched into one animation transaction.

### Animation Curves and Presets

| Animation | Behavior | Best For |
|-----------|----------|----------|
| `.easeInOut` | Slow start and end, fast middle | General-purpose; default for most UI changes |
| `.easeIn` | Slow start, abrupt end | Elements entering the screen |
| `.easeOut` | Abrupt start, slow end; decelerates | Elements leaving; menu dismissal |
| `.linear` | Constant speed | Progress indicators, continuous rotation |
| `.spring(duration:bounce:)` | Spring with duration + bounce control | Playful UI, toggles, cards |
| `.spring(response:dampingFraction:blendDuration:)` | Full spring physics tuning | Precise spring feel matching platform |
| `.interactiveSpring(response:dampingFraction:blendDuration:)` | Lower response for interactive (drag-coupled) animation | Follow-finger or gesture-driven animation |
| `.bouncy` | High bounce preset | macOS 14+; playful elements |
| `.smooth` | No bounce, gentle deceleration | macOS 14+; subtle property transitions |
| `.snappy` | Fast, minimal overshoot | macOS 14+; quick toggles, switches |
| `.timingCurve(_:_:_:_:)` | Custom cubic Bézier control points | Replicating platform motion curves |

```swift
// Spring variations compared
withAnimation(.spring(response: 0.5, dampingFraction: 0.8)) { ... } // Smooth settle
withAnimation(.spring(duration: 0.4, bounce: 0.4)) { ... }          // Bouncy
withAnimation(.interactiveSpring(response: 0.4, dampingFraction: 0.85)) { ... } // Drag feel
withAnimation(.snappy(duration: 0.25)) { ... }                       // Fast toggle
withAnimation(.timingCurve(0.25, 0.1, 0.25, 1.0)) { ... }          // Custom curve
```

### Modifying Animation Behavior

Chain modifiers on `Animation` values for additional control:

```swift
var body: some View {
    AnimatedCircle()
        .animation(
            .easeInOut(duration: 0.4)
                .delay(0.1)         // Delay before animation begins
                .repeatCount(3, autoreverses: true)
                .repeatForever(autoreverses: true),
            value: trigger
        )
}
```

- `.delay(_:)` — waits before starting.
- `.repeatCount(_:autoreverses:)` — play N times; if `autoreverses` reverses direction each cycle.
- `.repeatForever(autoreverses:)` — loops indefinitely. Use with caution: the animation holds a reference to the view.
- `.speed(_:)` — multiplies speed (>1 = faster, <1 = slower).

**Repeat forever cleanup:** Views with `.repeatForever` animations are not destroyed if removed from the hierarchy while hidden. Use a `@State` toggle to stop infinite animations explicitly.

---

## matchedGeometryEffect

`matchedGeometryEffect` creates a geometry link between two views in different parts of the hierarchy. During a transition, SwiftUI interpolates frame, position, and size — producing a "hero" animation where one view appears to morph into another.

### Basic Setup

Requires three things: a shared `Namespace`, `.matchedGeometryEffect(id:in:)` on both source and target views, and a state change that swaps which view is visible.

```swift
struct GalleryView: View {
    @Namespace private var imageTransition
    @State private var selectedPhoto: Photo?

    var body: some View {
        ZStack {
            // Thumbnail grid
            if selectedPhoto == nil {
                Grid {
                    ForEach(photos) { photo in
                        Thumbnail(photo: photo)
                            .matchedGeometryEffect(id: photo.id, in: imageTransition)
                            .onTapGesture {
                                withAnimation(.spring(duration: 0.5, bounce: 0.2)) {
                                    selectedPhoto = photo
                                }
                            }
                    }
                }
            }

            // Fullscreen detail
            if let selectedPhoto {
                PhotoDetail(photo: selectedPhoto)
                    .matchedGeometryEffect(id: selectedPhoto.id, in: imageTransition)
                    .onTapGesture {
                        withAnimation(.spring(duration: 0.5, bounce: 0.2)) {
                            self.selectedPhoto = nil
                        }
                    }
            }
        }
    }
}
```

### Sidebar-to-Detail Transition

In a `NavigationSplitView`, use matched geometry to create a hero transition from a sidebar row to a detail view:

```swift
struct BrowserView: View {
    @Namespace private var navTransition
    @State private var selectedItem: Item?

    var body: some View {
        NavigationSplitView {
            List(selection: $selectedItem) {
                ForEach(items) { item in
                    ItemRow(item: item)
                        .matchedGeometryEffect(
                            id: "row-\(item.id)",
                            in: navTransition,
                            properties: .position
                        )
                }
            }
        } detail: {
            if let selectedItem {
                ItemDetail(item: selectedItem)
                    .matchedGeometryEffect(
                        id: "row-\(selectedItem.id)",
                        in: navTransition,
                        properties: .position
                    )
                    .toolbar {
                        ToolbarItem(placement: .navigation) {
                            Button("Back") {
                                withAnimation(.spring(duration: 0.4, bounce: 0.15)) {
                                    self.selectedItem = nil
                                }
                            }
                        }
                    }
            }
        }
    }
}
```

### Z-Index Control

Without explicit z-ordering, the receiving view may render behind siblings during the transition. Use `.zIndex(_:)` to control depth during the animation:

```swift
ForEach(items) { item in
    ItemCard(item: item)
        .matchedGeometryEffect(id: item.id, in: namespace)
        .zIndex(selectedItem == item ? 1 : 0)
}
```

### Properties

The `properties` parameter controls which aspects of geometry are interpolated:

| Property | Interpolates |
|----------|-------------|
| `.position` | Center position only — view size stays constant |
| `.size` | Width and height only — position stays constant |
| `.positionAndSize` | Both — full frame morph (default) |

**Important constraints:**

- Only one view per namespace ID may be visible at a time. If both source and target exist simultaneously, the behavior is undefined.
- The matched geometry effect only works when one view appears and the other disappears within the same animation transaction.
- Do not apply `matchedGeometryEffect` to views inside a `List` cell that has selection highlighting — the cell's background modifier interferes with implicit animations. Wrap the matched view in a `ZStack` inside the cell.
- The `isSource:` parameter marks which view provides the geometry source. Exactly one view per ID should be the source. Consumers (non-source views) adopt the source's geometry.
- Use the `anchor:` parameter to control alignment point (.center, .topLeading, etc.) for precise positioning during the morph.

---

## @Animatable Macro

The `@Animatable` macro (macOS 26+) automatically synthesizes `Animatable` conformance for custom shapes and views. It eliminates manual `animatableData` boilerplate:

```swift
// Before (macOS 13–25):
struct Wave: Shape, Animatable {
    var amplitude: Double
    var phase: Double

    var animatableData: AnimatablePair<Double, Double> {
        get { AnimatablePair(amplitude, phase) }
        set { amplitude = newValue.first; phase = newValue.second }
    }

    func path(in rect: CGRect) -> Path { /* ... */ }
}

// After (macOS 26+):
@Animatable
struct Wave: Shape {
    var amplitude: Double
    var phase: Double

    func path(in rect: CGRect) -> Path { /* ... */ }
}
```

Use `@AnimatableIgnored` on properties that should not animate:

```swift
@Animatable
struct LoadingArc: Shape {
    var startAngle: Double
    var endAngle: Double
    @AnimatableIgnored var drawClockwise: Bool  // Not animated

    func path(in rect: CGRect) -> Path {
        Path { p in
            p.addArc(center: ..., radius: ..., startAngle: ..., endAngle: ..., clockwise: drawClockwise)
        }
    }
}
```

## PhaseAnimator

`PhaseAnimator` (SwiftUI 5 / macOS 14+) cycles through a sequence of phases, providing each phase's value to a view builder. Unlike `withAnimation` (which animates between two states), PhaseAnimator drives multi-step animations declaratively.

### Basic Usage

```swift
struct PulsingIndicator: View {
    var body: some View {
        PhaseAnimator([0, 1, 0, 1]) { phase in
            Circle()
                .fill(.blue)
                .scaleEffect(phase == 1 ? 1.4 : 1.0)
                .opacity(phase == 1 ? 1.0 : 0.4)
        } animation: { phase in
            .easeInOut(duration: 0.6)
        }
    }
}
```

Each phase value is passed to both the view builder and the animation closure. The animation closure returns the animation to use *from the previous phase to this one*.

### Trigger-Based Phase Animation

Use a `trigger` value to restart the animation when an external condition changes:

```swift
struct LoadingDots: View {
    @State private var triggerCount = 0

    var body: some View {
        HStack(spacing: 8) {
            ForEach(0..) { index in
                PhaseAnimator([0, 1, 0], trigger: triggerCount) { phase in
                    Text("•")
                        .font(.largeTitle)
                        .opacity(phase == 1 ? 1.0 : 0.3)
                        .offset(y: phase == 1 ? -8 : 0)
                } animation: { _ in
                    .easeInOut(duration: 0.4)
                        .delay(Double(index) * 0.15)
                }
            }
        }
        .onAppear {
            // Restart periodically
            Timer.scheduledTimer(withTimeInterval: 2.0, repeats: true) { _ in
                triggerCount += 1
            }
        }
    }
}
```

### macOS Use Cases

- **Loading states:** Progress indicators, skeleton shimmer, sequential reveal.
- **Attention indicators:** Badge bounce, notification ping, selection highlight.
- **Multi-step morphing:** Icon state changes (play → pause → loading → done).

```swift
// Attention pulse that fires once when a new item arrives
struct NewItemBadge: View {
    let newItem: Item

    var body: some View {
        PhaseAnimator([0, 1, 2], trigger: newItem.id) { phase in
            Label(newItem.name, systemImage: "sparkles")
                .padding(.horizontal, 12)
                .padding(.vertical, 6)
                .background(.blue.opacity(phase == 1 ? 0.3 : 0.1), in: Capsule())
                .scaleEffect(phase == 1 ? 1.1 : 1.0)
        } animation: { phase in
            switch phase {
            case 1: return .spring(duration: 0.3, bounce: 0.5)
            case 2: return .easeOut(duration: 0.2)
            default: return .linear(duration: 0)
            }
        }
    }
}
```

---

## KeyframeAnimator

`KeyframeAnimator` (SwiftUI 5 / macOS 14+) provides fine-grained control over multi-step animations using `KeyframeTrack`. Each keyframe specifies a value at a relative time (0.0–1.0) with its own interpolation.

### KeyframeTrack Structure

```swift
struct ComplexAnimation: View {
    @State private var trigger = false

    var body: some View {
        KeyframeAnimator(initialValue: 0.0, trigger: trigger) { value in
            RoundedRectangle(cornerRadius: 12)
                .fill(.blue)
                .rotationEffect(.degrees(value))
                .offset(x: value * 2)
        } keyframes: { _ in
            KeyframeTrack(\.self) {
                CubicKeyframe(0.0, relativeTime: 0.0)
                CubicKeyframe(45.0, relativeTime: 0.25)
                CubicKeyframe(45.0, relativeTime: 0.5)
                CubicKeyframe(-20.0, relativeTime: 0.75)
                CubicKeyframe(0.0, relativeTime: 1.0)
            }
        }
    }
}
```

### Keyframe Types

| Type | Behavior | Use When |
|------|----------|----------|
| `LinearKeyframe(value:relativeTime:)` | Linear interpolation from previous keyframe | Constant-speed motion |
| `CubicKeyframe(value:relativeTime:)` | Smooth cubic Bézier ease | Natural-feeling motion |
| `SpringKeyframe(value:relativeTime:duration:)` | Spring physics from previous keyframe | Bouncy, organic motion |
| `MoveKeyframe(value:relativeTime:)` | Instant jump (no interpolation) | Discrete state changes |

### Multi-Property Keyframes

Use multiple `KeyframeTrack` instances to animate different properties with independent timing:

```swift
KeyframeAnimator(initialValue: AnimationState(), trigger: trigger) { state in
    CardView()
        .rotation3DEffect(.degrees(state.rotationY), axis: (x: 0, y: 1, z: 0))
        .scaleEffect(state.scale)
        .opacity(state.opacity)
} keyframes: { _ in
    KeyframeTrack(\.rotationY) {
        CubicKeyframe(0, relativeTime: 0)
        SpringKeyframe(360, relativeTime: 0.5, duration: 0.4)
        CubicKeyframe(360, relativeTime: 1.0)
    }
    KeyframeTrack(\.scale) {
        LinearKeyframe(1.0, relativeTime: 0)
        SpringKeyframe(1.3, relativeTime: 0.3, duration: 0.3)
        LinearKeyframe(1.0, relativeTime: 1.0)
    }
    KeyframeTrack(\.opacity) {
        MoveKeyframe(0.0, relativeTime: 0)
        LinearKeyframe(1.0, relativeTime: 0.15)
        MoveKeyframe(1.0, relativeTime: 0.85)
        LinearKeyframe(0.0, relativeTime: 1.0)
    }
}
```

**Key rules:**

- `relativeTime` must be monotonically increasing within a track.
- The first keyframe should start at `relativeTime: 0.0`.
- The last keyframe should end at `relativeTime: 1.0` for a complete cycle.
- `SpringKeyframe` uses `duration` as the spring's own duration, not relative to the overall timeline.

---

## Transitions

Transitions control how views appear and disappear in the hierarchy. Apply with `.transition(_:)` on conditional views.

### Built-in Transitions

| Transition | Effect |
|-----------|--------|
| `.opacity` | Fade in/out |
| `.slide` | Slide from leading edge, slide out to leading edge |
| `.scale` | Scale from 0 to full (or reverse) |
| `.move(edge:)` | Slide from a specific edge |
| `.offset(x:y:)` | Translate by offset |
| `.blurReplace` | Blur transition (macOS 14+) |
| `.push(from:)` | Push new view in, old view out |

```swift
if isVisible {
    DetailView()
        .transition(.move(edge: .trailing).combined(with: .opacity))
}
```

### Asymmetric Transitions

Use different transitions for insertion and removal:

```swift
.transition(.asymmetric(
    insertion: .scale(scale: 0.1).combined(with: .opacity),
    removal: .slide.combined(with: .opacity)
))
```

### Combining Transitions

Chain transitions with `.combined(with:)`:

```swift
.transition(
    .scale(scale: 0.8)
        .combined(with: .opacity)
        .combined(with: .offset(y: 20))
)
```

### Custom Transitions with `AnyTransition`

Extend `AnyTransition` with custom modifiers:

```swift
extension AnyTransition {
    static var slideAndFade: AnyTransition {
        .asymmetric(
            insertion: .move(edge: .bottom).combined(with: .opacity),
            removal: .move(edge: .top).combined(with: .opacity)
        )
    }

    static func pivot(angle: Double) -> AnyTransition {
        .modifier(
            active: PivotModifier(angle: angle),
            identity: PivotModifier(angle: 0)
        )
    }
}

struct PivotModifier: ViewModifier {
    let angle: Double

    func body(content: Content) -> some View {
        content
            .rotationEffect(.degrees(angle), anchor: .bottomLeading)
            .clipped()
    }
}
```

**Transition constraints:**

- Transitions only work inside containers that animate their children: `ZStack`, `VStack`, `HStack`, `Group`, or `if/else` blocks. They do NOT work in `List` rows or `ForEach` without an explicit animation context.
- The container must have an animation context (either `withAnimation` or `.animation(_:value:)`).
- Transitions on views inside `NavigationSplitView` detail column may not animate as expected — the detail column uses its own transition logic. Use `withAnimation` around the selection change.

---

## macOS Window Transitions

### Default Window Open/Close Behavior

macOS applies default animations when windows appear and disappear. `WindowGroup` windows animate in with a subtle scale+fade from the position where they're placed. `Window` scenes use the same animation. These are managed by AppKit's window server and cannot be directly customized through SwiftUI's animation API.

### Customizing Window Animation Behavior

Control window placement and default size to influence the perceived animation:

```swift
WindowGroup {
    ContentView()
}
.defaultSize(width: 900, height: 600)
.defaultPosition(.center)
```

For window-level animation control, use `NSWindow` appearance customization via `AppKit` interop:

```swift
// In AppDelegate or scene delegate
func applicationDidFinishLaunching(_ notification: Notification) {
    // Adjust window animation speed
    NSAnimationContext.runAnimationGroup { context in
        context.duration = 0.25
    }
}
```

### Window Transition Gotchas

- **No SwiftUI-level window transition API.** Unlike view transitions, window open/close animations are controlled by AppKit. You cannot replace the default scale+fade with a custom SwiftUI transition.
- **Window restoration.** When macOS restores windows on relaunch, the animation is skipped — windows appear instantly at their saved positions.
- **`openWindow` / `dismissWindow`.** These environment actions trigger the standard window animation. There is no parameter to customize the animation curve.

---

## Sheet & Popover Transitions

### Default Sheet Animation

Sheets on macOS slide up from the bottom of the parent window. This is the system default and uses AppKit's sheet presentation animation.

### Custom Sheet Presentation

For custom sheet transitions, use `.presentationDetents(_:)` combined with `.presentationBackground(_:)` and `.presentationCornerRadius(_:)` (macOS 14+):

```swift
.sheet(isPresented: $showingSheet) {
    SheetContent()
        .presentationDetents([.medium, .large])
        .presentationBackground(.regularMaterial)
        .presentationCornerRadius(16)
        .presentationDragIndicator(.visible)
}
```

### Popover Transitions

Popovers use a scale+fade animation by default. Control the permitted arrow edges:

```swift
.popover(isPresented: $showingPopover, arrowEdge: .trailing) {
    PopoverContent()
        .frame(width: 280, height: 200)
}
```

### Custom Transition for Sheet-Like Presentation

For a fully custom presentation (e.g., a panel that slides from the side), avoid `.sheet` and instead use a `ZStack` overlay with a custom transition:

```swift
struct SidePanel<Content: View>: View {
    @Binding var isPresented: Bool
    let content: Content

    var body: some View {
        ZStack(alignment: .trailing) {
            // Main content
            Color.clear

            // Panel overlay
            if isPresented {
                content
                    .frame(width: 320)
                    .background(.regularMaterial)
                    .transition(.move(edge: .trailing).combined(with: .opacity))
                    .zIndex(1)
            }
        }
        .animation(.spring(duration: 0.4, bounce: 0.1), value: isPresented)
    }
}
```

**Sheet transition constraints:**

- `.presentationDetents` on macOS only supports `.medium` and `.large` — custom detents are iOS-only.
- Sheet animations cannot be replaced with custom SwiftUI transitions. The sheet presentation is managed by AppKit.
- For popovers, the `.transition()` modifier has no effect — popover animation is controlled by AppKit.

---

## Reduced Motion

### Checking the Accessibility Setting

Use `@Environment(\.accessibilityReduceMotion)` to detect the user's reduced motion preference:

```swift
struct AnimatedView: View {
    @Environment(\.accessibilityReduceMotion) private var reduceMotion
    @State private var isActive = false

    var body: some View {
        Circle()
            .scaleEffect(isActive ? 1.5 : 1.0)
            .animation(
                reduceMotion ? .none : .spring(duration: 0.5, bounce: 0.3),
                value: isActive
            )
    }
}
```

### `prefersReducedMotion` on Animation

SwiftUI 5 / macOS 14+ provides `.animation(_:value:)` that respects the setting automatically when you use the `prefersReducedMotion` parameter:

```swift
.animation(
    .spring(duration: 0.5, bounce: 0.3),
    value: isActive
)
// This already respects accessibilityReduceMotion in most cases.
// For explicit control, check the environment value.
```

### Graceful Degradation Pattern

For complex animations, provide a simplified or instant alternative:

```swift
struct LoadingView: View {
    @Environment(\.accessibilityReduceMotion) private var reduceMotion

    var body: some View {
        if reduceMotion {
            // Static or minimal animation
            ProgressView()
                .progressViewStyle(.circular)
        } else {
            // Full animation
            PhaseAnimator([0, 1, 2]) { phase in
                LoadingDots()
                    .opacity(phase == 1 ? 0.3 : 1.0)
            } animation: { _ in .easeInOut(duration: 0.4) }
        }
    }
}
```

### Reduced Motion Best Practices

- **Never remove all feedback.** Replace motion with opacity changes, color shifts, or static state indicators.
- **Test with "Reduce motion" enabled** in System Settings → Accessibility → Display.
- **PhaseAnimator and KeyframeAnimator** should be replaced with static views or simple opacity transitions.
- **matchedGeometryEffect** should be replaced with a simple `.opacity` crossfade.
- **Window and sheet transitions** are automatically reduced by the system — no code change needed.

---

## Constraints & Gotchas

### Animation Performance

- **Prefer animatable properties.** `offset`, `opacity`, `scaleEffect`, `rotationEffect`, and `frame` are GPU-accelerated. Avoid animating `padding`, `layoutPriority`, or any property that triggers a layout pass.
- **Use `.drawingGroup()`** for complex views that animate together. This flattens the view hierarchy into a single Metal-rendered layer. Use sparingly — it increases memory usage.
- **Avoid animating `List` or `ScrollView` content directly.** These containers manage their own layout and can conflict with SwiftUI's animation system. Animate the data model, not the container.

### AppKit Interop

- **NSViewRepresentable views do not participate in SwiftUI animations.** If you need to animate an AppKit view's property, use `NSAnimationContext` inside the representable's coordinator.
- **Layer-backed views.** Setting `wantsLayer = true` on an `NSView` inside `NSViewRepresentable` enables Core Animation, which can conflict with SwiftUI's animation timing. Prefer one system or the other.

### Coordinate Space

- **matchedGeometryEffect** requires both views to be in the same coordinate space context. If one view is inside a `ScrollView` and the other is not, the geometry matching may produce unexpected results.
- **GeometryReader breaks matched geometry.** Do not wrap matched views in `GeometryReader` — the reader's coordinate space transformation interferes with the geometry matching calculation.

### Transaction and State

- **Nested `withAnimation` blocks.** The innermost `withAnimation` wins. If you need different animations for different properties, use `withAnimation` for the outer state and explicit `.animation(_:value:)` modifiers for individual views.
- **Transaction flags.** Use `withAnimation(.none)` to suppress animations for a specific state change, even when an outer animation context is active.

---

## Common Mistakes

1. **Using implicit `.animation()` without a `value` parameter.** The deprecated `.animation(Animation)` (without `value:`) animates everything and causes unpredictable behavior. Always use `.animation(_:value:)`.

2. **Forgetting `withAnimation` around state changes.** Transitions and animations only fire when the state change happens inside an animation context. Changing a `@State` without `withAnimation` produces an instant update.

3. **matchedGeometryEffect with both views visible.** The source and target must never be in the hierarchy simultaneously. Use `if/else` or `ZStack` with opacity 0 — not two visible views with the same ID.

4. **Animating non-animatable properties.** `font`, `foregroundColor`, `padding`, and `layoutPriority` are not implicitly animatable. Wrap them in a `Group` and animate `opacity` or `scaleEffect` instead.

5. **Ignoring reduced motion.** Always check `@Environment(\.accessibilityReduceMotion)` for non-decorative animations. Users with vestibular disorders experience discomfort from excessive motion.

6. **Using `.repeatForever` without a stop condition.** Infinite animations hold strong references and prevent view deallocation. Always provide a way to stop them.

7. **Applying transitions to `List` rows.** Transitions on `List` row content are ignored. Use `withAnimation` on the data source change instead, and the `List` will animate insertions/removals automatically.

8. **Overusing spring animations.** Every element bouncing creates visual noise. Reserve springs for interactive or playful elements; use `.easeInOut` or `.snappy` for standard UI transitions.

---

## Review Checklist

- [ ] All implicit animations use `.animation(_:value:)` with an explicit value parameter
- [ ] `withAnimation` wraps state changes that should animate
- [ ] `matchedGeometryEffect` IDs are unique and only one view per ID is visible at a time
- [ ] `PhaseAnimator` phases are monotonically progressing and the animation closure returns appropriate curves
- [ ] `KeyframeAnimator` keyframe tracks have monotonically increasing `relativeTime` values
- [ ] Transitions are applied inside an animation context (`withAnimation` or `.animation(_:value:)`)
- [ ] Asymmetric transitions define both `insertion` and `removal`
- [ ] `@Environment(\.accessibilityReduceMotion)` is checked for all non-decorative animations
- [ ] Reduced motion fallback provides visual feedback (not just removal of animation)
- [ ] No `.repeatForever` without a stop condition
- [ ] Animatable properties are GPU-friendly (offset, opacity, scale, rotation — not padding or layout)
- [ ] Window/sheet transitions respect that AppKit controls the presentation animation
- [ ] Tested with "Reduce motion" enabled in System Settings

---

## References

- [Apple Docs: Animation](https://developer.apple.com/documentation/swiftui/animation)
- [Apple Docs: withAnimation](https://developer.apple.com/documentation/swiftui/withanimation())
- [Apple Docs: matchedGeometryEffect](https://developer.apple.com/documentation/swiftui/view/matchedgeometryeffect(id:in:properties:anchor:issource:))
- [Apple Docs: PhaseAnimator](https://developer.apple.com/documentation/swiftui/phaseanimator)
- [Apple Docs: KeyframeAnimator](https://developer.apple.com/documentation/swiftui/keyframeanimator)
- [Apple Docs: AnyTransition](https://developer.apple.com/documentation/swiftui/anytransition)
- [Apple Docs: accessibilityReduceMotion](https://developer.apple.com/documentation/swiftui/environmentvalues/accessibilityreducemotion)
- [Apple Docs: animation(_:value:)](https://developer.apple.com/documentation/swiftui/view/animation(_:value:))
- [Apple HIG: Animation](https://developer.apple.com/design/human-interface-guidelines/animation)
- [Controlling animation timing — Apple Developer](https://developer.apple.com/SwiftUI/Controlling-the-timing-and-movements-of-your-animations)
- [What's new in SwiftUI — WWDC25](https://developer.apple.com/videos/play/wwdc2025/256/)
- [How to create animatable views — Hacking with Swift](https://www.hackingwithswift.com/quick-start/swiftui/how-to-create-animatable-views-modifiers-and-more)
- [prefersCrossFadeTransitions — Apple Developer](https://developer.apple.com/documentation/uikit/uiaccessibility/preferscrossfadetransitions)
- [interactiveSpring — Apple Developer](https://developer.apple.com/documentation/SwiftUI/Animation/interactiveSpring(response:dampingFraction:blendDuration:))
- [Spring Animations Guide — Swift Crafted](https://swiftcrafted.dev/article/swiftui-animations-complete-guide-springs-keyframes-transitions)
- [Supporting Reduced Motion — Wesley de Groot](https://wesleydegroot.nl/blog/supporting-reduced-motion-accessibility-setting-in-swiftui)
