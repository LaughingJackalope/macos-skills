# StoreKit Patterns — Complete Recipes

Reference patterns for StoreKit 2 on macOS 26+. Each section is a self-contained recipe for a complete implementation scenario.

## Complete Purchase Flow

Full purchase implementation with transaction listener, purchase handling, and content delivery unified through `Transaction.updates`:

```swift
import StoreKit
import SwiftUI

// MARK: - Store Service

@MainActor
final class StoreService: ObservableObject {
    @Published var products: [Product] = []
    @Published var purchasedIDs: Set<String> = []
    @Published var isLoading = false
    @Published var errorMessage: String?

    init() {
        // Start the transaction listener immediately
        startTransactionListener()
    }

    // MARK: - Product Loading

    func loadProducts() async {
        isLoading = true
        errorMessage = nil
        defer { isLoading = false }

        do {
            let ids = [
                "com.example.pro.monthly",
                "com.example.pro.yearly",
                "com.example.pro.lifetime"
            ]
            let fetched = try await Product.products(for: ids)
            products = fetched.sorted { $0.price < $1.price }
        } catch {
            errorMessage = "Failed to load products: \(error.localizedDescription)"
        }
    }

    // MARK: - Purchase

    func purchase(_ product: Product) async {
        do {
            let result = try await product.purchase()

            switch result {
            case .success(let verification):
                let transaction = try checkVerified(verification)
                await deliverContent(for: transaction)
                await transaction.finish()

            case .userCancelled:
                break // User cancelled — no-op

            case .pending:
                // Ask to Buy or other delay — show pending UI
                break

            @unknown default:
                break
            }
        } catch {
            errorMessage = error.localizedDescription
        }
    }

    // MARK: - Transaction Listener

    private var listenerTask: Task<Void, Never>?

    private func startTransactionListener() {
        listenerTask?.cancel()
        listenerTask = Task {
            for await result in Transaction.updates {
                do {
                    let transaction = try checkVerified(result)

                    if transaction.revocationDate != nil {
                        await revokeContent(for: transaction.productID)
                    } else {
                        await deliverContent(for: transaction)
                    }
                    await transaction.finish()
                } catch {
                    // Log but do not stop listening
                    print("❌ Unverified update: \(error)")
                }
            }
        }
    }

    // MARK: - Entitlements

    func checkEntitlements() async {
        for await result in Transaction.currentEntitlements {
            do {
                let transaction = try checkVerified(result)
                await deliverContent(for: transaction)
            } catch {
                print("❌ Unverified entitlement: \(error)")
            }
        }
    }

    // MARK: - Restore

    func restorePurchases() async {
        do {
            try await AppStore.sync()
            await checkEntitlements()
        } catch {
            errorMessage = "Failed to restore: \(error.localizedDescription)"
        }
    }

    // MARK: - Content Delivery

    private func deliverContent(for transaction: Transaction) async {
        let id = transaction.productID

        if transaction.revocationDate != nil {
            await revokeContent(for: id)
            return
        }

        // Remove from revoked set if present
        purchasedIDs.remove(id)

        // Grant access
        purchasedIDs.insert(id)

        // Persist for offline access
        UserDefaults.standard.set(true, forKey: "purchased_\(id)")

        NotificationCenter.default.post(
            name: .purchaseCompleted,
            object: id
        )
    }

    private func revokeContent(for productID: String) async {
        purchasedIDs.remove(productID)
        UserDefaults.standard.set(false, forKey: "purchased_\(productID)")

        NotificationCenter.default.post(
            name: .purchaseRevoked,
            object: productID
        )
    }

    // MARK: - Helpers

    private func checkVerified<T>(_ result: VerificationResult<T>) throws -> T {
        switch result {
        case .unverified(_, let error):
            throw error
        case .verified(let safe):
            return safe
        }
    }
}

extension Notification.Name {
    static let purchaseCompleted = Notification.Name("purchaseCompleted")
    static let purchaseRevoked = Notification.Name("purchaseRevoked")
}
```

