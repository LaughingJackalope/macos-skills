---
name: macos-ipc-xpc
description: "Inter-process communication on macOS — XPC services (NSXPCConnection, NSXPCInterface, @objc protocol), privilege separation with XPC, App Groups (com.apple.security.application-groups), DistributedNotificationCenter, CFMessagePort. Use when implementing XPC services, helper tools, or inter-process communication on macOS."
sources: https://developer.apple.com/documentation/xpc, https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingXPCServices.html, https://developer.apple.com/documentation/xpc/xpcsession, https://developer.apple.com/documentation/technotes/tn3113-testing-xpc-code-with-an-anonymous-listener, https://developer.apple.com/documentation/foundation/distributednotificationcenter
researched: 2026-06-06 (enhanced 2026-06-06)
platform: macOS 26+
---

# macOS IPC & XPC

Inter-process communication patterns for macOS 26+. Covers XPC service lifecycle, `NSXPCConnection` protocol negotiation, privileged helper tools via SMJobBless, App Groups for shared state, and `DistributedNotificationCenter` for cross-process signals. This skill focuses on Mach-based XPC. For higher-level coordinating mechanisms, see `macos-system-integration`.

See [references/xpc-patterns.md](references/xpc-patterns.md) for complete code patterns: full `NSXPCConnection` + `NSXPCInterface` setup with `@objc` protocol, SMJobBless helper tool, App Group shared container, distributed notifications, and a Swift async wrapper.

## Contents

