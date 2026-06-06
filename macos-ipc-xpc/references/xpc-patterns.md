# XPC Code Patterns

Complete working patterns for macOS IPC: full `NSXPCConnection` + `NSXPCInterface` setup, SMJobBless helper tool, App Group shared container, distributed notifications, and a Swift async XPC wrapper.

## Full NSXPCConnection + NSXPCInterface Setup

The standard pattern: an `@objc` protocol, a host-side connection, and a service-side listener.

### Protocol Definition

```swift
import Foundation

@objc protocol CalculatorServiceProtocol: NSObjectProtocol {
    func add(_ a: Int, to b: Int, withReply reply: @escaping (Int) -> Void)
    func fetchConfiguration(withReply reply: @escaping (NSDictionary?) -> Void)
    func process(files: [URL], withReply reply: @escaping (NSError?) -> Void)
}
```

Key rules:
- Must inherit `NSObjectProtocol`.
- All parameters must be ObjC-compatible (`Int` bridges to `NSNumber` automatically in ObjC signatures, but be explicit if using `NSInteger` or `NSNumber`)`.
- `[URL]` bridges to `NSArray<NSURL *> *` — you must whitelist `NSURL` and `NSArray` via `setClasses`.
- `NSDictionary` bridges from Swift `Dictionary<String, Any>` but the dictionary contents must also be ObjC-compatible.

### Host Side

```swift
final class CalculatorClient {
    private var connection: NSXPCConnection?

    func connect() {
        connection = NSXPCConnection(serviceName: "com.example.MyApp.CalculatorService")
        guard let conn = connection else { return }

        let iface = NSXPCInterface(with: CalculatorServiceProtocol.self)
        // Whitelist NSArray and NSURL for the `files:` parameter of process(files:withReply:)
        let allowedClasses = NSSet(array: [NSArray.self, NSURL.self]) as Set
        iface.setClasses(allowedClasses,
                         for: #selector(CalculatorServiceProtocol.process(files:withReply:)),
                         argumentIndex: 0,
                         ofReply: false)

        conn.remoteObjectInterface = iface
        conn.interruptionHandler = { [weak self] in
            NSLog("Calculator service interrupted — will reconnect on next call")
            self?.connection = nil
        }
        conn.invalidationHandler = { [weak self] in
            NSLog("Calculator service connection invalidated")
            self?.connection = nil
        }
        conn.resume()
    }

    func add(_ a: Int, to b: Int) {
        guard let conn = connection else {
            connect()
            return
        }
        guard let proxy = conn.remoteObjectProxyWithErrorHandler { err in
            NSLog("XPC error: \(err)")
        } as? CalculatorServiceProtocol else {
            return
        }
        proxy.add(a, to: b) { result in
            DispatchQueue.main.async {
                print("Result: \(result)")
            }
        }
    }

    func disconnect() {
        connection?.invalidate()
        connection = nil
    }
}
```

### Service Side

```swift
import Foundation

final class CalculatorService: NSObject, CalculatorServiceProtocol {
    func add(_ a: Int, to b: Int, withReply reply: @escaping (Int) -> Void) {
        reply(a + b)
    }

    func fetchConfiguration(withReply reply: @escaping (NSDictionary?) -> Void) {
        let config: NSDictionary = [
            "version": "1.0" as NSString,
            "debug": NSNumber(value: false)
        ]
        reply(config)
    }

    func process(files: [URL], withReply reply: @escaping (NSError?) -> Void) {
        DispatchQueue.global(qos: .userInitiated).async {
            for url in files {
                // Process each file
                guard FileManager.default.fileExists(atPath: url.path) else {
                    let error = NSError(domain: "com.example.MyApp",
                                        code: 404,
                                        userInfo: [NSLocalizedDescriptionKey: "File not found: \(url.path)"])
                    reply(error)
                    return
                }
            }
            reply(nil) // Success
        }
    }
}

// main.swift for the XPC service:
final class ServiceDelegate: NSObject, NSXPCListenerDelegate {
    func listener(_ listener: NSXPCListener, shouldAcceptNewConnection conn: NSXPCConnection) -> Bool {
        let iface = NSXPCInterface(with: CalculatorServiceProtocol.self)
        let allowedClasses = NSSet(array: [NSArray.self, NSURL.self]) as Set
        iface.setClasses(allowedClasses,
                         for: #selector(CalculatorServiceProtocol.process(files:withReply:)),
                         argumentIndex: 0,
                         ofReply: false)

        conn.exportedInterface = iface
        conn.exportedObject = CalculatorService()
        conn.resume()
        return true
    }
}