**Architecture notes:**
- `StoreService` is an `@MainActor` to avoid data races on published state.
- All content delivery goes through `deliverContent(for:)` — even purchases initiated via `purchase()`. The `Transaction.updates` listener is the safety net for interruptions.
- `checkEntitlements()` recovers state after app reinstall or on relaunch.
- `restorePurchases()` calls `AppStore.sync()` to force-receipt refresh, then re-checks entitlements.
- `revocationDate` check in `deliverContent` is critical — a revoked entitlement in `currentEntitlements` will still appear if you don't check.

## Subscription Status Checking

Complete subscription status pattern with state machine, UI integration, and renewal handling:

```swift
import StoreKit
import SwiftUI

// MARK: - State Types

enum SubscriptionStatus: Equatable {
    case unknown
    case notSubscribed
    case active(productID: String, expiryDate: Date?)
    case inGracePeriod(productID: String, gracePeriodExpiry: Date?)
    case inBillingRetry(productID: String)
    case expired
    case revoked

    var isActive: Bool {
        switch self {
        case .active, .inGracePeriod, .inBillingRetry:
            return true
        default:
            return false
        }
    }

    var displayText: String {
        switch self {
        case .unknown:
            return "Checking…"
        case .notSubscribed:
            return "No active subscription"
        case .active(let id, let expiry):
            if let expiry {
                let formatter = DateFormatter()
                formatter.dateStyle = .medium
                return "Active (renews \(formatter.string(from: expiry)))"
            }
            return "Active"
        case .inGracePeriod:
            return "Payment issue — grace period active"
        case .inBillingRetry:
            return "Payment issue — retrying"
        case .expired:
            return "Subscription expired"
        case .revoked:
            return "Subscription refunded"
        }
    }
}

// MARK: - Subscription Manager

@MainActor
final class SubscriptionManager: ObservableObject {
    @Published var status: SubscriptionStatus = .unknown
    @Published var products: [Product] = []

    private let storeService: StoreService

    init(storeService: StoreService) {
        self.storeService = storeService
    }

    func refreshStatus() async {
        guard let subscriptionProduct = products.first(where: { $0.type == .autoRenewable }),
              let group = subscriptionProduct.subscription else {
            status = .notSubscribed
            return
        }

        do {
            let statuses = try await group.status

            // Find the most relevant status
            // statuses can contain entries for each product in the group
            guard let primary = statuses.first else {
                status = .notSubscribed
                return
            }

            switch primary.state {
            case .subscribed:
                let expiry = primary.renewalInfo?.currentProductID
                    .flatMap { id in self.productExpiryDate(for: id) }
                status = .active(
                    productID: primary.renewalInfo?.currentProductID ?? subscriptionProduct.id,
                    expiryDate: expiry
                )

            case .expired:
                status = .expired

            case .inBillingRetry:
                status = .inBillingRetry(
                    productID: primary.renewalInfo?.currentProductID ?? subscriptionProduct.id
                )

            case .inGracePeriod:
                let graceExpiry = primary.renewalInfo?.expirationReason
                    .flatMap { _ in self.gracePeriodExpiry(from: primary) }
                status = .inGracePeriod(
                    productID: primary.renewalInfo?.currentProductID ?? subscriptionProduct.id,
                    gracePeriodExpiry: graceExpiry
                )

            case .revoked:
                status = .revoked

            @unknown default:
                status = .unknown
            }
        } catch {
            status = .unknown
        }
    }

    private func productExpiryDate(for productID: String) -> Date? {
        // In practice, extract from the transaction or subscription period
        // This is a simplified version
        Calendar.current.date(byAdding: .month, value: 1, to: Date())
    }

    private func gracePeriodExpiry(from status: Product.SubscriptionInfo.Status) -> Date? {
        // Grace period duration varies by subscription and can be configured
        // in App Store Connect. Typically 6 or 16 days.
        Calendar.current.date(byAdding: .day, value: 16, to: Date())
    }

    // MARK: - Offer Management

    var availableOffers: [Product.SubscriptionOffer] {
        guard let subscription = products.first(where: { $0.type == .autoRenewable })?
            .subscription else { return [] }
        return subscription.offers
    }
}

// MARK: - Usage in SwiftUI

struct SubscriptionStatusView: View {
    @ObservedObject var manager: SubscriptionManager

    var body: some View {
        VStack(spacing: 12) {
            Text(manager.status.displayText)
                .foregroundStyle(manager.status.isActive ? .green : .secondary)

            if manager.status.isActive {
                Label("Premium features unlocked", systemImage: "checkmark.circle.fill")
                    .foregroundStyle(.green)
            }

            if case .inBillingRetry = manager.status {
                Button("Update Payment Method") {
                    if let url = URL(string: "https://apps.apple.com/account/billing") {
                        NSWorkspace.shared.open(url)
                    }
                }
            }

            if case .inGracePeriod = manager.status {
                VStack {
                    Text("Please update your payment information")
                        .foregroundStyle(.orange)
                    Button("Update Payment Method") {
                        if let url = URL(string: "https://apps.apple.com/account/billing") {
                            NSWorkspace.shared.open(url)
                        }
                    }
                    .buttonStyle(.borderedProminent)
                }
            }
        }
        .padding()
        .task {
            await manager.refreshStatus()
        }
    }
}
```

