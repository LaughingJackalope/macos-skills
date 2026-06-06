# SwiftUI Animation Patterns — Reference

Complete working patterns for common macOS animation scenarios. Swift 6.3 / macOS 26+.

---

## 1. matchedGeometryEffect for Sidebar-to-Detail Transition

Hero animation from a sidebar row to a full detail view using `NavigationSplitView`. The row's icon and title smoothly expand into the detail header.

```swift
import SwiftUI

struct BrowserView: View {
    @Namespace private var hero
    @State private var selectedItem: Item?

    var body: some View {
        NavigationSplitView {
            List(selection: $selectedItem) {
                ForEach(items) { item in
                    HStack(spacing: 12) {
                        ItemIcon(item: item)
                            .matchedGeometryEffect(
                                id: "icon-\(item.id)",
                                in: hero,
                                properties: .positionAndSize
                            )
                        VStack(alignment: .leading) {
                            Text(item.title)
                                .font(.headline)
                                .matchedGeometryEffect(
                                    id: "title-\(item.id)",
                                    in: hero,
                                    properties: .position
                                )
                            Text(item.subtitle)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                        Spacer()
                    }
                    .padding(.vertical, 4)
                    .contentShape(Rectangle())
                    .onTapGesture {
                        withAnimation(.spring(duration: 0.45, bounce: 0.15)) {
                            selectedItem = item
                        }
                    }
                }
            }
            .navigationTitle("Items")
        } detail: {
            if let item = selectedItem {
                DetailView(item: item, namespace: hero)
                    .toolbar {
                        ToolbarItem(placement: .navigation) {
                            Button("Back") {
                                withAnimation(.spring(duration: 0.4, bounce: 0.1)) {
                                    selectedItem = nil
                                }
                            }
                        }
                    }
            } else {
                ContentUnavailableView(
                    "Select an Item",
                    systemImage: "sidebar.left"
                )
            }
        }
    }
}

struct DetailView: View {
    let item: Item
    var namespace: Namespace.ID

    var body: some View {
        ScrollView {
            VStack(alignment: .leading, spacing: 16) {
                // Hero header — matched geometry target
                HStack(spacing: 16) {
                    ItemIcon(item: item)
                        .matchedGeometryEffect(
                            id: "icon-\(item.id)",
                            in: namespace,
                            properties: .positionAndSize
                        )
                    Text(item.title)
                        .font(.largeTitle)
                        .fontWeight(.bold)
                        .matchedGeometryEffect(
                            id: "title-\(item.id)",
                            in: namespace,
                            properties: .position
                        )
                }
                .padding(.bottom, 8)

                // Detail content
                Text(item.body)
                    .font(.body)
            }
            .frame(maxWidth: .infinity, alignment: .leading)
            .padding(24)
        }
    }
}
```

**Key points:**

- The `Namespace` is created in the parent and passed to the detail view. Namespaces cannot be created inside `@ViewBuilder` closures — they must be `@State` or `@Namespace` on a real view.
- Use `.position` for text elements (prevents font-size interpolation artifacts) and `.positionAndSize` for icons or shapes.
- The `if/else` in the detail column ensures only one of the matched views exists at a time.
- `.contentShape(Rectangle())` on the row ensures the tap target covers the full row width.

---

## 2. PhaseAnimator for Loading States

A multi-phase loading indicator that cycles through visual states, with a completion phase that transitions to the loaded content.