let delegate = ServiceDelegate()
let listener = NSXPCListener.service()
listener.delegate = delegate
listener.resume() // Starts the run loop; never returns
```

## SMJobBless Privileged Helper Tool Pattern

A privileged helper tool runs as root. The host app installs it via `SMJobBless` and communicates over XPC.

### Helper's Info.plist

```xml
<!-- Helper Info.plist -->
<key>Label</key>
<string>com.example.MyApp.PrivilegedHelper</string>
<key>MachServices</key>
<dict>
    <key>com.example.MyApp.PrivilegedHelper</key>
    <true/>
</dict>
<key>SMAuthorizedClients</key>
<array>
    <string>TEAMID.com.example.MyApp</string>
</array>
```

### Host Side: Install and Connect

```swift
import ServiceManagement
importFoundation

final class PrivilegedHelperManager {
    private let helperID = "com.example.MyApp.PrivilegedHelper"
    private var connection: NSXPCConnection?

    @discardableResult
    func installHelper() -> Bool {
        var authRef: AuthorizationRef?
        var authItem = AuthorizationItem(
            name: kSMRightBlessPrivilegedHelper,
            valueLength: 0,
            value: nil,
            flags: 0
        )
        var authRights = AuthorizationRights(count: 1, items: &authItem)
        let flags: AuthorizationFlags = [.interactionAllowed, .extendRights, .preAuthorize]

        let status = AuthorizationCreate(&authRights, nil, flags, &authRef)
        guard status == errAuthorizationSuccess, let auth = authRef else {
            return false
        }
        defer { AuthorizationFree(auth, []) }

        var error: Unmanaged<CFError>?
        let success = SMJobBless(kSMDomainSystemLaunchd, helperID as CFString, auth, &error)
        if !success, let error = error?.takeRetainedValue() {
            NSLog("SMJobBless failed: \(error)")
        }
        return success
    }

    func connect() -> PrivilegedHelperProtocol? {
        connection = NSXPCConnection(machServiceName: helperID, options: [])
        connection?.remoteObjectInterface = NSXPCInterface(with: PrivilegedHelperProtocol.self)
        connection?.invalidationHandler = { [weak self] in
            self?.connection = nil
        }
        connection?.resume()

        guard let proxy = connection?.remoteObjectProxyWithErrorHandler({ err in
            NSLog("Helper connection error: \(err)")
        }) as? PrivilegedHelperProtocol else {
            return nil
        }
        return proxy
    }
}
```

### Validate Caller on the Helper Side

```swift
import Security

func listener(_ listener: NSXPCListener, shouldAcceptNewConnection conn: NSXPCConnection) -> Bool {
    // Get the audit token
    var auditToken = conn.auditSessionIdentifier

    // Use SecCode to verify the connecting client
    var clientCode: SecCode?
    let status = SecCodeCopyGuestWithAttributes(
        nil,
        [kSecGuestAttributeAudit: auditToken] as CFDictionary,
        [],
        &clientCode
    )
    guard status == errSecSuccess, let code = clientCode else {
        return false
    }

    // Check that the client is signed with our team
    var requirement: SecRequirement?
    let reqStr = "anchor apple generic and certificate leaf[field.1.2.840.113635.100.6.1.9] exists and certificate 1[field.1.2.840.113635.100.6.2.6] exists and certificate leaf[subject.OU] = TEAMID"
    SecRequirementCreateWithString(reqStr as CFString, [], &requirement)

    if let req = requirement {
        let valid = SecCodeCheckValidity(code, [], req) == errSecSuccess
        if !valid { return false }
    }

    conn.exportedInterface = NSXPCInterface(with: PrivilegedHelperProtocol.self)
    conn.exportedObject = PrivilegedHelperService()
    conn.resume()
    return true
}
```

## App Group Shared Container with NSUserDefaults

Both an app and its XPC service share state via App Group containers.

### Entitlements

Both the app and the service target need:

```xml
<key>com.apple.security.application-groups</key>
<array>
    <string>TEAMID.com.example.myapp</string>
</array>
```

### Shared Defaults

```swift
import Foundation

