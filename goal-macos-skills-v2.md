# Build macos-skills — Complete, Self-Contained macOS 26 Skill Corpus

## Philosophy

`macos-skills` is a **standalone corpus**. An agent building a macOS app should
never need to consult `swift-ios-skills` to fill a gap. Where topics overlap
(Swift concurrency, SwiftUI, SwiftData, networking, security) the macOS skill
covers the full topic with macOS-specific depth — not a pointer to an iOS file.

Use `swift-ios-skills` as a **research reference and quality bar**, not as a
dependency. Study its structure, depth, and reference-file patterns. Match or
exceed that quality. Do not copy content verbatim; write macOS-first.

---

## Research Protocol

Before writing each skill, run the specified Exa queries. Every non-trivial
API claim, build flag, and behavior assertion must trace to a live Exa source.
Record source URLs in the `sources:` frontmatter field. Note the macOS version
and Xcode version where behavior was confirmed (target: **macOS 26 / Xcode 16+**,
Swift **6.3+**). If sources conflict, surface the conflict in
**Constraints & Gotchas** rather than silently resolving it.

---

## SKILL.md Format

Match the format of `swift-ios-skills` exactly:

```markdown
---
name: <skill-name>
description: "<single sentence: precise trigger conditions and what it enables>"
sources: <comma-separated Exa source URLs>
researched: <ISO 8601 date>
platform: macOS 26+
---

# <Title>

<One paragraph summary + scope boundary: what this skill owns and what
adjacent skills own.>

## Contents
- [Section](#anchor)
...

## <Section>

<Substantive content. Decision tables, code examples in Swift 6.3,
### subsections for depth. No padding — every sentence is agent-usable.>

## Constraints & Gotchas

## Common Mistakes

## Review Checklist
- [ ] ...

## References
- [references/filename.md](references/filename.md) — description
```

Each skill has a `references/` subdirectory with deep-dive `.md` files for
content that overflows the main `SKILL.md`. The `SKILL.md` is the router and
decision layer; `references/` files carry the full patterns and code examples.

Minimum **500 words** of substantive content per `SKILL.md`.
Minimum **one `references/` file** per skill.

---

## Complete Skill Manifest

### TIER 1 — Swift Language & Concurrency

---

#### `swift-language-macos/SKILL.md`
**Covers:** Modern Swift language patterns for macOS — value types, generics,
result builders, macros, `@dynamicMemberLookup`, property wrappers, `Sendable`,
`@retroactive` conformances, Swift 6.3 language changes. macOS-relevant
interop: `@objc`, `@objcMembers`, `NSObject` subclassing patterns needed for
AppKit delegation and KVO.

**Exa queries:**
- `Swift 6.3 language features macOS 2025 new`
- `Swift @objc NSObject AppKit delegation KVO macOS`
- `Swift macros attached freestanding macOS 2025`
- `Swift property wrappers macOS AppKit patterns`
- `Swift result builders DSL macOS SwiftUI commands`

---