```swift
struct ContentLoader: View {
    @State private var loadPhase: LoadPhase = .idle
    @State private var items: [Item] = []

    enum LoadPhase: Int, CaseIterable {
        case idle = 0
        case connecting = 1
        case fetching = 2
        case processing = 3
        case complete = 4
    }

    var body: some View {
        Group {
            if loadPhase == .complete {
                ItemList(items: items)
                    .transition(.opacity.combined(with: .scale(scale: 0.98)))
            } else {
                LoadingState(phase: loadPhase)
            }
        }
        .animation(.easeInOut(duration: 0.3), value: loadPhase)
        .task {
            await performLoad()
        }
    }

    private func performLoad() async {
        withAnimation { loadPhase = .connecting }
        try? await Task.sleep(for: .milliseconds(800))

        withAnimation { loadPhase = .fetching }
        let fetched = await DataService.fetchItems()
        try? await Task.sleep(for: .milliseconds(600))

        withAnimation { loadPhase = .processing }
        try? await Task.sleep(for: .milliseconds(400))

        items = fetched
        withAnimation(.spring(duration: 0.4, bounce: 0.2)) {
            loadPhase = .complete
        }
    }
}

struct LoadingState: View {
    let phase: ContentLoader.LoadPhase

    var body: some View {
        VStack(spacing: 20) {
            PhaseAnimator(ContentLoader.LoadPhase.allCases) { currentPhase in
                VStack(spacing: 16) {
                    // Animated icon
                    Image(systemName: iconForPhase(currentPhase))
                        .font(.system(size: 40))
                        .foregroundStyle(.blue)
                        .scaleEffect(currentPhase == phase ? 1.0 : 0.6)
                        .opacity(currentPhase <= phase ? 1.0 : 0.3)

                    // Phase label
                    Text(labelForPhase(currentPhase))
                        .font(.headline)
                        .foregroundStyle(currentPhase <= phase ? .primary : .secondary)
                }
            } animation: { currentPhase in
                currentPhase == .complete
                    ? .spring(duration: 0.5, bounce: 0.3)
                    : .easeInOut(duration: 0.35)
            }

            // Progress dots
            HStack(spacing: 8) {
                ForEach(0..) { index in
                    Circle()
                        .fill(phase.rawValue > index ? .blue : .gray.opacity(0.3))
                        .frame(width: 8, height: 8)
                        .scaleEffect(phase.rawValue > index ? 1.2 : 1.0)
                        .animation(
                            .spring(duration: 0.3, bounce: 0.4)
                                .delay(Double(index) * 0.05),
                            value: phase
                        )
                }
            }
        }
        .frame(maxWidth: .infinity, maxHeight: .infinity)
        .background(.background)
    }

    private func iconForPhase(_ phase: ContentLoader.LoadPhase) -> String {
        switch phase {
        case .idle: return "circle"
        case .connecting: return "network"
        case .fetching: return "arrow.down.circle"
        case .processing: return "gearshape"
        case .complete: return "checkmark.circle.fill"
        }
    }

    private func labelForPhase(_ phase: ContentLoader.LoadPhase) -> String {
        switch phase {
        case .idle: return "Ready"
        case .connecting: return "Connecting…"
        case .fetching: return "Fetching data…"
        case .processing: return "Processing…"
        case .complete: return "Done"
        }
    }
}
```

**Key points:**

- The `PhaseAnimator` cycles through enum cases, providing each phase to the view builder.
- The `animation` closure returns different animations per phase — spring for completion, easeInOut for intermediate steps.
- The outer `Group` with `.animation(.easeInOut, value: loadPhase)` handles the transition from loading to content.
- `Task.sleep` simulates async work; replace with actual network calls.

---

## 3. KeyframeAnimator for Complex Multi-Step Animation

A card flip-and-bounce animation using `KeyframeAnimator` with multiple tracks for independent property timing.