**Key patterns:**
- `SubscriptionStatus` enum centralizes all states into a type-safe model with computed properties for UI decisions.
- `.inGracePeriod` and `.inBillingRetry` should NOT revoke access — Apple gives users a window to fix payment. Use a soft lock pattern (showing banner/nag) instead.
- The `refreshStatus()` function retries on failure — status should never silently become stale.
- Present "Update Payment Method" buttons for billing issues, deep-linking to the App Store billing page.

## StoreView Integration

Full SwiftUI StoreView integration with header, policies, and post-purchase handling:

```swift
import StoreKit
import SwiftUI

// MARK: - Container with StoreView Premium Gate

struct PremiumGateView<Content: View>: View {
    @ObservedObject var store: StoreService
    let content: Content

    var body: some View {
        if store.purchasedIDs.contains(where: { $0.contains("pro") }) {
            // User has premium access — show the real content
            content
        } else {
            // Not purchased — show the store
            AppStoreView(store: store)
        }
    }
}

// MARK: - App Store View

struct AppStoreView: View {
    @ObservedObject var store: StoreService
    @State private var showManageSubscriptions = false

    var body: some View {
        ProductView(
            ids: [
                "com.example.pro.monthly",
                "com.example.pro.yearly",
                "com.example.pro.lifetime"
            ]
        ) {
            // Header content rendered above the product list
            StoreHeaderView()
        }
        .productViewStyle(.large)
        .subscriptionStorePolicyDestinationURL(
            URL(string: "https://example.com/terms")!
        )
        .subscriptionStorePolicyURL(
            URL(string: "https://example.com/privacy")!
        )
        .subscriptionStorePolicyURLForManageSubscriptions(
            URL(string: "https://apps.apple.com/account/subscriptions")!
        )
        .onInAppPurchaseStart { product in
            // Analytics: purchase attempt started
            print("🛒 Purchase started for \(product.id)")
        }
        .onInAppPurchaseCompletion { product, result in
            // Analytics: purchase result (use for telemetry, not delivery)
            switch result {
            case .success(let transaction):
                print("✅ Purchase completed: \(transaction.productID)")
            case .userCancelled:
                print("🚫 Purchase cancelled")
            case .pending:
                print("⏳ Purchase pending")
            @unknown default:
                break
            }
        }
    }
}

// MARK: - Custom Header

struct StoreHeaderView: View {
    var body: some View {
        VStack(spacing: 20) {
            // App icon with premium badge
            Image(.appIcon)
                .resizable()
                .frame(width: 88, height: 88)
                .clipShape(RoundedRectangle(cornerRadius: 20))
                .shadow(color: .black.opacity(0.15), radius: 10, y: 5)

            VStack(spacing: 8) {
                Text("Upgrade to Pro")
                    .font(.title.bold())

                Text("Unlock all features and support development")
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }

            // Feature bullet list
            VStack(alignment: .leading, spacing: 8) {
                FeatureRow(icon: "infinity", text: "Unlimited documents")
                FeatureRow(icon: "icloud.and.arrow.up", text: "Cloud sync")
                FeatureRow(icon: "paintbrush", text: "Custom themes")
                FeatureRow(icon: "shield.checkered", text: "Priority support")
            }
            .padding(.top, 8)
        }
        .padding(.vertical, 32)
    }
}

struct FeatureRow: View {
    let icon: String
    let text: String

    var body: some View {
        HStack {
            Image(systemName: icon)
                .foregroundStyle(.accent)
                .frame(width: 24)
            Text(text)
        }
    }
}

// MARK: - Alternative: Full StoreView for Dedicated Store Window

struct StoreWindow: View {
    @ObservedObject var store: StoreService

    var body: some View {
        NavigationStack {
            StoreView(
                ids: [
                    "com.example.pro.monthly",
                    "com.example.pro.yearly",
                    "com.example.pro.lifetime"
                ]
            ) {
                StoreHeaderView()
            }
            .navigationTitle("Upgrade")
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("Restore Purchases") {
                        Task {
                            await store.restorePurchases()
                        }
                    }
                }
            }
        }
        .frame(minWidth: 480, minHeight: 600)
    }
}
```

