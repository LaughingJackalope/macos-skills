# Debugging Patterns for macOS

Deep-dive reference for LLDB breakpoint commands, Instruments Hangs trace reading, os_log/signpost with Instruments integration, dtrace probes for filesystem and network, and ASan/TSan build settings. Targets macOS 26+ with Swift 6.3.

## Contents

- [LLDB Breakpoint Commands for AppKit Debugging](#lldb-breakpoint-commands-for-appkit-debugging)
- [Instruments Hangs Trace Reading Workflow](#instruments-hangs-trace-reading-workflow)
- [os_log & os_signpost with Instruments Integration](#os_log--os_signpost-with-instruments-integration)
- [dtrace Probes for Filesystem and Network Operations](#dtrace-probes-for-filesystem-and-network-operations)
- [ASan/TSan Build Settings and Interpretation](#asantsan-build-settings-and-interpretation)

## LLDB Breakpoint Commands for AppKit Debugging

### NSView/NSWindow Lifecycle Breakpoints

These breakpoint commands automatically log lifecycle transitions without pausing execution. Add them via the LLDB console or an `.lldbinit` file in your home directory.

```bash
# Track every NSView added to a hierarchy — log the view class and its superview
(lldb) breakpoint set --name "-[NSView addSubview:]"
(lldb) breakpoint command add 1
Enter your debugger command(s). Type 'DONE' to end.
> po (id)$arg1
> po [(id)$arg1 superview]
> continue
> DONE

# Track key window changes
(lldb) breakpoint set --name "-[NSWindow becomeKeyWindow]"
(lldb) breakpoint command add 2
> expr -l objc -O -- (id)[NSApp keyWindow]
> continue
> DONE

# Track first responder changes
(lldb) breakpoint set --name "-[NSWindow makeFirstResponder:]"
(lldb) breakpoint command add 3
> po (id)$arg1
> continue
> DONE
```

### Conditional Breakpoints for Retain Cycle Detection

Break when a specific object's retain count hits an unexpected value. The count value includes the temporary retain from `return`:

```bash
# Break when a specific VC is retained (not the debugDescription one)
(lldb) breakpoint set --name "-[NSObject(NSKeyValueObserverNotification) didChangeValueForKey:]" --condition '(BOOL)[(id)$arg0 isKindOfClass:[MyViewController class]]'
(lldb) breakpoint command add 4
> po (id)$arg0
> po [(id)$arg0 retainCount]
> bt 5
> continue
> DONE
```

### Method Swizzling Detection via Breakpoint

Break when the same method is called more than N times in rapid succession (potential infinite loop through KVO or layout):

```bash
(lldb) breakpoint set --selector "-[NSView layout]"
(lldb) breakpoint modifier --auto-continue true
(lldb) breakpoint command add 5
> expression -O -- (int)$dumpCount++
> expr -l objc -O -- (int)$dumpCount > 100 && (void)(int)$dumpCount = 0
> continue
> DONE
```

### Printing Responder Chain Programmatically

In LLDB, execute this to walk and print the current responder chain:

```bash
(lldb) expr -l objc -- @import Foundation
(lldb) expr -l objc -- NSLog(@"%@", [[[NSApp keyWindow] firstResponder] _subtreeDescription])
(lldb) expr -l objc -- {
    NSResponder *current = [[NSApp keyWindow] firstResponder];
    int depth = 0;
    while (current) {
        printf("%.*s %s\n", depth * 2, "", [[current description] UTF8String]);
        current = [current nextResponder];
        depth++;
    }
}
```

### Swift-Era LLDB: Address Sanitizer Interaction

When ASan is enabled, LLDB's expression evaluator can conflict with ASan's shadow memory. If `po` or `hangs` with an `EXC_BAD_ACCESS` that does not occur without the debugger, try:

```bash
# Disable LLDB's expression timeout for ASan builds
(lldb) set set target.process.thread.step-avoid-regexp ""

# If ASan reports a false positive from an LLDB expression, use memory read instead
(lldb) memory read --size 8 --format x `(void *)myPointer`
```

## Instruments Hangs Trace Reading Workflow

### Step-by-Step Hangs Analysis

1. **Profile with Hangs template.** In Xcode: Product → Profile → Hangs. Or CLI: `instruments -t "Hangs" -D hangs.trace /path/to/MyApp.app`.

2. **Reproduce the hang.** Perform the user action that causes the stall (scrolling, typing, opening a document, resizing a window).

3. **Stop recording.** The trace opens automatically.

4. **Read the Hangs track:**
   - The top-level timeline shows "Hangs" as red blocks. Each block is one stall event.
   - The height of the block indicates duration (taller = longer stall).
   - The number on the block is the count of hangs at that call stack.

5. **Select a hang block.** The detail pane (bottom) shows:
   - **Heaviest Stack Trace**: The call stack that was on the main thread during the hang.
   - **Duration**: How long the main thread was blocked.
   - **Count**: How many times this exact stack trace caused a hang.

6. **Double-click a frame** in the heaviest stack to jump to source code.

7. **Common hang patterns and their signatures:**

   | Hang Signature | Likely Cause | Fix |
   |---|---|---|
   | `-[NSView drawRect:]` taking > 16ms | Expensive custom drawing | Use `setNeedsDisplay:`, cache rendered content, use `CALayer` with `drawsAsynchronously` |
   | `-[NSTableView reloadData]` | Full table reload on every change | Use `reloadDataForRowIndexes:columnIndexes:` or view-based diffable updates |
   | `dispatch_sync(main_queue)` from background | Synchronous main thread dispatch | Replace with `async`, or restructure to avoid needing the result synchronously |
   | `-[NSDocument readFromURL:]` on main thread | Synchronous file I/O on main | Use `NSDocument`'s async reading (`read(from:ofType:)` with completion handler) |
   | `-[NSWindow display]` or `-[NSView display]` | Forced display pass | Remove explicit `display` calls; use `setNeedsDisplay:` |
   | `xpc_connection_send_message_with_reply_sync` | Synchronous XPC | Use async XPC with reply block |
   | `CFReadStreamRead` / `NSInputStream` on main | Synchronous network I/O | Move to background thread or use `URLSession` async API |

8. **Correlate with Points of Interest.** If you added `os_signpost` intervals, the Points of Interest track shows your app's logical operations overlaid on the hangs timeline. This tells you *which operation* was running when the hang occurred.

9. **Export the trace.** File → Export to share with teammates or attach to a radar. The `.trace` file can be opened by anyone with the same Xcode version.

### CLI Analysis with `instruments`

```bash
# Record a 30-second Hangs trace
instruments -t "Hangs" -D /tmp/hangs.trace -l 30000 /path/to/MyApp.app

# List all available templates
instruments -s templates

# List all available instruments
instruments -s instruments
```

## os_log & os_signpost with Instruments Integration

### Structured Logging with Privacy

```swift
import OSLog

// Define loggers at file scope or as static properties
extension Logger {
    static let ui = Logger(subsystem: Bundle.main.bundleIdentifier!, category: "ui")
    static let database = Logger(subsystem: Bundle.main.bundleIdentifier!, category: "database")
    static let rendering = Logger(subsystem: Bundle.main.bundleIdentifier!, category: "rendering")
}

// Privacy: %{public}@ is visible in release builds; %{private}@ is redacted
Logger.ui.info("User tapped button: %{public}@", buttonIdentifier)
Logger.ui.debug("View frame: %{private}@", String(describing: view.frame))
```

### Signpost Intervals for Instruments

```swift
import os.signpost

// Create a signpost log — use .pointsOfInterest for the Points of Interest track
private let signpostLog = OSLog(
    subsystem: Bundle.main.bundleIdentifier!,
    category: .pointsOfInterest
)

// For custom signpost instruments, use a dedicated category
private let customLog = OSLog(
    subsystem: Bundle.main.bundleIdentifier!,
    category: "custom-rendering"
)

func performExpensiveOperation(items: [Item]) {
    // Generate a unique signpost ID for this interval
    let id = os_signpost_id_generate(signpostLog)
    
    os_signpost(.begin, log: signpostLog, name: "ProcessItems", signpostID: id,
                "Starting %{public}d items", items.count)
    
    // ... work ...
    
    os_signpost(.end, log: signpostLog, name: "ProcessItems", signpostID: id,
                "Completed %{public}d items", items.count)
}

// Event (instantaneous, not an interval)
func userDidSelectItem(_ item: Item) {
    os_signpost(.event, log: signpostLog, name: "ItemSelection",
                "Selected %{public}@", item.id)
}
```

### Viewing Signposts in Instruments

1. Open Instruments → choose "os_signpost" instrument (or use Hangs/Time Profiler which include the Points of Interest track).
2. Record your app.
3. The Points of Interest track shows begin/end intervals as horizontal bars.
4. Hover over a bar to see the name and any format string arguments.
5. Use the "Inspection Head" (the vertical line you scrub) to see which signpost intervals were active at any point in time.

### `log` CLI for Post-Hoc Analysis

```bash
# Stream live from the Mac
log stream --predicate 'subsystem == "com.example.MyApp"' --level debug --style compact

# Show only signpost-related entries
log stream --predicate 'subsystem == "com.example.MyApp" AND signpostID != 0'

# Export historical logs to a file
log show --predicate 'subsystem == "com.example.MyApp" AND eventMessage CONTAINS "ProcessItems"' \
    --last 24h --style json > /tmp/log_export.json

# Show fault-level (highest severity) entries only
log show --predicate 'subsystem == "com.example.MyApp" AND messageType == "Fault"' --last 7d
```

## dtrace Probes for Filesystem and Network Operations

### Filesystem Tracing

```bash
# Trace every file open by your app, with full path
sudo dtrace -n '
    syscall::open*:entry /execname == "MyApp"/ {
        printf("%s opened: %s", execname, copyinstr(arg0));
    }
'

# Trace file reads with byte counts
sudo dtrace -n '
    syscall::read:entry /execname == "MyApp"/ {
        self->fd = arg0;
        self->buf = arg1;
        self->nbyte = arg2;
    }
    syscall::read:return /execname == "MyApp" && self->fd/ {
        printf("read(%d) = %d bytes", self->fd, arg1);
        self->fd = 0;
    }
'

# Trace file coordination (NSFileCoordinator)
sudo dtrace -n '
    objc$target:NSFileCoordinator:-coordinate*Reading*Entry:entry {
        printf("FileCoordinator: coordinated read");
        ustack(10);
    }
' -p $(pgrep MyApp)

# Trace all vnode operations (kernel-level file access)
sudo dtrace -n '
    fbt::vnode_open:entry /execname == "MyApp"/ {
        printf("vnode_open: %s", stringof(args[0]->v_name));
    }
'
```

### Network Tracing

```bash
# Trace TCP connections with remote address
sudo dtrace -n '
    syscall::connect:entry /execname == "MyApp"/ {
        this->sa = (struct sockaddr *)copyin(arg1, arg2);
        this->sin = (struct sockaddr_in *)this->sa;
        printf("connect to %d.%d.%d.%d:%d",
            this->sin->sin_addr.s_addr & 0xff,
            (this->sin->sin_addr.s_addr >> 8) & 0xff,
            (this->sin->sin_addr.s_addr >> 16) & 0xff,
            (this->sin->sin_addr.s_addr >> 24) & 0xff,
            ntohs(this->sin->sin_port));
    }
'

# Trace sendto/recvfrom (UDP or raw socket)
sudo dtrace -n '
    syscall::sendto:return /execname == "MyApp"/ {
        @bytes[probefunc] = sum(arg1);
    }
    syscall::recvfrom:return /execname == "MyApp"/ {
        @bytes[probefunc] = sum(arg1);
    }
'

# Trace URLSession task creation (Objective-C method)
sudo dtrace -n '
    objc$target:NSURLSession:dataTaskWithRequest*:entry {
        printf("URLSession data task created");
        ustack(5);
    }
' -p $(pgrep MyApp)
```

### Process and Thread Tracing

```bash
# Trace thread creation
sudo dtrace -n '
    proc:::create /execname == "MyApp"/ {
        printf("New thread created by %s", execname);
        ustack(5);
    }
'

# Trace dispatch queue activity
sudo dtrace -n '
    objc$target:OS_dispatch_queue:-*:entry {
        printf("dispatch: %s", probefunc);
    }
' -p $(pgrep MyApp)
```

### dtrace Gotchas on macOS

- **SIP requirement:** `syscall`, `pid`, `objc`, and `fbt` providers all require SIP to be at least partially disabled. Run from Recovery Mode: `csrutil enable --without dtrace`. A full SIP disable (`csrutil disable`) also works but is not recommended.
- **App Sandbox:** Sandboxed apps cannot be traced with `pid` provider unless the tracing process has the `com.apple.security.cs.debugger` entitlement. Use `syscall` provider instead (it operates at kernel level).
- **System Integrity Protection on macOS 15+:** Some `fbt` probes are blocked even with `csrutil enable --without dtrace`. Use `dtrace -l` to list available probes and verify your target is present before building complex scripts.
- **Swift symbol mangling:** Swift symbols in dtrace `objc` probes must use the mangled name. Find it with: `nm /path/to/MyApp | grep MyClass` or `swift demangle` the output.

## ASan/TSan Build Settings and Interpretation

### Xcode Build Settings

| Setting | ASan Value | TSan Value | Notes |
|---------|-----------|-----------|-------|
| `ENABLE_ADDRESS_SANITIZER` | `YES` | `NO` | Mutually exclusive with TSan |
| `ENABLE_THREAD_SANITIZER` | `NO` | `YES` | Mutually exclusive with ASan |
| `CLANG_ADDRESS_SANITIZER_CONTAINER_OVERFLOW` | `YES` (default) | — | Detects overflow in Foundation containers |
| `CLANG_ADDRESS_SANITIZER_USE_AFTER_RETURN` | `YES` (default) | — | Detects use-after-return (stack) |
| `Malloc Scribble` | `NO` (conflicts) | `NO` (conflicts) | Disable when using any sanitizer |
| `Malloc Guard Edges` | `NO` (conflicts) | `NO` (conflicts) | Disable when using any sanitizer |
| `Zombie Objects` | `NO` (conflicts) | `NO` (conflicts) | Disable when using any sanitizer |
| `ENABLE_DTRACE` | `NO` | `NO` | DTrace conflicts with sanitizer instrumentation |

### xcodebuild Flags

```bash
# ASan build
xcodebuild -scheme MyApp \
    OTHER_CFLAGS="-fsanitize=address -fno-omit-frame-pointer" \
    OTHER_LDFLAGS="-fsanitize=address" \
    ENABLE_ADDRESS_SANITIZER=YES \
    ENABLE_THREAD_SANITIZER=NO \
    build

# TSan build
xcodebuild -scheme MyApp \
    OTHER_CFLAGS="-fsanitize=thread -fno-omit-frame-pointer" \
    OTHER_LDFLAGS="-fsanitize=thread" \
    ENABLE_THREAD_SANITIZER=YES \
    ENABLE_ADDRESS_SANITIZER=NO \
    build
```

### Interpreting ASan Reports

ASan output has a consistent structure:

```
=================================================================
==PID==ERROR: AddressSanitizer: <error-type> on address 0xADDR
<ACCESS-TYPE> of size <N> at 0xADDR thread T<id>
    #0 0xADDR in <function> <file>:<line>
    #1 0xADDR in <function> <file>:<line>
    ...

<ALLOCATION-THREAD>:
    #0 0xADDR in malloc
    #1 0xADDR in <allocating-function> <file>:<line>

<DEALLOCATION-THREAD> (if applicable):
    #0 0xADDR in free
    #1 0xADDR in <deallocating-function> <file>:<line>

SUMMARY: AddressSanitizer: <error-type> <file>:<line> in <function>
```

**Error types and what they mean:**

| Error Type | Meaning | Typical Cause |
|---|---|---|
| `heap-buffer-overflow` | Write/read past end of heap allocation | Off-by-one in array access, wrong size passed to `memcpy` |
| `stack-buffer-overflow` | Write past end of stack array | Fixed-size buffer too small, unbounded `strcpy` |
| `heap-use-after-free` | Access to deallocated heap memory | Object released but still referenced; missing `weak` or `unowned` |
| `stack-use-after-return` | Access to stack variable after function returned | Pointer to local variable escapes the function |
| `use-after-poison` | Access to ASan-poisoned memory | Similar to use-after-free but caught earlier |
| `alloc-dealloc-mismatch` | `delete` on `new[]` or `free` on `malloc` with wrong form | Mismatched allocation/deallocation functions |
| `memcpy-param-overlap` | Overlapping source and destination in `memcpy` | Should use `memmove` instead |
| `new-delete-type-mismatch` | `delete` on wrong type (base class without virtual destructor) | Polymorphic deletion without virtual destructor |

**Suppressions:** If ASan reports a false positive (common with certain system frameworks), create a suppression file:

```bash
# Create a suppression file
echo "leak:MyFramework" > /tmp/asan_suppressions.txt

# Use it
ASAN_OPTIONS="suppressions=/tmp/asan_suppressions.txt" /path/to/MyApp.app/Contents/MacOS/MyApp
```

### Interpreting TSan Reports

TSan output shows two threads that raced:

```
WARNING: ThreadSanitizer: data race (pid=12345)
  Write of size 8 at 0x7b0400003020 by thread T1:
    #0 -[MyModel setCount:] MyModel.m:42 (MyApp+0x100004ab2)
    #1 __24-[MyController update]_block_invoke MyController.m:88 (MyApp+0x100004c10)

  Previous read of size 8 at 0x7b0400003020 by thread T2:
    #0 -[MyModel count] MyModel.m:38 (MyApp+0x100004a90)
    #1 -[StatsView drawRect:] StatsView.m:156 (MyApp+0x100005e20)

  Location is heap block of size 64 at 0x7b0400003020 allocated by main thread:
    #0 malloc <null>
    #1 -[MyModel init] MyModel.m:20 (MyApp+0x100004900)

  Thread T1 (tid=12346, running) created by main thread at:
    #0 dispatch_async <null>
    #1 -[MyController startUpdates] MyController.m:70 (MyApp+0x100004b00)

  Thread T2 (tid=12347, running) created by main thread at:
    #0 pthread_create <null>
    #1 -[StatsView startRefreshTimer] StatsView.m:100 (MyApp+0x100005d00)
```

**Key fields:**
- **"Write of size N"** — the racing write and its size in bytes.
- **"Previous read"** — the racing read. "Previous" means it happened before the write in wall-clock time, but without synchronization, the order is not guaranteed.
- **"Location is heap block"** — where the racing variable was allocated. This helps you find the ivar or property.
- **Thread creation site** — where each racing thread was spawned. This helps you understand the concurrency model.

**Common TSan false positives:**
- Dispatch queue label comparison (use `dispatch_queue_set_specific` instead of comparing `dispatch_queue_get_label`).
- `retainCount` reading (TSan flags the unsynchronized read; don't use `retainCount` anyway).
- Singleton initialization without `dispatch_once` (use `dispatch_once` or `static let` in Swift).

### Combining Sanitizers in a CI Pipeline

Since ASan and TSan cannot run simultaneously, use separate build configurations:

```yaml
# Example CI matrix (GitHub Actions syntax)
strategy:
  matrix:
    sanitizer: [asan, tsan]
steps:
  - name: Build with ${{ matrix.sanitizer }}
    run: |
      if [ "${{ matrix.sanitizer }}" = "asan" ]; then
        export SANITIZER_FLAGS="ENABLE_ADDRESS_SANITIZER=YES"
      else
        export SANITIZER_FLAGS="ENABLE_THREAD_SANITIZER=YES"
      fi
      xcodebuild -scheme MyApp $SANITIZER_FLAGS build-for-testing

  - name: Test with ${{ matrix.sanitizer }}
    run: xcodebuild -scheme MyApp test-without-building
    env:
      ASAN_OPTIONS: detect_leaks=1:halt_on_error=0
      TSAN_OPTIONS: halt_on_error=0
```

`halt_on_error=0` makes the sanitizer report all issues rather than stopping at the first one. This is critical for CI — you want the full list of issues, not just the first.