```swift
struct CardFlipAnimation: View {
    @State private var isFlipped = false
    @State private var flipTrigger = false

    var body: some View {
        VStack(spacing: 24) {
            KeyframeAnimator(
                initialValue: CardState(),
                trigger: flipTrigger
            ) { state in
                ZStack {
                    CardBack()
                        .opacity(state.showBack ? 1 : 0)
                    CardFront()
                        .opacity(state.showBack ? 0 : 1)
                }
                .rotation3DEffect(
                    .degrees(state.rotationY),
                    axis: (x: 0, y: 1, z: 0),
                    perspective: 0.3
                )
                .scaleEffect(state.scale)
                .offset(y: state.verticalOffset)
            } keyframes: { _ in
                // Y-axis rotation: 0 → 90 → 180
                KeyframeTrack(\.rotationY) {
                    CubicKeyframe(0, relativeTime: 0.0)
                    CubicKeyframe(90, relativeTime: 0.35)
                    SpringKeyframe(180, relativeTime: 0.7, duration: 0.3)
                }

                // Scale: pulse up then settle
                KeyframeTrack(\.scale) {
                    LinearKeyframe(1.0, relativeTime: 0.0)
                    SpringKeyframe(1.15, relativeTime: 0.2, duration: 0.15)
                    LinearKeyframe(1.0, relativeTime: 0.5)
                    SpringKeyframe(1.05, relativeTime: 0.85, duration: 0.15)
                    LinearKeyframe(1.0, relativeTime: 1.0)
                }

                // Vertical offset: lift during flip, drop with bounce
                KeyframeTrack(\.verticalOffset) {
                    LinearKeyframe(0, relativeTime: 0.0)
                    CubicKeyframe(-40, relativeTime: 0.3)
                    SpringKeyframe(0, relativeTime: 0.75, duration: 0.25)
                }

                // Show back face: instant swap at the midpoint
                KeyframeTrack(\.showBack) {
                    MoveKeyframe(false, relativeTime: 0.0)
                    MoveKeyframe(true, relativeTime: 0.36) // Just past 90°
                    MoveKeyframe(true, relativeTime: 1.0)
                }
            }

            Button(isFlipped ? "Flip Back" : "Flip Card") {
                flipTrigger.toggle()
                isFlipped.toggle()
            }
            .keyboardShortcut("f", modifiers: .command)
        }
        .padding(40)
    }
}

struct CardState {
    var rotationY: Double = 0
    var scale: Double = 1.0
    var verticalOffset: Double = 0
    var showBack: Bool = false
}

struct CardFront: View {
    var body: some View {
        RoundedRectangle(cornerRadius: 16)
            .fill(.blue.gradient)
            .frame(width: 200, height: 280)
            .overlay {
                VStack {
                    Image(systemName: "star.fill")
                        .font(.system(size: 48))
                    Text("Front")
                        .font(.title2)
                        .fontWeight(.semibold)
                }
                .foregroundStyle(.white)
            }
    }
}

struct CardBack: View {
    var body: some View {
        RoundedRectangle(cornerRadius: 16)
            .fill(.gray.gradient)
            .frame(width: 200, height: 280)
            .overlay {
                VStack {
                    Image(systemName: "swift")
                        .font(.system(size: 48))
                    Text("Back")
                        .font(.title2)
                        .fontWeight(.semibold)
                }
                .foregroundStyle(.white)
            }
    }
}
```

**Key points:**

- `CardState` is a plain struct holding all animatable properties. `KeyframeAnimator` interpolates between instances.
- `MoveKeyframe` for `showBack` creates an instant swap at 90° (when the card is edge-on and invisible).
- `SpringKeyframe` on `rotationY` and `verticalOffset` adds organic bounce to the landing.
- The `trigger` value (`flipToggle`) causes the animation to replay each time it changes. The `initialValue` is the starting state.
- `rotation3DEffect` with `perspective: 0.3` gives depth to the 3D flip.

---

## 4. Reduced Motion Fallback Pattern

A reusable pattern for providing graceful animation degradation when the user has "Reduce motion" enabled.

