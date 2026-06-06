---
name: storekit-macos
description: "StoreKit 2 on macOS — Product.products/purchase/Transaction.updates, subscription status, AppStore/sync, SKOverlay, StoreView/ProductView SwiftUI, receipt validation, sandbox testing (Xcode scheme setup). Use when implementing in-app purchases or managing subscriptions in macOS apps."
sources: https://developer.apple.com/documentation/storekit, https://developer.apple.com/documentation/storekit/transaction
researched: 2026-06-06
platform: macOS 26+
---

# StoreKit 2 on macOS

Implement in-app purchases and subscription management using StoreKit 2 on macOS 26+ with Swift 6.3. Covers product fetching, purchase flows, transaction handling, subscription status checking, App Store sync, SwiftUI store views, receipt validation, and sandbox testing. This skill focuses on macOS-specific considerations; for iOS StoreKit patterns, see the `storekit` skill.

See [references/storekit-patterns.md](references/storekit-patterns.md) for complete code patterns including a full purchase flow, subscription status checking, StoreView integration, receipt validation, and sandbox testing configuration.

## Contents

- [StoreKit 2 Product Fetch](#storekit-2-product-fetch)
- [Purchase Flow](#purchase-flow)
- [Transaction Handling](#transaction-handling)
- [Subscription Status](#subscription-status)
- [AppStore.sync](#appstoresync)
- [UI Components](#ui-components)
- [Receipt Validation](#receipt-validation)
- [Sandbox Testing](#sandbox-testing)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## StoreKit 2 Product Fetch

StoreKit 2 replaces the older SKProductsRequest with a Swift Concurrency-based API. Fetch products by their string identifiers using `Product.products(for:)`.

```swift
import StoreKit

@MainActor
final class StoreService {
    private var products: [Product] = []

    func loadProducts() async throws {
        let identifiers = [
            "com.example.app.pro.monthly",
            "com.example.app.pro.yearly",
            "com.example.app.lifetime"
        ]
        let storeProducts = try await Product.products(for: identifiers)
        products = storeProducts.sorted(by: { $0.price < $1.price })
    }

    var subscriptions: [Product] {
        products.filter { $0.type == .autoRenewable }
    }

    var nonConsumables: [Product] {
        products.filter { $0.type == .nonConsumable }
    }
}
```

**Product types:**

| Type | Enum Case | Description |
|------|-----------|-------------|
| Consumable | `.consumable` | One-time purchase consumed by the app (currency, boosts). Must be purchased again each time. |
| Non-consumable | `.nonConsumable` | Permanent purchase unlocked forever (pro features, ad removal). Does not expire. |
| Auto-renewable subscription | `.autoRenewable` | Recurring subscription until cancelled (monthly, yearly). Auto-renews unless user cancels. |
| Non-renewing subscription | `.nonRenewing` | Time-limited access that does not auto-renew. Must be repurchased after expiry. Rare on macOS. |

**Key properties on `Product`:**
- `id` — The string identifier matching App Store Connect.
- `type` — The `Product.ProductType` enum.
- `displayName` — Localized display name from App Store Connect.
- `description` — Localized description.
- `price` — The `Decimal` price in the user's local currency.
- `displayPrice` — A localized string suitable for display (includes currency symbol).
- `subscription` — A `Product.SubscriptionInfo?` (non-nil for `.autoRenewable`).

**Ordering:** `Product.products(for:)` returns products in an arbitrary order. Sort them yourself — typically by price or by your preferred display sequence. Never assume the order matches your identifier array.

**Caching:** StoreKit does not cache product results. If you need products available across your app, store them in an `@Observable` or `@Published` property. Re-fetch when the app becomes foregrounded.

## Purchase Flow

Purchase a product with `Product.purchase(options:)`. The system handles authentication (biometrics or password) and returns a `Product.PurchaseResult`.

```swift
@MainActor
final class StoreService {
    // ... products loaded as above

    func purchase(_ product: Product) async throws -> PurchaseResult {
        let result = try await product.purchase()

        switch result {
        case .success(let verification):
            // Verify the transaction
            let transaction = try checkVerified(verification)

            // Deliver content to the user
            await deliverPurchase(productID: transaction.productID)

            // Always finish the transaction after delivering content
            await transaction.finish()

            return .purchased(transaction)

        case .userCancelled:
            return .cancelled

        case .pending:
            // Waiting for approval (e.g., Ask to Buy)
            return .pending

        @unknown default:
            return .unknown
        }
    }

    private func checkVerified<T>(_ result: VerificationResult<T>) throws -> T {
        switch result {
        case .unverified:
            throw PurchaseError.verificationFailed
        case .verified(let safe):
            return safe
        }
    }

    func deliverPurchase(productID: String) async {
        // Unlock features, grant credits, etc.
        switch productID {
        case "com.example.app.pro.monthly", "com.example.app.pro.yearly":
            UserDefaults.standard.set(true, forKey: "isPro")
        case "com.example.app.lifetime":
            UserDefaults.standard.set(true, forKey: "isProLifetime")
        default:
            break
        }
    }
}

enum PurchaseResult {
    case purchased(Transaction)
    case cancelled
    case pending
    case unknown
}

enum PurchaseError: Error {
    case verificationFailed
    case productNotFound
}
```

**Purchase options:**

```swift
// Application username — for server-side tracking
let option = Product.PurchaseOption.applicationUsername("user_123_abc")

// Simulates a successful purchase in sandbox (Xcode only)
let option = Product.PurchaseOption.simulatesSuccessfulPurchase()

let result = try await product.purchase(options: [option])
```

**`Product.PurchaseOption` variants:**
- `.applicationUsername(_:)` — Opaque string passed to server-side validation. Useful for mapping transactions to your user accounts.
- `.quantity(_:)` — Purchase multiple of a consumable. Must be 1 for non-consumables and subscriptions.
- `.promotionalOffer(_:)` — Apply a promotional offer from App Store Connect.
- `.simulatesSuccessfulPurchase()` — Debug-only. Use in Xcode scheme for testing without a sandbox account.

**Error handling:** `product.purchase()` throws if the device cannot make purchases (restrictions, no network, etc.) or if the product identifier is invalid. The `Result` covers logical outcomes (success, cancel, pending), while the `throws` covers infrastructure failures.

## Transaction Handling

StoreKit 2 provides two ways to access transactions: `Transaction.updates` for real-time changes and `Transaction.currentEntitlements` for existing purchases.

**Transaction listener — start at app launch:**

```swift
@MainActor
final class StoreService {
    var listening = false

    func startTransactionListener() {
        guard !listening else { listening = true; return }

        Task {
            for await result in Transaction.updates {
                do {
                    let transaction = try checkVerified(result)

                    // Handle the transaction
                    await deliverPurchase(productID: transaction.productID)
                    await transaction.finish()
                } catch {
                    // Log verification failure but keep listening
                    print("⚠️ Unverified transaction from update: \(error)")
                }
            }
        }
    }
}
```

**`Transaction.updates`** is an `AsyncSequence<VerificationResult<Transaction>>`. It emits:
- New purchases
- Subscription renewals
- Refunds and revocations
- Ask-to-Buy approvals

**Current entitlements — check at launch:**

```swift
func checkCurrentEntitlements() async {
    for await result in Transaction.currentEntitlements {
        do {
            let transaction = try checkVerified(result)
            await deliverPurchase(productID: transaction.productID)
        } catch {
            print("⚠️ Unverified entitlement: \(error)")
        }
    }
}
```

**`Transaction.currentEntitlements`** iterates over all transactions that entitle the user to content. It starts from the oldest and includes the most recent transaction for each product. Non-consumables and auto-renewable subscriptions appear here as long as the user owns them. Consumables do not — they are finished after delivery.

**Always call `finish()`:** After delivering content, call `await transaction.finish()`. Until a transaction is finished, StoreKit will continue to deliver it through `Transaction.updates` at every app launch. Forgetting to finish transactions leads to repeated delivery.

**Revocation:** When a purchase is refunded via Apple's customer support or an Ask-to-Buy is declined, `Transaction.updates` delivers a transaction with `transaction.revocationDate` set. Handle this:

```swift
for await result in Transaction.updates {
    let transaction = try checkVerified(result)

    if transaction.revocationDate != nil {
        // Refund or cancellation — revoke access
        await revokeAccess(productID: transaction.productID)
    } else if transaction.expirationDate?.timeIntervalSinceNow.sign == .minus {
        // Subscription expired — do not renew access
        await revokeAccess(productID: transaction.productID)
    } else {
        // Valid transaction — deliver or extend access
        await deliverPurchase(productID: transaction.productID)
    }

    await transaction.finish()
}
```

**Transaction properties key to check:**
- `transaction.productID` — The product identifier.
- `transaction.purchaseDate` — When the purchase was made.
- `transaction.expirationDate` — For subscriptions, when the subscription expires (nil for non-consumables).
- `transaction.revocationDate` — If refunded/cancelled.
- `transaction.isUpgraded` — True if the user upgraded to a higher-tier subscription.
- `transaction.environment` — `.sandbox` or `.production`.
- `transaction.ownershipType` — `.purchased` or `.familyShared`.

## Subscription Status

For auto-renewable subscriptions, access detailed status through `Product.SubscriptionInfo`:

```swift
func checkSubscriptionStatus() async throws -> SubscriptionState {
    guard let product = subscriptions.first,
          let subscriptionInfo = product.subscription else {
        return .notSubscribed
    }

    let statuses = try await subscriptionInfo.status

    // SubscriptionStatus is an array — a subscription group can have
    // multiple products, each with its own status
    for status in statuses {
        switch status.state {
        case .subscribed:
            return .active(expires: status.renewalInfo?.currentProductID)

        case .expired:
            return .expired

        case .inBillingRetry:
            return .billingRetry(expiry: status.renewalInfo?.currentProductID)

        case .inGracePeriod:
            return .gracePeriod

        case .revoked:
            return .revoked

        @unknown default:
            break
        }
    }

    return .notSubscribed
}

enum SubscriptionState {
    case notSubscribed
    case active(expires: String?)
    case expired
    case billingRetry(expiry: String?)
    case gracePeriod
    case revoked
}
```

**`SubscriptionStatus.State` values:**

| State | Meaning | What to do |
|-------|---------|-----------|
| `.subscribed` | Active subscription | Grant access |
| `.expired` | Lapsed, user didn't renew | Revoke access, offer restore |
| `.inBillingRetry` | Payment failed, Apple retrying | Grant temporary access (soft lock) |
| `.inGrace Period` | Payment failed, grace period active | Grant access, prompt to update payment |
| `.revoked` | User got a refund | Revoke access immediately |

**`RenewalInfo`** (from `status.renewalInfo`):
- `currentProductID` — The product the user is currently subscribed to.
- `autoProductID` — The product the subscription will renew to (changes if user cross-grades).
- `willAutoRenew` — Whether auto-renew is enabled.
- `isInBillingRetryPeriod` — Whether Apple is retrying billing.
- `expirationReason` — Why the subscription expired (if applicable).
- `gracePeriodExpirationDate` — When the grace period ends.
- `offerID` — Any applied promotional offer ID.
- `isUpgraded` — Whether the user upgraded during this period.

**Period unit:** Access subscription length via `subscriptionInfo.subscriptionPeriod`:
```swift
let period = subscriptionInfo.subscriptionPeriod
// period.unit: .day, .week, .month, .year
// period.value: Int (e.g., 1 for monthly, 12 for yearly)
```

## AppStore.sync

On macOS, `AppStore.sync()` forces the App Store to refresh pending transactions. This is critical in scenarios where a purchase was made but the transaction wasn't fully processed (e.g., the app was terminated during purchase).

```swift
func syncWithAppStore() async throws {
    try await AppStore.sync()
}
```

**When to call `sync()`:**
- When the user taps a "Restore Purchases" button
- When `Transaction.updates` has been silent but you know a purchase was initiated
- After receiving a StoreKit external purchase notification (macOS)

**macOS vs iOS differences:**
- On iOS, purchases are typically processed within the same app session. On macOS (especially with App Store distribution), there's a higher chance of interrupted transactions.
- macOS supports external purchase flows (linking to a website) in some regions — handle these via `onStorefrontChange` and `onExternalPurchase`.
- Unlike iOS, macOS does not have a dedicated "Restore Purchases" button guideline, but providing one that calls `AppStore.sync()` is still good practice.
- `AppStore.sync()` on macOS syncs the local receipt with the App Store. It can be slow — show a progress indicator.

**Restore pattern:**

```swift
func restorePurchases() async throws {
    try await AppStore.sync()
    // After sync, re-check entitlements
    await checkCurrentEntitlements()
}
```

## UI Components

**ProductView (SwiftUI):**

```swift
import StoreKit
import SwiftUI

struct SubscriptionOfferView: View {
    let products: [Product]

    var body: some View {
        VStack(spacing: 16) {
            ForEach(products, id: \.id) { product in
                ProductView(product) {
                    // Custom icon or badge
                    Image(systemName: "crown.fill")
                        .foregroundStyle(.yellow)
                }
                .productViewStyle(.compact)
                .onInAppPurchaseCompletion { product, result in
                    // Called after purchase completes (success or failure)
                    // Use this for analytics, not for delivering content
                    // (use Transaction.updates for actual delivery)
                    logPurchaseAttempt(product: product, result: result)
                }
            }
        }
        .padding()
    }
}
```

**StoreView (SwiftUI — for full store experience):**

```swift
import StoreKit
import SwiftUI

struct StoreContainerView: View {
    var body: some View {
        StoreView(ids: [
            "com.example.app.pro.monthly",
            "com.example.app.pro.yearly",
            "com.example.app.lifetime"
        ]) {
            // Header content above the store
            VStack {
                Image(.premiumBadge)
                    .resizable()
                    .frame(width: 80, height: 80)
                Text("Go Premium")
                    .font(.title.bold())
            }
            .padding(.top, 40)
        }
        .storePolicyForegroundStyle(.secondary)
        .subscriptionStorePolicyDestinationURL(
            URL(string: "https://example.com/terms")
        )
    }
}
```

**SKOverlay (macOS available):**

```swift
import StoreKit

func presentAppStoreOverlay(for product: Product) {
    guard let scene = NSApplication.shared.connectedScene else { return }

    let config = SKOverlay.AppConfiguration(
        appIdentifier: "YOUR_APP_ID", // Your app's Apple ID from App Store Connect
        position: .bottom
    )

    let overlay = SKOverlay(configuration: config)
    overlay.present(in: scene)
}

// For a specific product recommendation:
func presentOverlay(for product: Product, in scene: SKScene) {
    let config = SKOverlay.AppProductConfiguration(
        appIdentifier: "YOUR_APP_ID",
        campaignToken: "your-campaign-token", // Optional
        providerToken: "your-provider-token", // Optional
        position: .bottom
    )
    let overlay = SKOverlay(configuration: config)
    overlay.present(in: scene)
}
```

**macOS-specific UI notes:**
- `ProductView` and `StoreView` are available on macOS 14+. They render native SwiftUI controls.
- `SKOverlay` on macOS presents an App Store card at the specified position. Unlike iOS, it overlays the window, not the full screen.
- `StoreView` handles purchase UI, including subscription management. Users can upgrade, downgrade, or cancel directly from the StoreView sheet.
- For subscription management, you can deep-link to the App Store management page: `NSApp.open(URL(string: "https://apps.apple.com/account/subscriptions")!)`.

## Receipt Validation

On macOS, the app receipt is a file at `Bundle.main.appReceiptURL` named `receipt`. This is an opaque PKCS 7 container signed by Apple.

**Local receipt validation (basic):**

```swift
import Foundation
import Security
import CryptoKit

struct ReceiptValidator {

    /// Verifies the receipt file exists and is signed by Apple.
    /// This is a basic check — for production, always validate server-side.
    static func validateLocalReceipt() -> ReceiptCheck {
        guard let receiptURL = Bundle.main.appReceiptURL else {
            return .noReceipt
        }

        guard FileManager.default.fileExists(atPath: receiptURL.path) else {
            return .noReceipt
        }

        // Read the receipt data
        guard let receiptData = try? Data(contentsOf: receiptURL) else {
            return .unreadable
        }

        // Basic PKCS 7 container check — verify Apple signature
        guard verifyAppleSignature(in: receiptData) else {
            return .invalidSignature
        }

        return .valid
    }

    /// Verifies the PKCS 7 container is signed by Apple's root CA.
    /// This does NOT replace server-side validation.
    private static func verifyAppleSignature(in receiptData: Data) -> Bool {
        guard let secData = SecCMSCreateDecryptedData(
            receiptData as CFData,
            nil,  // detached content (nil for embedded)
            nil   // output (nil = only verify)
        ) else {
            // Simplified: in production, use the full SecCMS verification chain
            // with Apple's intermediate and root certificates.
            return true // Placeholder — implement full chain validation
        }

        return secData != nil
    }
}

enum ReceiptCheck {
    case valid
    case noReceipt
    case unreadable
    case invalidSignature
}
```

**macOS receipt gotchas:**
- **App Store distribution:** The receipt is automatically present after download.
- **Developer ID distribution:** The receipt is NOT present. Do not require receipt validation for Developer ID builds.
- **Xcode debug builds:** The receipt is NOT present. Use StoreKit Testing Framework or sandbox for testing.
- **TestFlight:** The receipt IS present, signed by the sandbox environment.

**Server-side validation (recommended):**

```swift
func validateWithServer(_ transaction: Transaction) async throws -> Bool {
    // For StoreKit 2, send the JWS (JSON Web Signature) directly to your server
    let jwsPayload = transaction.jwsRepresentation // macOS 14+ / iOS 17+

    guard let url = URL(string: "https://your-server.com/verify") else {
        throw URLError(.badURL)
    }

    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.httpBody = jwsPayload.data(using: .utf8)
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")

    let (data, _) = try await URLSession.shared.data(for:: request)
    let response = try JSONDecoder().decode(ValidationResponse.self, from: data)

    return response.isValid
}

struct ValidationResponse: Decodable {
    let isValid: Bool
}
```

**Server-side flow:**
1. Get the JWS representation from `transaction.jwsRepresentation`
2. Send it to your server
3. Your server verifies the JWS signature with Apple's public key
4. Server decodes the payload and checks the transaction details
5. Server responds with validity and any content to unlock

## Sandbox Testing

**Xcode scheme setup for sandbox:**

1. In Xcode, select your scheme → Edit Scheme → Run → Options
2. Set "StoreKit Configuration" to your `.storekit` configuration file
3. If no `.storekit` file exists: File → New → File → StoreKit Configuration File
4. Define your products in the `.storekit` file with matching identifiers
5. Run the app from Xcode — purchases will use the sandbox environment

**No StoreKit Testing Framework on macOS:**

On iOS, you can use `SKTestSession` and `SKTestTransaction` for programmatic testing. **This is NOT available on macOS.** On macOS, sandbox testing relies on:
- `.storekit` configuration files (local testing without Apple ID)
- Sandbox tester accounts (TestFlight-style testing)

**Sandbox tester account setup (Xcode scheme):**
1. In the `.storekit` file, enable "Ask for Payment Failure" to test billing issues
2. Set subscription renewal rate (accelerated — 5-minute months, etc.)
3. Test interruption behavior by killing the app mid-purchase

**TestFlight testing:**
1. Upload to App Store Connect
2. Add internal testers (no review needed) or external testers (requires beta review)
3. TestFlight builds use the sandbox environment automatically
4. Sandbox subscriptions renew at accelerated rates:
   - 1-month → renews every 5 minutes
   - 3-month → renews every 15 minutes
   - 6-month → renews every 30 minutes
   - 1-year → renews every 1 hour
5. Sandbox subscriptions auto-cancel after 12 renewals (regardless of duration)

**Testing checklist:**
- [ ] Purchase a consumable → transaction appears in `.updates`, delivered, finished
- [ ] Purchase a non-consumable → appears in `currentEntitlements` after app restart
- [ ] Subscribe → `.subscribed` status delivered via `SubscriptionInfo.status`
- [ ] Cancel subscription (via Settings → Manage Subscriptions) → `.expired` status
- [ ] Force-expire via `.storekit` config → expiry handled correctly
- [ ] Ask-to-Buy decline → `.pending` → revocation in `.updates`
- [ ] Billing retry → access granted during retry period
- [ ] Restore Purchases → `AppStore.sync()` → all entitlements restored
- [ ] App killed mid-purchase → transaction delivered via `.updates` on relaunch

## Constraints & Gotchas

**macOS sandbox restrictions:**
- If your app uses the macOS App Sandbox (required for App Store distribution), StoreKit works. There are no additional entitlements needed for StoreKit.
- Developer ID-distributed apps (outside App Store) MUST use a special entitlement: `com.apple.developer.storekit.external-purchase` if linking to an external purchase page. Direct in-app purchase via StoreKit does NOT work for Developer ID apps.

**App Store vs Developer ID:**
- **App Store distribution:** Full StoreKit 2 support. Receipt validation works. `AppStore.sync()` works.
- **Developer ID distribution:** StoreKit 2 works for purchase within the app, but the app receipt is NOT present. If you need receipt validation, either require App Store distribution or skip receipt validation and rely on transaction JWS verification only.
- **Both distributions:** Use `#if APP_STORE` compiler flags or a runtime check to conditionally enable receipt validation.

**Receipt refresh:**
- On macOS, `SKReceiptRefreshRequest` (from StoreKit 1) is still available but deprecated. Prefer StoreKit 2's `AppStore.sync()` instead.
- If you need a receipt for server-side validation of legacy calls, use `AppStore.sync()` which ensures the receipt is current.

**Family Sharing:**
- `Transaction.ownershipType == .familyShared` indicates a family-shared purchase.
- Handle this in your transaction listener just like a regular purchase.
- Family Sharing must be enabled in App Store Connect for each product.

**Ask to Buy:**
- Transactions from children with Ask to Buy enabled arrive as `.pending` from `purchase()`.
- The parent approves or declines → the result appears in `Transaction.updates`.
- Do not deliver content for `.pending` results.

**Subscription upgrades/downgrades:**
- When a user changes subscription tier, `transaction.isUpgraded == true` on the old subscription.
- The new subscription starts immediately; the old one's remaining time is forfeited (unless proration applies).
- `Product.SubscriptionInfo.Status.renewalInfo.currentProductID` reflects the active product.

## Common Mistakes

1. **Not starting the transaction listener at launch.** If you don't iterate `Transaction.updates` early, you'll miss renewals, revocations, and interrupted purchases. Start it in `init` or at app launch.

2. **Forgetting to call `transaction.finish()`.** Unfinished transactions are re-delivered on every app launch via `Transaction.updates`. This causes repeated content delivery.

3. **Delivering content from `purchase()` result instead of `Transaction.updates`.** While delivering from the purchase result works for the current session, only the update stream handles renewals, revocations, and interrupted purchases. Use `Transaction.updates` as the single source of truth.

4. **Not checking `revocationDate` and `expirationDate`.** A transaction from `currentEntitlements` could be revoked or expired. Always check dates before granting access.

5. **Using `Transaction.currentEntitlements` without also using `Transaction.updates`.** Current entitlements give you the state at this moment; updates handle future changes. Always use both.

6. **Validating receipts locally for DRM purposes.** Local receipt validation on macOS is trivially bypassed for Developer ID apps. For production security, always validate server-side using the JWS representation.

7. **Not handling `.pending` purchases.** Ask-to-Buy flows return `.pending`. Show appropriate UI — don't deliver content until the transaction state is confirmed.

8. **Assuming product order from `Product.products(for:)`.** The returned array is not sorted by your input order. Sort by price or maintain your own ordering.

9. **Calling `AppStore.sync()` without UI feedback.** On macOS, the sync can take several seconds. Show a spinner or disable the Restore button while syncing.

10. **Testing only with `.storekit` files.** Always run final tests with TestFlight to verify the real end-to-end purchase flow, including receipt generation and sandbox renewal behavior.

## Review Checklist

- [ ] `Product.products(for:)` errors are caught and surfaced with user-facing messages
- [ ] Transaction listener started at app launch (in `@main` `App` init or early lifecycle)
- [ ] `Transaction.updates` is the single source of truth for content delivery
- [ ] `transaction.finish()` called after every successful content delivery
- [ ] `revocationDate` and `expirationDate` checked before granting access
- [ ] Subscription status checked via `Product.SubscriptionInfo.status` before granting access
- [ ] All subscription states (subscribed, expired, billing retry, grace period, revoked) handled
- [ ] `AppStore.sync()` called for "Restore Purchases" with loading UI
- [ ] Server-side receipt/JWS validation implemented for production
- [ ] Local receipt validation skipped for Developer ID builds
- [ ] Sandbox testing completed with `.storekit` file and TestFlight
- [ ] Subscription lifecycle tested: subscribe, renew, cancel, upgrade, restore
- [ ] Ask-to-Buy flow tested (`.pending` handling)
- [ ] Family Shared purchases handled
- [ ] Unicode/localized product names and prices displayed correctly
- [ ] Purchase flow works when app is killed mid-purchase (recovers via `Transaction.updates`)

## References

- [Apple — StoreKit Documentation](https://developer.apple.com/documentation/storekit)
- [Apple — Transaction Documentation](https://developer.apple.com/documentation/storekit/transaction)
- [Apple — Product Documentation](https://developer.apple.com/documentation/storekit/product)
- [Apple — Validating receipts on the device](https://developer.apple.com/documentation/storekit/original_api_for_in-app_purchase/validating_receipts_with_the_app_store)
- [Apple — AppStore.sync()](https://developer.apple.com/documentation/storekit/appstore/sync())
- [Apple — Product.SubscriptionInfo](https://developer.apple.com/documentation/storekit/product/subscriptioninfo)
- [Apple — SKOverlay](https://developer.apple.com/documentation/storekit/skoverlay)
- [Apple — StoreView](https://developer.apple.com/documentation/storekit/storeview)
- [Apple — WWDC23: Meet StoreKit 2](https://developer.apple.com/videos/play/wwdc2023/10115/)
- [Apple — WWDC23: What's new in StoreKit testing](https://developer.apple.com/videos/play/wwdc2023/10145/)

## Research Addendum — 2026 Updates

### StoreKit 2 Purchase & Transaction (WWDC25)

- **`Transaction.currentEntitlements(for:)`** (iOS 18.4+/macOS 15.4+) — Replaces `currentEntitlement(for:)`. Returns an `AsyncSequence` of all transactions entitling the customer to a given product. Handles multiple entitlements per product.
- **`AppTransaction`** — New in iOS 18.4+ providing `appTransactionID` (globally unique per Apple Account + Family Sharing) and `originalPlatform` (iOS/macOS/tvOS/visionOS). Useful for cross-device entitlement tracking.
- **`advancedCommerceInfo`** — For apps using Apple's Advanced Commerce API (large catalogs, creator experiences, optional add-ons). New field on `Transaction`.
- **`RenewalInfo` enhancements** — Four new fields added. The `SubscriptionStatus` API supports querying statuses by `Transaction.ID` starting with iOS 18.4.

### Sandbox Testing on macOS (2026)

- **StoreKit Testing in Xcode** — Available for macOS. Use `.storekit` configuration files for rapid iteration without App Store Connect or network. Set scheme option "StoreKit Configuration" to select your `.storekit` file.
- **No SKTestSession on macOS** — The programmatic `SKTestSession`/`SKTestTransaction` API is iOS-only. On macOS, testing relies on `.storekit` files and sandbox accounts.
- **Sandbox renewal rates** (same as before, confirmed current):
  - 1-month → 5 minutes
  - 3-month → 15 minutes
  - 6-month → 30 minutes
  - 1-year → 1 hour
- **Sandbox auto-cancel after 12 renewals** regardless of subscription duration.
- **Development builds** use APNs sandbox; **TestFlight and App Store builds** use production. Tokens are environment-specific.
- **Download testing advice**: Always test final flow with TestFlight to verify real end-to-end purchase behavior, receipt generation, and sandbox renewal.

### ProductView & StoreView (2026)

- **`SubscriptionStoreView`** — For subscription-only stores, provides built-in plan comparison, subscriber status, and introductory offer display.
- **`ProductView.Style`** — Three styles: `.compact`, `.regular`, `.large` (via `.productViewStyle(_:)`).
- **`StoreButton(_:for:)` modifier** — Add common buttons (restore purchases, manage subscriptions) to StoreView or SubscriptionStoreView toolbar.
- **`onStorefrontChange` / `onExternalPurchase`** — Handle external purchase flows (required in some EU markets). macOS StoreKit supports linking to web-based purchase pages.
- **Layout**: StoreView arranges in rows/columns based on available horizontal space, grows to fit container, scrolls when limited. The rendering on macOS follows platform conventions.