struct AppGroupStore {
    static let suiteName = "TEAMID.com.example.myapp"

    static var shared: UserDefaults {
        UserDefaults(suiteName: suiteName)!
    }

    // App side
    func saveState(_ value: String, forKey key: String) {
        Self.shared.set(value, forKey: key)
        Self.shared.synchronize() // Force immediate write for cross-process visibility
    }

    // Service side
    func loadState(forKey key: String) -> String? {
        Self.shared.string(forKey: key)
    }
}
```

### Shared File Container

```swift
import Foundation

struct SharedContainer {
    static let identifier = "TEAMID.com.example.myapp"

    static var containerURL: URL? {
        FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: identifier)
    }

    static func write<T: Encodable>(_ value: T, to filename: String) throws {
        guard let url = containerURL?.appendingPathComponent(filename) else {
            throw SharedContainerError.containerNotFound
        }
        let data = try JSONEncoder().encode(value)
        try data.write(to: url, options: .atomic)
        // Restrict permissions for sensitive data
        try FileManager.default.setAttributes([.posixPermissions: 0o600],
                                              ofItemAtPath: url.path)
    }

    static func read<T: Decodable>(_ type: T.Type, from filename: String) throws -> T {
        guard let url = containerURL?.appendingPathComponent(filename) else {
            throw SharedContainerError.containerNotFound
        }
        let data = try Data(contentsOf: url)
        return try JSONDecoder().decode(type, from: data)
    }
}

enum SharedContainerError: Error {
    case containerNotFound
}
```

## DistributedNotificationCenter Cross-Process Notifications

Lightweight signaling without an XPC connection.

### Notification Name Convention

```swift
import Foundation

extension static let configDidChange = Notification.Name("com.example.myapp.config-change")
```

### Poster

```swift
import Foundation

func postConfigChangedNotification() {
    let center = CFNotificationCenterGetDarwinNotifyCenter()
    CFNotificationCenterPostNotification(
        center,
        CFNotificationName(AppNotificationNames.configDidChange.rawValue as CFString),
        nil,  // No object
        nil,  // No userInfo — Darwin notifications carry no payload
        true  // Deliver immediately
    )
}
```

### Observer

```swift
import Foundation

final class DarwinNotificationObserver {
    private let callback: () -> Void

    init(name: String, callback: @escaping () -> Void) {
        self.callback = callback
        let center = CFNotificationCenterGetDarwinNotifyCenter()
        let observer = UnsafeRawPointer(Unmanaged.passUnretained(self).toOpaque())

        CFNotificationCenterAddObserver(
            center, observer,
            { _, observerRef, _, _, _ in
                guard let ref = observerRef else { return }
                let instance = Unmanaged<DarwinNotificationObserver>
                    .fromOpaque(ref)
                    .takeUnretainedValue()
                instance.callback()
            },
            name as CFString,
            nil,
            .deliverImmediately
        )
    }

    deinit {
        let center = CFNotificationCenterGetDarwinNotifyCenter()
        let observer = UnsafeRawPointer(Unmanaged.passUnretained(self).toOpaque())
        CFNotificationCenterRemoveEveryObserver(center, observer)
    }
}

// Usage:
let observer = DarwinNotificationObserver(name: "com.example.myapp.config-change") {
    // Reload config from shared container
    DispatchQueue.main.async {
        self.reloadConfiguration()
    }
}
```

**Combined pattern — notification + shared container:**

```swift
// App writes new config
func updateConfiguration(_ config: AppConfig) {
    try? SharedContainer.write(config, to: "config.json")
    postConfigChangedNotification() // Signal other processes
}

// Service observes and reloads
let observer = DarwinNotificationObserver(name: "com.example.myapp.config-change") {
    guard let config = try? SharedContainer.read(AppConfig.self, from: "config.json") else {
        return
    }
    applyConfiguration(config)
}
```

## Swift Async Wrapper for XPC

Bridge `NSXPCConnection` to Swift concurrency with `withCheckedContinuation`.

### Core Wrapper

```swift
import Foundation

enum XPCErrors: Error {
    case proxyTypeMismatch
    case nilResult
    case connectionNotReady
}

