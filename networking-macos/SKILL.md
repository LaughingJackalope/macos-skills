---
name: networking-macos
description: "Networking on macOS — URLSession async/await, background sessions for daemons, Network.framework (NWConnection/NWListener/NWBrowser), WebSocket, Bonjour/mDNS, NWPathMonitor, App Transport Security, local network entitlement. Use when implementing network communication in macOS apps."
sources: https://developer.apple.com/documentation/network, https://developer.apple.com/documentation/foundation/urlsession, https://developer.apple.com/videos/play/wwdc2025/250/, https://developer.apple.com/documentation/technotes/tn3151-choosing-the-right-networking-api, https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy, https://developer.apple.com/documentation/foundation/urlsessionwebsockettask
researched: 2026-06-06
platform: macOS 26+
---

# Networking on macOS

Covers the full macOS networking stack for macOS 26+ with Swift 6.3: URLSession with async/await, background sessions (critical for daemons), Network.framework (NWConnection, NWListener, NWBrowser), WebSocket, Bonjour/mDNS service discovery, NWPathMonitor, ATS, TLS pinning, and local network entitlements. For HTTP/HTTPS workloads, start with URLSession; drop to Network.framework only when you need custom protocols, QUIC parameters, or raw socket control. For non-HTTP networking (game servers, custom TCP protocols), Network.framework is the right choice from the start.

See [references/networking-patterns.md](references/networking-patterns.md) for deep-dive code patterns: URLSession with error handling, NWConnection with TLS and state management, NWBrowser service discovery, NWPathMonitor network-aware behavior, and WebSocket bidirectional communication.

## Contents

