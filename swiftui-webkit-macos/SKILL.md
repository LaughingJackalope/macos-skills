---
name: swiftui-webkit-macos
description: "WebKit on macOS — WebView (SwiftUI-native macOS 26), WKWebView as NSViewRepresentable, WKNavigationDelegate/WKUIDelegate, JavaScript bridge (WKScriptMessageHandler, callAsyncJavaScript), custom URL schemes, download handling, PDF rendering. Use when embedding web content in macOS apps."
sources: https://developer.apple.com/documentation/webkit/wkwebview, https://developer.apple.com/documentation/webkit/wkscriptmessagehandler, https://developer.apple.com/documentation/webkit/wknavigationdelegate, https://developer.apple.com/documentation/webkit/wkuidelegate, https://developer.apple.com/documentation/webkit/wkdownloaddelegate, https://developer.apple.com/documentation/webkit/wkurlschemehandler, https://developer.apple.com/documentation/webkit/wkpdfconfiguration, https://developer.apple.com/videos/play/wwdc2025/231/
researched: 2026-06-06
platform: macOS 26+
---

# WebKit on macOS

Embed web content in macOS apps using the SwiftUI-native `WebView` (macOS 26+) or wrap `WKWebView` via `NSViewRepresentable` for fine-grained control and backward compatibility. This skill covers the full WebKit surface: navigation control, JavaScript bridge, custom URL schemes, download handling, PDF rendering, and configuration.

**Owns:** WebKit integration, JavaScript-to-native bridge, web content rendering.
**Owned by:** `swiftui-appkit-interop` for the representable wrapping pattern; `macos-security-entitlements` for sandbox + network entitlements.

## Contents

