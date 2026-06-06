# macOS Skills

Production macOS 26+ development skills covering the full spectrum of macOS development: Swift language & concurrency, app architecture, AppKit, SwiftUI, data persistence, networking, security, system integration, hardware, testing, tooling, and shared Apple frameworks. Designed to complement `swift-ios-skills` with zero redundancy — each skill covers the full topic with macOS-specific depth.
:Skills are grounded in live web research (Apple Developer Documentation, WWDC sessions, Tech Notes, and community deep-dives) — not just model training data. Each skill's `sources:` frontmatter lists 2–56 verified URLs.

## Skills

| Skill | Trigger | Reference |
|-------|---------|-----------|
| `appkit-foundations` | NSWindow, NSViewController, NSView hierarchy, responder chain, NSDocument, layer-backed drawing, drag & drop, printing | [SKILL.md](appkit-foundations/SKILL.md) |
| `swiftui-appkit-interop` | NSViewRepresentable, NSViewControllerRepresentable, NSHostingController, NSHostingView, coordinator pattern, AppKit/SwiftUI state sync | [SKILL.md](swiftui-appkit-interop/SKILL.md) |
| `macos-windows-scenes` | WindowGroup, Window, Settings, MenuBarExtra, openWindow, window restoration, WindowStyle, DocumentGroup | [SKILL.md](macos-windows-scenes/SKILL.md) |
| `macos-menus-commands` | .commands, CommandMenu, CommandGroup, keyboard shortcuts, context menus, NSMenu, dock menu, services menu | [SKILL.md](macos-menus-commands/SKILL.md) |
| `macos-toolbar-sidebar` | NavigationSplitView on macOS, toolbar placements, inspector, sidebar style, OutlineGroup, NSSplitViewController | [SKILL.md](macos-toolbar-sidebar/SKILL.md) |
| `macos-system-integration` | SMAppService, NSWorkspace, security-scoped bookmarks, NSPasteboard, Core Spotlight, Quick Look, AppleScript, NSFileCoordinator | [SKILL.md](macos-system-integration/SKILL.md) |
| `macos-text-and-input` | NSTextView, TextKit 2, NSAttributedString, find bar, keyboard events, NSFont, font panel | [SKILL.md](macos-text-and-input/SKILL.md) |
| `macos-security-entitlements` | Sandbox entitlements, Hardened Runtime, notarization, code signing, SMJobBless, SecureEnclave, SIP | [SKILL.md](macos-security-entitlements/SKILL.md) |
| `xcode-macos-toolchain` | xcodebuild macOS destinations, archive/export, notarytool, Mac Catalyst, universal binaries | [SKILL.md](xcode-macos-toolchain/SKILL.md) |
| `macos-metal-gpu` | MTKView, GPU selection, render/compute pipelines, MTLHeap, synchronization, CAMetalLayer, MSL | [SKILL.md](macos-metal-gpu/SKILL.md) |
| `swift-language-macos` | Swift 6.3 language patterns, macros, result builders, property wrappers, Sendable, @objc/AppKit interop | [SKILL.md](swift-language-macos/SKILL.md) |
| `swift-concurrency-macos` | Swift concurrency, actors, @MainActor, Task/TaskGroup, AsyncStream, strict concurrency, AppKit bridging, XPC concurrency | [SKILL.md](swift-concurrency-macos/SKILL.md) |
| `swift-architecture-macos` | @Observable MV, MVVM, NSDocument architecture, multi-window state, NSApplicationDelegate, dependency injection | [SKILL.md](swift-architecture-macos/SKILL.md) |
| `swiftui-macos-patterns` | @Observable MV, @State/@Binding/@Environment, .task/.onChange, @FocusedValue/@FocusedBinding, defaultFocus | [SKILL.md](swiftui-macos-patterns/SKILL.md) |
| `swiftui-macos-navigation` | NavigationSplitView 3-column, NavigationStack, TabView, sheet/popover, openWindow, deep linking | [SKILL.md](swiftui-macos-navigation/SKILL.md) |
| `swiftui-macos-layout` | VStack/HStack/ZStack, Table, List sidebar style, OutlineGroup, Form styles, Grid, GeometryReader | [SKILL.md](swiftui-macos-layout/SKILL.md) |
| `swiftui-macos-controls` | Button/Picker/Toggle styles, ColorPicker, FontPicker, PathControl, LevelIndicator, ControlGroup, controlSize | [SKILL.md](swiftui-macos-controls/SKILL.md) |
| `swiftui-macos-liquid-glass` | GlassEffect, GlassEffectContainer, glass toolbar/sidebar, NSVisualEffectView vibrancy, glassBackgroundDisplayMode | [SKILL.md](swiftui-macos-liquid-glass/SKILL.md) |
| `swiftui-macos-animation` | withAnimation, matchedGeometryEffect, PhaseAnimator, KeyframeAnimator, transitions, reduced motion | [SKILL.md](swiftui-macos-animation/SKILL.md) |
| `swiftui-macos-gestures` | TapGesture, DragGesture, onHover, cursor control, onScrollWheel, secondary click, NSEvent bridge | [SKILL.md](swiftui-macos-gestures/SKILL.md) |
| `swiftui-macos-performance` | Redraw prevention, EquatableView, TimelineView, lazy containers, Instruments, MainActor hop cost | [SKILL.md](swiftui-macos-performance/SKILL.md) |
| `swiftui-macos-accessibility` | VoiceOver, accessibility modifiers/actions/children, AXCustomContent, full keyboard access, Accessibility Inspector | [SKILL.md](swiftui-macos-accessibility/SKILL.md) |
| `swiftui-webkit-macos` | WebView (macOS 26), WKWebView NSViewRepresentable, navigation/UI delegates, JS bridge, PDF, downloads | [SKILL.md](swiftui-webkit-macos/SKILL.md) |
| `swiftdata-macos` | @Model, @Query, ModelContainer, CloudKit sync, migrations, multi-window containers, XPC history tracking | [SKILL.md](swiftdata-macos/SKILL.md) |
| `core-data-macos` | NSPersistentContainer, NSFetchedResultsController, batch ops, staged migration, NSPersistentCloudKitContainer | [SKILL.md](core-data-macos/SKILL.md) |
| `macos-file-system` | Security-scoped bookmarks, NSOpenPanel/NSSavePanel, NSFileCoordinator/NSFilePresenter, NSMetadataQuery, UTType | [SKILL.md](macos-file-system/SKILL.md) |
| `cloudkit-macos` | CKContainer/CKRecord/CKQuery, CKSyncEngine, NSPersistentCloudKitContainer, push sync, iCloud Drive, conflicts | [SKILL.md](cloudkit-macos/SKILL.md) |
| `networking-macos` | URLSession async/await, background sessions, Network.framework (NWConnection/NWBrowser), Bonjour, NWPathMonitor, ATS | [SKILL.md](networking-macos/SKILL.md) |
| `swift-security-macos` | Keychain, biometrics (Touch ID, Apple Watch), SecureEnclave, CryptoKit, Hardened Runtime, passkeys | [SKILL.md](swift-security-macos/SKILL.md) |
| `macos-ipc-xpc` | NSXPCConnection, NSXPCInterface, SMJobBless, App Groups, DistributedNotificationCenter | [SKILL.md](macos-ipc-xpc/SKILL.md) |
| `macos-iokit-hardware` | IOKit, HID devices (IOHIDManager), USB enumeration, power management (IOPMAssertion), CoreAudio, NSSound | [SKILL.md](macos-iokit-hardware/SKILL.md) |
| `swift-testing-macos` | Swift Testing (@Test/@Suite/#expect), XCTest UI testing, AppKit/XPC testing, xcodebuild test | [SKILL.md](swift-testing-macos/SKILL.md) |
| `debugging-macos` | LLDB, Memory Graph, Instruments (Hangs/Time Profiler/Metal), os_log/os_signpost, dtrace, crash symbolication, sanitizers | [SKILL.md](debugging-macos/SKILL.md) |
| `swiftlint-macos` | .swiftlint.yml, SPM/Xcode build plugin, macOS custom rules, CI integration, baseline, swiftlint analyze | [SKILL.md](swiftlint-macos/SKILL.md) |
| `swift-charts-macos` | Chart types, selection, scrollableAxes, chartOverlay, annotations, dark mode, print rendering, accessibility | [SKILL.md](swift-charts-macos/SKILL.md) |
| `swift-codable-macos` | Codable, JSON/PropertyList, NSCoding migration, NSKeyedArchiver bridge, NSUserDefaults, NSDocument serialization | [SKILL.md](swift-codable-macos/SKILL.md) |
| `swift-api-design-macos` | API design guidelines, AppKit delegate naming, @objc protocol design, error types (CocoaError/POSIXError), package design | [SKILL.md](swift-api-design-macos/SKILL.md) |
| `storekit-macos` | StoreKit 2, Product.products/purchase, Transaction.updates, subscriptions, SKOverlay, StoreView, sandbox testing | [SKILL.md](storekit-macos/SKILL.md) |
| `apple-on-device-ai-macos` | Foundation Models, @Generable, tool calling, CoreML (MLComputeUnits, MLTensor), model deployment, XPC isolation | [SKILL.md](apple-on-device-ai-macos/SKILL.md) |
| `localization-macos` | String Catalogs, LocalizedStringKey, plural rules (.stringsdict), RTL layout, menu localization, Xcode workflow | [SKILL.md](localization-macos/SKILL.md) |
| `swift-formatstyle-macos` | Date/Number/Measurement/List/Duration FormatStyle, custom FormatStyle, locale-aware formatting, macOS display patterns | [SKILL.md](swift-formatstyle-macos/SKILL.md) |
| `push-notifications-macos` | UNUserNotificationCenter, local/remote notifications, categories/actions, foreground delivery, Focus modes | [SKILL.md](push-notifications-macos/SKILL.md) |

## Format

Each skill directory contains:
- `SKILL.md` — router with overview, key patterns, constraints, common mistakes, review checklist
- `references/*.md` — deep-dive patterns, complete code examples, decision tables

All skills target **macOS 26+**, **Swift 6.3+**, and **Xcode 16+**. Code examples are macOS-first with no dependencies on iOS skill files.
