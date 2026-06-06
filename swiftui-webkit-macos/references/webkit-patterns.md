# WebKit Patterns

Complete working patterns for common WebKit integration scenarios on macOS. Each pattern includes the full implementation with Coordinator, delegate methods, and SwiftUI usage.
These patterns are designed for macOS 26+ with Swift 6.3, using NSViewRepresentable for WKWebView integration. Each pattern is self-contained and can be copied into a project directly. All code assumes a SwiftUI app structure with MainActor isolation where required by WebKit threading model.

The five patterns progress from basic to advanced: start with the complete NSViewRepresentable wrapper to get a working web view, then layer on the JavaScript bridge for two-way communication, add custom URL schemes for bundled resource serving, and finally implement PDF export and download handling for file transfer scenarios.

Key architectural decisions across all patterns: the Coordinator holds every delegate reference as a reference type that outlives the representable struct; weak web view references prevent retain cycles between the view and its delegate; MainActor isolation satisfies Swift 6.3 compile-time concurrency checks for WebKit APIs; and dismantleNSView cleans up message handlers and observers to prevent leaks.

---

## Contents

- [1. Complete WKWebView NSViewRepresentable](#1-complete-wkwebview-nsviewrepresentable)
- [2. JavaScript Bridge with Type-Safe Protocol](#2-javascript-bridge-with-type-safe-protocol)
- [3. Custom URL Scheme Handler](#3-custom-url-scheme-handler)
- [4. PDF Export Pattern](#4-pdf-export-pattern)
- [5. Download with Save Panel](#5-download-with-save-panel)

---

## 1. Complete WKWebView NSViewRepresentable

Full wrapper with navigation delegate, loading state binding, error handling, and back/forward support.

```swift
import SwiftUI
import WebKit

struct WebView: NSViewRepresentable {
    @Binding var url: URL
    @Binding var isLoading: Bool
    @Binding var error: Error?
    @Binding var canGoBack: Bool
    @Binding var canGoForward: Bool
    @Binding var pageTitle: String

    func makeCoordinator() -> Coordinator { Coordinator(self) }

    func makeNSView(context: Context) -> WKWebView {
        let config = WKWebViewConfiguration()
        config.preferences.isFraudulentWebsiteWarningEnabled = true
        config.defaultWebpagePreferences.allowsContentJavaScript = true
        config.mediaTypesRequiringUserActionForPlayback = [.video]

        let webView = WKWebView(frame: .zero, configuration: config)
        webView.navigationDelegate = context.coordinator
        webView.uiDelegate = context.coordinator
        webView.allowsBackForwardNavigationGestures = true
        #if DEBUG
        webView.isInspectable = true
        #endif

        context.coordinator.webView = webView
        return webView
    }

    func updateNSView(_ webView: WKWebView, context: Context) {
        if webView.url != url {
            webView.load(URLRequest(url: url))
        }
    }

    static func dismantleNSView(_ webView: WKWebView, coordinator: Coordinator) {
        coordinator.cancellables.removeAll()
    }

    @MainActor
    final class Coordinator: NSObject {
        var parent: WebView
        weak var webView: WKWebView?
        var cancellables: Set<AnyCancellable> = []

        init(_ parent: WebView) { self.parent = parent }
    }
}

// MARK: - WKNavigationDelegate

extension WebView.Coordinator: WKNavigationDelegate {
    func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping (WKNavigationActionPolicy) -> Void) {
        guard let url = navigationAction.request.url else {
            decisionHandler(.cancel); return
        }
        // Allow https and about:blank only
        if url.scheme == "https" || url.scheme == "about" {
            decisionHandler(.allow)
        } else if url.scheme == "http" {
            // Upgrade to https
            var components = URLComponents(url: url, resolvingAgainstBaseURL: false)
            components?.scheme = "https"
            if let upgraded = components?.url {
                parent.url = upgraded
            }
            decisionHandler(.cancel)
        } else {
            // Open mailto:, tel:, etc. in system handler
            NSWorkspace.shared.open(url)
            decisionHandler(.cancel)
        }
    }

    func webView(_ webView: WKWebView, didStartProvisionalNavigation navigation: WKNavigation!) {
        parent.isLoading = true
        parent.error = nil
    }

    func webView(_ webView: WKWebView, didFinish navigation: WKNavigation!) {
        parent.isLoading = false
        parent.pageTitle = webView.title ?? ""
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
}

// MARK: - WKUIDelegate

extension WebView.Coordinator: WKUIDelegate {
    func webView(_ webView: WKWebView, runJavaScriptAlertPanelWithMessage message: String, initiatedByFrame frame: WKFrameInfo, completionHandler: @escaping () -> Void) {
        let alert = NSAlert()
        alert.messageText = frame.request.url?.host ?? "Alert"
        alert.informativeText = message
        alert.addButton(withTitle: "OK")
        alert.runModal()
        completionHandler()
    }

    func webView(_ webView: WKWebView, runJavaScriptConfirmPanelWithMessage message: String, initiatedByFrame frame: WKFrameInfo, completionHandler: @escaping (Bool) -> Void) {
        let alert = NSAlert()
        alert.messageText = frame.request.url?.host ?? "Confirm"
        alert.informativeText = message
        alert.addButton(withTitle: "OK")
        alert.addButton(withTitle: "Cancel")
        completionHandler(alert.runModal() == .alertFirstButtonReturn)
    }
}
```

### Usage

```swift
struct BrowserView: View {
    @State private var url = URL(string: "https://apple.com")!
    @State private var isLoading = false
    @State private var error: Error?
    @State private var canGoBack = false
    @State private var canGoForward = false
    @State private var pageTitle = ""

    var body: some View {
        VStack(spacing: 0) {
            // Toolbar
            HStack {
                Button(action: { /* goBack via binding */ }) {
                    Image(systemName: "chevron.left")
                }
                .disabled(!canGoBack)

                Button(action: { /* goForward */ }) {
                    Image(systemName: "chevron.right")
                }
                .disabled(!canGoForward)

                if isLoading {
                    ProgressView()
                        .controlSize(.small)
                        .padding(.horizontal, 4)
                }

                Text(pageTitle)
                    .lineLimit(1)
                    .truncationMode(.middle)
            }
            .padding(8)

            // WebView
            WebView(
                url: $url,
                isLoading: $isLoading,
                error: $error,
                canGoBack: $canGoBack,
                canGoForward: $canGoForward,
                pageTitle: $pageTitle
            )
        }
    }
}
```

---

## 2. JavaScript Bridge with Type-Safe Message Protocol

A structured bridge using `WKScriptMessageHandler` with JSON-encoded messages and callback support.

### Protocol Definition

```swift
import Foundation

// Messages from JS → Swift
struct JSInboundMessage: Codable {
    let action: String
    let payload: [String: String]?
    let callbackId: String?
}

// Messages from Swift → JS
struct JSOutboundMessage: Codable {
    let callbackId: String
    let result: [String: String]?
    let error: String?
}

// Supported actions
enum BridgeAction: String {
    case getTheme = "getTheme"
    case submitForm = "submitForm"
    case log = "log"
}
```

### Coordinator Bridge

```swift
extension WebView.Coordinator: WKScriptMessageHandler {
    func userContentController(_ userContentController: WKUserContentController, didReceive message: WKScriptMessage) {
        guard message.name == "bridge" else { return }

        do {
            let data = try JSONSerialization.data(withJSONObject: message.body)
            let inbound = try JSONDecoder().decode(JSInboundMessage.self, from: data)

            switch inbound.action {
            case BridgeAction.getTheme.rawValue:
                handleGetTheme(callbackId: inbound.callbackId)

            case BridgeAction.submitForm.rawValue:
                if let payload = inbound.payload {
                    handleSubmitForm(payload: payload, callbackId: inbound.callbackId)
                }

            case BridgeAction.log.rawValue:
                if let msg = inbound.payload?["message"] {
                    print("[JS Log]", msg)
                }

            default:
                sendToJS(callbackId: inbound.callbackId, error: "Unknown action: \(inbound.action)")
            }
        } catch {
            print("Bridge decode error:", error)
        }
    }

    private func handleGetTheme(callbackId: String?) {
        let result = ["theme": NSApp.effectiveAppearance.bestMatch(from: [.darkAqua, .aqua]) == .darkAqua ? "dark" : "light"]
        sendToJS(callbackId: callbackId, result: result)
    }

    private func handleSubmitForm(payload: [String: String], callbackId: String?) {
        // Process form data
        print("Form submitted:", payload)
        sendToJS(callbackId: callbackId, result: ["status": "ok"])
    }

    private func sendToJS(callbackId: String?, result: [String: String]? = nil, error: String? = nil) {
        guard let callbackId, let webView else { return }
        let outbound = JSOutboundMessage(callbackId: callbackId, result: result, error: error)
        guard let data = try? JSONEncoder().encode(outbound),
              let json = String(data: data, encoding: .utf8) else { return }

        let js = "window.__bridge.resolve(\(json))"
        Task { @MainActor in
            webView.evaluateJavaScript(js, completionHandler: nil)
        }
    }
}
```

### JavaScript Bridge Client

```javascript
// bridge.js — include in your web content
window.__bridge = {
    callbacks: {},
    nextId() { return crypto.randomUUID(); },

    call(action, payload) {
        return new Promise((resolve, reject) => {
            const id = this.nextId();
            this.callbacks[id] = { resolve, reject };
            window.webkit.messageHandlers.bridge.postMessage({
                action: action,
                payload: payload || {},
                callbackId: id
            });
        });
    },

    resolve(message) {
        const { callbackId, result, error } = message;
        const cb = this.callbacks[callbackId];
        if (!cb) return;
        if (error) cb.reject(new Error(error));
        else cb.resolve(result);
        delete this.callbacks[callbackId];
    }
};

// Usage from web page:
// const { theme } = await window.__bridge.call('getTheme');
// await window.__bridge.call('submitForm', { name: 'Alice', email: 'a@b.com' });
```

### Registration in makeNSView

```swift
func makeNSView(context: Context) -> WKWebView {
    let config = WKWebViewConfiguration()
    let contentController = WKUserContentController()
    contentController.add(context.coordinator, name: "bridge")
    config.userContentController = contentController

    let webView = WKWebView(frame: .zero, configuration: config)
    webView.navigationDelegate = context.coordinator
    webView.uiDelegate = context.coordinator
    context.coordinator.webView = webView
    return webView
}
```

### Cleanup

```swift
static func dismantleNSView(_ webView: WKWebView, coordinator: Coordinator) {
    webView.configuration.userContentController.removeScriptMessageHandler(forName: "bridge")
    coordinator.cancellables.removeAll()
}
```

---

## 3. Custom URL Scheme Handler

Serve bundled assets via a custom `myapp://` scheme.

```swift
import WebKit

@MainActor
final class AppSchemeHandler: NSObject, WKURLSchemeHandler {
    func webView(_ webView: WKWebView, start urlSchemeTask: any WKURLSchemeTask) {
        guard let url = urlSchemeTask.request.url else {
            urlSchemeTask.didFailWithError(URLError(.badURL))
            return
        }

        // Map URL path to bundle resource
        // e.g., myapp://assets/images/logo.png → Assets.xcassets or bundle resource
        let resourcePath = url.path // "/assets/images/logo.png"
        let resourceName = String(resourcePath.dropFirst()) // "assets/images/logo.png"
        let fileExtension = (resourceName as NSString).pathExtension
        let baseName = (resourceName as NSString).deletingPathExtension

        guard let resourceURL = Bundle.main.url(forResource: baseName, withExtension: fileExtension) else {
            urlSchemeTask.didFailWithError(URLError(.resourceUnavailable))
            return
        }

        do {
            let data = try Data(contentsOf: resourceURL)
            let mimeType = Self.mimeType(for: fileExtension)
            let response = HTTPURLResponse(
                url: url,
                statusCode: 200,
                httpVersion: "HTTP/1.1",
                headerFields: [
                    "Content-Type": mimeType,
                    "Content-Length": "\(data.count)"
                ]
            )!
            urlSchemeTask.didReceive(response)
            urlSchemeTask.didReceive(data)
            urlSchemeTask.didFinish()
        } catch {
            urlSchemeTask.didFailWithError(error)
        }
    }

    func webView(_ webView: WKWebView, stop urlSchemeTask: any WKURLSchemeTask) {
        // Cancel any in-flight work for this task
    }

    private static func mimeType(for ext: String) -> String {
        switch ext.lowercased() {
        case "html", "htm": return "text/html"
        case "css":         return "text/css"
        case "js":          return "application/javascript"
        case "json":        return "application/json"
        case "png":         return "image/png"
        case "jpg", "jpeg": return "image/jpeg"
        case "svg":         return "image/svg+xml"
        case "gif":         return "image/gif"
        case "woff2":       return "font/woff2"
        case "woff":        return "font/woff"
        default:            return "application/octet-stream"
        }
    }
}
```

### Registration

```swift
let config = WKWebViewConfiguration()
config.setURLSchemeHandler(AppSchemeHandler(), forURLScheme: "myapp")
```

### HTML Usage

```html
<!-- In your bundled HTML content -->
<img src="myapp://assets/images/logo.png" />
<link rel="stylesheet" href="myapp://assets/css/style.css" />
<script src="myapp://assets/js/app.js"></script>
```

> **Important:** Do not register schemes that WebKit already handles (`http`, `https`, `file`, `data`, `blob`, `about`). Check with `WKWebView.handlesURLScheme(_:)` before registering.

---

## 4. PDF Export Pattern

Export web content as PDF data, then save via `NSSavePanel`.

```swift
import WebKit
import PDFKit

struct PDFExporter {
    let webView: WKWebView

    func exportPDF(pageSize: PDFPageSize = .a4) async throws -> Data {
        // Wait for content to fully load
        while webView.isLoading {
            try await Task.sleep(for: .milliseconds(100))
        }

        // Additional wait for dynamic content
        _ = try await webView.callAsyncJavaScript(
            "document.readyState === 'complete'", [:], in: nil, contentWorld: .page
        )

        let config = WKPDFConfiguration()
        let size = pageSize.pointSize
        config.rect = CGRect(x: 0, y: 0, width: size.width, height: size.height)
        config.allowTransparentBackground = false

        return try await withCheckedThrowingContinuation { continuation in
            webView.createPDF(configuration: config) { result in
                continuation.resume(with: result)
            }
        }
    }

    func exportAndSave(pageSize: PDFPageSize = a4) async throws {
        let data = try await exportPDF(pageSize: pageSize)

        await MainActor.run {
            let panel = NSSavePanel()
            panel.nameFieldStringValue = "export.pdf"
            panel.allowedContentTypes = [.pdf]
            guard panel.runModal() == .OK, let url = panel.url else { return }
            try? data.write(to: url)
        }
    }
}

enum PDFPageSize {
    case a4, letter, legal
    case custom(width: CGFloat, height: CGFloat)

    var pointSize: CGSize {
        switch self {
        case .a4:     return CGSize(width: 595.28, height: 841.89)
        case .letter: return CGSize(width: 612, height: 792)
        case .legal:  return CGSize(width: 612, height: 1008)
        case .custom(let w, let h): return CGSize(width: w, height: h)
        }
    }
}
```

### SwiftUI Integration

```swift
struct ExportableWebView: View {
    @State private var webView: WKWebView?
    @State private var isExporting = false

    var body: some View {
        WebView(url: .constant(URL(string: "https://example.com")!),
                isLoading: .constant(false),
                error: .constant(nil),
                canGoBack: .constant(false),
                canGoForward: .constant(false),
                pageTitle: .constant(""))
        .overlay(alignment: .topTrailing) {
            Button("Export PDF") {
                isExporting = true
            }
            .disabled(isExporting)
        }
        .task(id: isExporting) {
            guard isExporting, let webView else { return }
            let exporter = PDFExporter(webView: webView)
            try? await exporter.exportAndSave()
            isExporting = false
        }
    }
}
```

---

## 5. Download with Save Panel

Handle file downloads from web content using `WKDownloadDelegate`.

```swift
extension WebView.Coordinator: WKNavigationDelegate {
    // Detect download-triggering navigations
    func webView(
        _ webView: WKWebView,
        decidePolicyFor navigationAction: WKNavigationAction,
        decisionHandler: @escaping (WKNavigationActionPolicy) -> Void
    ) {
        // Check for download attribute or known download extensions
        let downloadExtensions = ["pdf", "zip", "dmg", "pkg", "docx", "xlsx", "pptx"]
        if let ext = navigationAction.request.url?.pathExtension.lowercased(),
           downloadExtensions.contains(ext) {
            decisionHandler(.download)
            return
        }
        decisionHandler(.allow)
    }

    // Wire the download delegate when a navigation becomes a download
    func webView(
        _ webView: WKWebView,
        navigationAction: WKNavigationAction,
        didBecome download: WKDownload
    ) {
        download.delegate = self
    }

    // Also handle response-level downloads (Content-Disposition: attachment)
    func webView(
        _ webView: WKWebView,
        navigationResponse: WKNavigationResponse,
        didBecome download: WKDownload
    ) {
        download.delegate = self
    }
}

// MARK: - WKDownloadDelegate

extension WebView.Coordinator: WKDownloadDelegate {
    func download(
        _ download: WKDownload,
        decideDestinationUsing response: URLResponse,
        suggestedFilename: String,
        completionHandler: @escaping (URL?) -> Void
    ) {
        let panel = NSSavePanel()
        panel.nameFieldStringValue = suggestedFilename
        panel.canCreateDirectories = true
        panel.isExtensionHidden = false

        // Determine allowed types from MIME type
        if response.mimeType == "application/pdf" {
            panel.allowedContentTypes = [.pdf]
        }

        guard panel.runModal() == .OK else {
            download.cancel()
            completionHandler(nil)
            return
        }
        completionHandler(panel.url)
    }

    func downloadDidFinish(_ download: WKDownload) {
        // Optionally show a notification
        let notification = UNMutableNotificationContent()
        notification.title = "Download Complete"
        notification.body = "File saved successfully"
        let request = UNNotificationRequest(identifier: UUID().uuidString, content: notification, trigger: nil)
        UNUserNotificationCenter.current().add(request)
    }

    func download(_ download: WKDownload, didFailWithError error: Error, resumeData: Data?) {
        Task { @MainActor in
            parent.error = error
        }
    }

    func download(
        _ download: WKDownload,
        willPerformHTTPRedirection response: HTTPURLResponse,
        newRequest request: URLRequest,
        decisionHandler: @escaping (WKDownload.RedirectPolicy) -> Void
    ) {
        decisionHandler(.allow)
    }

    func download(
        _ download: WKDownload,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard challenge.protectionSpace.authenticationMethod == NSURLAuthenticationMethodServerTrust,
              let trust = challenge.protectionSpace.serverTrust else {
            completionHandler(.performDefaultHandling, nil)
            return
        }
        completionHandler(.useCredential, URLCredential(trust: trust))
    }
}
```

### Required Entitlements

For downloads to work in a sandboxed app, add to your `.entitlements`:

```xml
<key>com.apple.security.network.client</key>
<true/>
<key>com.apple.security.files.user-selected.read-write</key>
<true/>
```

---

## Summary Table

| Pattern | Key Protocol | macOS Version | Coordinator Role |
|---|---|---|---|
| NSViewRepresentable | `WKNavigationDelegate`, `WKUIDelegate` | 10.10+ | Delegate + state bridge |
| JS Bridge | `WKScriptMessageHandler` | 10.10+ | Message handler + evaluator |
| Custom Scheme | `WKURLSchemeHandler` | 11.0+ | Resource loader |
| PDF Export | `createPDF(configuration:)` | 11.0+ | Async continuation |
| Download | `WKDownloadDelegate` | 11.3+ | Save panel + progress |

## Choosing Between Patterns

For a read-only web content display (help pages, documentation, marketing content), the NSViewRepresentable wrapper alone is sufficient. Add the navigation delegate if you need to intercept link clicks or enforce HTTPS.

For interactive web apps embedded in your macOS app (dashboards, forms, web-based tools), layer the JavaScript bridge on top of the wrapper. The type-safe protocol pattern prevents stringly-typed message passing and makes the bridge testable.

When your web content references local assets (images, stylesheets, fonts) that ship in your app bundle, add a custom URL scheme handler. This avoids the security restrictions of `file://` URLs in WebKit and gives you full control over resource loading.

For document generation workflows (invoices, reports, printable content), use the PDF export pattern. The key gotcha is waiting for `isLoading == false` and `document.readyState === 'complete'` before calling `createPDF`, otherwise the PDF may be blank or truncated.

For file download scenarios (user-initiated downloads from web content), implement `WKDownloadDelegate`. The critical detail is that you must return `.download` from `decidePolicyFor navigationAction` AND implement `navigationAction:didBecomeDownload:` to wire the download delegate. Missing either step silently drops the download.

## Migration to macOS 26 Native WebView

When your deployment target reaches macOS 26, you can replace the NSViewRepresentable wrapper with the native `WebView` + `WebPage` API. The JavaScript bridge patterns translate directly: `callAsyncJavaScript` becomes `page.callJavaScript()`, and `WKScriptMessageHandler` is replaced by `WebPage.Configuration.urlSchemeHandlers` for custom schemes. PDF export becomes `page.pdf()`, and download handling is managed by the `WebPage` transferable conformance. The delegate-based patterns in this file remain relevant for apps that need to support macOS 15 and earlier, or for scenarios requiring fine-grained navigation control that the native API does not expose.
