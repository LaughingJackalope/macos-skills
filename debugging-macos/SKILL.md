---
name: debugging-macos
description: "Debugging on macOS — LLDB for AppKit, Memory Graph Debugger, Instruments (Hangs, Time Profiler, Allocations, Metal System Trace), os_log/os_signpost, dtrace, crash report symbolication, sandbox violation debugging, Address Sanitizer, Thread Sanitizer. Use when debugging performance, memory, or crash issues in macOS apps."
sources: https://developer.apple.com/documentation/xcode/identifying-responsiveness-issues-in-your-app, https://developer.apple.com/documentation/os/logging, https://developer.apple.com/videos/play/wwdc2023/10248/, https://developer.apple.com/videos/play/wwdc2025/306/, https://developer.apple.com/documentation/os/ossignposter, https://developer.apple.com/documentation/xcode/adding-identifiable-symbol-names-to-a-crash-report
>
> **Updated for 2026:** Instruments 26 (Xcode 26) introduces a new SwiftUI instrument and enhanced Hangs analysis. WWDC25 session "Optimize SwiftUI performance with Instruments" covers the new workflow. The `OSSignposter` API (macOS 12+) is the modern replacement for the C-based `os_signpost()` functions.
researched: 2026-06-06
platform: macOS 26+
---

# Debugging on macOS

Debugging and profiling toolkit for macOS 26+ with Swift 6.3. Covers LLDB for AppKit and Swift, Memory Graph Debugger for retain cycles, Instruments templates for hangs/memory/GPU/energy, os_log and os_signpost for structured logging, dtrace for kernel-level tracing, crash report symbolication, sandbox violation diagnostics, and sanitizers (ASan, TSan, UBSan, Main Thread Checker). This skill is the primary reference for diagnosing crashes, performance stalls, memory growth, threading violations, and entitlement/sandbox issues in macOS apps.

See [references/debugging-patterns.md](references/debugging-patterns.md) for deep-dive LLDB breakpoint commands for AppKit, Instruments Hangs trace reading, os_log/signpost with Instruments integration, dtrace probes for filesystem and network, and ASan/TSan build settings and interpretation.

## Contents