- [URLSession Async/Await](#urlsession-asyncawait)
- [Background URLSession](#background-urlsession)
- [Network.framework](#networkframework)
- [WebSocket](#websocket)
- [Bonjour/mDNS](#bonjourmdns)
- [NWPathMonitor](#nwpathmonitor)
- [HTTP/3 & ATS](#http3--ats)
- [TLS Pinning](#tls-pinning)
- [Local Network Entitlement](#local-network-entitlement)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## URLSession Async/Await

URLSession's async/await API (available since macOS 12) is the primary entry point for HTTP networking. Five task types cover all common patterns:

```swift
// Data task — short responses (JSON APIs, WebSocket upgrades)
let (data, response) = try await URLSession.shared.data(for: request)

// Upload — from Data or file URL
let (data, response) = try await URLSession.shared.upload(for: request, from: bodyData)
let (data, response) = try await URLSession.shared.upload(for: request, fromFile: fileURL)

// Download — to a temporary file, returns file URL
let (fileURL, response) = try await URLSession.shared.download(for: request)

// Bytes — streaming response body
let asyncBytes = try await URLSession.shared.bytes(for: request)
for try await byte in asyncBytes {
    // process each UInt8
}

// Lines — line-delimited streaming (SSE, NDJSON)
let asyncBytes = try await URLSession.shared.bytes(for: request)
for try await line in asyncBytes.lines {
    // process each String
}
```

**`data(for:)` vs `data(from:)`:** `data(for:)` takes a `URLRequest` (allows method, headers, body configuration); `data(from:)` takes a plain `URL` (GET only). The overload resolution can be ambiguous at call sites — be explicit:

```swift
// Ambiguous — both URL and URLRequest are URLRequestConvertible
let (data, _) = try await URLSession.shared.data(for: url)

// Explicit:
let request = URLRequest(url: url)
let (data, _) = try Session.shared.data(for: request)   // from URLRequest
let (data, _) = try await Session.shared.data(from: url) // from URL
```

**Response validation:** The async API returns `(Data, URLResponse)`. Always cast and check `HTTPURLResponse.statusCode` before decoding:

```swift
let (data, response) = try await URLSession.shared.data(for: request)
guard let httpResponse = response as? HTTPURLResponse else {
    throw NetworkError.unexpectedResponse(response)
}
guard (200...299).contains(httpResponse.statusCode) else {
    throw NetworkError.serverError(httpResponse.statusCode, data)
}
let decoded = try JSONDecoder().decode(ResponseType.self, from: data)
```

**Configuration types:**

| Configuration | Cookies/Cache/Credentials | Background Support | Use When |
|---|---|---|---|
| `.shared` (default) | Persistent | No | Simple requests from foreground apps |
| `.ephemeral` | None (in-memory) | No | Privacy-sensitive sessions (incognito, daemons) |
| `.background` | Persistent | Yes | Transfers that must survive app suspension/termination |

```swift
// Ephemeral session — no cookies, no persistent cache
let config = URLSessionConfiguration.ephemeral
config.httpAdditionalHeaders = ["Authorization": "Bearer \(token)"]
config.timeoutIntervalForRequest = 15
config.timeoutIntervalForResource = 60
let session = URLSession(configuration: config)
```

## Background URLSession

Background transfers use a delegate-based API even when your app calls the async wrappers. The system daemon (`nsurlsessiond`) manages the actual transfer — your app can be suspended or terminated and the transfer continues.

> **Important:** Background sessions do **not** support the async/await convenience methods (`data(for:)`, `download(for:)`, etc.) — they are delegate-only. You cannot use `async/await` overloads with background session tasks; use `downloadTask(with:)` and the delegate callbacks instead. Reconnect to the session via its unique identifier if the app quits and relaunches.

```swift
final class BackgroundDownloadManager: NSObject, URLSessionDownloadDelegate {
    var backgroundSession: URLSession!
    var completionHandlers: [URLSessionTask: (Result<URL, Error>) -> Void] = [:]

    override init() {
        super.init()
        let config = URLSessionConfiguration.background(
            withIdentifier: "com.example.app.background"
        )
        config.isDiscretionary = true  // System-optimizes timing (power, WiFi)
        config.sessionSendsLaunchEvents = true  // App relaunched in background on completion
        backgroundSession = URLSession(
            configuration: config,
            delegate: self,
            delegateQueue: nil
        )
    }

    // Usage from calling code:
    func download(_ request: URLRequest, completion: @escaping (Result<URL, Error>) -> Void) {
        let task = backgroundSession.downloadTask(with: request)
        completionHandlers[task] = completion
        task.resume()
    }

    // URLSessionDownloadDelegate
    func urlSession(_ session: URLSession, downloadTask: URLSessionDownloadTask,
                    didFinishDownloadingTo location: URL) {
        completionHandlers[downloadTask]?(.success(location))
        completionHandlers.removeValue(forKey: downloadTask)
    }

    func urlSession(_ session: URLSession, task: URLSessionTask, didCompleteWithError error: Error?) {
        if let error {
            completionHandlers[task]?(.failure(error))
            completionHandlers.removeValue(forKey: task)
        }
    }

    // Required: AppDelegate must implement this
    func urlSessionDidFinishEvents(forBackgroundURLSession session: URLSession) {
        DispatchQueue.main.async {
            // Call the system-provided completion handler stored in AppDelegate
            (NSApp.delegate as? AppDelegate)?
                .backgroundSessionCompletionHandler?()
        }
    }
}
```

In `AppDelegate`, capture the system completion handler:

```swift
var backgroundSessionCompletionHandler: (() -> Void)?

func application(_ application: NSApplication,
                 handleEventsForBackgroundURLSession identifier: String,
                 completionHandler: @escaping () -> Void) {
    backgroundSessionCompletionHandler = completionHandler
}
```

**Daemon implication on macOS:** Unlike iOS, macOS daemons (`launchd` agents) are not suspended by the app lifecycle. A background URLSession in a daemon behaves identically to a foreground session — `nsurlsessiond` manages transfers regardless. However, if your daemon is `UIElement` or background-only (`LSBackgroundOnly = true`), it should still use background sessions for large transfers to avoid blocking the process.

**Unique configuration identifiers:** Each background session must have a unique identifier. Re-creating a session with the same identifier returns the existing session (and its pending tasks). Use a different identifier to create a separate session — otherwise tasks from the old delegate may be orphaned.

## Network.framework

Network.framework provides low-level TCP/UDP, TLS, and Bonjour integration. It is the right choice for custom protocols, game networking, non-HTTP RPC, or when you need direct control over connection parameters.

**ATS bypass:** ATS only applies to URLSession and CFNetwork. NWConnection and NWListener bypass ATS entirely — you are responsible for TLS configuration. If you use `NWParameters.tls`, you get system trust evaluation by default. If you disable the default verifier, you must implement your own.

### NWConnection (TCP/UDP)

```swift
let connection: NWConnection

// TCP connection
let tcpOptions = NWProtocolTCP.Options()
tcpOptions.enableKeepalive = true
tcpOptions.keepaliveInterval = 30

let tlsOptions = NWProtocolTLS.Options()
// Configure TLS parameters via sec_protocol_options_t
sec_protocol_options_set_verify_block(
    tlsOptions.securityProtocolOptions,
    { (metadata, trust, completionHandler) in
        let secTrust = sec_trust_copy_ref(trust).takeRetainedValue()
        // Custom trust evaluation here
        completionHandler(true) // or false to reject
    },
    DispatchQueue.main
)

let params = NWParameters(tls: tlsOptions, tcp: tcpOptions)
params.requiredInterfaceType = .wifi  // Constraint: WiFi only

connection = NWConnection(
    host: NWEndpoint.Host("api.example.com"),
    port: NWEndpoint.Port(integerLiteral: 443),
    using: params
)
```

**State management — always observe state changes:**

```swift
connection.stateUpdateHandler = { [weak self] newState in
    switch newState {
    case .ready:
        self?.receiveLoop()
    case .failed(let error):
        self?.handleError(error)
    case .waiting(let error):
        // Transient: no viable path, TLS failure, DNS failure
        // Network.framework will retry automatically for .noPath
        // For .tls, often requires new connection
        if error == .posix(.ENETDOWN) || error == .dns(.cancel) {
            // Fatal — recreate connection
        }
    case .cancelled:
        break  // Expected on explicit cancel
    case .preparing, .setup:
        break  // Transitional, usually brief
    @unknown default:
        break
    }
}
connection.start(queue: .global(qos: .userInitiated))
```

**Send/receive loop:**

```swift
func receiveLoop() {
    connection.receive(minimumIncompleteLength: 1, maximumLength: 65536) { data, _, isComplete, error in
        if let data, !data.isEmpty {
            self.process(data)
        }
        if isComplete {
            self.handleEOF()
        } else if let error {
            self.handleError(error)
        } else {
            self.receiveLoop()  // Continue reading
        }
    }
}

connection.send(content: messageData, completion: .contentProcessed { error in
    if let error { self.handleError(error) }
})
```

### NWListener (Incoming Connections)

```swift
let listener: NWListener
let params = NWParameters.tls  // Or .tcp for plaintext

listener = try NWListener(using: params, on: NWEndpoint.Port(integerLiteral: 8080))

listener.newConnectionHandler = { connection in
    connection.stateUpdateHandler = { state in
        switch state {
        case .ready:
            self.handleNewConnection(connection)
        case .cancelled:
            self.removeConnection(connection)
        case .failed(let error):
            self.removeConnection(connection)
        default: break
        }
    }
    connection.start(queue: .global(qos: .utility))
}
listener.start(queue: .main)
```

### NWBrowser (Service Discovery)

```swift
let browser = NWBrowser(
    for: .bonjour(type: "_myservice._tcp", domain: nil),
    using: .tcp
)

browser.browseResultsChangedHandler = { results, changes in
    for change in changes {
        switch change {
        case .add(let result):
            self.discovered(result.endpoint, txtRecord: result.metadata)
        case .remove(let result):
            self.lost(result.endpoint)
        case .changed(let old, let oldInterface, let new, let newInterface, let flags):
            if flags.contains(.interfaceChanged) {
                self.endpointChanged(new.endpoint)
            }
        case .identical:
            break
        @unknown default: break
        }
    }
}
browser.start(queue: .main)
```

**New in macOS 26 / iOS 26 (WWDC25):** Structured concurrency wrappers `NetworkConnection`, `NetworkListener`, and `NetworkBrowser` provide async/await-first alternatives to the callback-based NW* APIs. These support `Codable` types via a built-in TLV framer and handle per-connection message processing in dedicated subtasks. Prefer these for new macOS 26+ code:

```swift
// macOS 26+ only — async-first, Codable-native
let connection = try await NetworkConnection(
    to: NWEndpoint.hostPort(host: "example.com", port: .https),
    using: .tls
)
try await connection.send(MyMessage.request)
for try await message: MyMessage in connection.messages {
    process(message)
}
```

**WWDC25 "Use structured concurrency with Network framework"** also demonstrates `NetworkListener` for server-side connections with per-connection async subtasks, and `NetworkBrowser` for endpoint discovery (via Wi‑Fi Aware, Bonjour, etc.) with a cleaner async pattern than `NWBrowser`'s callback-based API.

### Service Publishing with NWListener

```swift
let params = NWParameters.tcp
let listener = try NWListener(using: params, on: .any)

// Publish Bonjour service
listener.service = NWListener.Service(
    name: "MyMac",
    type: "_myservice._tcp",
    domain: nil,
    txtRecord: TXTEncoder.dictionaryToTXTRecord(["version": "2", "api": "v3"])
)

listener.serviceRegistrationUpdateHandler = { change in
    switch change {
    case .add(let endpoint):
        print("Published: \(endpoint)")
    case .remove(let endpoint):
        print("Unpublished: \(endpoint)")
    @unknown default: break
    }
}

listener.start(queue: .main)
```

### NetService (Legacy Fallback)

NetService still works and provides a simpler API for publishing, but it is effectively in maintenance mode:

```swift
// Publishing
let service = NetService(domain: "local.", type: "_myservice._tcp", name: "MyMac", port: 8080)
service.setTXTRecordData(NetService.data(fromTXTRecord: ["version": "2"]))
service.publish(options: .listenForConnections)

// Browsing
let browser = NetServiceBrowser()
browser.delegate = self  // Use NetServiceBrowserDelegate / NetServiceDelegate
browser.searchForServices(ofType: "_myservice._tcp", inDomain: "local.")
```

**TXT record limit:** TXT records are limited to ~400 bytes total. Large payloads should reference a URL or use a different discovery mechanism.

## WebSocket

> **TN3151 guidance:** For new WebSocket code on Apple platforms, **prefer `Network.framework`** over `URLSessionWebSocketTask`. The Network framework is the favored choice. Use URLSession only if you're constrained to HTTP-layer WebSocket or need URLSession-specific features (cookies, auth challenges, ATS).

### URLSession WebSocket (for HTTP-layer needs)

`URLSessionWebSocketTask` integrates with URLSession authentication and cookie handling — use it when your WebSocket upgrade needs HTTP auth, cookies, or ATS compliance.

```swift
let wsSession = URLSession(configuration: .default)
let task = wsSession.webSocketTask(with: URL(string: "wss://echo.example.com")!)
task.resume()

// Send
task.send(.string("hello")) { error in
    if let error { print(error) }
}

// Receive (callback-based — no native async sequence)
func receiveNext() {
    task.receive { result in
        switch result {
        case .success(.string(let text)):
            handle(text)
            self.receiveNext()  // Re-enqueue
        case .success(.data(let data)):
            handle(data)
            self.receiveNext()
        case .failure(let error):
            handleError(error)
        }
    }
}

// Ping/pong
task.sendPing { error in
    if let error { print("Ping failed: \(error)") }
}
```

**AsyncSequence wrapper:** Wrap the callback-based receive into an `AsyncThrowingStream` for idiomatic async iteration (see [references/networking-patterns.md](references/networking-patterns.md) for a complete implementation):

```swift
extension URLSessionWebSocketTask {
    func asyncMessages() -> AsyncThrowingStream<Message, Error> {
        AsyncThrowingStream { continuation in
            let task = Task {
                while !Task.isCancelled {
                    do {
                        let message = try await self.receive()
                        continuation.yield(message)
                    } catch {
                        continuation.finish(throwing: error)
                        return
                    }
                }
                continuation.finish()
            }
            continuation.onTermination = { _ in task.cancel() }
        }
    }
}

// Usage:
for try await message in task.asyncMessages() {
    switch message {
    case .string(let text): handle(text)
    case .data(let data):   handle(data)
    @unknown default:       break
    }
}
```

**Closing:**

```swift
task.close(with: .normalClosure, reason: "shutting down".data(using: .utf8))
// Or for abnormal close: task.cancel(with: .invalidPayloadData, reason: nil)
```

### Network.framework WebSocket Extension

For custom framing, non-HTTP negotiation, or when you need WebSocket inside an existing NWConnection:

```swift
let connection = NWConnection(host: .name("example.com", nil), port: .https, using: .tcp)
let wsOptions = NWProtocolWebSocket.Options()
wsOptions.autoReplyPing = true
// Attach WebSocket as application protocol layer in NWParameters
```

### WebSocket Considerations

- **No automatic reconnect:** Neither `URLSessionWebSocketTask` nor Network.framework WebSocket reconnects automatically. Implement your own logic: catch errors in the receive loop, wait with backoff, create a new connection.
- **Heartbeats:** Send periodic pings via `sendPing()`. The server should respond with a pong. If you don't receive pongs, the connection is stale (common with HTTP proxies).
- **Large messages:** `URLSessionWebSocketTask` has a `maximumMessageSize` property (default 16KB). For larger payloads, fragment at the application layer.
- **Background on macOS:** WebSocket connections are severed when an app is suspended. Background sessions do **not** maintain WebSocket connections after suspension. Use a `launchd` agent (daemon) for persistent connections, or when the app becomes active again, recreate the WebSocket task.
- **`resume()` without completion handler:** `URLSessionWebSocketTask.resume()` has no async/await equivalent or completion handler. You must use completion handlers (via delegate) to know when the connection is established.

## Bonjour/mDNS

Bonjour service discovery on macOS uses NWBrowser (new) or NetService (legacy). NWBrowser is the recommended path for new code.

> **Privacy note:** On macOS 11+, browsing for Bonjour services triggers a Local Network Privacy permission prompt. The user must grant access in System Settings → Privacy & Security → Local Network. If denied, `NWBrowser.state` transitions to `.failed(.dns(DNSServiceErrorType(-65563)))`.

### Service Discovery with NWBrowser

```swift
let browser = NWBrowser(for: .bonjourWithTXTRecord(type: "_http._tcp", domain: nil), using: .tcp)

browser.stateUpdateHandler = { state in
    switch state {
    case .ready:
        print("Browser active")
    case .failed(let error):
        // Failed to start — usually missing local network entitlement on macOS
        print("Browser failed: \(error)")
    case .cancelled:
        break
    default: break
    }
}

// Assign browseResultsChangedHandler before calling start()
browser.start(queue: .main)
```

**Info.plist requirements for Bonjour:**

```xml
<!-- Declare the Bonjour service types you browse for -->
<key>NSBonjourServices</key>
<array>
    <string>_http._tcp</string>
    <string>_myservice._tcp</string>
</array>

<!-- Explain why you need local network access -->
<key>NSLocalNetworkUsageDescription</key>
<string>This app discovers local services to connect to your devices.</string>
```

**Broad mDNS discovery limitation:** `NWBrowser` cannot discover all mDNS service types (e.g., via `_services._dns-sd._udp.`). Browsing for all services is privacy-sensitive and requires the `com.apple.developer.networking.multicast` entitlement. Without it, limit discovery to specific service types you care about.

### Service Publishing with NWListener

```swift
let params = NWParameters.tcp
let listener = try NWListener(using: params, on: .any)

listener.service = NWListener.Service(
    name: "MyMac",
    type: "_myservice._tcp",
    domain: nil,
    txtRecord: TXTEncoder.dictionaryToTXTRecord(["version": "2", "api": "v3"])
)

listener.serviceRegistrationUpdateHandler = { change in
    switch change {
    case .add(let endpoint):
        print("Published: \(endpoint)")
    case .remove(let endpoint):
        print("Unpublished: \(endpoint)")
    @unknown default: break
    }
}

listener.start(queue: .main)
```

## NWPathMonitor

Replace the deprecated SystemConfiguration `SCNetworkReachability` API with `NWPathMonitor` for all reachability monitoring:

```swift
let monitor = NWPathMonitor()
// Or for a specific interface:
// let monitor = NWPathMonitor(requiredInterfaceType: .wifi)

monitor.pathUpdateHandler = { path in
    let isReachable = path.status == .satisfied
    let isWiFi = path.availableInterfaces.contains { $0.type == .wifi }
    let isWired = path.availableInterfaces.contains { $0.type == .wiredEthernet }
    let isExpensive = path.isExpensive  // Cellular on iOS; not typically true on macOS
    let isConstrained = path.isConstrained  // Low Data Mode

    // Check for specific DNS/HTTPS connectivity:
    let usesDNS = path.supportsDNS
    let usesIPv4 = path.supportsIPv4
    let usesIPv6 = path.supportsIPv6
}

// Access current path synchronously at any time:
let currentStatus = monitor.currentPath.status

monitor.start(queue: .global(qos: .utility))
// monitor.cancel() when done
```

**Key distinction — `satisfied` ≠ internet access:** `path.status == .satisfied` only means there is a valid route. The network could be a captive portal or VPN that drops packets. For true connectivity verification, send a lightweight probe (e.g., GET to a known endpoint) rather than relying solely on NWPathMonitor.

**False positives/negatives:** NWPathMonitor can produce false positives (reporting connected when internet is unreachable) and false negatives. Use it as an **advisory signal** — warn or retry, but don't gate functionality solely on its status. Test under scenarios like WWAN off, WiFi connected but no internet, etc.

**Daemon usage:** NWPathMonitor works in daemons. The path update handler fires on the queue you specify — do not dispatch to `.main` in a daemon that has no run loop on the main thread. Use a dedicated serial queue instead.

**Async/await integration:** Convert NWPathMonitor into an `AsyncStream` for idiomatic consumption (see [references/networking-patterns.md](references/networking-patterns.md) for a complete `ConnectivityMonitor` implementation with `AsyncStream<State>` and transition callbacks).

## HTTP/3 & ATS

### ATS Requirements

App Transport Security (ATS) is enabled by default and enforces:

- TLS 1.2+ only (TLS 1.0/1.1 are blocked)
- Certificate: RSA ≥ 2048-bit or ECC ≥ 256-bit, signed with SHA-256+
- Cipher: AES-128 or AES-256 with Perfect Forward Secrecy (ECDHE)
- Certificate must be trusted by the system trust store (or your custom trust evaluation)

Failed ATS connections throw `NSURLErrorServerCertificateUntrusted`, `-1202`, or log a message like: *"ATS failed system trust"`

### ATS Exceptions (Info.plist)

Add exceptions only as a last resort. Each exception weakens security:

```xml
<key>NSAppTransportSecurity</key>
<dict>
    <!-- Global exception — avoid if possible -->
    <key>NSAllowsArbitraryLoads</key>
    <false/>

    <!-- Per-domain exception for a legacy server -->
    <key>NSExceptionDomains</key>
    <dict>
        <key>legacy-api.example.com</key>
        <dict>
            <key>NSExceptionMinimumTLSVersion</key>
            <string>TLSv1.1</string>
            <key>NSExceptionRequiresForwardSecrecy</key>
            <false/>
        </dict>
    </dict>
</dict>
```

**Important:** `NSAllowsArbitraryLoads = true` is accepted for macOS but rejected for iOS App Store submissions. On macOS, it affects Gatekeeper notarization only indirectly (the notary may flag it as a security concern).

**`NSAllowsLocalNetworking`:** Allows connections to unqualified domains, `.local` domains, and IP addresses. On macOS 14+, ATS no longer allows IP addresses by default — add specific IPs and CIDR ranges in `NSExceptionDomains`. Setting `NSAllowsLocalNetworking = YES` is recommended to declare intent even when not supporting older OS versions.

### HTTP/3 via URLSession

URLSession supports HTTP/3 (QUIC) automatically when the server advertises it via Alt-Svc or DNS HTTPS records. To prefer HTTP/3 on the first connection:

```swift
var request = URLRequest(url: url)
request.assumesHTTP3Capable = true  // Skip H1/H2 negotiation attempt

let (data, response) = try await URLSession.shared.data(for: request)

// Check what protocol was actually used:
if let metrics = response as? HTTPURLResponse,
   let taskMetrics = /* captured URLSessionTaskMetrics */ {
    if let txn = taskMetrics.transactionMetrics.last {
        print("Protocol: \(txn.networkProtocolName ?? "unknown")")
        // "h3" for HTTP/3, "h2" for HTTP/2, "http/1.1"
    }
}
```

URLSession falls back to H2 or H/1.1 if HTTP/3 is not available.

## TLS Pinning

Implement TLS (certificate or public key) pinning via `URLSessionDelegate`:

```swift
final class PinningURLSessionDelegate: NSObject, URLSessionDelegate {
    // Store the SPKI hash (SHA-256 of the public key info, base64)
    let pinnedSPKIHashes: Set<String>>

    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard challenge.protectionSpace.authenticationMethod ==
                NSURLAuthenticationMethodServerTrust,
              let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        var error: CFError?
        guard SecTrustEvaluateWithError(serverTrust, &error) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        // Certificate pinning: pin a specific certificate
        guard let serverCert = SecTrustGetCertificateAtIndex(serverTrust, 0) else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }

        let serverCertData = SecCertificateCopyData(serverCert) as Data
        let pinnedCertData = loadPinnedCertificateData()  // Load from bundle

        if serverCertData == pinnedCertData {
            completionHandler(.useCredential, URLCredential(trust: serverTrust))
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }

        // Alternative: pin public key (survives certificate rotation within same keypair)
        // if let serverKey = SecCertificateCopyKey(serverCert),
        //    let serverKeyData = SecKeyCopyExternalRepresentation(serverKey, nil) {
        //     let hash = SHA256.hash(data: serverKeyData as Data)
        //     if pinnedSPKIHashes.contains(Data(hash).base64EncodedString()) {
        //         completionHandler(.useCredential, URLCredential(trust: serverTrust))
        //         return
        //     }
        // }
    }
}
```

**Pitfalls with pinning + ATS:** If you pin a certificate but the certificate meets ATS requirements, everything works. If the certificate does NOT meet ATS requirements (e.g., SHA-1 signature), you need BOTH an ATS exception AND pinning — the ATS exception disables system trust validation, and pinning replaces it with your own. This is intentionally difficult as Apple discourages connecting to insecure servers.

## Local Network Entitlement

macOS requires explicit permission to access the local network and multicast. Local Network Privacy (LNP) is tracked per user/account and per responsible code (the app, including helper tools). The system shows a consent alert the first time a local network operation occurs.

| Entitlement | Key | Required For |
|---|---|---|
| Network Client | `com.apple.security.network.client` | All outbound HTTP/TCP/UDP connections |
| Network Server | `com.apple.security.network.server` | NWListener, inbound connections |
| Multicast | `com.apple.developer.networking.multicast` | NWBrowser, mDNS TXT record changes, raw multicast |

**macOS vs iOS behavior:**

| Behavior | iOS | macOS |
|---|---|---|
| Bonjour discovery triggers privacy prompt | Yes (since iOS 14) | Yes (since macOS 11) |
| `com.apple.developer.networking.multicast` required | Yes for mDNS browse/publish | Yes for mDNS browse/publish |
| Prompt shown on first NWBrowser/NWListener use | System alert | System alert |
| User can deny in Settings | Settings → Privacy → Local Network | System Settings → Privacy & Security → Local Network |
| NWPathMonitor local reachability | Requires LocalNetwork entitlement on iOS | Same on macOS 11+ |

**macOS Sequoia+ (15+):** Local Network Privacy protections are enforced more strictly. Apps must be properly signed (Developer ID/notarization) for network access to work reliably. Some users report access failures on Sequoia unless the app is rebuilt/distributed with proper signing.

**When the user denies local network access:** `NWBrowser.state` transitions to `.failed(.dns(DNSServiceErrorType(-65563)))` — a DNSService-specific "no permission" error. Always handle this state and fall back gracefully (e.g., allow manual IP entry).

**Info.plist requirements:**

```xml
<!-- Required: explain why you access the local network -->
<key>NSLocalNetworkUsageDescription</key>
<string>This app discovers and connects to devices on your local network.</string>

<!-- Required: declare Bonjour service types you browse for -->
<key>NSBonjourServices</key>
<array>
    <string>_myservice._tcp</string>
</array>
```

**Note:** For standalone executables that aren't the main app bundle (e.g., command-line tools, helper processes), `NSLocalNetworkUsageDescription` is often unnecessary. Daemons started by `launchd` may be automatically allowed by the system.

## Constraints & Gotchas

- **Background daemon networking:** macOS daemons managed by `launchd` are not subject to iOS-style background execution modes. Network sessions (both URLSession and NWConnection) run continuously. However, energy impact matters — use `discretionary` background sessions and enable TCP keepalive to maintain NAT mappings.
- **Bonjour and firewall:** macOS application firewall can block NWListener incoming connections. The user is prompted to allow/deny. If the firewall is set to "Block all incoming connections," NWListener will not receive connections.
- **DNS deadlock:** Calling `NWEndpoint.Host("hostname")` performs synchronous DNS resolution internally. For a large number of concurrent connections, resolve DNS asynchronously via `NWConnection` with a `.name(host, nil)` endpoint which is the async path, or pre-resolve with `NWHostEndpoint`.
- **URLSession background task creation from a daemon:** Background URLSession tasks can only be created while the session's delegate is alive and the process is running. In a daemon, this is usually fine. In an app, the user may force-quit — the tasks will be relaunched by `nsurlsessiond`.
- **ATS and Network.framework:** ATS only applies to URLSession and CFNetwork. NWConnection and NWListener bypass ATS entirely — you are responsible for TLS configuration. If you use `NWParameters.tls`, you get system trust evaluation by default. If you disable the default verifier, you must implement your own.
- **NWPathMonitor false positives:** `path.status == .satisfied` means a network interface has an address. It does not guarantee connectivity to the internet. Always have a fallback strategy (retry with backoff, manual URL override).
- **Multicast entitlement and App Store:** The `com.apple.developer.networking.multicast` entitlements are available to Developer ID-signed apps (non-App Store) but require App Review approval for App Store distribution.
- **Maximum background sessions:** The system limits the number of background sessions per app. Typically 3–4 concurrently. Creating more silently fails or reuses existing sessions.
- **Background WebSocket:** WebSocket connections are severed when an app is suspended. Background URLSession does not maintain WebSocket connections. Reconnect when the app becomes active again.
- **Broad mDNS discovery:** NWBrowser cannot discover all mDNS service types without the multicast entitlement. Limit to specific service types.

## Common Mistakes

- **Ignoring `NWConnection.stateUpdateHandler`:** Failing to observe state means you can't detect failures, waiting states, or connection readiness. Always set a state handler before calling `start()`.
- **Calling `receive()` once instead of looping:** A single `receive()` call returns one chunk. For a persistent connection, re-enqueue receive in the completion handler.
- **Using `URLSession.shared` in daemons:** The shared session persists cookies across all requests. In a daemon handling multiple user contexts, create isolated sessions with `.ephemeral` configuration.
- **Confusing `NWBrowser.cancel()` and memory management:** Cancelling an NWBrowser stops discovery but doesn't remove existing results from your cache. Track results separately and clear them on `.cancelled`.
- **Forgetting `connection.start(queue: ...)`:** An NWConnection does nothing until started. This is the #1 reason connections appear to "hang."
- **ATS exception without pinning:** Adding an ATS exception enables insecure connections without adding any compensating security. If you must use an exception, at minimum implement TLS pinning.
- **Not handling `NWListener.Service.BrowseResults.changed`:** Service endpoints update (IP change, TXT record change). Ignoring `.changed` results in stale endpoint references.
- **Pinning the leaf certificate (too tight):** If the server rotates certificates (Let's Encrypt every 90 days), leaf pinning breaks. Pin the intermediate CA or the public key SPKI hash instead.
- **Using async/await with background URLSession:** Background sessions are delegate-only. The async convenience methods (`data(for:)`, `download(for:)`) do not work with background configurations.
- **Missing `NSBonjourServices` in Info.plist:** Without declaring your Bonjour service types, NWBrowser may fail silently on macOS 14+.
- **Not handling local network permission denial:** Always handle `.failed(.dns(DNSServiceErrorType(-65563)))` in `NWBrowser.stateUpdateHandler` and provide a fallback (manual IP entry).

## Review Checklist

- [ ] URLSession requests use explicit `URLRequest` for non-GET methods
- [ ] HTTP status codes are checked before JSON decoding (no silent decode failures)
- [ ] Background URLSession uses unique identifiers and captures the system completion handler in AppDelegate
- [ ] Background URLSession uses delegate-based API (not async/await convenience methods)
- [ ] NWConnection has a `stateUpdateHandler` set before `start()`
- [ ] Receive loops re-enqueue `receive()` in the completion handler
- [ ] NWBrowser handler processes add, remove, and changed results
- [ ] `NWBrowser.stateUpdateHandler` handles `.failed` (including local network permission denied)
- [ ] NWPathMonitor handler uses `path.status == .satisfied` and understands it's interface-level, not internet-level
- [ ] TLS pinning checks certificate/public key before calling `.useCredential`
- [ ] `com.apple.security.network.client` and `.server` entitlements are set
- [ ] `com.apple.developer.networking.multicast` entitlement is set for Bonjour (NWBrowser/NWListener)
- [ ] `NSBonjourServices` declared in Info.plist for Bonjour service types
- [ ] `NSLocalNetworkUsageDescription` set in Info.plist
- [ ] ATS exceptions in Info.plist are domain-specific (not `NSAllowsArbitraryLoads = true`)
- [ ] NWConnection is properly cancelled in `deinit` or scene teardown
- [ ] WebSocket reconnection logic implemented (no auto-reconnect)
- [ ] For new macOS 26+ code, `NetworkConnection`/`NetworkListener`/`NetworkBrowser` preferred over callback-based NW* APIs

## References

- [references/networking-patterns.md](references/networking-patterns.md) — Deep-dive code patterns for URLSession, NWConnection, NWBrowser, NWPathMonitor, and WebSocket
- [Network Framework Documentation](https://developer.apple.com/documentation/network)
- [URLSession Documentation](https://developer.apple.com/documentation/foundation/urlsession)
- [URLSessionWebSocketTask Documentation](https://developer.apple.com/documentation/foundation/urlsessionwebsockettask)
- [WWDC25: Use structured concurrency with Network framework](https://developer.apple.com/videos/play/wwdc2025/250/)
- [TN3151: Choosing the right networking API](https://developer.apple.com/documentation/Technotes/tn3151-choosing-the-right-networking-api)
- [TN3179: Understanding local network privacy](https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy)
- [TN3102: HTTP/3 in your app](https://developer.apple.com/documentation/technotes/tn3102-http3-in-your-app)
- [Preventing Insecure Network Connections](https://developer.apple.com/documentation/security/preventing-insecure-network-connections)