- [SwiftUI WebView (macOS 26)](#swiftui-webview-macos-26)
- [WKWebView as NSViewRepresentable](#wkwebview-as-nsviewrepresentable)
- [WKNavigationDelegate](#wknavigationdelegate)
- [WKUIDelegate](#wkuidelegate)
- [JavaScript Bridge](#javascript-bridge)
- [Custom URL Schemes](#custom-url-schemes)
- [Download Handling](#download-handling)
- [PDF Rendering](#pdf-rendering)
- [WKWebViewConfiguration](#wkwebviewconfiguration)
- [Constraints & Gotchas](#constraints-and-gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## SwiftUI WebView (macOS 26)

Apple introduced a native SwiftUI `WebView` in macOS 26 (WWDC25, session 231). It replaces the need for `NSViewRepresentable` wrapping in most cases.

### WebView + WebPage

```swift
import SwiftUI
import WebKit

struct BrowserView: View {
    @State private var page = WebPage()

    var body: some View {
        WebView(page)
            .onAppear {
                page.load(URLRequest(url: URL(string: "https://apple.com")!))
            }
    }
}
```

### Loading Content

```swift
// From URL
page.load(URLRequest(url: url))

// From HTML string
let html = "<h1>Hello</h1>"
page.load(html.data(using: .utf8)!, mimeType: "text/html", characterEncodingName: "utf-8", baseURL: URL(string: "about:blank")!)

// From WebArchive data
page.load(webArchiveData, mimeType: "application/x-webarchive", characterEncodingName: "utf-8", baseURL: URL(string: "about:blank")!)
```

### Observability

```swift
struct BrowserView: View {
    @State private var page = WebPage()

    var body: some View {
        WebView(page)
            .inspector(isPresented: .constant(true)) {
                // Inspector sidebar content
                VStack {
                    Text(page.title ?? "No title")
                    Text(page.url?.absoluteString ?? "")
                    ProgressView(value: page.estimatedProgress)
                }
            }
            .toolbar {
                // SwiftUI toolbar items alongside web content
            }
    }
}
```

`WebPage` is `@Observable` (macOS 26). Key properties: `title`, `url`, `isLoading`, `estimatedProgress`, `themeColor`, `mediaType`, `canGoBack`, `canGoForward`, `hasOnlySecureContent`.

### JavaScript Execution

```swift
let result = await page.callJavaScript("document.title")
```

For complex exchanges, use a `WebPage.Configuration` with `urlSchemeHandlers` for custom schemes.

> When to use `WebView` vs `NSViewRepresentable`: prefer `WebView` for new macOS 26-only apps. Fall back to `NSViewRepresentable` if you need `WKNavigationDelegate` fine-grained control, `WKUIDelegate` panel customization, or backward compatibility below macOS 26.

## WKWebView as NSViewRepresentable

Wrap `WKWebView` for pre-26 macOS, or when you need direct access to delegates, `WKWebViewConfiguration`, or download delegates.

### Basic Wrapper

```swift
import SwiftUI
import WebKit

struct WebViewRepresentable: NSViewRepresentable {
    let url: URL
    @Binding var isLoading: Bool
    @Binding var error: Error?

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    func makeNSView(context: Context) -> WKWebView {
        let config = WKWebViewConfiguration()
        config.preferences.isFraudulentWebsiteWarningEnabled = true
        config.defaultWebpagePreferences.allowsContentJavaScript = true

        let webView = WKWebView(frame: .zero, configuration: config)
        webView.navigationDelegate = context.coordinator
        webView.uiDelegate = context.coordinator
        webView.isInspectable = true // Safari Web Inspector

        context.coordinator.webView = webView
        return webView
    }

    func updateNSView(_ webView: WKWebView, context: Context) {
        if webView.url != url {
            webView.load(URLRequest(url: url))
        }
    }

    @MainActor
    final class Coordinator: NSObject {
        var parent: WebViewRepresentable
        weak var webView: WKWebView?

        init(_ parent: WebViewRepresentable) {
            self.parent = parent
        }
    }
}
```

### Sizing

`WKWebView` wraps its content height by default. For auto-sizing, observe content size via KVO on `scrollView.contentView.documentView.frame`:

```swift
// In Coordinator
var sizeObserver: NSKeyValueObserving?

func startObservingSize() {
    sizeObserver = webView?.scrollView.contentView.documentView?
        .observe(\.frame, options: [.new]) { [weak self] view, change in
            guard let self, let height = change.newValue?.height else { return }
            DispatchQueue.main.async {
                self.parent.contentHeight = height
            }
        }
}
```

Set `frame` in `updateNSView` when you need a fixed height:

```swift
func updateNSView(_ webView: WKWebView, context: Context) {
    if webView.url != context.coordinator.lastURL {
        context.coordinator.lastURL = webView.url
    }
}
```

### Navigation Delegate Binding

```swift
@Binding var canGoBack: Bool
@Binding var canGoForward: Bool

// In Coordinator's didFinish:
func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
    parent.isLoading = false
    parent.canGoBack = webView.canGoBack
    parent.canGoForward = webView.canGoForward
}
```

## WKNavigationDelegate

`WKNavigationDelegate` controls navigation policy and tracks loading lifecycle.

### Deciding Navigation Policy

```swift
func webView(
    _ webView: WKWebView,
    decidePolicyFor navigationAction: WKNavigationAction,
    decisionHandler: @escaping (WKNavigationActionPolicy) -> Void
) {
    guard let url = navigationAction.request.url else {
        decisionHandler(.cancel); return
    }

    // Block non-HTTPS
    if url.scheme != "https" && url.scheme != "about" {
        decisionHandler(.cancel)
        // Optionally open in system browser
        NSWorkspace.shared.open(url)
        return
    }

    // Allow
    decisionHandler(.allow)
}
```

### Loading Lifecycle

```swift
func webView(_ webView: WKWebView, didStartProvisionalNavigation navigation: WKNavigation!) {
    parent.isLoading = true
    parent.error = nil
}

func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
    parent.isLoading = false
    parent.title = webView.title
    parent.canGoBack = webView.canGoBack
    parent.canGoForward = webView.canGoForward
}

func webView(_ webView: WKWebView, didFail navigation: WKNavigation!, withError error: Error) {
    parent.isLoading = false
    parent.error = error
}

func webView(_ webView: WKWebView, didFailProvisionalNavigation navigation: WKNavigation!, withError error: Error) {
    parent.isLoading = false
    parent.error = error
}
```

### HTTPS Enforcement

```swift
func webView(
    _ webView: WKWebView,
    didReceive challenge: URLAuthenticationChallenge,
    completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
) {
    // For known pinned certificates, validate here.
    // Otherwise, default handling:
    guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
          let trust = challenge.protectionSpace.serverTrust else {
        completionHandler(.performDefaultHandling, nil)
        return
    }

    let credential = URLCredential(trust: trust)
    completionHandler(.useCredential, credential)
}
```

## WKUIDelegate

`WKUIDelegate` presents native UI for JavaScript dialogs and window open requests.

### JavaScript Alerts, Confirms, Prompts

Use `NSAlert` on macOS (not `UIAlertController` which is iOS-only):

```swift
func webView(
    _ webView: WKWebView,
    runJavaScriptAlertPanelWithMessage message: String,
    initiatedByFrame frame: WKFrameInfo,
    completionHandler: @escaping () -> Void
) {
    let alert = NSAlert()
    alert.messageText = frame.request.url?.host ?? "Alert"
    alert.informativeText = message
    alert.addButton(withTitle: "OK")
    alert.runModal()
    completionHandler()
}

func webView(
    _ webView: WKWebView,
    runJavaScriptConfirmPanelWithMessage message: String,
    initiatedByFrame frame: WKFrameInfo,
    completionHandler: @escaping (Bool) -> Void
) {
    let alert = NSAlert()
    alert.messageText = frame.request.url?.host ?? "Confirm"
    alert.informativeText = message
    alert.addButton(withTitle: "OK")
    alert.addButton(withTitle: "Cancel")
    completionHandler(alert.runModal() == .alertFirstButtonReturn)
}

func webView(
    _ webView: WKWebView,
    runJavaScriptTextInputPanelWithPrompt prompt: String,
    defaultText: String?,
    initiatedByFrame frame: WKFrameInfo,
    completionHandler: @escaping (String?) -> Void
) {
    let alert = NSAlert()
    alert.messageText = frame.request.url?.host ?? "Input"
    alert.informativeText = prompt
    alert.addButton(withTitle: "OK")
    alert.addButton(withTitle: "Cancel")

    let input = NSTextField(frame: NSRect(x: 0, y: 0, width: 300, height: 24))
    input.stringValue = defaultText ?? ""
    alert.accessoryView = input

    if alert.runModal() == .alertFirstButtonReturn {
        completionHandler(input.stringValue)
    } else {
        completionHandler(nil)
    }
}
```

### Window.open Handling

```swift
func webView(
    _ webView: WKWebView,
    createWebViewWith configuration: WKWebViewConfiguration,
    for navigationAction: WKFrameInfo,
    windowFeatures: WKWindowFeatures
) -> WKWebView? {
    // Open in same view
    if let url = navigationAction.request.url {
        parent.externalURL = url // Set in SwiftUI binding
    }
    return nil // Return a new WKWebView if you want multi-window
}
```

## JavaScript Bridge

Two approaches: bidirectional `WKScriptMessageHandler` and async `callAsyncJavaScript`.

### WKScriptMessageHandler (Bidirectional)

**Registration** — register in `makeNSView`, add to user content controller:

```swift
func makeNSView(context: Context) -> WKWebView {
    let config = WKWebViewConfiguration()
    let contentController = WKUserContentController()
    contentController.add(context.coordinator, name: "nativeBridge")
    config.userContentController = contentController

    let webView = WKWebView(frame: .zero, configuration: config)
    webView.navigationDelegate = context.coordinator
    return webView
}
```

**Handling messages** — Coordinator conforms to `WKScriptMessageHandler`:

```swift
extension Coordinator: WKScriptMessageHandler {
    func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
        guard message.name == "nativeBridge",
              let body = message.body as? [String: Any],
              let action = body["action"] as? String else { return }

        switch action {
        case "log":
            print("[JS]", body["message"] ?? "")
        case "getData":
            Task { @MainActor in
                let result = await fetchData(messageId: body["id"] as? String)
                let js = "window.__resolveBridge('\(body["id"] ?? "")', \(result))"
                self.webView?.evaluateJavaScript(js, completionHandler: nil)
            }
        default:
            break
        }
    }
}
```

**Cleanup** — remove handler on dismantle:

```swift
static func dismantleNSView(_ webView: WKWebView, coordinator: Coordinator) {
    webView.configuration.userContentController.removeScriptMessageHandler(forName: "nativeBridge")
}
```

**JavaScript side**:

```javascript
window.webkit.messageHandlers.nativeBridge.postMessage({
    action: "log",
    message: "Hello from JS"
});

// With callback
const id = crypto.randomUUID();
window.__callbacks[id] = (result) => console.log(result);
window.webkit.messageHandlers.nativeBridge.postMessage({
    action: "getData",
    id: id
});
```

### callAsyncJavaScript (macOS 11+)

For simple Swift→JS calls with return values:

```swift
// In Coordinator or from SwiftUI via binding method
do {
    let result = try await webView.callAsyncJavaScript(
        "document.getElementById('content').innerText", 
        arguments: [:], 
        in: nil, 
        contentWorld: .page
    )
    print(result) // Any? decoded from JS
} catch {
    print("JS error:", error)
}
```

Arguments are passed as `[String: Any]` and accessed in JS as named parameters:

```swift
let result = try await webView.callAsyncJavaScript(
    "return $0 + $1",
    arguments: ["$0": 10, "$1": 20],
    in: nil,
    contentWorld: .page
) // Result: 30
```

### Type-Safe Bridge Pattern

```swift
enum BridgeAction: String, Codable {
    case submitForm = "submitForm"
    case fetchData = "fetchData"
    case log = "log"
}

struct InboundMessage: Codable {
    let action: String
    let payload: [String: String]?
    let callbackId: String?
}

struct OutboundMessage: Codable {
    let callbackId: String
    let result: [String: String]?
    let error: String?
}
```

Encode/decode messages via `JSONDecoder` in the coordinator's `userContentController(_:didReceive:)`.

## Custom URL Schemes

Serve bundled assets via a custom scheme instead of HTTP.

### WKURLSchemeHandler

```swift
@MainActor
final class BundleSchemeHandler: NSObject, WKURLSchemeHandler {
    func webView(_ webView: WKWebView, start urlSchemeTask: WKURLSchemeTask) {
        guard let url = urlSchemeTask.request.url else {
            urlSchemeTask.didFailWithError(URLError(.badURL))
            return
        }

        // Map custom scheme URL to bundle resource
        let path = url.path // e.g., "/images/logo.png"
        guard let data = NSDataAsset(name: String(path.dropFirst()))?.data else {
            urlSchemeTask.didFailWithError(URLError(.resourceUnavailable))
            return
        }

        let response = HTTPURLResponse(
            url: url, statusCode: 200, httpVersion: nil,
            headerFields: ["Content-Type": mimeType(for: path)]
        )!
        urlSchemeTask.didReceive(response)
        urlSchemeTask.didReceive(data)
        urlSchemeTask.didFinish()
    }

    func webView(_ webView: WKWebView, stop urlSchemeTask: WKURLSchemeTask) {
        // Cancel in-flight work if needed
    }

    private func mimeType(for path: String) -> String {
        switch (path as NSString).pathExtension {
        case "png": return "image/png"
        case "jpg", "jpeg": return "image/jpeg"
        case "css": return "text/css"
        case "js": return "application/javascript"
        case "html": return "text/html"
        default: return "application/octet-stream"
        }
    }
}
```

### Registration

```swift
let config = WKWebViewConfiguration()
config.setURLSchemeHandler(BundleSchemeHandler(), forURLScheme: "myapp")
```

> **Cannot** register `http`, `https`, `file`, or other schemes WebKit already handles. Check with `WKWebView.handlesURLScheme(_:)` first.

### SwiftUI WebPage (macOS 26)

```swift
var configuration = WebPage.Configuration()
configuration.urlSchemeHandlers.append(MySchemeHandler())
let page = WebPage(configuration: configuration)
```

## Download Handling

### WKDownloadDelegate

```swift
extension Coordinator: WKDownloadDelegate {
    func webView(
        _ webView: WKWebView,
        decidePolicyFor navigationAction: WKNavigationAction,
        decisionHandler: @escaping (WKNavigationActionPolicy) -> Void
    ) {
        // Detect download-triggering navigations
        if navigationAction.request.url?.pathExtension == "pdf" {
            decisionHandler(.download)
        } else {
            decisionHandler(.allow)
        }
    }

    func webView(
        _ webView: WKWebView,
        navigationAction: WKNavigationAction,
        didBecome download: WKDownload
    ) {
        download.delegate = self
    }

    func download(
        _ download: WKDownload,
        decideDestinationUsing response: URLResponse,
        suggestedFilename: String,
        completionHandler: @escaping (URL?) -> Void
    ) {
        let panel = NSSavePanel()
        panel.nameFieldStringValue = suggestedFilename
        panel.begin { result in
            completionHandler(result == .OK ? panel.url : nil)
        }
    }

    func downloadDidFinish(_ download: WKDownload) {
        // Show notification, update UI
    }

    func download(_ download: WKDownload, didFailWithError error: Error, resumeData: Data?) {
        parent.error = error
    }
}
```

## PDF Rendering

### createPDF (macOS 11+)

```swift
func exportPDF(from webView: WKWebView) async throws -> Data {
    let config = WKPDFConfiguration()
    config.rect = CGRect(x: 0, y: 0, width: 595.28, height: 841.89) // A4 in points
    config.allowTransparentBackground = false

    return try await withCheckedThrowingContinuation { continuation in
        webView.createPDF(configuration: config) { result in
            continuation.resume(with: result)
        }
    }
}
```

### Usage Pattern

```swift
// In SwiftUI view
.task {
    // Wait for page to load
    while webView.isLoading { try await Task.sleep(for: .milliseconds(100)) }
    let pdfData = try await exportPDF(from: webView)
    // Save or display
    let panel = NSSavePanel()
    panel.nameFieldStringValue = "export.pdf"
    if panel.runModal() == .OK, let url = panel.url {
        try pdfData.write(to: url)
    }
}
```

### macOS 26 WebPage

```swift
let transferable = page.pdf(allowTransparentBackground: false)
// Use with .draggable or ShareLink
```

## WKWebViewConfiguration

Central configuration object for all WebKit behavior:

```swift
let config = WKWebViewConfiguration()

// Preferences
config.preferences.isFraudulentWebsiteWarningEnabled = true
config.preferences.minimumFontSize = 12
config.preferences.javaScriptCanOpenWindowsAutomatically = false

// Page-level defaults
config.defaultWebpagePreferences.allowsContentJavaScript = true
config.defaultWebpagePreferences.preferredContentMode = .desktop // macOS 11+

// Media
config.mediaTypesRequiringUserActionForPlayback = [.video, .audio]
// .all, .none, or specific types

// User content controller (for JS bridge)
let contentController = WKUserUserContentController()
contentController.add(coordinator, name: "bridge")
config.userContentController = contentController

// Custom schemes
config.setURLSchemeHandler(BundleSchemeHandler(), forURLScheme: "myapp")

// Process pool (share state across web views)
config.processPool = WKProcessPool.shared // Define a shared instance
```

## Constraints & Gotchas

| Constraint | Detail |
|---|---|
| **App Sandbox** | Enable `com.apple.security.network.client` for outgoing network. Enable `com.apple.security.files.user-selected.read-write` for download save panels. |
| **Message handler memory** | `add(_:name:)` retains the handler. Always call `removeScriptMessageHandler(forName:)` in `dismantleNSView` or `deinit`. Leaks the entire web view otherwise. |
| **Blank view on tab switch** | `WKWebView` inside `NSTabView` can go blank on re-selection. Workaround: recreate the web view or use `isHidden` instead of add/remove. |
| **evaluateJavaScript** | Must run on main thread. Use `DispatchQueue.main.async` if calling from background. |
| **Custom scheme conflicts** | Cannot override `http`, `https`, `file`. Raises an exception at runtime. |
| **HTTPS by default** | WebKit requires TLS 1.2+. Self-signed certs require explicit trust handling in `didReceive challenge`. |
| **JavaScript disabled** | `allowsContentJavaScript` defaults to true on macOS. Set via `defaultWebpagePreferences`. |
| **Process isolation** | Each `WKProcessPool` is isolated. Share a pool across web views to share cookies/cache. |
| **isInspectable** | Only for debug builds. App Store rejects apps with `isInspectable = true` in release. |
| **WKDownloadDelegate** | Must set `decisionHandler(.download)` in `decidePolicyFor` AND implement `navigationAction:didBecomeDownload:` to wire the delegate. |

## Common Mistakes

1. **Forgetting to remove message handler.** Causes retain cycles and web view leaks. Always pair `add` with `removeScriptMessageHandler(forName:)`.
2. **Setting delegate in `updateNSView`.** Causes redundant assignment on every state change. Set delegates in `makeNSView` only.
3. **Not calling `completionHandler`** in `WKUIDelegate` dialog methods. JavaScript execution hangs indefinitely.
4. **Calling `evaluateJavaScript` from background thread.** Crashes or undefined behavior. Always dispatch to main.
5. **Using `UIAlertController` on macOS.** Use `NSAlert` instead. `UIAlertController` is UIKit/iOS only.
6. **Not waiting for `didFinish` before PDF export.** Results in empty or partial PDF. Check `isLoading == false`.
7. **Registering `http`/`https` as custom schemes.** Runtime exception. Use `WKWebView.handlesURLScheme(_:)` to check first.
8. **Ignoring `stop` in `WKURLSchemeHandler`.** If the task is cancelled and you continue loading, you waste resources and may crash accessing a deallocated task.

## Review Checklist

- [ ] Message handlers removed in `dismantleNSView` or `deinit`
- [ ] Delegates set in `makeNSView`, not `updateNSView`
- [ ] All `WKUIDelegate` completion handlers called
- [ ] `evaluateJavaScript` called on main thread
- [ ] PDF export waits for `didFinish` / `isLoading == false`
- [ ] Custom URL scheme doesn't conflict with built-in schemes
- [ ] Sandbox entitlements include network client + file access
- [ ] `isInspectable` gated behind `#if DEBUG`
- [ ] `WKDownloadDelegate` wired via both `decidePolicyFor` and `didBecomeDownload`
- [ ] `NSAlert` used (not `UIAlertController`) for JS dialogs on macOS

## References

- [references/webkit-patterns.md](references/webkit-patterns.md) — Complete NSViewRepresentable recipe, JS bridge with type-safe protocol, custom URL scheme handler, PDF export, download with save panel
- [WKWebView | Apple Developer Documentation](https://developer.apple.com/documentation/webkit/wkwebview)
- [WKNavigationDelegate | Apple Developer Documentation](https://developer.apple.com/documentation/webkit/wknavigationdelegate)
- [WKUIDelegate | Apple Developer Documentation](https://developer.apple.com/documentation/webkit/wkuidelegate)
- [WKScriptMessageHandler | Apple Developer Documentation](https://developer.apple.com/documentation/webkit/wkscriptmessagehandler)
- [WKDownloadDelegate | Apple Developer Documentation](https://developer.apple.com/documentation/webkit/wkdownloaddelegate)
- [WKURLSchemeHandler | Apple Developer Documentation](https://developer.apple.com/documentation/webkit/wkurlschemehandler)
- [WKPDFConfiguration | Apple Developer Documentation](https://developer.apple.com/documentation/webkit/wkpdfconfiguration)
- [Meet WebKit for SwiftUI - WWDC25 Session 231](https://developer.apple.com/videos/play/wwdc2025/231/)