- [LLDB for macOS](#lldb-for-macos)
- [Memory Graph Debugger](#memory-graph-debugger)
- [Instruments Templates](#instruments-templates)
- [os_log & os_signpost](#os_log--os_signpost)
- [dtrace](#dtrace)
- [Crash Reports](#crash-reports)
- [Sandbox Debugging](#sandbox-debugging)
- [Sanitizers](#sanitizers)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## LLDB for macOS

LLDB is the primary debugger for macOS apps. Xcode's debugger is a GUI over LLDB — everything achievable in the GUI can be done in the LLDB console, and many tasks (watchpoints, memory inspection, scriptable breakpoints) are CLI-only.

**Setting breakpoints:**

```bash
# By symbol name — works for Swift and Objective-C
(lldb) breakpoint set --name "-[NSWindow setContentView:]"
(lldb) b NSMakeRect           # shorthand

# By file and line
(lldb) breakpoint set --file MyController.swift --line 42
(lldb) b MyController.swift:42

# By Objective-C selector
(lldb) breakpoint set --selector "-[NSView mouseDown:]"

# Regex breakpoint — all methods containing "draw" in any class
(lldb) breakpoint set --func-regex "draw"

# Conditional breakpoint
(lldb) breakpoint set --name "-[NSView addSubview:]" --condition '(BOOL)[(id)$arg1 isKindOfClass:[NSButton class]]'
```

Swift function names are mangled. Use `image lookup -r -n "MyClass\.methodName"` to find the mangled form, or use the GUI to set the breakpoint and read the resolved name from the console.

>
> **Watchpoints** — halt when a variable's memory is read or written. Hardware-limited (typically 4 watchpoints per architecture):
>
> ```bash
> # Watch a variable for writes (default)
> (lldb) watchpoint set variable myView.bounds
> (lldb) watchpoint set expression -- (void*)0x12345678
>
> # Watch for reads only
> (lldb) watchpoint set expression -s 8 -w read -- myPointer
>
> # Watch for both reads and writes with size
> (lldb) watchpoint set expression -s 4 -w read_write -- globalCounter
>
> # List all watchpoints
> (lldb) watchpoint list
>
> # Watch a condition on a watched variable
> (lldb) watchpoint modify -c '(int)count == 3'
>
> # Delete when done (frees hardware resources)
> (lldb) watchpoint delete 1
> ```
>
> **Limitations:** Watchpoints are hardware-backed — most architectures support only 4 simultaneous watchpoints. Watched memory size is capped at 8 bytes. Works reliably on reference types (`NSObject` subclasses, `AnyObject`). Swift value types (structs, enums) do not have stable memory addresses unless heap-allocated.

**Inspection:**

```bash
# Print object description (equivalent of -description / debugDescription)
(lldb) po myArray
(lldb) expression -- myArray          # raw output (no description)
(lldb) expr -O -- myArray             # same as po

# Print typed value
(lldb) p (int)window.level
(lldb) p (BOOL)[myView isFlipped]

# Swift expression: use `expression` with import context
(lldb) expression -l swift -- import Foundation
(lldb) expression -l swift -- let frame = NSMakeRect(0,0,100,100)
```

**View hierarchy:**

```bash
# Print the entire view hierarchy from a window
(lldb) po [[[NSApp keyWindow] contentView] _subtreeDescription]

# Layer tree (for layer-backed views)
(lldb) po [myView _subtreeDescription]

# Print first responder chain
(lldb) expr -l objc -O -- [keyWindow firstResponder]
(lldb) po [keyWindow firstResponder]
```

**Backtraces:**

```bash
# All threads
(lldb) thread backtrace all
(lbbs) bt all

# Current thread
(lldb) bt

# Select a frame and inspect locals
(lldb) frame select 3
(lldb) frame variable
```

For AppKit-specific patterns including breakpoint commands for NSView/NSWindow lifecycle and first responder debugging, see [references/debugging-patterns.md](references/debugging-patterns.md).

## Memory Graph Debugger

The Memory Graph Debugger (Xcode ≥ 8) captures a snapshot of all live objects and their references, making it the fastest way to find retain cycles in Swift and over-retained AppKit objects.

**To capture the graph:**

1. Run the app in Xcode.
2. Reproduce the leak or retain cycle.
3. Click the **Debug Memory Graph** button (three connected circles) in the debug bar, or press `⌃⌘M`.
4. The left panel shows all live objects grouped by type.
5. The right panel shows the reference graph — arrows indicate strong references.
6. Purple ⚠️ icons indicate potential memory issues (cycles or leaked objects).

**What to look for:**

- **Retained NSView/NSViewController that should be deallocated.** If you dismiss a view controller and its view stays alive, select the controller in the graph and trace what still holds it. Common culprits:
  - A `displayLink` or `Timer` retaining `self` via the target. Use `[weak self]` or `CADisplayLink(target:proxy, selector:)`.
  - A notification center block observer that captures `self`. Always store the observer token and remove it in `deinit`.
  - The responder chain — `nextResponder` is strong for views. Ensure no view controller's view references the controller outside the responder chain contract.
  - Core Animation animations (`CAAnimation`) — their `delegate` is strong. Use a weak proxy.

- **Swift reference cycles in closures.** Capture lists `[weak self]` or `[unowned self]` break cycles. Look at the graph for closure objects (`(closure #1 in ...)`) that reference a view controller VC that also references the closure.

- **Foundation objects with unexpected retainers.** `NSAttributedString` holding a reference to a view via a custom attribute, `NSFileCoordinator` holding onto a presenter, `NSXPCConnection` not invalidated.

**Swift-specific:** In Xcode 15+, the Memory Graph Debugger shows Swift reference types (strong, weak, unowned) with distinct arrow styles. Weak/unowned references are dashed. A cycle of solid arrows is a guaranteed leak.

**Exporting:** Right-click in the graph area → Export to `.memgraph` file. Can be opened later in Xcode or analyzed with `leaks` command-line tool.

## Instruments Templates

Instruments is the profiling suite bundled with Xcode. Each "template" is a collection of recording instruments. Launch with `⌘I` in Xcode or via `instruments` CLI.

**Key templates for macOS:**

| Template | Detects | Key Detail |
|----------|---------|------------|
| **Time Profiler** | CPU hotspots, slow functions | Statistical sampling of the call tree. Not real-time — samples every 1ms by default. |
>
> ### Advanced Hangs Analysis (WWDC23/25)
>
> WWDC23 "Analyze hangs with Instruments" and WWDC25 "Optimize SwiftUI performance with Instruments" refined the hangs workflow:
>
> **Multi-instrument correlation:** Combine Hangs + Time Profiler + Swift Tasks + Thread State in a single document. The Thread State instrument reveals blocked/sleeping periods that Time Profiler misses entirely — a thread showing 0% CPU in Time Profiler might be blocked on a lock, not idle.
>
> **Narrative view:** Instruments generates a textual narrative explaining why each thread was blocked ("Thread was waiting for the main queue" or "Thread was sleeping"). This replaces manual stack trace interpretation for common hang patterns.
>
> **Swift Tasks integration:** The Swift Tasks instrument (Xcode 15+) shows async task lifecycle alongside hangs. A hang on the main thread correlated with a suspended Swift Task indicates an `await` that is blocked, not just slow synchronous code.
>
> **Instruments 26 (Xcode 26, 2026):** Adds a new SwiftUI instrument for detailed body re-evaluation tracking on macOS. Requires the latest macOS and Xcode 26.
| **Hangs** | Main thread stalls > 1ms | Records every time the main thread is blocked. Pinpoints the exact call stack at the moment of the stall. |
| **Allocations** | Memory growth, transient peaks | Tracks every `malloc`/`alloc`/`new`. Use "Generation" markers (`⌘G`) to see allocations between two points. |
| **Leaks** | Leaked heap memory | Periodically scans heap for unreachable objects. Works on macOS but less precise than the Memory Graph Debugger for reference cycles. |
| **Metal System Trace** | GPU stalls, pipeline waits, command buffer issues | Records GPU timeline, shader duration, resource binding, and CPU-side Metal calls. |
| **Energy Log** | Excessive CPU/GPU/disk/network energy | Good for diagnosing battery drain on laptops. |
| **SwiftUI** | Body re-evaluations, state mutations | Only available in macOS 14+. Shows which `@State`/`@ObservedObject` mutations trigger body re-evaluation and how long each evaluation takes. |
| **Network** | URL session activity, connection lifecycle | Replaces the deprecated Network template. |
| **File Activity** | File opens, reads, writes, locks | Useful for diagnosing coordinated file access issues. |

**Workflow for Hangs (main thread stalls):**

1. Profile with the Hangs template.
2. Reproduce the stall. The trace shows "Hangs" with red bars for stalls > threshold.
3. Click a hang → the detail pane shows the **heaviest stack trace** during the stall.
4. Double-click a frame to jump to the source code.
5. Common culpacts from the hang itself (use `DispatchQueue.main.async` for work from background threads), synchronous XPC calls, `dispatch_sync` to main from background.

For detailed Hangs trace reading workflow, see [references/debugging-patterns.md](references/debugging-patterns.md).

## os_log & os_signpost

`os_log` is Apple's unified logging system, replacing `NSLog` and `print` for production diagnostics. `os_signpost` defines intervals for Instruments integration.

```swift
import OSLog

// Logger with subsystem and category
let logger = Logger(subsystem: "com.example.MyApp", category: "rendering")
let networkLogger = Logger(subsystem: "com.example.MyApp", category: "network")

// Log levels
logger.trace("Detailed frame info: \(frame)")          // `.debug` level, not in release by default
logger.debug("Frame rendered in \(ms)ms")
logger.info("Viewport ready")
logger.notice("Cache warmed")                          // default level
logger.error("Render failed: \(error)")
logger.critical("GPU device lost")                      // highest severity; always preserved
```

**Subsystem and category** let you filter logs in Console.app and `log` CLI. Use reverse-DNS for the subsystem (matching your bundle ID). Categories group related log statements (e.g., "rendering", "network", "persistence").

**Signpost intervals** — link log statements into Instruments timelines:

```swift
import os.signpost

let signpostLog = OSLog(subsystem: "com.example.MyApp", category: .pointsOfInterest)

func renderFrame() {
    let id = os_signpost_id_generate(signpostLog.log)
    os_signpost(.begin, log: signpostLog, name: "RenderFrame", signpostID: id)
    
    // ... actual rendering ...
    
    os_signpost(.end, log: signpostLog, name: "RenderFrame", signpostID: id)
>
> ### Modern OSSignposter API (macOS 12+)
>
> The `OSSignposter` class (Swift) replaces the C-based `os_signpost()` functions for most use cases. It provides a cleaner, type-safe interface:
>
> ```swift
> import OSLog
>
> struct PerformanceLogger {
>     private let signposter = OSSignposter(logger: Logger(subsystem: "com.example.MyApp", category: "rendering"))
>
>     func renderFrame() {
>         let id = signposter.makeSignpostID()
>         let state = signposter.beginInterval("RenderFrame", id: id, "Frame \(id)")
>
>         // ... actual rendering ...
>
>         signposter.endInterval("RenderFrame", state)
>     }
>
>     // One-shot event
>     func userDidSelect(_ item: Item) {
>         signposter.emitEvent("ItemSelection", "Selected \(item.id)")
>     }
> }
> ```
>
> **Advantages over `os_signpost()`:** Automatic signpost ID generation, type-safe interval state object, works with Swift Concurrency (`async` intervals via `beginInterval` returning a `OSSignpostIntervalState`).
}
```

The signpost intervals appear in the Points of Interest track in Instruments (Time Profiler, Hangs, or the custom os_signpost instrument). This is the most effective way to correlate your app's logical operations with system-level performance data.

**Viewing logs:**

```bash
# Stream live from a device
log stream --predicate 'subsystem == "com.example.MyApp"' --level debug

# Filter by category
log stream --predicate 'subsystem == "com.example.MyApp" AND category == "network"'

# Show a process's historical logs
log show --predicate 'subsystem == "com.example.MyApp"' --last 1h
```

For build-time structured logging integration with Xcode organizers and Patterns, see the `networking-macos` and `coredata-macos` skills respectively.

## dtrace

dtrace provides kernel-level dynamic tracing — you can probe any syscall, any kernel function, any user-space function with a probe. On macOS, dtrace requires **SIP to be partially disabled** for most providers (see Constraints).

**Syscall tracing:**

```bash
# Trace all file-opens by your process
sudo dtrace -n 'syscall::open*:entry /execname == "MyApp"/ { printf("%s %s", execname, copyinstr(arg0)); }'

# Trace network connects
sudo dtrace -n 'syscall::connect:entry /execname == "MyApp"/ { printf("connect to %d", arg1); }'
```

**Swift function probing** (requires the function to not be inlined; use `@_optimize(none)` for critical probes):

```bash
# Probe a specific Swift function (mangled name)
sudo dtrace -n 'pid$target::*MyClass*myMethod*:entry { printf("entered"); }' -p $(pgrep MyApp)
```

**One-liners for common tasks:**

```bash
# Top syscalls by frequency for a process
sudo dtrace -n 'syscall:::entry /execname == "MyApp"/ { @num[probefunc] = count(); }'

# Files opened (with full path)
sudo dtrace -n 'syscall::open*:entry /execname == "MyApp"/ { @[copyinstr(arg0)] = count(); }'

# Print stack trace on a specific exception/AppKit error
sudo dtrace -n 'objc$target:-[NSView mouseDown:]:entry { ustack(10); }' -p $(pgrep MyApp)
```

For comprehensive dtrace probes for filesystem, network, and specific AppKit operations, see [references/debugging-patterns.md](references/debugging-patterns.md).

## Crash Reports

macOS writes crash reports to `~/Library/Logs/DiagnosticReports/` for per-user crashes and `/Library/Logs/DiagnosticReports/` for system daemons.

**Crash log anatomy:**

```
Exception Type:  EXC_BAD_ACCESS (SIGSEGV)
Exception Codes: KERN_INVALID_ADDRESS at 0x0000000000000010
Crashed Thread:  0  Dispatch queue: com.apple.main-thread

Thread 0 Crashed:
0   libobjc.A.dylib                 objc_msgSend + 32
1   MyApp                           0x100004ab2 -[MyController updateView] + 138
2   MyApp                           0x100004a1 -[MyController viewDidLoad]
3   AppKit                          -[NSViewController _loadViewIfRequired] + 82
...

Binary Images:
0x100000000 -        0x10001ffff MyApp (1.0 - 123) <UUID-HERE> /path/to/MyApp
0x1a2b3c000 -        0x1a2b7ffff libobjc.A.dylib <UUID> /usr/lib/libobjc.A.dylib
```

Key sections:
- **Exception Type**: `EXC_BAD_ACCESS` (memory error), `EXC_CRASH (SIGABRT)` (failed assertion/uncaught exception), `EXC_BREAKPOINT (SIGTRAP)` (Swift runtime trap — force unwrap nil, array out of bounds, precondition failure), `EXC_GUARD` (guarded_file_description violation).
- **Crashed Thread**: The thread number and queue name. Thread 0 is the main thread.
- **Backtrace**: Stack at crash. Addresses are hex, without symbols by default.
- **Binary Images**: Load addresses and UUIDs for every loaded library/framework. Used for symbolication.

**Symbolication with `atos`:**

```bash
# Symbolicate a single address
atos -o MyApp.app.dSYM/Contents/Resources/DWARF/MyApp -l 0x100000000 0x100004ab2

# Symbolicate the full crash log (when the app binary is available)
# Match the UUID in the crash log to the dSYM dwarfdump --uuid MyApp.app.dSYM
```

>
> **Symbolicating kernel/KEXT backtraces:**
>
> For macOS kernel panic reports, `atos -l` uses the load address relative to the first `__TEXT` segment. Adjust the load address accordingly:
>
> ```bash
> # For kernel backtraces, find the kernel base from the "Kernel slide" in the panic log
> atos -arch arm64 -o /System/Library/Kernels/kernel.release.t610 -l 0xfffffe0007004000 0xfffffe0007abcdef
> ```
>
> Always verify the UUID match: `dwarfdump --uuid` must match the binary UUID in the crash log.
**`symbolicatecrash`** — Apple's batch symbolication script (if available — not bundled with recent Xcode, may need to be copied from Xcode 14 or extracted from Xcode.app):

```bash
# Set the DEVELOPER_DIR
export DEVELOPER_DIR=$(xcode-select -p)

# Symbolicate
/Applications/Xcode.app/Contents/SharedFrameworks/DVTFoundation.framework/Versions/A/Resources/symbolicatecrash myapp.crash MyApp.app.dSYM > symbolicated.crash
```

**Keeping .dSYM files:**
- Archive builds in Xcode (Product → Archive) — Organizer stores the dSYM alongside the archive.
- On CI, export `ENABLE_DSYM=YES` in your xcodebuild settings and copy the `dSYM.zip` artifact.
- Use Spotlight to find dSYMs: `mdfind "com_apple_xcode_dsym_uuids == *"` or filter by UUID.
- Without the dSYM, crash logs show hex addresses instead of symbols. Always store them.

For specific build settings to preserve debug info for distribution builds, see the `xcode-macos-toolchain` skill.

## Sandbox Debugging

When a sandboxed app is denied file access or a system resource, the denial is logged to the system log and (on macOS 14+) shown in the Console app under your process.

**Console.app filter:**

1. Open Console.app ( `/Applications/Utilities/Console.app`).
2. Start streaming.
3. In the search bar, enter `subsystem:com.apple.TCC` or `category:sandbox`.
4. Reproduce the violation — denied operations appear with the sandbox violation, the path, and the required entitlement.

Example log entry:

```
Sandbox: MyApp(1234) deny(1) file-read-data /Users/me/Documents/secret.txt
```

**Using `sandbox-exec` with a custom profile:**

Create a sandbox profile (SBPL format):

```
(version 1)
(debug deny)
(deny file-read*
    (subpath "/Users/me/Private")
    (with no-log))  ; suppress logging for this rule
(allow file-read*
    (subpath "/Users/me/Documents"))
```

```bash
# Run under the profile
sandbox-exec -f my.sbpl /Applications/MyApp.app/Contents/MacOS/MyApp
```

This lets you test a stricter or different sandbox without changing entitlements.

**Provenance and container:**

```bash
# Check the container paths
ls -la ~/Library/Containers/com.example.MyApp/

# Verify the container manager
container_managerctl list
```

## Sanitizers

Sanitizers instrument your binary at compile time to detect undefined behavior, memory corruption, and threading violations at runtime. Enable in Xcode: Edit Scheme → Run → Diagnostics. Set via `xcodebuild` with sanitizer-specific flags.

**Address Sanitizer (ASan)** — Detects:

- Stack and heap buffer overflows
- Use-after-free and use-after-return
- Use-after-scope (variable used after end of scope)
- Double-free and memory leaks (with `detect_leaks=1`)

Enable: Xcode Diagnostics → Address Sanitizer, or `xcodebuild OTHER_CFLAGS="-fsanitize=address" OTHER_LDFLAGS="-fsanitize=address"`.

At runtime, ASan adds a shadow memory byte for every 8 bytes of app memory. Overhead: ~2× memory, ~2× CPU. Output includes the bad address, the allocation site, and the deallocation site.

Output example:
```
=================================================================
==12345==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000002ab4
WRITE of size 4 at 0x602000002ab4 thread T0
    #0 0x100004ab2 in processItems + 138
    #1 0x100004910 in -[MyController viewDidLoad]
  ...
```

**Thread Sanitizer (TSan)** — Detects:

- Data races (two threads access the same memory without synchronization, at least one write)
- Thread leaks
- Misuse of grand central dispatch (dispatch_sync to current queue — deadlock)

Enable: Xcode Diagnostics → Thread Sanitizer. Cannot be combined with ASan or Malloc Scribble in the same scheme. Overhead: ~5-15× memory, ~5-15× CPU.

**Main Thread Checker** — Detects AppKit/AppKit calls made from background threads:

- `NSView`, `NSWindow`, `NSViewController`, `NSTableView` accessed off main thread.
- Enable: Xcode Diagnostics → Main Thread Checker. Low overhead; leave on during development.

**Undefined Behavior Sanitizer (UBSan)** — Detects:

- Integer overflow
- Null pointer dereference (for non-ObjC/Swift)
- Misaligned pointer
- Invalid enum value
- Division by zero
- Out-of-bounds array access ObjC only with the ObjC ARC sub-kind)

Enable: Xcode Diagnostics → Undefined Behavior Sanitizer. Combine with ASan for maximum coverage.

**Conflicts and exclusions:**

| Combination | Works? |
|-------------|--------|
| ASan + TSan | ❌ Mutual exclusion |
| ASan + UBSan | ✅ |
| ASan + Main Thread Checker | ✅ |
| TSan + Main Thread Checker | ✅ |
| All three | ❌ (ASan+TSan conflict) |

## Constraints & Gotchas

- **dtrace and SIP:** Most dtrace providers (syscall, pid, fbt, objc) require disabling SIP. SIP can be partially disabled via `csrutil enable --without dtrace` from Recovery Mode. Without this, dtrace probes fail with "dtrace: failed to initialize dtrace: DTrace requires additional privileges." Some fbt or dtrace providers remain restricted even with this flag on macOS 15+.
- **dtrace and System Extensions:** The `mac:’` provider on macOS 15+ requires the client task to be CS_PLATFORM, 1) or SIP fully disabled.
- **Instruments SwiftUI template:** Only available in macOS 14+ and Xcode 15+. On macOS 13, the SwiftUI instrument does not appear but replay does not crash.
- **ASan in Release builds:** Never ship with ASan enabled. The runtime is not App Store-compatible. Always use DEBUG or a dedicated profiling scheme.
- **Sanitizer and Debug:** Sanitizers require debug symbols (dSYM or unstripped). If you strip the binary, sanitizer output will lack symbolicated stack traces.
- **watchpoint on Swift value types:** Swift structs and enums do not have stable memory addresses for watchpoint purposes unless stored in `inout` or heap-allocated (`AnyObject`, `UnsafeMutablePointer`). Watchpoints work reliably only on reference types (` NSObject subclass, AnyObject`).
- **`po` in LLDB with Swift 6:** `po` invokes `debugDescription` / `CustomDebugStringConvertible`. In release-optimized builds, `po` may produce incomplete output. Use `expression -l swift --` for raw memory reads.
- **Instruments and bitcode:** If your app framework uses bitcode (Xcode generates bitcode by default for macOS), Instruments may fail to attribute symbols. Disable bitcode with `ENABLE_BITCODE=NO` for profiling builds.
- **os_log density:** `debug` and `trace` level messages are stored (not just in-memory buffers) and subject to privacy redaction. Default `.private` strings are redacted for non-`{public}` values in release builds. Use `%{public}@` format specifier in format strings if the value must be visible in the log.

## Common Mistakes

1. **`NSLog` in production.** Use `os_log`. `NSLog` writes to stderr with no level control, no subsystem filtering, and no privacy management. It is visible to every process that can read stderr — a data leak vector.
2. **Ignoring the hang threshold in Instruments Hangs template.** The default threshold is 1ms. A hang of 500µs is invisible. Lower the threshold in the instrument config if you need to catch sub-millisecond stalls (note: USB HID mice run at 125Hz = 8ms, so tracking Hangs < 1ms requires a custom HID rate or ProMotion display).
3. **Running ASan-enabled builds on a different machine than the developer's.** ASan-instrumented binaries depend on the ASan runtime shipped with the OS. A macOS 13 machine cannot run a macOS 15 ASan binary without the matching runtime.
4. **Disabling Main Thread Checker in CI.** The checker has negligible overhead and catches real bugs. Leave it on in all Debug schemes on CI.
5. **Using `atos` with the wrong UUID.** The dSYM UUID must match the binary UUID in the crash log, not the app wrapper. Verify with `dwarfdump --uuid`.
6. **Storing debugDescription` that references other objects.** `po` calls `debugDescription`, which if it reads properties, can trigger lazy initialization or access deallocated memory. Keep `debugDescription` a pure string formatting operation.
7. **Attaching Instruments to a hardened + notarized binary.** Hardened runtime with `com.apple.security.cs.debugger` denied (the default in release) prevents profiling. Use a Development-signed build for Instruments profiling.
8. **Using dispatch_sync to main queue from a background thread in Swift.** This is a deadlock if the main queue is busy. Use `DispatchQueue.main.async` instead. TSan detects this deadlock. Worse: dispatching synchronously from main to any queue that targets main causes an instant crash.
9. **Not checking `.dSYM` arch.** For universal binaries (Apple Silicon + Intel invest in an arm64), the dSYM may be per-architecture. Use `dwarfdump --uuid --all` to see all embedded architectures.

## Review Checklist

- [ ] Crash symbolication: every crash log reviewed with `atos` or `symbolicatecrash`, not raw hex addresses
- [ ] Memory: Memory Graph Debugger captured after each major user flow (open document, close document, present modal, dismiss modal)
- [ ] Instruments runs: Hangs template captured with threshold ≤ 1ms for all interactive flows (scrolling, typing, window resize)
- [ ] os_log coverage: every error path has at least one `.error` or `.fault` log; every user-visible flow has a signpost interval
- [ ] dtrace: if investigating a system-level issue, SIP has been partially disabled and probes validated on target OS version
- [ ] Sanitizers: ASan run for every Debug CI build; TSan run at least nightly; Main Thread Checker always on in Debug
- [ ] Watchpoints: used to track unexpected mutation of view properties, controller state, or shared model objects during reproduction
- [ ] Sandbox: Console.app checked for `sandbox deny` entries after each test run in the sandbox
- [ ] dSYM archives: matched and stored for every shipped build

## References

- [Identifying Responsiveness Issues in Your App](https://developer.apple.com/documentation/xcode/identifying-responsiveness-issues-in-your-app)
- [Logging (os_log)](https://developer.apple.com/documentation/os/logging)
- [Instruments User Guide](https://help.apple.com/instruments/mac/current/)
- [LLDB Quick Start](https://lldb.llvm.org/use/tutorial.html)
- [Address Sanitizer](https://developer.apple.com/documentation/xcode/diagnosing_memory_thread-and-crash-issues-early)
- [Diagnosing Issues Using Crash Reports and Device Logs](https://developer.apple.com/documentation/xcode/diagnosing-issues-using-crash-reports-and-device-logs)
- [WWDC 22: Detect and diagnose memory issues](https://developer.apple.com/videos/play/wwdc2022/10106/)
- [WWDC 23: Analyze hangs with Instruments and Xcode Organizer](https://developer.apple.com/videos/play/wwdc2023/10058/)
- [EC2 dtrace restrictions on macOS](https://developer.apple.com/forums/thread/719862)
- [WWDC 23: Analyze hangs with Instruments](https://developer.apple.com/videos/play/wwdc2023/10248/)
- [WWDC 25: Optimize SwiftUI performance with Instruments](https://developer.apple.com/videos/play/wwdc2025/306/)
- [OSSignposter | Apple Developer Documentation](https://developer.apple.com/documentation/os/ossignposter)
- [Symbolicating crash reports with atos | Apple Developer Documentation](https://developer.apple.com/documentation/xcode/adding-identifiable-symbol-names-to-a-crash-report)
- [How to Manually Symbolicate a macOS Crash Log](https://daveceddia.com/manually-symbolicate-crash-log-macos-app/)
- [LLDB Watchpoints | Apple Developer Documentation](https://developer.apple.com/library/archive/documentation/General/Conceptual/lldb-guide/chapters/C3-Breakpoints.html)