```swift
struct AdaptiveAnimationModifier: ViewModifier {
    @Environment(\.accessibilityReduceMotion) private var reduceMotion
    let animation: Animation
    let value: AnyHashable

    func body(content: Content) -> some View {
        if reduceMotion {
            content
        } else {
            content
                .animation(animation, value: value)
        }
    }
}

extension View {
    func adaptiveAnimation(
        _ animation: Animation,
        value: some Hashable
    ) -> some View {
        modifier(AdaptiveAnimationModifier(
            animation: animation,
            value: AnyHashable(value)
        ))
    }
}

// Usage in a view
struct NotificationBanner: View {
    let notification: Notification
    @State private var isVisible = false

    var body: some View {
        HStack {
            Image(systemName: notification.icon)
            Text(notification.message)
            Spacer()
            Button("Dismiss") { dismiss() }
        }
        .padding()
        .background(.regularMaterial, in: RoundedRectangle(cornerRadius: 12))
        .offset(y: isVisible ? 0 : -100)
        .opacity(isVisible ? 1 : 0)
        .adaptiveAnimation(.spring(duration: 0.4, bounce: 0.2), value: isVisible)
        .onAppear {
            isVisible = true
        }
    }

    private func dismiss() {
        withAnimation(.easeOut(duration: 0.2)) {
            isVisible = false
        }
    }
}

// For PhaseAnimator — replace with static view
struct AdaptiveLoader: View {
    @Environment(\.accessibilityReduceMotion) private var reduceMotion
    let isComplete: Bool

    var body: some View {
        if reduceMotion {
            // Static fallback: just show the current state
            Image(systemName: isComplete ? "checkmark.circle.fill" : "arrow.triangle.2.circlepath")
                .font(.title2)
                .foregroundStyle(isComplete ? .green : .blue)
        } else {
            // Full animation
            PhaseAnimator([0, 1, 0, 1]) { phase in
                Image(systemName: isComplete ? "checkmark.circle.fill" : "arrow.triangle.2.circlepath")
                    .font(.title2)
                    .foregroundStyle(isComplete ? .green : .blue)
                    .rotationEffect(.degrees(phase == 1 ? 180 : 0))
                    .scaleEffect(phase == 1 ? 1.2 : 1.0)
            } animation: { _ in
                .easeInOut(duration: 0.5)
            }
        }
    }
}

// For matchedGeometryEffect — replace with crossfade
struct AdaptiveHeroView: View {
    @Environment(\.accessibilityReduceMotion) private var reduceMotion
    @Namespace private var hero
    @State private var selectedItem: Item?

    var body: some View {
        if reduceMotion {
            // Simple crossfade instead of hero animation
            if let selectedItem {
                ItemDetail(item: selectedItem)
                    .transition(.opacity)
            } else {
                ItemGrid { item in
                    withAnimation(.easeInOut(duration: 0.2)) {
                        selectedItem = item
                    }
                }
                .transition(.opacity)
            }
        } else {
            // Full matched geometry hero animation
            if let selectedItem {
                ItemDetail(item: selectedItem)
                    .matchedGeometryEffect(id: selectedItem.id, in: hero)
            } else {
                ItemGrid { item in
                    withAnimation(.spring(duration: 0.45, bounce: 0.15)) {
                        selectedItem = item
                    }
                }
            }
        }
    }
}
```

**Key points:**

- The `AdaptiveAnimationModifier` is a reusable wrapper that conditionally applies animation based on the environment value.
- For `PhaseAnimator`, replace with a static view that shows the current state — no motion at all.
- For `matchedGeometryEffect`, replace with a simple `.opacity` crossfade. The user still sees the state change, just without the spatial morphing.
- Always test with System Settings → Accessibility → Display → Reduce motion enabled.
- The `reduceMotion` environment value is dynamic — if the user changes the setting while the app is running, the view tree rebuilds automatically.

---

## 5. Spring Animation Tuning for macOS Feel

macOS has a distinct animation personality: snappy, precise, with minimal overshoot. iOS tends toward bouncier springs. Use these tuning guidelines to match platform conventions.