- [XPC Service Architecture](#xpc-service-architecture)
- [NSXPCConnection](#nsxpcconnection)
- [NSXPCInterface](#nsxpcinterface)
- [Privilege Separation](#privilege-separation)
- [App Groups](#app-groups)
- [DistributedNotificationCenter](#distributednotificationcenter)
- [XPC Error Handling](#xpc-error-handling)
- [Swift Concurrency Bridge](#swift-concurrency-bridge)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## XPC Service Architecture

XPC on macOS is built on Mach ports and managed by launchd. An XPC service is a separate binary (`.xpc` bundle) or an embedded `.app` bundle that the system launches on demand, isolated from the host process.

**Lifecycle:**

1. The host app creates an `NSXPCConnection` to a service by name.
2. launchd checks the service's `launchd.plist` for matching job specifications.
3. If the service is not running, launchd spawns it.
4. The connection is established via a Mach port pair — one end in the host, one in the service.
5. When all connections close and no work is pending, launchd may terminate the service.

**Audit session requirement:** Every XPC connection carries an audit token (`au_asid`, pid, uid, gid) from the connecting process. The service can inspect the audit token via `NSXPCConnection.auditSessionIdentifier` or the `audit_token_t` from `xpc_connection_get_audit_token` to verify the caller's identity. Always validate the caller before performing sensitive operations — XPC provides isolation but not automatic trust. A compromised host process can still send arbitrary messages.

**Protocol negotiation:** Both sides define an `@objc` protocol. The host sets this as the `remoteObjectInterface` on the connection. The service sets it as the `exportedInterface` on its listener. The runtime enforces that only methods declared in the protocol can be called across the boundary.

## NSXPCConnection

The primary object for establishing and managing an XPC connection.

**Creating a connection:**

```swift
// Connect to an XPC service embedded in the app bundle
let connection = NSXPCConnection(
    serviceName: "com.example.MyApp.MyService"
)
connection.remoteObjectInterface = NSXPCInterface(with: MyServiceProtocol.self)
connection.resume()  // Must call resume() — the connection starts suspended
```

For Mach services (launched by launchd from a `launchd.plist`):

```swift
let connection = NSXPCConnection(machServiceName: "com.example.MyApp.MyService", options: [])
```

**Getting a remote proxy:**

```swift
guard let proxy = connection.remoteObjectProxyWithErrorHandler { error in
    NSLog("XPC connection error: \(error)")
} as? MyServiceProtocol else {
    return
}
```

**Interruption and invalidation:**

```swift
connection.interruptionHandler = {
    // Service was killed (e.g., memory pressure, crash).
    // Can attempt to reconnect.
    NSLog("XPC connection interrupted")
}

connection.invalidationHandler = {
    // Connection is permanently broken.
    // Clean up references; cannot reconnect with the same NSXPCConnection.
    NSLog("XPC connection invalidated")
}
```

**Synchronous vs. async:** Methods called on the proxy are synchronous from the caller's perspective but execute asynchronously in the service. Use a completion-handler pattern inside the protocol for results:

```swift
@objc protocol MyServiceProtocol: NSObjectProtocol {
    func fetchData(for query: String, withReply reply: @escaping (Data?, Error?) -> Void)
}
```

The `withReply` parameter (named `reply:` by convention) is required for any method that returns data. XPC transparently handles the reply block across the Mach channel.

**Service side — NSXPCListener:**

```swift
class ServiceDelegate: NSObject, NSXPCListenerDelegate {
    func listener(_ listener: NSXPCListener, shouldAcceptNewConnection newConnection: NSXPCConnection) -> Bool {
        newConnection.exportedInterface = NSXPCInterface(with: MyServiceProtocol.self)
        newConnection.exportedObject = MyServiceImplementation()
        newConnection.resume()
        return true
    }
}

let listener = NSXPCListener.service()
listener.delegate = ServiceDelegate()
listener.resume()  // Blocks and runs the listener loop
```

Return `false` from `shouldAcceptNewConnection:` to reject a caller. Use this to enforce that only your team's code connects.

## NSXPCInterface

`NSXPCInterface` configures which types are allowed to cross the XPC boundary. It is defined by pointing at an `@objc` protocol, plus explicit class whitelisting for complex types.

**@objc protocol definition:**

```swift
@objc protocol FileServiceProtocol: NSObjectProtocol {
    func readFile(at url: URL, withReply reply: @escaping (Data?, Error?) -> Void)
    func listDirectory(at url: URL, withReply reply: @escaping ([URL]?, Error?) -> Void)
}
```

The protocol must inherit `NSObjectProtocol`. All method parameters and return values (via `reply:` blocks) must be representable in the Objective-C runtime — Swift value types like enums without `@objc` raw values, structs, generics, and tuples are not allowed.

**Allowed classes with NSSet classes:**

When passing collections or specific class types, you must explicitly whitelist them:

```swift
let iface = NSXPCInterface(with: FileServiceProtocol.self)
let allowedClasses: Set<AnyHashable> = [
    NSArray.self,
    NSURL.self,
    NSString.self,
    NSData.self,
    NSError.self
]
// For a method parameter of type NSSet
iface.setClasses(allowedClasses,
                 for: #selector(FileServiceProtocol.processItems(_:withReply:)),
                 argumentIndex: 0,
                 ofReply: false)
```

Forgetting to call `setClasses(_:for:argumentIndex:ofReply:)` results in a runtime exception when the non-whitelisted object is encoded. The exception message names the class and selector, making it straightforward to diagnose.

**Progress reporting via NSProgress:**

For long-running operations, pass an `NSProgress` proxy:

```swift
@objc protocol ExportServiceProtocol: NSObjectProtocol {
    func export(source: URL, progress: NSProgress, withReply reply: @escaping (URL?, Error?) -> Void)
}
```

NSProgress automatically bridges across XPC — the host can observe `fractionCompleted` and call `cancel()`. KVO notifications propagate transparently.

## Privilege Separation

The primary reason to use XPC on macOS is privilege separation: a sandboxed app delegates privileged operations to an unsandboxed (or differently-signed) helper.

**Pattern: Privileged helper tool via SMJobBless**

```
MyApp.app (sandboxed)
  → connects via NSXPCConnection
  → to HelperTool (runs as root, installed via SMJobBless)
```

The helper tool is a separate `launchd` job (via `SMJobBless` or `SMLoginItemSetEnabled`). The XPC channel carries a audit token the helper can inspect to verify the connecting app's identity.

```swift
// On the helper (service) side
func listener(_ listener: NSXPCListener, shouldAcceptNewConnection conn: NSXPCConnection) -> Bool {
    // Extract the audit token
    let token = conn.auditSessionIdentifier
    // Or use the XPC C API for full audit_token_t
    // Compare against a known-asid/pid if needed

    // Simpler: check the code signature of the connecting client
    // via the SecCode API and the audit token
    return codeMatchesExpectedCaller(conn)
}
```

**Security considerations:**
- The helper's `SMAuthorizedClients` (in its Info.plist) limits which calling bundles can install it.
- Validate the caller's audit token on every connection — do not trust the Mach port alone.
- Keep the helper minimal. The larger the helper's attack surface, the greater the privilege escalation risk.
- The helper should expose a narrow, well-audited protocol — not a general-purpose shell.

## App Groups

App Groups allow multiple apps (and XPC services) from the same development team to share data.

**Entitlement:** Both the app and the service need the `com.apple.security.application-groups` entitlement with a shared group identifier:

```xml
<key>com.apple.security.application-groups</key>
<array>
    <string>TEAMID.com.example.myapp</string>
</array>
```

The group identifier must start with the team ID followed by a period.

**Shared NSUserDefaults:**

```swift
let suiteName = "TEAMID.com.example.myapp"
let sharedDefaults = UserDefaults(suiteName: suiteName)!
sharedDefaults.set("value", forKey: "sharedKey")

// On the service side, use the same suite name
let value = UserDefaults(suiteName: suiteName)?.string(forKey: "sharedKey")
```

**Shared file container:**

```swift
let containerURL = FileManager.default
    .containerURL(forSecurityApplicationGroupIdentifier: "TEAMID.com.example.myapp")!

// Both the app and the XPC service can read/write this directory
let sharedFile = containerURL.appendingPathComponent("shared-data.json")
```

**Security:** Both the app and service must share the same App Group entitlement. If the service runs as root (via SMJobBless), the shared container's POSIX permissions are still controlled by the owning team — ensure files are created with restrictive permissions (`0o600`) if sensitive data is stored.

**iOS-style App Group IDs (Feb 2025+):** Apple now supports iOS-style App Group identifiers (e.g., `group.com.example.myapp`) on macOS provisioning profiles. The Developer Portal auto-adds `TEAMID.*` to authorized App Group IDs in Mac provisioning profiles. Validate your group identifier with `containerURL(forSecurityApplicationGroupIdentifier:)` before using it — it returns `nil` for invalid IDs without logging an error. On macOS 15+, container access without user intervention requires distribution via Mac App Store/TestFlight or a group ID starting with your Team ID.

## DistributedNotificationCenter

`DistributedNotificationCenter` (reached via `CFNotificationCenterGetDarwinNotifyCenter`) provides lightweight cross-process signals. Unlike `NSNotificationCenter`, notifications posted here go through the Darwin notification system — they cross process boundaries without explicit connection setup.

**Posting:**

```swift
import Foundation

func postDarwinNotification(name: String) {
    let center = CFNotificationCenterGetDarwinNotifyCenter()
    let notificationName = name as CFString
    CFNotificationCenterPostNotification(center, CFNotificationName(notificationName), nil, nil, true)
}
```

The `deliverImmediately: true` parameter (last arg) delivers the notification synchronously. Use `false` for coalesced delivery.

**Receiving:**

```swift
import Foundation

func observeDarwinNotification(name: String, callback: @escaping () -> Void) {
    let center = CFNotificationCenterGetDarwinNotifyCenter()
    let observer = UnsafeRawPointer(Unmanaged.passUnretained(self).toOpaque())

    CFNotificationCenterAddObserver(center, observer,
        { center, observer, name, object, userInfo in
            guard let observer else { return }
            let mySelf = Unmanaged<YourClass>.fromOpaque(observer).takeUnretainedValue()
            mySelf.handleNotification()
        },
        name as CFString, nil, .deliverImmediately)
}

func removeDarwinObserver() {
    let center = CFNotificationCenterGetDarwinNotifyCenter()
    let observer = UnsafeRawPointer(Unmanaged.passUnretained(self).toOpaque())
    CFNotificationCenterRemoveObserver(center, observer, nil, nil)
}
```

**Foundation-level `NSDistributedNotificationCenter` (login.10.15+):**

The higher-level `NSDistributedNotificationCenter.default` API provides the same cross-process signal mechanism with ObjC/Foundation integration. Key behavioral caveats on modern macOS:

- **Execution-context boundary:** GUI apps and launchd daemons run in separate execution contexts. `NSDistributedNotificationCenter` notifications do not cross context boundaries — a notification posted from a GUI app will not reach a root daemon, and vice versa. For system-wide notifications across contexts, use Darwin notifications (`CFNotificationCenterGetDarwinNotifyCenter`).
- **Suspension behavior:** Distributed notifications are not delivered while your app is suspended. Use `addObserver(_:selector:name:object:suspensionBehavior:)` with `.deliverImmediately` to receive notifications while backgrounded, or use Darwin notifications for always-delivery.
- **`nil` names no longer supported:** As of Catalina (10.15), using `nil` for the notification name when registering an observer is a privileged operation. Always register for specific notification names.
- **`NSDistributedNotificationPostToAllSessions`:** Required if your notification must reach observers across all user sessions (fast user switching, screen sharing).

**Compared to NSNotificationCenter:**

| Property | `NSNotificationCenter.default` | `DistributedNotificationCenter` |
|---|---|---|
| Scope | Same process only | Cross-process |
| UserInfo | Can carry a `Dictionary` | No `userInfo` payload — signal only |
| Delivery | Synchronous by default | Synchronous (`true`) or coalesced (`false`) |
| Reliability | Always delivered if observer is alive | Best-effort; not queued if no observers |
| Execution context | N/A | Boundary-limited (GUI vs. daemon) |

**Payload limitation:** Darwin notifications carry no `userInfo` object. If data must be transferred, pair the notification with shared memory, a file in a shared container, or a `NSUserDefaults` change.
## XPC Error Handling

Every XPC message has a reply. If the service throws an exception or errors, the reply contains an `NSError` object. XPC defines special error types for connection failures. The XPC C API defines three lifecycle error constants delivered via event handlers:

| Error Constant | Meaning |
|---|---|
| `XPC_ERROR_CONNECTION_INTERRUPTED` | Remote service exited (crash, memory pressure). Reconnect with a new `NSXPCConnection`. |
| `XPC_ERROR_CONNECTION_INVALID` | Connection is permanently broken. Cannot reuse this `NSXPCConnection`. |
| `XPC_ERROR_TERMINATION_IMMINENT` | XPC runtime determined the program must exit. Finish outstanding transactions; do not start new ones. |

Use `xpc_connection_copy_invalidation_reason()` on the C connection to get a human-readable invalidation reason string for logging. At the `NSXPCConnection` level, invalidation reasons surface in the `invalidationHandler` callback.

```swift
let proxy = connection.remoteObjectProxyWithErrorHandler { error in
    if let nsError = error as NSError? {
        switch nsError.code {
        case NSXPCConnectionInterrupted:
            // Connection was interrupted — service may have crashed
            // Reconnect by creating a new NSXPCConnection
            self.reconnect()
        case NSXPCConnectionInvalid:
            // Connection is permanently invalid
            // Cannot reconnect — must handle cleanup
            self.handlePermanentFailure()
        default:
            // Service-returned error
            NSLog("Service error: \(nsError)")
        }
    }
} as? MyServiceProtocol
```

**Reconnection strategy:**

```swift
var connection: NSXPCConnection?

func ensureConnected() -> MyServiceProtocol? {
    if let conn = connection, !conn.isInvalid {
        return conn.remoteObjectProxy as? MyServiceProtocol
    }
    connection = NSXPCConnection(serviceName: "com.example.MyApp.MyService")
    connection?.remoteObjectInterface = NSXPCInterface(with: MyServiceProtocol.self)
    connection?.resume()
    return connection?.remoteObjectProxy as? MyServiceProtocol
}

extension NSXPCConnection {
    var isInvalid: Bool {
        // NSXPCConnection does not expose an invalidation flag directly.
        // Track it via the invalidationHandler.
        // Alternatively, check whether calling remoteObjectProxy returns nil.
        return false // Placeholder — track state manually
    }
}
```

Track invalidation state manually via a flag set in `invalidationHandler`.

**Debugging with Console.app:**

1. Open Console.app and filter by your service's subsystem: `subsystem == "com.example.MyApp.MyService"`.
2. Include info-level messages: toggle "Info" in the toolbar.
3. Add `os_log` points in both the client and service:

```swift
import os.log

let xpcLog = OSLog(subsystem: "com.example.MyApp.MyService", category: "xpc")

os_log("Received request: %{public}@", log: xpcLog, type: .info, query)
```

## Swift Concurrency Bridge

`NSXPCConnection` is callback-based. Wrap it in `withCheckedContinuation` for async/await:

```swift
extension NSXPCConnection {
    func call<ServiceProtocol, Result>(
        to protocolType: ServiceProtocol.Type,
        _ invocation: @escaping (ServiceProtocol, @escaping (Result?, Error?) -> Void) -> Void
    ) async throws -> Result {
        try await withCheckedThrowingContinuation { continuation in
            guard let proxy = remoteObjectProxyWithErrorHandler { error in
                continuation.resume(throwing: error)
            } as? ServiceProtocol else {
                continuation.resume(throwing: XPCError.proxyTypeMismatch)
                return
            }
            invocation(proxy) { result, error in
                if let error = error {
                    continuation.resume(throwing: error)
                } else if let result = result {
                    continuation.resume(returning: result)
                } else {
                    continuation.resume(throwing: XPCError.nilResult)
                }
            }
        }
    }
}

enum XPCError: Error {
    case proxyTypeMismatch
    case nilResult
}
```

**Actor-isolated service wrapper:**

```swift
actor XpcService {
    private var connection: NSXPCConnection?

    func getData(for query: String) throws -> Data {
        if connection == nil {
            connection = NSXPCConnection(serviceName: "com.example.MyApp.MyService")
            connection?.remoteObjectInterface = NSXPCInterface(with: MyServiceProtocol.self)
            connection?.resume()
        }
        guard let conn = connection else { throw XPCError.connectionFailed }
        return try await conn.call(to: MyServiceProtocol.self) { proxy, reply in
            proxy.fetchData(for: query, withReply: reply)
        }
    }

    func disconnect() {
        connection?.invalidate()
        connection = nil
    }
}
```

The actor serializes access to the connection, preventing race conditions from concurrent XPC calls on the same `NSXPCConnection` instance.

### XPCSession (Modern Alternative)

`XPCSession` is the modern session-based XPC API that provides first-class Swift concurrency support. It replaces the callback-based `NSXPCConnection` pattern with `async/await` and `Codable` message passing:

```swift
import XPC

// Connect to an embedded XPC service
let session = XPCSession(serviceName: "com.example.MyApp.MyService")

// Send a Codable message and await a Codable response
let result = try await session.send(MyRequest(query: "data"))
// result is decoded as MyResponse
```

Key differences from `NSXPCConnection`:

| Feature | `NSXPCConnection` | `XPCSession` |
|---|---|---|
| Protocol definition | `@objc` protocol + `NSXPCInterface` | `Codable` structs |
| Type safety | Runtime (ObjC bridging) | Compile-time (Swift generics) |
| Concurrency | Callbacks + manual `withCheckedContinuation` | Native `async/await` |
| Lifecycle | `resume()` / `invalidate()` | `activate()` / `cancel()` |
| Error handling | `NSError` in reply blocks | Typed Swift errors |
| Cancellation | Manual invalidation | Structured concurrency cancellation |

`XPCSession` supports three message modes:
- **Decodable replies** — `send(_:)` returns a decoded response
- **Dictionary messages** — `send(_:)` with `[String: Any]` for dynamic payloads
- **Fire-and-forget** — `send(_:)` with no reply handler

Always call `session.cancel()` when done to avoid resource leaks. Cancellation discards unsent messages and triggers the optional `cancellationHandler`.

## Constraints & Gotchas

- **XPC message size limit:** Individual XPC messages have no fixed byte limit, but large payloads (> ~1 MB) serialize through `NSCoding` and cross Mach boundaries — this is slow. For large data, pass a file URL in a shared container and let the other process read it directly.
- **`serviceName` lifetime:** An XPC service launched by `serviceName:` lives as long as any connection references it. If the host does not invalidate the connection, the service stays resident in memory.
- **Launchd job debugging:** Use `launchctl print gui/$UID/com.example.MyApp.MyService` to inspect a running XPC service job. `launchctl kickstart -k gui/$UID/...` to force-restart it after code changes.
- **App Group identifier format:** Must be `<TEAMID>.<identifier>`. The team ID is the 10-character prefix from the Developer Portal. Using a lowercase bundle ID prefix (instead of the numeric team ID) will silently fail — the shared container path returns `nil`.
- **NSXPCConnection is not thread-safe:** Call `resume()` on the main thread. The proxy methods can be called from any thread, but the connection lifecycle methods (`resume`, `invalidate`) should be serialized.
- **No Swift-only types across XPC:** All types must be ObjC-compatible (`@objc` classes, `@objc` protocols, `NSObject` subclasses, `NSString`, `NSArray`, `NSData`, etc.). Swift structs, enums without Int raw values, and generics are rejected at runtime.
- **Darwin notifications are not entitlements-protected:** Any process can observe or post Darwin notifications. Do not send sensitive data or make security decisions based solely on these notifications.
- **`NSProgress` proxying:** `NSProgress` bridging is automatic for parameters. For return values via `reply:` blocks, the progress must be explicitly created on the service side and passed back; it does not auto-proxy through the reply.

## Common Mistakes

- **Forgetting `resume()`:** The connection starts suspended. Calling any proxy method before `resume()` silently does nothing. Always call `resume()` immediately after setting up the interface.
- **Using Swift-only types in the protocol:** Defining a protocol method with a Swift enum or struct parameter compiles fine but crashes at runtime with `[aSelector]: unrecognized class for argument`. Audit all protocol methods for ObjC compatibility.
- **Not setting `exportedInterface` on the listener:** The service will accept connections but all method calls will fail with an `NSInvalidArgumentException`.
- **Assuming Darwin notifications carry data:** They carry nothing beyond the name. Use a shared store if data synchronization is needed.
- **Ignoring `invalidationHandler`:** If the service crashes, the host keeps a stale connection reference. Without the invalidation handler, subsequent proxy calls produce no error and silently fail.
- **App Group typo:** A mismatched group identifier (wrong team ID uppercase vs. lowercase) produces no error — `FileManager.containerURL(forSecurityApplicationGroupIdentifier:)` simply returns nil.
- **Blocking the listener's `shouldAcceptNewConnection`:** This callback runs on the main thread. Do not perform synchronous I/O or long computation here — it blocks subsequent connections.
- **Strong reference cycles with reply blocks:** A reply block that captures `self` while `self` captures the connection creates a cycle. Use `[weak self]` inside reply blocks.
- **XPC_ERROR_TERMINATION_IMMINENT not handled:** This error (delivered via the XPC C API event handler) signals the runtime is shutting down. If ignored, outstanding transactions silently fail. Handle it the same as `XPC_ERROR_CONNECTION_INTERRUPTED` for cleanup.
## Review Checklist

- [ ] `NSXPCConnection.resume()` is called immediately after setup
- [ ] Protocol inherits `NSObjectProtocol` and all param/return types are ObjC-compatible
- [ ] `setClasses(_:for:argumentIndex:ofReply:)` is called for any collection or custom class parameters
- [ ] `exportedInterface` is set on the service listener delegate
- [ ] `interruptionHandler` and `invalidationHandler` are set on the connection
- [ ] Service validates the caller's audit token or code signature in `shouldAcceptNewConnection:`
- [ ] App Group identifier uses the numeric team ID prefix (not bundle ID)
- [ ] Shared container files have restrictive POSIX permissions when sensitive
- [ ] Darwin notifications are used for signals only, never for data transfer
- [ ] Reply blocks use `[weak self]` to avoid retain cycles
- [ ] `os_log` is added at service entry/exit points for Console.app debugging

## References

- [references/xpc-patterns.md](references/xpc-patterns.md) — Full NSXPCConnection + NSXPCInterface code, SMJobBless helper tool pattern, App Group container, distributed notifications, Swift async wrapper
- [Apple: Creating XPC Services](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingXPCServices.html)
- [Apple: NSXPCConnection documentation](https://developer.apple.com/documentation/foundation/nsxpcconnection)
- [Apple: XPC Services](https://developer.apple.com/documentation/xpc)
- [Apple: XPCSession documentation](https://developer.apple.com/documentation/xpc/xpcsession)
- [Apple: TN3113 — Testing XPC code with an anonymous listener](https://developer.apple.com/documentation/technotes/tn3113-testing-xpc-code-with-an-anonymous-listener)
- [Apple: XPC connections (error codes)](https://developer.apple.com/documentation/xpc/xpc-connections)
- [Apple: DistributedNotificationCenter](https://developer.apple.com/documentation/foundation/distributednotificationcenter)