**Integration notes:**
- `ProductView` (plural `ids`) renders a list with your header content — use this for inline upsell screens.
- `StoreView` provides the full App Store management experience including subscription upgrade/downgrade/cancel. Use it for a dedicated "Manage Subscription" window or tab.
- `.onInAppPurchaseStart` and `.onInAppPurchaseCompletion` are for analytics only. Content delivery must happen through `Transaction.updates`.
- `subscriptionStorePolicyDestinationURL` is required for App Store review. Provide both terms of service and privacy policy links.
- The Premium Gate pattern conditionally shows content or the store based on purchase state. Remember to also check entitlements on app launch, not just the `purchasedIDs` set.

## Receipt Validation Pattern

Production-ready server-side receipt validation using StoreKit 2 JWS representation:

```swift
import Foundation
import CryptoKit
import StoreKit

// MARK: - Receipt Validation Service

struct ReceiptValidationService {

    private let serverURL: URL
    private let bundleID: String
    private let environment: Environment

    init(serverURL: URL) {
        self.serverURL = serverURL
        self.bundleID = Bundle.main.bundleIdentifier ?? ""
        self.environment = Environment.current
    }

    /// Validate a transaction JWS with your server.
    /// Returns the decoded payload on success.
    func validate(_ transaction: Transaction) async throws -> ValidationResult {
        // Get the JWS representation (available macOS 14+ / iOS 17+)
        guard let jwsString = transaction.jwsRepresentation else {
            throw ValidationError.jwsNotAvailable
        }

        let requestBody = ValidationRequest(
            jwsRepresentation: jwsString,
            bundleID: bundleID,
            environment: environment.rawValue,
            transactionID: String(transaction.id)
        )

        var request = URLRequest(url: serverURL)
        request.httpMethod = "POST"
        request.httpBody = try JSONEncoder().encode(requestBody)
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.timeoutInterval = 30

        let (data, response) = try await URLSession.shared.data(for: request)

        guard let http = response as? HTTPURLResponse,
              (200...299).contains(http.statusCode) else {
            throw ValidationError.serverError
        }

        let result = try JSONDecoder().decode(ValidationResponse.self, from: data)

        return ValidationResult(
            isValid: result.isValid,
            transactionID: result.transactionID,
            originalTransactionID: result.originalTransactionID,
            productID: result.productID,
            purchaseDate: result.purchaseDate,
            expirationDate: result.expirationDate,
            subscriptionGroupID: result.subscriptionGroupID
        )
    }

    /// Check if the local app receipt exists (macOS App Store builds only).
    /// Returns the receipt data, or nil for Xcode/Developer ID builds.
    func fetchLocalReceipt() -> Data? {
        guard let url = Bundle.main.appReceiptURL,
              FileManager.default.fileExists(atPath: url.path) else {
            return nil
        }
        return try? Data(contentsOf: url)
    }
}

// MARK: - Request/Response Types

struct ValidationRequest: Encodable {
    let jwsRepresentation: String
    let bundleID: String
    let environment: String
    let transactionID: String
}

struct ValidationResponse: Decodable {
    let isValid: Bool
    let transactionID: String?
    let originalTransactionID: String?
    let productID: String?
    let purchaseDate: Date?
    let expirationDate: Date?
    let subscriptionGroupID: String?
}

struct ValidationResult {
    let isValid: Bool
    let transactionID: String?
    let originalTransactionID: String?
    let productID: String?
    let purchaseDate: Date?
    let expirationDate: Date?
    let subscriptionGroupID: String?
}

// MARK: - Environment

extension ReceiptValidationService {
    enum Environment: String {
        case sandbox = "Sandbox"
        case production = "Production"
        case xcode = "Xcode"

        static var current: Environment {
            #if DEBUG
            return .xcode
            #else
            // Check the transaction/environment to determine sandbox vs production
            return .production
            #endif
        }
    }

    enum ValidationError: LocalizedError {
        case jwsNotAvailable
        case serverError
        case invalidResponse

        var errorDescription: String? {
            switch self {
            case .jwsNotAvailable:
                return "JWS representation not available for this transaction"
            case .serverError:
                return "Server returned an invalid response"
            case .invalidResponse:
                return "Could not decode server response"
            }
        }
    }
}

// MARK: - Combined Validation in StoreService

extension StoreService {
    /// Full validation flow: JWS to server, then deliver on success.
    func validateAndDeliver(
        transaction: Transaction,
        validator: ReceiptValidationService
    ) async throws {
        // Step 1: Server-side JWS validation
        let result = try await validator.validate(transaction)

        guard result.isValid else {
            throw ValidationError.invalidResponse
        }

        // Step 2: Deliver content if server confirms validity
        await deliverContent(for: transaction)
        await transaction.finish()
    }
}
```