```swift
// MARK: - Spring Presets for macOS

extension Animation {
    /// Quick toggle — snappy with minimal overshoot.
    /// Use for: checkboxes, switches, small state changes.
    static var macosToggle: Animation {
        .spring(response: 0.25, dampingFraction: 0.85, blendDuration: 0)
    }

    /// Standard UI transition — smooth and controlled.
    /// Use for: panel open/close, sidebar expand, sheet presentation.
    static var macosStandard: Animation {
        .spring(response: 0.4, dampingFraction: 0.85, blendDuration: 0)
    }

    /// Emphasis animation — slightly more expressive.
    /// Use for: alert presentation, badge appearance, focus ring.
    static var macosEmphasis: Animation {
        .spring(response: 0.45, dampingFraction: 0.75, blendDuration: 0)
    }

    /// Playful — noticeable bounce for personality.
    /// Use for: onboarding, celebratory moments, drag-and-drop landing.
    static var macosPlayful: Animation {
        .spring(response: 0.5, dampingFraction: 0.65, blendDuration: 0)
    }

    /// Interactive — lower response for gesture-coupled animation.
    /// Use for: drag preview, rubber-banding, follow-cursor effects.
    static var macosInteractive: Animation {
        .interactiveSpring(response: 0.35, dampingFraction: 0.9, blendDuration: 0)
    }
}

// MARK: - Usage Examples

struct SpringExamples: View {
    @State private var isExpanded = false
    @State private var showAlert = false
    @State private var items: [Item] = []

    var body: some View {
        VStack(spacing: 20) {
            // Toggle — snappy
            Toggle("Enable Feature", isOn: $isExpanded)
                .animation(.macosToggle, value: isExpanded)

            // Panel expand — standard
            if isExpanded {
                SettingsPanel()
                    .transition(.move(edge: .top).combined(with: .opacity))
            }

            // Alert — emphasis
            if showAlert {
                AlertBanner()
                    .transition(.scale(scale: 0.9).combined(with: .opacity))
            }

            // List insertion — standard
            List {
                ForEach(items) { item in
                    ItemRow(item: item)
                }
                .onMove { from, to in
                    withAnimation(.macosStandard) {
                        items.move(fromOffsets: from, toOffset: to)
                    }
                }
            }
        }
        .animation(.macosStandard, value: isExpanded)
        .animation(.macosEmphasis, value: showAlert)
    }
}

// MARK: - Spring Physics Reference

/*
 Spring parameters explained:

 - response: Natural duration of the spring (lower = faster).
   macOS range: 0.25–0.5 for most UI.

 - dampingFraction: How quickly oscillation stops.
   1.0 = critically damped (no bounce)
   0.85 = slight settle (macOS standard feel)
   0.65 = noticeable bounce (playful)
   0.5  = very bouncy (avoid on macOS)

 - blendDuration: How long to blend from a previous animation.
   0 = immediate switch (recommended for discrete state changes)

 Platform comparison:
 ┌──────────────┬────────────┬────────────┐
 │              │ macOS      │ iOS        │
 ├──────────────┼────────────┼────────────┤
 │ response     │ 0.25–0.45  │ 0.4–0.6    │
 │ dampingFrac  │ 0.8–0.9    │ 0.7–0.8    │
 │ feel         │ Snappy     │ Bouncy     │
 └──────────────┴────────────┴────────────┘

 Testing tip: Use Xcode's "Slow Animations" (Debug → Slow Animations
 or ⌘T) to visually inspect spring curves during development.
 Damping fractions below 0.7 are almost always too bouncy for macOS.
*/
```

**Key points:**

- macOS favors higher damping fractions (0.8–0.9) and lower response values (0.25–0.45) compared to iOS.
- `blendDuration: 0` prevents lingering animation blends when state changes rapidly.
- Use `.interactiveSpring` for gesture-coupled animations — it has a lower effective response that feels more responsive during drag.
- Avoid `dampingFraction` below 0.7 on macOS — the resulting bounce feels out of place on the platform.
- The `bouncy`, `smooth`, and `snappy` presets (macOS 14+) are good starting points but may need tuning for specific use cases.
