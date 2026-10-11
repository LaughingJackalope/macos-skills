# Networking Patterns for macOS

Deep-dive code patterns for the major macOS networking APIs: URLSession with async/await and error handling, NWConnection with TLS and state management, NWBrowser service discovery, NWPathMonitor network awareness, and WebSocket bidirectional communication.

## Contents

- [URLSession with Async/Await and Error Handling](#urlsession-with-asyncawait-and-error-handling)
- [NWConnection with TLS and State Management](#nwconnection-with-tls-and-state-management)
- [NWBrowser Service Discovery Pattern](#nwbrowser-service-discovery-pattern)
- [NWPathMonitor for Network-Aware App Behavior](#nwpathmonitor-for-network-aware-app-behavior)
- [WebSocket Bidirectional Communication](#websocket-bidirectional-communication)

## URLSession with Async/Await and Error Handling

### Complete Service Layer Pattern

A production-quality layer wraps URLSession's async API with typed errors, request configuration, and response decoding:

```swift
enum NetworkError: Error, LocalizedError {
    case invalidURL(String)
    case invalidResponse(URLResponse, Data)
    case httpError(statusCode: Int, body: Data)
    case decodingFailed(Error, Data)
    case noConnection
    case requestTimeout
    case serverCertificateUntrusted
    case cancelled

    var errorDescription: String? {
        switch self {
        case .invalidURL(let raw): return "Invalid URL: \(raw)"
        case .invalidResponse: return "Unexpected response type"
        case .httpError(let code, _): return "HTTP \(code)"
        case .decodingFailed(let error, _): return "Decode failed: \(error.localizedDescription)"
        case .noConnection: return "No network connection"
        case .requestTimeout: return "Request timed out"
        case .serverCertificateUntrusted: return "Server certificate not trusted"
        case .cancelled: return "Request was cancelled"
        }
    }
}

struct APIClient {
    let session: URLSession
    let decoder: JSONDecoder
    let encoder: JSONEncoder

    init(session: URLSession = .shared, decoder: JSONDecoder = JSONDecoder()) {
        self.session = session
        self.decoder = decoder
        self.encoder = JSONEncoder()
    }

    func request<T: Decodable>(
        _ path: String,
        method: String = "GET",
        body: Encodable? = nil,
        headers: [String: String] = [:]
    ) async throws -> T {
        guard let url = URL(string: path) else {
            throw NetworkError.invalidURL(path)
        }

        var request = URLRequest(url: url)
        request.httpMethod = method
        request.timeoutInterval = 30
        for (key, value) in headers {
            request.setValue(value, forHTTPHeaderField: key)
        }

        if let body {
            request.setValue("application/json", forHTTPHeaderField: "Content-Type")
            request.httpBody = try encoder.encode(AnyEncodable(body))
        }

        let data: Data
        let response: URLResponse

        do {
            (data, response) = try await session.data(for: request)
        } catch let urlError as URLError {
            switch urlError.code {
            case .notConnectedToInternet, .networkConnectionLost:
                throw NetworkError.noConnection
            case .timedOut:
                throw NetworkError.requestTimeout
            case .serverCertificateUntrusted, .serverCertificateHasBadDate,
                 .serverCertificateHasUnknownRoot, .serverCertificateNotYetValid:
                throw NetworkError.serverCertificateUntrusted
            case .cancelled:
                throw NetworkError.cancelled
            default:
                throw urlError
            }
        }

        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse(response, data)
        }

        guard (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.httpError(statusCode: httpResponse.statusCode, body: data)
        }

        return try decoder.decode(T.self, from: data)
    }
}

// Type erasure for Encodable bodies
struct AnyEncodable: Encodable {
    private let _encode: (Encoder) throws -> Void
    init<T: Encodable>(_ wrapped: T) { _encode = wrapped.encode }
    func encode(to encoder: Encoder) throws { try _encode(encoder) }
}
```

### Retry with Exponential Backoff

```swift
func fetchWithRetry<T: Decodable>(
    request: URLRequest,
    maxRetries: Int = 3,
    initialDelay: Duration = .seconds(1),
    decoder: JSONDecoder = JSONDecoder()
) async throws -> T {
    var delay = initialDelay

    for attempt in 0...maxRetries {
        do {
            let (data, response) = try await URLSession.shared.data(for: request)
            guard let httpResponse = response as? HTTPURLResponse,
                  (200...299).contains(httpResponse.statusCode) else {
                let statusCode = (response as? HTTPURLResponse)?.statusCode ?? -1
                throw NetworkError.httpError(statusCode: statusCode, body: data)
            }
            return try decoder.decode(T.self, from: data)
        } catch {
            if attempt == maxRetries { throw error }
            // Don't retry on client errors (4xx) — they won't succeed
            if case NetworkError.httpError(let code, _) = error, (400...499).contains(code) {
                throw error
            }
            if case NetworkError.cancelled = error { throw error }
            try await Task.sleep(for: delay)
            delay *= 2  // 1s, 2s, 4s
        }
    }

    throw NetworkError.cancelled  // unreachable but required by compiler
}
```

### Streaming with `bytes(for:)`

`URLSession.shared` uses system trust. On a session with a custom trust delegate (pinning, self-signed servers), `bytes(for:)` / `bytes(from:)` consult only the task-level challenge method. Implement both methods (SKILL.md → TLS Pinning) or the stream fails with -1202.

```swift
func streamLines(from url: URL) -> AsyncThrowingStream<String, Error> {
    AsyncThrowingStream { continuation in
        let task = Task {
            do {
                let (asyncBytes, response) = try await URLSession.shared.bytes(from: url)
                guard let httpResponse = response as? HTTPURLResponse,
                      (200...299).contains(httpResponse.statusCode) else {
                    continuation.finish(throwing: NetworkError.httpError(
                        statusCode: (response as? HTTPURLResponse)?.statusCode ?? 0,
                        body: Data()
                    ))
                    return
                }
                for try await line in asyncBytes.lines {
                    continuation.yield(line)
                }
                continuation.finish()
            } catch {
                continuation.finish(throwing: error)
            }
        }
        continuation.onTermination = { _ in task.cancel() }
    }
}

// Usage:
for try await line in streamLines(from: url) {
    if let data = line.data(using: .utf8),
       let event = try? JSONDecoder().decode(ServerEvent.self, from: data) {
        handle(event)
    }
}
```

## NWConnection with TLS and State Management

### Full Connection Lifecycle

A complete NWConnection wrapper that handles DNS resolution, TLS, state transitions, and automatic reconnection:

```swift
actor Connection {
    private var connection: NWConnection?
    private let endpoint: NWEndpoint.Host
    private let port: NWEndpoint.Port
    private let parameters: NWParameters
    private var isConnecting = false

    var onMessage: @Sendable (Data) async -> Void = { _ in }
    var onError: @Sendable (Error) async -> Void = { _ in }

    init(host: String, port: UInt16, parameters: NWParameters = .tls) {
        self.endpoint = NWEndpoint.Host(host)
        self.port = NWEndpoint.Port(integerLiteral: port)
        self.parameters = parameters
    }

    func connect() async throws {
        guard !isConnecting else { return }
        isConnecting = true
        defer { isConnecting = false }

        let nwConnection = NWConnection(host: endpoint, port: port, using: parameters)
        self.connection = nwConnection

        try await withCheckedThrowingContinuation { (continuation: CheckedContinuation<Void, Error>) in
            nwConnection.stateUpdateHandler = { [weak self] state in
                Task { [weak self] in
                    await self?.handleState(state, continuation: continuation)
                }
            }
            nwConnection.start(queue: DispatchQueue(label: "com.example.networking"))
        }
    }

    private func handleState(
        _ state: NWConnection.State,
        continuation: CheckedContinuation<Void, Error>?
    ) {
        switch state {
        case .ready:
            continuation?.resume()
            Task { await receiveLoop() }
        case .failed(let error):
            continuation?.resume(throwing: error)
            Task { await onError(error); try? await reconnect() }
        case .waiting(let error):
            // .noPath: Network.framework retries automatically
            // .tls: TLS handshake failed — usually fatal
            if case .tls = error {
                continuation?.resume(throwing: error)
            }
        case .cancelled:
            continuation?.resume(throwing: NetworkError.cancelled)
        default: break
        }
    }

    func send(_ data: Data) async throws {
        guard let connection else { throw NetworkError.noConnection }
        return try await withCheckedThrowingContinuation { continuation in
            connection.send(content: data, completion: .contentProcessed { error in
                if let error { continuation.resume(throwing: error) }
                else { continuation.resume() }
            })
        }
    }

    private func receiveLoop() async {
        guard let connection else { return }
        while connection.state == .ready {
            do {
                let data = try await receive()
                if data.isEmpty { break }
                await onMessage(data)
            } catch {
                await onError(error)
                break
            }
        }
    }

    private func receive() async throws -> Data {
        let conn = self.connection!
        return try await withCheckedThrowingContinuation { continuation in
            conn.receive(minimumIncompleteLength: 1, maximumLength: .max) { data, _, _, error in
                if let error { continuation.resume(throwing: error) }
                else { continuation.resume(returning: data ?? Data()) }
            }
        }
    }

    private func reconnect() async throws {
        connection?.cancel()
        connection = nil
        try await Task.sleep(nanoseconds: 2_000_000_000)  // 2s
        try await connect()
    }

    func disconnect() {
        connection?.cancel()
        connection = nil
    }
}
```

### TLS Custom Trust with NWParameters

```swift
func buildParameters(pinningMode: PinningMode?) -> NWParameters {
    let tlsOptions = NWProtocolTLS.Options()

    sec_protocol_options_set_verify_block(
        tlsOptions.securityProtocolOptions,
        { (metadata, trust, completionHandler) in
            let secTrust = sec_trust_copy_ref(trust).takeRetainedValue()

            // 1. Standard system validation first
            var error: CFError?
            guard SecTrustEvaluateWithError(secTrust, &error) else {
                completionHandler(false)
                return
            }

            // 2. Optional: pin specific certificates
            if let pinningMode {
                let certs = (0..<SecTrustGetCertificateCount(secTrust)).compactMap {
                    SecTrustGetCertificateAtIndex(secTrust, $0)
                }
                let certData = certs.map { SecCertificateCopyData($0) as Data }

                if !certData.matchesAny(pinningMode.pinnedCertificates) {
                    completionHandler(false)
                    return
                }
            }

            completionHandler(true)
        },
        DispatchQueue.global(qos: .userInitiated)
    )

    let tcpOptions = NWProtocolTCP.Options()
    tcpOptions.enableKeepalive = true
    tcpOptions.keepaliveInterval = 30
    tcpOptions.keepaliveCount = 8

    return NWParameters(tls: tlsOptions, tcp: tcpOptions)
}

enum PinningMode {
    case certificates  // Pin full certificate data
    case spkiHash      // Pin public key hash

    var pinnedCertificates: [Data] {
        // Load from bundled certificates or embedded hashes
        []
    }
}
```

### Network.framework Research Notes (2026)
New in macOS 26 / iOS 26 (WWDC25): `NetworkConnection`, `NetworkListener`, and `NetworkBrowser` provide async/await-first, Codable-native wrappers over the callback-based NW* APIs. Use these for new macOS 26+ code.
Structured concurrency: WWDC25 "Use structured concurrency with Network framework" demonstrates per-connection async subtasks, type-safe message framing with Codable, and cleaner server/client patterns.
ATS bypass: NWConnection and NWListener bypass ATS entirely. You are responsible for TLS configuration. `NWParameters.tls` gives system trust evaluation by default; disabling it requires custom trust implementation.
Connection metrics: Use `startDataTransferReport()`, `requestEstablishmentReport()`, and `NWConnection.DataTransferReport` for performance monitoring.
## NWBrowser Service Discovery Pattern

### Complete Bonjour Discovery with State Tracking

```swift
actor BonjourDiscovery {
    private var browser: NWBrowser?
    private var isStarted = false

    /// Currently discovered services, keyed by name
    private(set) var services: [String: DiscoveredService] = [:]

    /// Callbacks for UI updates
    var onDiscoveryChanged: @Sendable ([DiscoveredService]) async -> Void = { _ in }

    struct DiscoveredService: Hashable, @unchecked Sendable {
        let endpoint: NWEndpoint
        let name: String
        let txtRecord: [String: String]
        let interfaces: [NWInterface]
    }

    func start(serviceType: String = "_http._tcp", domain: String? = nil) {
        guard !isStarted else { return }

        let descriptor = NWBrowser.Descriptor.bonjour(
            type: serviceType,
            domain: domain
        )
        let params = NWParameters.tcp

        browser = NWBrowser(for: descriptor, using: params)

        browser?.stateUpdateHandler = { [weak self] state in
            Task { [weak self] in
                await self?.handleBrowserState(state)
            }
        }

        browser?.browseResultsChangedHandler = { [weak self] results, changes in
            Task { [weak self] in
                await self?.handleChanges(results: results, changes: changes)
            }
        }

        browser?.start(queue: .main)
        isStarted = true
    }

    func stop() {
        browser?.cancel()
        browser = nil
        isStarted = false
        services.removeAll()
        Task { await onDiscoveryChanged([]) }
    }

    /// Connect to a discovered service
    func connectToService(_ service: DiscoveredService) async throws -> NWConnection {
        let params = NWParameters.tls
        // NWBrowser endpoints (like .service(name:,type:domain:)) can be used
        // directly in NWConnection init
        let connection = NWConnection(to: service.endpoint, using: params)

        return try await withCheckedThrowingContinuation { continuation in
            connection.stateUpdateHandler = { state in
                switch state {
                case .ready:
                    continuation.resume(returning: connection)
                case .failed(let error):
                    continuation.resume(throwing: error)
                case .waiting(let error):
                    if case .tls = error {
                        continuation.resume(throwing: error)
                    }
                default: break
                }
            }
            connection.start(queue: .global(qos: .userInitiated))
        }
    }

    private func handleBrowserState(_ state: NWBrowser.State) async {
        switch state {
        case .failed(let error):
            // Common: denied permission — DNSServiceErrorDomain, -65563
            if error == .dns(DNSServiceErrorType(kDNSServiceErr_Policy)) {
                print("Local network permission denied — prompt user to enable")
                // Fall back to manual connection
            }
            // With result: UI prompt
            await onDiscoveryChanged(Array(services.values))
        case .cancelled:
            services.removeAll()
            await onDiscoveryChanged([])
        case .setup, .ready:
            break
        @unknown default: break
        }
    }

    private func handleChanges(results: Set<NWBrowser.Result>, changes: Set<NWBrowser.Result.Change>) async {
        for change in changes {
            switch change {
            case .add(let result):
                let service = parseResult(result)
                services[service.name] = service
            case .remove(let result):
                if case .service(let name, _, _) = result.endpoint {
                    services.removeValue(forKey: name)
                }
            case .changed(_, _, let new, _, _):
                let service = parseResult(new)
                services[service.name] = service
            case .identical:
                break
            @unknown default: break
            }
        }
        await onDiscoveryChanged(Array(services.values))
    }

    private func parseResult(_ result: NWBrowser.Result) -> DiscoveredService {
        let name: String
        if case .service(let n, _, _) = result.endpoint {
            name = n
        } else {
            name = "\(result.endpoint)"
        }

        var txt: [String: String] = [:]
        if case .bonjour(let txtDict) = result.metadata {
            txt = txtDict
        }

        let interfaces = result.interfaces

        return DiscoveredService(
            endpoint: result.endpoint,
            name: name,
            txtRecord: txt,
            interfaces: interfaces
        )
    }
}

/// Helper to connect and immediately exchange data
extension BonjourDiscovery {
    func connectAndSend(
        to service: DiscoveredService,
        data: Data,
        responseHandler: @escaping (Data) -> Void
    ) async throws -> NWConnection {
        let connection = try await connectToService(service)

        connection.stateUpdateHandler = { [weak connection] state in
            if case .ready = state, let conn = connection {
                conn.receive(minimumIncompleteLength: 1, maximumLength: 65536) { data, _, _, _ in
                    if let data, !data.isEmpty { responseHandler(data) }
                }
            }
        }

        try await withCheckedThrowingContinuation { (c: CheckedContinuation<Void, Error>) in
            connection.send(content: data, completion: .contentProcessed { error in
                if let error { c.resume(throwing: error) }
                else { c.resume() }
            })
        }

        return connection
    }
}
```
```
### NWBrowser Research Notes (2026)
Cannot discover all mDNS services: NWBrowser does not expose `_services._dns-sd._udp.` (broad service type discovery). Browse specific types like `_http._tcp`. For broad discovery, request `com.apple.developer.networking.multicast` entitlement.
Privacy permission (macOS 11+): NWBrowser triggers Local Network Privacy prompt. If denied: `.failed(.dns(DNSServiceErrorType(-65563)))`. Always handle this and provide manual IP fallback.
Info.plist required: Declare browsed service types in `NSBonjourServices` and set `NSLocalNetworkUsageDescription` or NWBrowser may fail silently on macOS 14+.

## NWPathMonitor for Network-Aware App Behavior

### Production Connectivity Monitor

A reusable monitor that translates NWPathMonitor callbacks into Swift-friendly state:

```swift
@MainActor
final class ConnectivityMonitor: ObservableObject {
    struct State: Equatable {
        var isConnected: Bool = false
        var connectionType: ConnectionType = .unknown
        var isExpensive: Bool = false
        var isConstrained: Bool = false
        var supportsIPv4: Bool = false
        var supportsIPv6: Bool = false
        var supportsDNS: Bool = false

        enum ConnectionType: Equatable {
            case wifi, wiredEthernet, loopback, other, unknown
        }
    }

    @Published private(set) var state = State()
    @Published private(set) var connectionHistory: [(Date, State)] = []

    private let monitor: NWPathMonitor
    private let queue = DispatchQueue(label: "com.example.connectivity")

    var onConnectionRestored: (() -> Void)?
    var onConnectionLost: (() -> Void)?
    var onTypeChanged: ((State.ConnectionType) -> Void)?

    init(monitor: NWPathMonitor = NWPathMonitor()) {
        self.monitor = monitor
        self.monitor.pathUpdateHandler = { [weak self] path in
            Task { @MainActor [weak self] in
                self?.process(path: path)
            }
        }
    }

    func start() {
        monitor.start(queue: queue)
        process(path: monitor.currentPath)  // Initial state
    }

    func stop() {
        monitor.cancel()
    }

    /// Force a re-read of current path (e.g., when app becomes active)
    func refresh() {
        guard monitor.currentPath.status != .notReported else { return }
        process(path: monitor.currentPath)
    }

    private func process(path: NWPath) {
        let previous = state
        let interfaces = path.availableInterfaces

        let type: State.ConnectionType = {
            if interfaces.contains(where: { $0.type == .wifi }) { return .wifi }
            if interfaces.contains(where: { $0.type == .wiredEthernet }) { return .wiredEthernet }
            if interfaces.contains(where: { $0.type == .loopback }) { return .loopback }
            if !interfaces.isEmpty { return .other }
            return .unknown
        }()

        state = State(
            isConnected: path.status == .satisfied,
            connectionType: type,
            isExpensive: path.isExpensive,
            isConstrained: path.isConstrained,
            supportsIPv4: path.supportsIPv4,
            supportsIPv6: path.supportsIPv6,
            supportsDNS: path.supportsDNS
        )

        connectionHistory.append((Date(), state))
        // Keep last 100 entries
        if connectionHistory.count > 100 { connectionHistory.removeFirst() }

        // Fire callbacks for transitions
        if !previous.isConnected && state.isConnected {
            onConnectionRestored?()
        } else if previous.isConnected && !state.isConnected {
            onConnectionLost?()
        }
        if previous.connectionType != state.connectionType && state.isConnected {
            onTypeChanged?(state.connectionType)
        }
    }

    /// AsyncSequence-based observation
    var stateStream: AsyncStream<State> {
        AsyncStream { continuation in
            let monitor = self.monitor
            let queue = DispatchQueue(label: "com.example.connectivity.stream")

            let pathMonitor = NWPathMonitor()
            pathMonitor.pathUpdateHandler = { path in
                let isConnected = path.status == .satisfied
                let interfaces = path.availableInterfaces
                let type: State.ConnectionType = {
                    if interfaces.contains(where: { $0.type == .wifi }) { return .wifi }
                    if interfaces.contains(where: { $0.type == .wiredEthernet }) { return .wiredEthernet }
                    if interfaces.contains(where: { $0.type == .loopback }) { return .loopback }
                    if !interfaces.isEmpty { return .other }
                    return .unknown
                }()
                continuation.yield(State(
                    isConnected: isConnected,
                    connectionType: type,
                    isExpensive: path.isExpensive,
                    isConstrained: path.isConstrained,
                    supportsIPv4: path.supportsIPv4,
                    supportsIPv6: path.supportsIPv6,
                    supportsDNS: path.supportsDNS
                ))
            }
            pathMonitor.start(queue: queue)
            continuation.onTermination = { _ in pathMonitor.cancel() }
        }
    }
}

// Usage in SwiftUI:
// @StateObject private var connectivity = ConnectivityMonitor()
// .onAppear { connectivity.start() }
// .onChange(of: connectivity.state.isConnected) { _, connected in
//     if connected { resumePendingUploads() }
// }
```

### Network-Aware Request Scheduling

```swift
final class NetworkAwareScheduler {
    private let monitor: NWPathMonitor
    private var pendingWork: [() async -> Void] = []
    private var isOnline = false

    init() {
        monitor = NWPathMonitor()
        monitor.pathUpdateHandler = { [weak self] path in
            Task { [weak self] in
                self?.handlePathUpdate(path)
            }
        }
    }

    func start() {
        monitor.start(queue: .global(qos: .utility))
    }

    /// Schedule work to run immediately if online, or queued until connectivity returns
    func enqueue(_ work: @escaping () async -> Void) {
        if isOnline {
            Task { await work() }
        } else {
            pendingWork.append(work)
        }
    }

    private func handlePathUpdate(_ path: NWPath) {
        let wasOnline = isOnline
        isOnline = path.status == .satisfied && !path.isConstrained

        if !wasOnline && isOnline {
            // Connectivity restored — drain queue
            let work = pendingWork
            pendingWork.removeAll()
            Task {
                for op in work {
                    await op()
                }
            }
        }

        if path.isExpensive {
            // Defer large transfers — only do critical operations
            print("Expensive network — deferring non-critical work")
        }
    }
}
```
```
### NWPathMonitor Research Notes (2026)
False positives/negatives: NWPathMonitor can report connected when internet is unreachable (captive portal) or disconnected when a viable path exists. Use advisory-only: warn/retry, never gate functionality solely on its status.
Test scenarios: WWAN off, WiFi connected but no internet, VPN active, captive portal. NWPathMonitor may return stale or misleading results in these cases.
Async/await: Wrap in AsyncStream (see `ConnectivityMonitor.stateStream` above) for idiomatic async consumption. Avoid blocking waits on path status.
Daemon usage: Path update handler fires on the specified queue — do not dispatch to `.main` in a daemon without a main run loop. Use a dedicated serial queue.

## WebSocket Bidirectional Communication

### URLSessionWebSocketTask with AsyncIterator

Wrap the callback-based `URLSessionWebSocketTask.receive()` into a clean async sequence:

```swift
actor WebSocketClient {
    private var task: URLSessionWebSocketTask?
    private let session: URLSession

    var isConnected: Bool {
        task?.state == .running
    }

    init(session: URLSession = .default) {
        self.session = session
    }

    func connect(to url: URL) async throws {
        guard task == nil || task?.state != .running else { return }
        let newTask = session.webSocketTask(with: url)
        self.task = newTask
        newTask.resume()

        // Wait for connection to be established by attempting a ping
        do {
            try await withTimeout(.seconds(10)) {
                try await newTask.sendPing { error in
                    if let error { throw error }
                }
            }
        } catch {
            task?.cancel(with: .goingAway, reason: nil)
            task = nil
            throw error
        }
    }

    /// Send a text message
    func send(_ text: String) async throws {
        guard let task, task.state == .running else {
            throw WebSocketError.notConnected
        }
        try await task.send(.string(text))
    }

    /// Send binary data
    func send(_ data: Data) async throws {
        guard let task, task.state == .running else {
            throw WebSocketError.notConnected
        }
        try await task.send(.data(data))
    }

    /// AsyncStream of incoming messages
    func messages() -> AsyncThrowingStream<URLSessionWebSocketTask.Message, Error> {
        AsyncThrowingStream { continuation in
            let task = Task {
                guard let wsTask, wsTask.state == .running else {
                    continuation.finish(throwing: WebSocketError.notConnected)
                    return
                }
                continuation.onTermination = { @Sendable _ in
                    wsTask.cancel(with: .goingAway, reason: nil)
                }
                do {
                    for try await message in wsTask.asyncMessages() {
                        continuation.yield(message)
                    }
                    continuation.finish()
                } catch {
                    continuation.finish(throwing: error)
                }
            }
        }
    }

    func disconnect() async {
        task?.cancel(with: .normalClosure, reason: nil)
        task = nil
    }
}

enum WebSocketError: Error {
    case notConnected
    case timedOut
}

/// Convert callback-based receive() into AsyncSequence
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

/// Throttle/dedupe utility
func withTimeout<T>(_ duration: Duration, operation: @escaping () async throws -> T) async throws -> T {
    try await withThrowingTaskGroup(of: T.self) { group in
        group.addTask { try await operation() }
        group.addTask {
            try await Task.sleep(for: duration)
            throw WebSocketError.timedOut
        }
        let result = try await group.next()!
        group.cancelAll()
        return result
    }
}
```

### Usage

```swift
let client = WebSocketClient()
try await client.connect(to: URL(string: "wss://api.example.com/stream")!)

Task {
    for try await message in client.messages() {
        switch message {
        case .string(let text):
            print("Text: \(text)")
        case .data(let data):
            print("Binary: \(data.count) bytes")
        @unknown default: break
        }
    }
}

try await client.send("{\"action\":\"subscribe\",\"channel\":\"updates\"}")
```

### Important WebSocket Considerations

- **No automatic reconnect:** URLSessionWebSocketTask does NOT reconnect. Implement your own logic: catch errors in the receive loop, wait with backoff, create a new task.
- **Heartbeats:** Send periodic pings via `sendPing()`. The server should respond with a pong. If you don't receive pongs, the connection is stale (common with HTTP proxies).
- **Large messages:** WebSocket has a default 16KB message limit in URLSession. For larger payloads, fragment at the application layer or use chunked binary frames.
- **Background WebSocket on macOS:** When an app is suspended, the WebSocket connection is severed. Use a `launchd` agent (daemon) for persistent connections, or use `BGTaskScheduler` for periodic wake-ups (iOS only; on macOS, prefer daemons).
- **Prefer Network.framework for WebSocket (TN3151):** Apple recommends Network framework over `URLSessionWebSocketTask` for new WebSocket code. URLSession WebSocket is acceptable when you need HTTP-layer features (cookies, auth, ATS).
- **resume() has no completion:** `URLSessionWebSocketTask.resume()` is fire-and-forget — no async/await equivalent or completion handler. Use `sendPing()` after resume to verify the connection is established, wrapped in a timeout.
- **Background sessions do not maintain WebSockets:** Background URLSession configurations do not maintain WebSocket connections after app suspension. You must reconnect when the app relaunches.
- **AsyncSequence wrapping:** Wrap callback-based `receive()` in `AsyncThrowingStream` for clean async iteration. The callback-based approach and the AsyncStream wrapper have trade-offs: callback is more reliable for cancellation lifecycle; the AsyncStream wrapper is cleaner for consumption.