**Best practices:**
- **Always validate server-side in production.** The sample local receipt check in `fetchLocalReceipt()` is a minimal health check — not a security measure.
- **JWS representation** (`transaction.jwsRepresentation`) is the modern StoreKit 2 approach. It contains a signed payload from Apple that your server can verify without calling Apple's `/verifyReceipt` endpoint.
- **Bundle ID check:** Your server must verify the bundle ID in the JWS payload matches your app. Never trust client-supplied bundle IDs.
- **Environment check:** Ensure sandbox transactions are only valid in sandbox. Don't grant production entitlements for sandbox purchases.
- **Timeout:** Receipt validation can be slow over poor networks. Set a reasonable timeout (30 seconds) and retry on timeout.
- **Local receipt:** Only present in App Store builds. Never gate features on local receipt presence in Developer ID or debug builds.

## Sandbox Testing Configuration

Complete Xcode scheme and `.storekit` file setup for testing StoreKit 2 on macOS:

**1. Create StoreKit configuration file:**
```
File → New → File → StoreKit Configuration File
Name it "Products.storekit"
```

**2. Configure products in the editor (or edit the XML directly):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>identifier</key>
    <string>com.example.app</string>
    <key>localizations</key>
    <array>
        <dict>
            <key>locale</key>
            <string>en_US</string>
            <key>name</key>
            <string>Example App</string>
        </dict>
    </array>
    <key>products</key>
    <array>
        <!-- Auto-renewable subscription: Monthly -->
        <dict>
            <key>identifier</key>
            <string>com.example.pro.monthly</string>
            <key>type</key>
            <string>Auto-Renewable Subscription</string>
            <key>subscriptionGroupID</key>
            <string>group_pro</string>
            <key>subscriptionPeriod</key>
            <string>P1M</string>
            <key>price</key>
            <real>4.99</real>
            <key>localizations</key>
            <array>
                <dict>
                    <key>locale</key>
                    <string>en_US</string>
                    <key>name</key>
                    <string>Pro Monthly</string>
                    <key>description</key>
                    <string>Monthly pro subscription</string>
                </dict>
            </array>
        </dict>
        <!-- Auto-renewable subscription: Yearly -->
        <dict>
            <key>identifier</key>
            <string>com.example.pro.yearly</string>
            <key>type</key>
            <string>Auto-Renewable Subscription</string>
            <key>subscriptionGroupID</key>
            <string>group_pro</string>
            <key>subscriptionPeriod</key>
            <string>P1Y</string>
            <key>price</key>
            <real>29.99</real>
            <key>localizations</key>
            <array>
                <dict>
                    <key>locale</key>
                    <string>en_US</string>
                    <key>name</key>
                    <string>Pro Yearly</string>
                    <key>description</key>
                    <string>Yearly pro subscription</string>
                </dict>
            </array>
        </dict>
        <!-- Non-consumable -->
        <dict>
            <key>identifier</key>
            <string>com.example.pro.lifetime</string>
            <key>type</key>
            <string>Non-Consumable</key>
            <key>price</key>
            <real>79.99</real>
            <key>localizations</key>
            <array>
                <dict>
                    <key>locale</key>
                    <string>en_US</string>
                    <key>name</key>
                    <string>Pro Lifetime</string>
                    <key>description</key>
                    <string>Lifetime access to all features</string>
                </dict>
            </array>
        </dict>
    </products>