#### `swift-concurrency-macos/SKILL.md`
**Covers:** Full Swift concurrency for macOS — actor isolation, `@MainActor`,
approachable concurrency (SE-0466), `Task`, `TaskGroup`, `AsyncStream`,
`AsyncSequence`, `Sendable`, strict concurrency migration, and
**macOS-specific patterns**: `DispatchQueue` bridging for AppKit
(AppKit's threading model predates Swift concurrency), `NSRunLoop` interaction,
XPC service concurrency, background daemon concurrency patterns.

**Exa queries:**
- `Swift concurrency macOS AppKit main thread MainActor 2025`
- `Swift strict concurrency migration macOS 6.3 2025`
- `Swift async await AppKit NSRunLoop DispatchQueue bridge`
- `Swift actor isolation macOS XPC service`
- `Swift approachable concurrency SE-0466 macOS`

---

### TIER 2 — App Architecture

---

#### `swift-architecture-macos/SKILL.md`
**Covers:** Architecture patterns for macOS apps — MV with `@Observable`,
MVVM, document-based architecture (`NSDocument` + model), multi-window state
management, app delegate lifecycle (`NSApplicationDelegate`), scene-based
vs delegate-based app entry points, dependency injection for macOS, testing
architecture.

**macOS delta from iOS:** `NSApplicationDelegate` vs `UIApplicationDelegate`,
`applicationWillTerminate` / `applicationShouldTerminate` patterns,
`NSUserDefaults` suite sharing between app and helper, multi-window document
model as an architectural primitive.

**Exa queries:**
- `SwiftUI App NSApplicationDelegate macOS lifecycle 2025`
- `macOS multi-window state management SwiftUI @Observable`
- `NSDocument architecture macOS MVC model layer`
- `macOS app architecture document-based SwiftUI 2025`
- `macOS dependency injection SwiftUI environment service`

---

### TIER 3 — AppKit Foundations

---

#### `appkit-foundations/SKILL.md`
**Covers:** NSWindow, NSViewController, NSView hierarchy, responder chain,
NSDocument, layer-backed drawing, drag & drop, printing, KVO/KVC patterns
still used in AppKit, and the `NSApplication` run loop.

**Exa queries:**
- `AppKit NSWindow NSViewController lifecycle macOS 26 2025`
- `AppKit responder chain first responder NSResponder macOS`
- `NSDocument read write architecture macOS autosave`
- `AppKit NSView layer-backed wantsLayer drawRect CALayer`
- `AppKit drag drop NSDraggingDestination pasteboard macOS`
- `AppKit KVO KVC NSObject observeValue macOS patterns`

---

#### `swiftui-appkit-interop/SKILL.md`
**Covers:** `NSViewRepresentable`, `NSViewControllerRepresentable`,
`NSHostingController`, `NSHostingView`, coordinator delegate pattern, state
sync across the AppKit/SwiftUI boundary. Key differences from
`UIViewRepresentable` (flipped coordinates, `intrinsicContentSize`,
first responder, no `sizeThatFits`).

**Exa queries:**
- `NSViewRepresentable SwiftUI macOS coordinator 2025`
- `NSHostingController NSHostingView SwiftUI AppKit embed macOS`
- `NSViewRepresentable intrinsicContentSize sizing macOS`
- `SwiftUI AppKit first responder focus boundary macOS`
- `NSViewRepresentable vs UIViewRepresentable differences`

---

### TIER 4 — SwiftUI on macOS

---

#### `swiftui-macos-patterns/SKILL.md`
**Covers:** SwiftUI state and architecture patterns **as they work on macOS**
— `@Observable`, MV pattern, `@State`, `@Binding`, `@Environment`,
`@AppStorage`, `.task`, `.onChange`. macOS deltas: `@FocusedValue`,
`@FocusedBinding`, `@FocusedObject`, `defaultFocus`, `.focusedSceneValue`,
scene-scoped environment, `Settings` scene wiring.

**Exa queries:**
- `SwiftUI @Observable macOS MV pattern 2025`
- `SwiftUI @FocusedValue @FocusedBinding macOS menu commands`
- `SwiftUI @AppStorage macOS UserDefaults suite`
- `SwiftUI .task .onChange macOS lifecycle scene`
- `SwiftUI environment EnvironmentValues macOS custom keys`

---

#### `swiftui-macos-navigation/SKILL.md`
**Covers:** `NavigationSplitView` on macOS — three-column, sidebar/content/detail,
column visibility, programmatic control. `NavigationStack` on macOS.
`TabView` macOS behavior. Sheet, popover, alert, confirmationDialog, fullscreenCover
(absent on macOS — replacement patterns). `openURL`, `openWindow`,
`dismissWindow` environment actions. Deep linking on macOS.

**Exa queries:**
- `SwiftUI NavigationSplitView macOS 3-column sidebar 2025`
- `SwiftUI openWindow dismissWindow macOS environment 2025`
- `SwiftUI sheet popover macOS window presentation`
- `SwiftUI TabView macOS tabViewStyle 2025`
- `SwiftUI NavigationStack macOS deep link URL`

---

#### `swiftui-macos-layout/SKILL.md`
**Covers:** Layout primitives on macOS — `VStack`, `HStack`, `ZStack`,
`LazyVGrid`, `LazyHGrid`, `Grid`, `ViewThatFits`, `GeometryReader`.
macOS deltas: `Form` layout styles (`.grouped` vs `.automatic` on macOS),
`Table` (macOS-primary multi-column data display), `List` with source list
style, `OutlineGroup`, `DisclosureGroup`. macOS window sizing and
`fixedSize` / `frame` interactions.

**Exa queries:**
- `SwiftUI Table macOS multi-column sortable 2025`
- `SwiftUI List sidebarRowBackground sourceList macOS`
- `SwiftUI Form macOS grouped insetGrouped style`
- `SwiftUI OutlineGroup DisclosureGroup macOS`
- `SwiftUI GeometryReader window resize macOS`
- `SwiftUI Grid ViewThatFits macOS layout 2025`

---

#### `swiftui-macos-windows-scenes/SKILL.md`
**Covers:** `WindowGroup`, `Window`, `Settings`, `MenuBarExtra`,
`DocumentGroup` scene types. Multi-window management, window restoration,
`defaultSize`, `defaultPosition`, `windowStyle`, `windowToolbarStyle`,
`WindowStyle` types, `resizability`.

**Exa queries:**
- `SwiftUI WindowGroup Window scene macOS 26 multi-window 2025`
- `SwiftUI MenuBarExtra macOS status bar app window menu 2025`
- `SwiftUI Settings scene PreferencePane macOS Form`
- `SwiftUI DocumentGroup scene macOS document-based 2025`
- `SwiftUI windowStyle windowToolbarStyle macOS`
- `SwiftUI window restoration state defaultSize macOS`

---

#### `swiftui-macos-toolbar-sidebar/SKILL.md`
**Covers:** `.toolbar {}`, `ToolbarItem` placements on macOS
(`.principal`, `.primaryAction`, `.secondaryAction`, `.navigation`,
`.status`, `.confirmationAction`, `.destructiveAction`),
`ToolbarTitleMenu`, `searchable` in toolbar with tokens and suggestions,
inspector panel (`inspector(isPresented:)`), `inspectorColumnWidth`.

**Exa queries:**
- `SwiftUI toolbar ToolbarItem placement macOS principal 2025`
- `SwiftUI ToolbarTitleMenu macOS document title menu`
- `SwiftUI searchable toolbar macOS tokens suggestions 2025`
- `SwiftUI inspector panel macOS 26 inspectorColumnWidth`
- `SwiftUI toolbar overflow macOS customization`

---

#### `swiftui-macos-menus-commands/SKILL.md`
**Covers:** `.commands {}`, `CommandMenu`, `CommandGroup`, `CommandsBuilder`,
`KeyboardShortcut`, context menus with `primaryAction`, `Button.role`,
menu validation via focused values, Dock menu via `NSApplicationDelegate`,
Services menu, `NSMenu` escape hatch.

**Exa queries:**
- `SwiftUI commands CommandMenu CommandGroup macOS 2025`
- `SwiftUI keyboardShortcut macOS scope modifiers 2025`
- `SwiftUI contextMenu primaryAction macOS 2025`
- `NSMenu NSMenuItem AppKit macOS SwiftUI integration`
- `macOS menu validation first responder @FocusedValue`
- `macOS dock menu NSApplicationDelegate applicationDockMenu`

---

#### `swiftui-macos-controls/SKILL.md`
**Covers:** macOS-specific control behaviors and styling — `Button` styles
(`bordered`, `borderedProminent`, `plain`, `link`, macOS-only
`accessoryBar`), `Picker` styles (`segmented`, `menu`, `radioGroup`,
`inline`), `Toggle` styles, `Slider`, `Stepper`, `ColorPicker`,
`FontPicker`, `PathControl`, `LevelIndicator`. macOS control sizing
(`.controlSize`), `ControlGroup`.

**Exa queries:**
- `SwiftUI Button Picker Toggle macOS control styles 2025`
- `SwiftUI controlSize .mini .small .regular .large macOS`
- `SwiftUI ColorPicker FontPicker macOS 2025`
- `SwiftUI ControlGroup radioGroup picker macOS`
- `AppKit NSPathControl NSLevelIndicator SwiftUI macOS`

---

#### `swiftui-macos-liquid-glass/SKILL.md`
**Covers:** Liquid Glass on macOS 26 — `glassEffect`, `GlassEffectContainer`,
glass toolbar/sidebar/panel patterns, macOS-specific vibrancy materials
(`NSVisualEffectView` vs SwiftUI `.glassBackgroundDisplayMode`),
`windowBackgroundDragBehavior`, translucent sidebar, title bar blending.

**Exa queries:**
- `SwiftUI Liquid Glass macOS 26 glassEffect 2025`
- `SwiftUI glassBackgroundDisplayMode macOS window`
- `NSVisualEffectView material macOS SwiftUI 2025`
- `macOS 26 translucent sidebar toolbar glass SwiftUI`
- `SwiftUI GlassEffectContainer macOS panel`

---

#### `swiftui-macos-animation/SKILL.md`
**Covers:** SwiftUI animations on macOS — `withAnimation`, `Animation` types,
`matchedGeometryEffect`, `PhaseAnimator`, `KeyframeAnimator`, transitions,
`spring` and `interactiveSpring`. macOS deltas: reduced motion
(`accessibilityReduceMotion`), window open/close transitions, `popover`
and `sheet` custom transitions.

**Exa queries:**
- `SwiftUI animation macOS 26 KeyframeAnimator PhaseAnimator 2025`
- `SwiftUI matchedGeometryEffect macOS window transition`
- `SwiftUI accessibilityReduceMotion macOS prefersReducedMotion`
- `SwiftUI spring animation macOS interactive 2025`
- `SwiftUI transition sheet popover macOS custom`

---

#### `swiftui-macos-gestures/SKILL.md`
**Covers:** Gesture handling on macOS — `TapGesture`, `LongPressGesture`,
`DragGesture`, `MagnifyGesture`, `RotateGesture`. **macOS-specific:**
`onHover`, cursor control (`onContinuousHover`, `.cursor`), trackpad
gestures, scroll wheel (`onScrollWheel`), right-click / secondary click,
`EventModifiers`, `NSEvent` bridge for raw mouse/keyboard events.

**Exa queries:**
- `SwiftUI gesture macOS hover cursor onHover 2025`
- `SwiftUI DragGesture MagnifyGesture macOS trackpad`
- `SwiftUI onScrollWheel macOS scroll event 2025`
- `SwiftUI secondary click right click macOS contextMenu`
- `NSEvent SwiftUI bridge macOS raw keyboard mouse event`

---

#### `swiftui-macos-performance/SKILL.md`
**Covers:** SwiftUI performance on macOS — avoiding redundant redraws,
`equatable` view optimization, `TimelineView`, lazy containers with large
datasets (`Table` with 10k+ rows), Instruments for SwiftUI on macOS
(SwiftUI template, Hangs instrument), `MainActor` hop cost, off-main
rendering.

**Exa queries:**
- `SwiftUI performance macOS large data Table List 2025`
- `SwiftUI Instruments Hangs macOS performance profiling`
- `SwiftUI equatable body macOS re-render optimization`
- `SwiftUI TimelineView macOS animation performance`
- `SwiftUI MainActor performance macOS off-main rendering`

---

#### `swiftui-macos-accessibility/SKILL.md`
**Covers:** VoiceOver on macOS (different from iOS — keyboard-driven, rotor
behavior differs), accessibility modifiers (`.accessibilityLabel`,
`.accessibilityValue`, `.accessibilityAction`, `.accessibilityChildren`),
`AXCustomContent`, full keyboard access, `accessibilityFocused`,
Accessibility Inspector usage, Dynamic Type on macOS.

**Exa queries:**
- `SwiftUI accessibility VoiceOver macOS keyboard navigation 2025`
- `SwiftUI accessibilityLabel accessibilityAction macOS`
- `macOS VoiceOver rotor custom rotors SwiftUI`
- `macOS full keyboard access SwiftUI focus`
- `Accessibility Inspector macOS SwiftUI audit 2025`

---

### TIER 5 — Text, Input & Rich Content

---

#### `macos-text-and-input/SKILL.md`
**Covers:** `NSTextView` and TextKit 2 (`NSTextLayoutManager`,
`NSTextContentStorage`, `NSTextViewportLayoutController`),
rich text editing, `NSAttributedString`, `NSTextAttachment`,
find bar (`NSTextFinder`), `NSTextInputClient`, font panel,
SwiftUI `TextEditor` limitations, wrapping `NSTextView` as
`NSViewRepresentable`.

**Exa queries:**
- `NSTextView TextKit 2 NSTextLayoutManager macOS 26 2025`
- `NSTextView rich text NSAttributedString attachment macOS`
- `NSTextFinder usesFindBar macOS SwiftUI TextEditor`
- `NSTextInputClient keyboard input macOS custom view`
- `NSFontManager NSFontPanel macOS SwiftUI`

---

#### `swiftui-webkit-macos/SKILL.md`
**Covers:** `WebView` (new SwiftUI-native API, macOS 26), `WKWebView`
wrapped as `NSViewRepresentable`, `WKNavigationDelegate`,
`WKUIDelegate`, JavaScript bridging (`WKScriptMessageHandler`,
`callAsyncJavaScript`), custom URL schemes, local content loading,
`WKWebViewConfiguration`, download handling, PDF rendering via WebKit.

**Exa queries:**
- `SwiftUI WebView macOS 26 WKWebView native API 2025`
- `WKWebView NSViewRepresentable macOS navigation delegate`
- `WKWebView JavaScript bridge callAsyncJavaScript macOS`
- `WKWebView custom URL scheme WKURLSchemeHandler macOS`
- `WKWebView download PDF macOS 2025`

---

### TIER 6 — Data & Persistence

---

#### `swiftdata-macos/SKILL.md`
**Covers:** SwiftData on macOS — `@Model`, `@Query`, `ModelContainer`,
`ModelContext`, CloudKit sync, migrations, `#Predicate`,
`FetchDescriptor`, `SortDescriptor`, performance with large datasets.
macOS deltas: shared `ModelContainer` across windows, document-scoped
containers, `NSPersistentHistoryTracking` for XPC/multi-process access.

**Exa queries:**
- `SwiftData macOS ModelContainer multi-window 2025`
- `SwiftData CloudKit sync macOS 26 2025`
- `SwiftData migration schema macOS versioned 2025`
- `SwiftData NSPersistentHistoryTracking macOS XPC`
- `SwiftData performance large dataset macOS 2025`

---

#### `core-data-macos/SKILL.md`
**Covers:** Core Data on macOS — `NSPersistentContainer`, `NSManagedObject`,
fetch requests, `NSFetchedResultsController`, batch operations, history
tracking, staged migration, CloudKit (`NSPersistentCloudKitContainer`),
multi-process access via `NSPersistentHistoryTracking`, memory management
with large graphs.

**Exa queries:**
- `Core Data macOS NSPersistentContainer NSFetchedResultsController`
- `Core Data batch operations macOS NSBatchUpdateRequest 2025`
- `NSPersistentCloudKitContainer macOS sync 2025`
- `Core Data staged migration macOS NSMigrationStage`
- `Core Data NSPersistentHistoryTracking macOS multi-process`

---

#### `macos-file-system/SKILL.md`
**Covers:** macOS file system access — sandbox entitlements, security-scoped
bookmarks (`startAccessingSecurityScopedResource`), `NSOpenPanel` /
`NSSavePanel`, `FileManager`, `NSFileCoordinator` / `NSFilePresenter`,
`NSMetadataQuery` (Spotlight queries from code), file promises,
`NSFileProviderExtension` basics, `UTType` / `UTI` for document types.

**Exa queries:**
- `macOS sandbox security-scoped bookmark file access 2025`
- `NSOpenPanel NSSavePanel SwiftUI macOS 2025`
- `NSFileCoordinator NSFilePresenter macOS coordinated file`
- `NSMetadataQuery Spotlight macOS file search code`
- `UTType document type registration macOS 26 Info.plist`
- `NSFilePromise drag drop macOS 2025`

---

#### `cloudkit-macos/SKILL.md`
**Covers:** CloudKit on macOS — `CKContainer`, `CKDatabase`,
`CKRecord`, `CKQuery`, `CKSubscription`, `CKSyncEngine`,
`NSPersistentCloudKitContainer`, push-based sync via `CKDatabaseSubscription`,
macOS-specific: `CloudKit` entitlement + sandbox considerations,
iCloud Drive document sync, multi-window conflict resolution.

**Exa queries:**
- `CloudKit macOS CKSyncEngine 2025`
- `CloudKit CKSubscription macOS push notification sync`
- `NSPersistentCloudKitContainer macOS conflict resolution`
- `CloudKit iCloud Drive document macOS NSFilePresenter`
- `CloudKit entitlement macOS sandbox App Store 2025`

---

### TIER 7 — Networking

---

#### `networking-macos/SKILL.md`
**Covers:** `URLSession` with `async/await` on macOS,
`URLSessionConfiguration` (background sessions, macOS daemon background
transfer), `Network.framework` (`NWConnection`, `NWListener`,
`NWBrowser`), WebSocket, Bonjour / mDNS (`NWBrowser` for service discovery),
`NWPathMonitor` network reachability, HTTP/3, TLS pinning, App Transport
Security, local network entitlement.

**macOS delta:** Background URLSession on macOS differs from iOS (no app
suspension, but daemon/helper process implications). Local network privacy
prompt behavior differs.

**Exa queries:**
- `URLSession async await macOS background session daemon 2025`
- `Network framework NWConnection NWListener macOS 2025`
- `NWBrowser Bonjour mDNS macOS service discovery`
- `NWPathMonitor macOS network reachability 2025`
- `macOS App Transport Security local network entitlement`
- `URLSession WebSocket macOS async stream 2025`

---

### TIER 8 — Security

---

#### `swift-security-macos/SKILL.md`
**Covers:** Keychain on macOS (kSecUseDataProtectionKeychain,
`kSecAttrAccessGroup` for app group sharing, keychain sharing between
app and helper tool), biometrics (`LAContext`, Touch ID, Apple Watch
unlock), `SecureEnclave` on Apple Silicon, `CryptoKit`, Hardened Runtime
entitlements, notarization, App Sandbox, SIP implications,
`AuthenticationServices` (Sign in with Apple, passkeys on macOS).

**Exa queries:**
- `macOS Keychain kSecUseDataProtectionKeychain app group 2025`
- `LAContext Touch ID Apple Watch unlock macOS 26`
- `SecureEnclave macOS Apple Silicon SecKey access control`
- `CryptoKit macOS HPKE post-quantum 2025`
- `macOS Hardened Runtime entitlements notarization 2025`
- `AuthenticationServices passkey macOS 26 2025`

---

#### `macos-security-entitlements/SKILL.md`
**Covers:** App Sandbox entitlement map, Hardened Runtime exception
entitlements, `com.apple.security.*` namespace, notarization workflow
(`notarytool submit`, stapling), Developer ID vs Mac App Store signing,
`SMJobBless` privileged helper tools, XPC services for privilege separation,
System Integrity Protection interaction.

**Exa queries:**
- `macOS App Sandbox entitlements file network hardware 2025`
- `macOS Hardened Runtime JIT DYLD unsigned-executable 2025`
- `xcrun notarytool submit staple macOS 2025`
- `SMJobBless privileged helper XPC macOS 2025`
- `macOS Developer ID vs Mac App Store code signing`

---

### TIER 9 — System Integration

---

#### `macos-system-integration/SKILL.md`
**Covers:** `ServiceManagement` (`SMAppService` login items, launch agents),
`NSWorkspace` (open files, observe app lifecycle), `NSPasteboard`
(custom `UTType`, promised data), Core Spotlight (`CSSearchableItem`,
`CSSearchableIndex`), Quick Look (`QLPreviewPanel`, SwiftUI
`.quickLookPreview`), AppleScript (`NSUserAppleScriptTask`),
Finder integration (file tags, `NSFileProviderExtension`),
`NSUserNotification` → `UNUserNotificationCenter` on macOS.

**Exa queries:**
- `SMAppService login item launch agent macOS 13+ 2025`
- `NSWorkspace macOS open file app observe notification`
- `NSPasteboard custom UTType promised data macOS 2025`
- `Core Spotlight CSSearchableIndex macOS app indexing 2025`
- `Quick Look QLPreviewPanel SwiftUI macOS 2025`
- `UNUserNotificationCenter macOS push local notification`

---

#### `macos-ipc-xpc/SKILL.md`
**Covers:** XPC services (`NSXPCConnection`, `NSXPCInterface`,
`@objc protocol`-based interface), privilege separation with XPC,
App Groups (`com.apple.security.application-groups`), `DistributedNotificationCenter`,
`CFMessagePort`, `NSDistributedNotificationCenter`, and `NSConnection`
(legacy — when encountered in existing codebases).

**Exa queries:**
- `NSXPCConnection NSXPCInterface macOS service Swift 2025`
- `XPC service privilege separation macOS sandbox 2025`
- `App Groups NSUserDefaults suite macOS helper sharing`
- `DistributedNotificationCenter macOS inter-process 2025`
- `XPC error handling invalidation macOS Swift async`

---

### TIER 10 — Hardware & Low-Level

---

#### `macos-metal-gpu/SKILL.md`
**Covers:** Metal on macOS — `MTKView` setup as `NSViewRepresentable`,
render and compute pipelines, MSL authoring, resource heaps,
`MTLFence` / `MTLSharedEvent` synchronization, `CAMetalLayer` direct path,
GPU selection (`MTLCopyAllDevices`, `preferLowPower`, eGPU `isRemovable`),
GPU family enumeration for Apple Silicon vs Intel discrete.

**Exa queries:**
- `MTKView NSViewRepresentable SwiftUI macOS 2025 setup`
- `Metal MTLDevice GPU selection Apple Silicon macOS 2025`
- `Metal MTLHeap resource management macOS performance`
- `Metal MTLSharedEvent synchronization macOS multi-queue`
- `MSL shader macOS best practices threadgroup compute`
- `CAMetalLayer macOS AppKit off-screen rendering`

---

#### `macos-iokit-hardware/SKILL.md`
**Covers:** IOKit for hardware access — `IOServiceGetMatchingService`,
`IORegistryEntry`, USB device enumeration, HID devices
(`IOHIDManager`, `IOHIDDevice`), serial ports, power management
(`IOPMAssertion` — prevent sleep), battery / power source info.
Also: CoreAudio on macOS (`AVAudioEngine`, `AudioUnit`, `AudioDeviceID`
for device selection), `NSSound`.

**Exa queries:**
- `IOKit USB HID device macOS Swift 2025`
- `IOHIDManager macOS gamepad keyboard custom device Swift`
- `IOPMAssertion macOS prevent sleep Swift 2025`
- `CoreAudio AVAudioEngine macOS audio device selection`
- `IOKit power source battery macOS Swift`

---

### TIER 11 — Testing & Tooling

---

#### `swift-testing-macos/SKILL.md`
**Covers:** Swift Testing (`@Test`, `@Suite`, `#expect`, `#require`,
confirmation, traits, parameterized tests) on macOS. macOS-specific
testing patterns: testing AppKit views, testing `NSDocument`, XCTest
UI testing on macOS (`XCUIApplication`, macOS-specific element queries),
testing XPC services, `xcodebuild test` macOS destination, test plans.

**Exa queries:**
- `Swift Testing macOS @Test @Suite 2025 Swift 6.3`
- `XCTest UI testing macOS XCUIApplication element query`
- `xcodebuild test macOS destination platform 2025`
- `Swift Testing AppKit NSDocument macOS async test`
- `Swift Testing XPC service macOS mock`

---

#### `debugging-macos/SKILL.md`
**Covers:** LLDB on macOS, Memory Graph Debugger, Instruments templates
for macOS (Hangs, Time Profiler, Allocations, Leaks, Metal System Trace,
Energy Log), `os_log` and `os_signpost` on macOS, `dtrace` (macOS-only),
crash report symbolicaction (`atos`, `symbolicatecrash`), sandbox
violation debugging (`sandbox-exec`, Console.app), Address Sanitizer,
Thread Sanitizer, Main Thread Checker.

**Exa queries:**
- `Instruments macOS Hangs Time Profiler 2025 profiling`
- `LLDB macOS AppKit debugging breakpoint watchpoint`
- `dtrace macOS Swift tracing 2025`
- `macOS crash report symbolicate atos 2025`
- `macOS sandbox violation debugging Console log`
- `os_log os_signpost macOS Swift structured logging`

---

#### `xcode-macos-toolchain/SKILL.md`
**Covers:** `xcodebuild` for macOS targets — destination syntax
(`platform=macOS`, `variant=Mac Catalyst`, `arch=arm64`), archive/export
(Developer ID, Mac App Store, development), `exportOptionsPlist` keys,
`xcrun notarytool`, `xcrun stapler`, SPM in macOS workspace, build
settings delta (macOS-specific: `MACOSX_DEPLOYMENT_TARGET`,
`ENABLE_HARDENED_RUNTIME`, `CODE_SIGN_ENTITLEMENTS`),
Mac Catalyst, Universal Binary (`lipo`), `xcodebuild` test on macOS.

**Exa queries:**
- `xcodebuild macOS destination archive export 2025`
- `xcodebuild exportArchive Developer ID Mac App Store plist`
- `xcrun notarytool submit staple macOS 2025`
- `xcodebuild test platform macOS destination 2025`
- `Mac Catalyst xcodebuild variant macOS targetEnvironment`
- `lipo universal binary macOS arm64 x86_64 xcodebuild`

---

#### `swiftlint-macos/SKILL.md`
**Covers:** SwiftLint configuration for macOS projects —
`.swiftlint.yml` setup, build tool plugin integration (SPM and Xcode),
macOS-relevant rules (AppKit naming, `NSObject` patterns),
custom rules for macOS-specific anti-patterns, CI integration
(`xcodebuild` + SwiftLint), `--strict` mode, baseline files,
`swiftlint analyze` with the compiler log.

**Exa queries:**
- `SwiftLint macOS Xcode build plugin 2025`
- `SwiftLint custom rules macOS AppKit patterns`
- `SwiftLint CI xcodebuild integration macOS 2025`
- `SwiftLint analyze compiler log macOS`
- `SwiftLint baseline suppress violations macOS`

---

### TIER 12 — Shared Apple Frameworks (macOS-first treatment)

---

#### `swift-charts-macos/SKILL.md`
**Covers:** Swift Charts on macOS — all chart types, `ChartProxy`,
selection, `scrollableAxes`, `chartOverlay`, annotations,
macOS-specific: chart in `NSWindow` / panel, dark mode color handling,
print-ready chart rendering, accessibility for charts on macOS.

**Exa queries:**
- `Swift Charts macOS 26 selection scrollable 2025`
- `Swift Charts macOS dark mode accessibility`
- `Swift Charts ChartProxy macOS print PDF rendering`
- `Swift Charts annotation overlay macOS 2025`

---

#### `swift-codable-macos/SKILL.md`
**Covers:** `Codable`, `JSONDecoder`/`JSONEncoder`, `PropertyListDecoder`,
`CodingKey`, custom `encode`/`decode`, `@CodingKeyPath`, error handling.
macOS-relevant patterns: `NSCoding` migration to `Codable`,
`NSKeyedArchiver` bridge, `NSUserDefaults` Codable storage,
`NSDocument` Codable model serialization.

**Exa queries:**
- `Swift Codable macOS NSCoding migration NSKeyedArchiver`
- `Swift Codable NSUserDefaults storage macOS`
- `Swift Codable PropertyListEncoder macOS preferences`
- `Swift Codable @CodingKeyPath macOS 2025`

---

#### `swift-api-design-macos/SKILL.md`
**Covers:** Swift API Design Guidelines applied to macOS frameworks —
naming conventions for AppKit-style delegates, `NS`-prefixed type
interop, `@objc` protocol design for AppKit, error type design for
macOS system errors (`POSIXError`, `CocoaError`), Swift package design
for macOS libraries.

**Exa queries:**
- `Swift API design guidelines AppKit delegate naming macOS`
- `Swift @objc protocol AppKit delegate macOS patterns`
- `Swift error type CocoaError POSIXError macOS`
- `Swift package macOS library API design 2025`

---

#### `storekit-macos/SKILL.md`
**Covers:** StoreKit 2 on macOS — `Product.products`, `purchase`,
`Transaction.updates`, subscription status, `AppStore.sync`,
`SKOverlay` (available on macOS), `StoreView` / `ProductView`
SwiftUI components, receipt validation, sandbox testing on macOS
(different from iOS — Xcode scheme sandbox account setup).

**Exa queries:**
- `StoreKit 2 macOS purchase Transaction 2025`
- `StoreKit macOS sandbox testing Xcode 2025`
- `StoreKit ProductView StoreView SwiftUI macOS`
- `StoreKit subscription macOS App Store Mac Catalyst`

---

#### `apple-on-device-ai-macos/SKILL.md`
**Covers:** Foundation Models (`LanguageModelSession`, `@Generable`,
tool calling, `SystemLanguageModel`), CoreML (model conversion,
`MLModel`, `MLTensor`, Neural Engine on Apple Silicon), MLX Swift,
llama.cpp. macOS-specific: `MLComputeUnits` selection (CPU+NE+GPU),
`MLModelConfiguration`, model deployment in macOS app bundle vs
on-demand resources, `NSXPCConnection` for isolating model inference
in a helper process.

**Exa queries:**
- `Foundation Models LanguageModelSession macOS 26 2025`
- `CoreML macOS Apple Silicon Neural Engine MLComputeUnits`
- `MLX Swift macOS inference Apple Silicon 2025`
- `CoreML on-demand resources macOS bundle deployment`
- `CoreML XPC helper process macOS isolation inference`

---

#### `swift-concurrency-macos/SKILL.md` *(already listed in Tier 1)*

---

#### `localization-macos/SKILL.md`
**Covers:** String Catalogs, `LocalizedStringKey`, `String(localized:)`,
`FormatStyle`, plural rules, right-to-left layout on macOS,
`NSLocalizedString` migration, `.strings` / `.stringsdict` files,
Xcode localization export/import workflow, macOS-specific: menu item
localization, `NSMenu` title localization, accessibility label
localization, locale-aware `NSDateFormatter`.

**Exa queries:**
- `Swift String Catalogs macOS Xcode localization 2025`
- `SwiftUI LocalizedStringKey macOS RTL layout`
- `NSLocalizedString String Catalogs migration macOS`
- `macOS menu localization NSMenu title strings`
- `FormatStyle locale macOS date number currency`

---

#### `swift-formatstyle-macos/SKILL.md`
**Covers:** `FormatStyle` protocol — `Date.FormatStyle`,
`Number.FormatStyle`, `Measurement.FormatStyle`, `List.FormatStyle`,
`Duration.TimeFormatStyle`, custom `FormatStyle` conformances.
macOS display patterns: formatting for menu items, status bar,
inspector panels, large-number display in `Table` columns.

**Exa queries:**
- `Swift FormatStyle macOS Date Number Measurement 2025`
- `Swift FormatStyle custom conformance macOS`
- `Swift Duration TimeFormatStyle macOS 2025`
- `Swift FormatStyle Table column macOS display`

---

#### `push-notifications-macos/SKILL.md`
**Covers:** `UNUserNotificationCenter` on macOS — authorization
(different prompt behavior), `UNNotificationRequest`,
`UNUserNotificationCenterDelegate`, notification categories and actions,
APNs registration on macOS, remote notifications via `NSApplicationDelegate`
(`didReceiveRemoteNotification`), notification delivery when app is
in foreground (macOS delivers by default differently than iOS),
Focus modes impact on macOS notifications.

**Exa queries:**
- `UNUserNotificationCenter macOS 26 authorization push 2025`
- `APNs registration macOS NSApplicationDelegate remote notification`
- `macOS notification foreground delivery UNUserNotificationCenterDelegate`
- `macOS notification category action button 2025`
- `macOS Focus mode notification interruption level`

---

#### `swift-security-biometrics-macos/SKILL.md`
*(Can merge into `swift-security-macos` — include Touch ID, Apple Watch
unlock, `LAPolicy.deviceOwnerAuthenticationWithBiometrics` vs
`deviceOwnerAuthentication` on macOS. Cover separately only if depth
warrants.)*

---

## Output Structure

```
macos-skills/
├── README.md
├── swift-language-macos/
│   ├── SKILL.md
│   └── references/
├── swift-concurrency-macos/
│   ├── SKILL.md
│   └── references/
├── swift-architecture-macos/
│   ├── SKILL.md
│   └── references/
├── appkit-foundations/
│   ├── SKILL.md
│   └── references/
├── swiftui-appkit-interop/
│   ├── SKILL.md
│   └── references/
├── swiftui-macos-patterns/
│   ├── SKILL.md
│   └── references/
├── swiftui-macos-navigation/
│   ├── SKILL.md
│   └── references/
├── swiftui-macos-layout/
│   ├── SKILL.md
│   └── references/
├── swiftui-macos-windows-scenes/
│   ├── SKILL.md
│   └── references/
├── swiftui-macos-toolbar-sidebar/
│   ├── SKILL.md
│   └── references/
├── swiftui-macos-menus-commands/
│   ├── SKILL.md
│   └── references/
├── swiftui-macos-controls/
│   ├── SKILL.md
│   └── references/
├── swiftui-macos-liquid-glass/
│   ├── SKILL.md
│   └── references/
├── swiftui-macos-animation/
│   ├── SKILL.md
│   └── references/
├── swiftui-macos-gestures/
│   ├── SKILL.md
│   └── references/
├── swiftui-macos-performance/
│   ├── SKILL.md
│   └── references/
├── swiftui-macos-accessibility/
│   ├── SKILL.md
│   └── references/
├── macos-text-and-input/
│   ├── SKILL.md
│   └── references/
├── swiftui-webkit-macos/
│   ├── SKILL.md
│   └── references/
├── swiftdata-macos/
│   ├── SKILL.md
│   └── references/
├── core-data-macos/
│   ├── SKILL.md
│   └── references/
├── macos-file-system/
│   ├── SKILL.md
│   └── references/
├── cloudkit-macos/
│   ├── SKILL.md
│   └── references/
├── networking-macos/
│   ├── SKILL.md
│   └── references/
├── swift-security-macos/
│   ├── SKILL.md
│   └── references/
├── macos-security-entitlements/
│   ├── SKILL.md
│   └── references/
├── macos-system-integration/
│   ├── SKILL.md
│   └── references/
├── macos-ipc-xpc/
│   ├── SKILL.md
│   └── references/
├── macos-metal-gpu/
│   ├── SKILL.md
│   └── references/
├── macos-iokit-hardware/
│   ├── SKILL.md
│   └── references/
├── swift-testing-macos/
│   ├── SKILL.md
│   └── references/
├── debugging-macos/
│   ├── SKILL.md
│   └── references/
├── xcode-macos-toolchain/
│   ├── SKILL.md
│   └── references/
├── swiftlint-macos/
│   ├── SKILL.md
│   └── references/
├── swift-charts-macos/
│   ├── SKILL.md
│   └── references/
├── swift-codable-macos/
│   ├── SKILL.md
│   └── references/
├── swift-api-design-macos/
│   ├── SKILL.md
│   └── references/
├── storekit-macos/
│   ├── SKILL.md
│   └── references/
├── apple-on-device-ai-macos/
│   ├── SKILL.md
│   └── references/
├── localization-macos/
│   ├── SKILL.md
│   └── references/
├── swift-formatstyle-macos/
│   ├── SKILL.md
│   └── references/
└── push-notifications-macos/
    ├── SKILL.md
    └── references/
```

---

## Quality Bar

- Every API or build flag verified against a live Exa source.
- Code examples Swift 6.3 / macOS 26 unless a legacy pattern is
  explicitly labeled as such.
- No padding — every sentence is agent-actionable.
- Minimum 500 words substantive content per `SKILL.md`.
- At least one `references/` file per skill with deep patterns and
  code examples that would overflow the main file.
- Where the iOS repo covers the same topic, match or exceed its depth.
  Study its structure; do not copy its content.
