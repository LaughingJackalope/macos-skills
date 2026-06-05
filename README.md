# macOS Skills

Production macOS 26+ development skills covering the AppKit foundation, SwiftUI-AppKit interop, window/scene architecture, menus, toolbars, system integration, text editing, security/entitlements, Xcode toolchain, and Metal/GPU programming. Designed to complement `swift-ios-skills` with zero redundancy.

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

## Scope Boundaries

These skills cover macOS-exclusive or macOS-differentiated surfaces. Shared Apple platform foundations (Swift language, SwiftUI state, NavigationSplitView basics, SwiftData, Core Data, Keychain, CryptoKit, CloudKit, StoreKit, Instruments, etc.) are owned by `swift-ios-skills` and referenced, not duplicated.

## Format

Each skill directory contains:
- `SKILL.md` — router with overview, key patterns, constraints, common mistakes, review checklist
- `references/*.md` — deep-dive patterns, complete code examples, decision tables