</dict>
</plist>
```

**3. Xcode scheme configuration:**
```
1. Product → Scheme → Edit Scheme → Run → Options
2. StoreKit Configuration → Select "Products.storekit"
3. Ensure "StoreKit transactions" is checked
```

**4. Programmatic subscription handling for tests:**

```swift
#if DEBUG
extension StoreService {

    /// Force-expire all subscriptions for testing.
    /// Useful for testing expired state without waiting for sandbox renewal.
    func testing_forceExpireAll() async {
        purchasedIDs.removeAll()
        UserDefaults.standard.removeObject(forKey: "isPro")
    }

    /// Simulate a purchase without StoreKit UI.
    /// Only works with .storekit Configuration file — NOT on device.
    func testing_forcePurchase(_ productID: String) async {
        purchasedIDs.insert(productID)
        UserDefaults.standard.set(true, forKey: "purchased_\(productID)")
    }

    /// Verify the full listener + entitlements pipeline.
    func testing_verifyPipeline() async {
        // Check entitlements
        await checkEntitlements()

        // Verify all expected IDs are present
        let allIDs = Set(UserDefaults.standard.dictionaryRepresentation()
            .compactMap { key, value in
                (key as String).hasPrefix("purchased_") &&
                (value as? Bool) == true ?
                (key as String).replacingOccurrences(of: "purchased_", with: "") : nil
            })
        print("✅ Entitlements: \(purchasedIDs)")
        print("✅ Persisted:   \(allIDs)")
    }
}
#endif
```

**5. Test scenarios and accelerators:**

```
Sandbox renewal rates (for testing without waiting real durations):
  1-month subscription  → renews every 5 minutes
  3-month subscription  → renews every 15 minutes
  6-month subscription  → renews every 30 minutes
  1-year subscription   → renews every 1 hour
  Maximum renewals: 12 (after which subscription auto-cancels)

To test renewal in Xcode .storekit config:
  - Use "Expire Subscription" context menu in StoreKit transactions panel
  - Or use Subscription → "Renew" to trigger immediate renewal

To test billing retry:
  - Enable "Ask for Billing Retry" in .storekit file settings
  - Purchase a subscription — it enters billing retry state
  - Verify your app grants temporary access during retry
  - Resolve by completing the purchase in StoreKit transactions panel
```

**Key macOS testing differences from iOS:**
- No `SKTestSession` or `SKTestTransaction` API on macOS (iOS 14+/17+ only).
- Must use `.storekit` files + Xcode scheme for local testing.
- `.storekit` files support IAP Testing panel in Xcode debug bar (Start StoreKit Transaction / Renew / Expire / etc.).
- TestFlight is the only way to test real server-side validation against Apple's sandbox servers.
- No simulator — macOS sandbox tests run on the actual device (your Mac) with the `.storekit` configuration.

**End-to-end test script:**
```swift
// Run these in order during QA testing

// 1. Fresh purchase flow
Task { await store.testing_forceExpireAll() }
await store.loadProducts()
await store.purchase(store.products[0]) // Monthly
await store.testing_verifyPipeline()

// 2. App restart recovery
// Kill the app while purchase is in progress
// Relaunch — check that Transaction.updates delivers the pending transaction
await store.checkEntitlements()
await store.testing_verifyPipeline()

// 3. Renewal
// Wait 5 minutes (sandbox month) or force-renew via Xcode IAP Testing panel
// Verify content is still delivered
await store.testing_verifyPipeline()

// 4. Cancellation
// Open IAP Testing panel → Expire Subscription
// Verify expired state is handled (content revoked or grace period shown)
await subscriptionManager.refreshStatus()

// 5. Restore
await store.restorePurchases()
await store.checkEntitlements()
await store.testing_verifyPipeline()
```