/// Wraps a two-way XPC call in an async/await context.
func callXPC<T>(
    on connection: NSXPCConnection,
    protocol proto: T.Type,
    _ invocation: @escaping (T, @escaping (Any?, Error?) -> Void) -> Void
) async throws -> Any {
    try await withCheckedThrowingContinuation { continuation in
        guard connection.remoteObjectInterface != nil else {
            continuation.resume(throwing: XPCErrors.connectionNotReady)
            return
        }
        let errorHandler: (Error) -> Void = { error in
            continuation.resume(throwing: error)
        }
        guard let proxy = connection.remoteObjectProxyWithErrorHandler(errorHandler) as? T else {
            continuation.resume(throwing: XPCErrors.proxyTypeMismatch)
            return
        }
        invocation(proxy) { result, error in
            if let error {
                continuation.resume(throwing: error)
            } else if let result {
                continuation.resume(returning: result)
            } else {
                continuation.resume(throwing: XPCErrors.nilResult)
            }
        }
    }
}
```

### Typed Extension

```swift
extension NSXPCConnection {
    /// Type-safe XPC call with automatic casting.
    func xcall<Proto, Result>(
        _ proto: Proto.Type = Proto.self,
        _ invocation: @escaping (Proto, @escaping (Result?, Error?) -> Void) -> Void
    ) async throws -> Result {
        let raw = try await callXPC(on: self, protocol: proto, invocation)
        guard let result = raw as? Result else {
            throw XPCErrors.proxyTypeMismatch
        }
        return result
    }
}

// Usage:
let sum: Int = try await connection.xcall(CalculatorServiceProtocol.self) { proxy, reply in
    proxy.add(2, to: 3, withReply: reply)
}
print("Sum: \(sum)") // Sum: 5
```

### Actor-Isolated Service Handle

```swift
actor CalculatorXPCService {
    private var connection: NSXPCConnection?

    private func ensureConnection() throws -> NSXPCConnection {
        if let conn = connection { return conn }
        let conn = NSXPCConnection(serviceName: "com.example.MyApp.CalculatorService")
        conn.remoteObjectInterface = NSXPCInterface(with: CalculatorServiceProtocol.self)
        conn.invalidationHandler = { [weak self] _ in
            Task { await self?.reset() }
        }
        conn.resume()
        connection = conn
        return conn
    }

    func reset() {
        connection?.invalidate()
        connection = nil
    }

    func add(_ a: Int, to b: Int) async throws -> Int {
        let conn = try ensureConnection()
        return try await conn.xcall(CalculatorServiceProtocol.self) { proxy, reply in
            proxy.add(a, to: b, withReply: reply)
        }
    }

    func process(files: [URL]) async throws {
        let conn = try ensureConnection()
        let iface = NSXPCInterface(with: CalculatorServiceProtocol.self)
        let allowed = NSSet(array: [NSArray.self, NSURL.self]) as Set
        iface.setClasses(allowed,
                         for: #selector(CalculatorServiceProtocol.process(files:withReply:)),
                         argumentIndex: 0,
                         ofReply: false)
        conn.remoteObjectInterface = iface

        // The `NSError?` result: nil means success, non-nil means failure.
        let error: NSError? = try await conn.xcall(CalculatorServiceProtocol.self) { proxy, reply in
            proxy.process(files: files, withReply: reply)
        }
        if let error { throw error }
    }
}
```

### NSProgress Proxying

```swift
// Service side: create progress and report
@objc protocol ExportProtocol: NSObjectProtocol {
    func export(source: URL, withReply reply: @escaping (URL?, Error?) -> Void) -> NSProgress
}

final class ExportService: NSObject, ExportProtocol {
    func export(source: URL, withReply reply: @escaping (URL?, Error?) -> Void) -> NSProgress {
        let progress = Progress(totalUnitCount: 100)
        DispatchQueue.global(qos: .userInitiated).async {
            // ... do work ...
            progress.completedUnitCount = 50
            // ... more work ...
            progress.completedUnitCount = 100
            reply(outputURL, nil)
        }
        return progress
    }
}

// Host side: observe the returned progress
let progress: NSProgress = proxy.export(source: url) { outputURL, error in
    print("Export finished")
}

progress.addObserver(self, forKeyPath: "fractionCompleted", options: .new, context: nil)

// observeValue(forKeyPath:) will fire as the service updates progress.
// Call progress.cancel() to signal cancellation to the service.
```

When an `NSProgress` is returned from an XPC method (not as a parameter, but as the method return value), `NSXPCConnection` automatically creates a proxy. KVO updates cross the connection boundary with no additional code.
