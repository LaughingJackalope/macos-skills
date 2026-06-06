---
name: swift-security-macos
description: "Security on macOS — Keychain (kSecUseDataProtectionKeychain, kSecAttrAccessGroup for sharing), biometrics (LAContext, Touch ID, Apple Watch unlock), SecureEnclave (SecKey, crypto operations), CryptoKit, Hardened Runtime entitlements, notarization, App Sandbox, AuthenticationServices (passkeys, Sign in with Apple). Use when implementing security features in macOS apps."
sources: https://developer.apple.com/documentation/security, https://developer.apple.com/documentation/cryptokit, https://developer.apple.com/documentation/authenticationservices, https://support.apple.com/guide/security/keychain-data-protection-secb0694df1a/web, https://support.apple.com/guide/security/quantum-secure-cryptography-apple-devices-secc7c82e533/web
researched: 2026-06-06
platform: macOS 26+
---

# Swift Security on macOS

Keychain access, biometric authentication, SecureEnclave key storage, CryptoKit cryptography, Hardened Runtime, App Sandbox, and AuthenticationServices (passkeys, Sign in with Apple) for macOS 26+ with Swift 6.3. This skill covers the Swift-level security APIs; for the full entitlements/notarization/signing pipeline, see `macos-security-entitlements`.

See [references/security-patterns.md](references/security-patterns.md) for deep-dive code patterns: keychain helper tool with App Groups sharing, Touch ID with Apple Watch unlock, SecureEnclave key generation and signing, CryptoKit AES-GCM + HKDF + post-quantum HPKE/ML-KEM/ML-DSA, Apple Watch companion policy migration notes, and passkey integration.

## Contents

- [Keychain Access](#keychain-access)
- [Keychain App Group Sharing](#keychain-app-group-sharing)
- [Biometrics](#biometrics)
- [SecureEnclave](#secureenclave)
- [CryptoKit](#cryptokit)
- [Hardened Runtime Entitlements](#hardened-runtime-entitlements)
- [App Sandbox](#app-sandbox)
- [AuthenticationServices](#authenticationservices)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Keychain Access

The Keychain is the primary secure storage mechanism on macOS. Every query is a `CFDictionary` of `kSec` attributes evaluated by the Security framework.

**Basic storage with `SecItemAdd`:**

```swift
import Foundation
import Security

func storePassword(account: String, password: String) throws {
    guard let passwordData = password.data(using: .utf8) else {
        throw KeychainError.encoding
    }

    // Remove existing item first to avoid errSecDuplicateItem
    try deletePassword(account: account)

    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecAttrService as String: Bundle.main.bundleIdentifier!,
        kSecValueData as String: passwordData,
        kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly,
        // macOS 12+: use Data Protection Keychain (stronger, syncs to iCloud Keychain)
        kSecUseDataProtectionKeychain as String: true
    ]

    let status = SecItemAdd(query as CFDictionary, nil)
    guard status == errSecSuccess else {
        throw KeychainError.unexpectedStatus(status)
    }
}
```

**Reading with `SecItemCopyMatching`:**

```swift
func readPassword(account: String) throws -> String {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: account,
        kSecAttrService as String: Bundle.main.bundleIdentifier!,
        kSecReturnData as String: true,
        kSecMatchLimit as String: kSecMatchLimitOne,
        kSecUseDataProtectionKeychain as String: true
    ]

    var result: AnyObject?
    let status = SecItemCopyMatching(query as CFDictionary, &result)
    guard status == errSecSuccess,
          let data = result as? Data,
          let password = String(data: data, encoding: .utf8) else {
        throw KeychainError.unexpectedStatus(status)
    }

    return password
}
```

**Key `kSecClass` values:**

| Class | Use |
|---|---|
| `kSecClassGenericPassword` | Passwords, API tokens, secrets. Pair with `kSecAttrAccount` and `kSecAttrService`. |
| `kSecClassInternetPassword` | URL-specific credentials. Pair with `kSecAttrServer`, `kSecAttrProtocol`. |
| `kSecClassCertificate` | X.509 certificates. |
| `kSecClassKey` | Symmetric keys (use SecureEnclave for asymmetric). |

**Accessibility levels (`kSecAttrAccessible`):**

| Constant | Description |
|---|---|
| `kSecAttrAccessibleWhenUnlocked` | Accessible while device is unlocked. Backed up. |
| `kSecAttrAccessibleAfterFirstUnlock` | Accessible after first unlock until restart. Backed up. |
| `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` | Not backed up. Stays on device. |
| `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` | Not backed up. Survives re-lock. |
| `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` | Only if passcode is set. Deleted when passcode removed. Not backed up. |

**Biometric protection with `kSecAttrAccessControl`:**

Replace `kSecAttrAccessible` with `kSecAttrAccessControl` to require biometric authentication before each access. If you set both, `kSecAttrAccessControl` wins and `kSecAttrAccessible` is ignored.

```swift
var error: Unmanaged<CFError>?

guard let accessControl = SecAccessControlCreateWithFlags(
    nil,
    kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
    [.biometryCurrentSet, .privateKeyUsage],  // invalidated by biometric enrollment changes
    &error
) else {
    throw KeychainError.accessControlCreation(error?.takeRetainedValue())
}

let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrAccount as String: account,
    kSecValueData as String: passwordData,
    kSecAttrAccessControl as String: accessControl,
    kSecUseDataProtectionKeychain as String: true,
    // Prompt shown to the user during LAContext authentication evaluation
    kSecUseOperationPrompt as String: "Authenticate to access your credentials"
]
```

**Key `SecAccessControlCreateFlags`:**

| Flag | Behavior |
|---|---|
| `.biometryCurrentSet` | Touch ID / Apple Watch. Invalidated if biometric enrollment changes. |
| `.biometryAny` | Touch ID / Apple Watch. NOT invalidated by enrollment changes (weaker). |
| `.devicePasscode` | Device passcode fallback. |
| `.privateKeyUsage` | Required for SecureEnclave key operations (asymmetric keys). |
| `.userPresence` | Any user verification (biometric or passcode). |

 For macOS 15+, when resolving an app group container programmatically, ensure one of: MAS/distributed, TestFlight on macOS 15.1+, app group ID starts with your Team ID, or provisioning profile explicitly authorizes the claim. Unauthorized claims trigger a one-shot user consent prompt (valid only for the current session).
 **Dealing with `kSecUseDataProtectionKeychain`:**

 On macOS 12+, setting `kSecUseDataProtectionKeychain: true` opts into the modern keychain backend. Benefits include iCloud Keychain sync and better crash protection. This attribute must be present on ALL queries for a given item — if you add with it, you must query/update/delete with it. Omitting it on a lookup causes `errSecItemNotFound` even though the item exists. Always include it consistently.

 **iOS-style App Group IDs as keychain access groups (macOS 15+):**
 As of Feb 2025, iOS-style app group IDs (e.g. `group.com.example.MyApp`) can be used as keychain access groups on macOS when the provisioning profile authorizes the claim. Previously only macOS-style `TEAMID.*` groups worked. Apps using the data protection keychain get implicit access to `kSecAttrAccessGroupToken` (`com.apple.token`) — no need to list it in keychain-access-groups entitlement.
 For programmatic container access on macOS 15+, ensure one of: MAS distribution, TestFlight on macOS 15.1+, app group ID starts with your Team ID, or provisioning profile explicitly authorizes. Unauthorized claims trigger a one-shot user consent prompt.
## Keychain App Group Sharing

To share keychain items between a main app and a privileged helper tool, use App Groups (`kSecAttrAccessGroup`). Both targets must have the same App Groups container configured in their entitlements.

**Entitlements for both app and helper:**

```xml
<!-- Both .entitlements files -->
<key>com.apple.security.application-groups</key>
<array>
    <string>TEAMID.com.example.MyAppSuite</string>
</array>
```

**Storing with access group:**

```swift
let query: [String: Any] = [
    kSecClass as String: kSecClassGenericPassword,
    kSecAttrAccount as String: "shared-token",
    kSecValueData as String: tokenData,
    kSecAttrAccessGroup as String: "TEAMID.com.example.MyAppSuite",
    kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly,
    kSecUseDataProtectionKeychain as String: true
]
```

The helper tool must have the same App Groups entitlement and must itself be signed with the same team identity and be sandboxed (or have the keychain-access-groups entitlement).

**SMJobBless sharing pattern:**

```
MyApp (sandboxed, App Group)
  └── kSecAttrAccessGroup: "TEAMID.com.example.MyAppSuite"

MyHelper (installed via SMJobBless, sandboxed, App Group)
  └── kSecAttrAccessGroup: "TEAMID.com.example.MyAppSuite"
```

Both can read and write the same keychain items. See [references/security-patterns.md](references/security-patterns.md) for a complete pattern.

## Biometrics

`LAContext` evaluates biometric authentication requests. On macOS, biometrics means Touch ID (built-in or Magic Keyboard) and optionally Apple Watch unlock.

**Basic biometric check:**

```swift
import LocalAuthentication

func authenticateWithBiometrics() async throws -> Bool {
    let context = LAContext()
    context.localizedFallbackTitle = "Use Password" // passcode fallback button title
    context.localizedCancelTitle = "Cancel"

    // Check availability first
    guard context.canEvaluatePolicy(
        .deviceOwnerAuthenticationWithBiometrics,
        error: nil
    ) else {
        throw BiometryError.notAvailable
    }

    do {
        return try await context.evaluatePolicy(
            .deviceOwnerAuthenticationWithBiometrics,
            localizedReason: "Unlock your secure data"
        )
    } catch let error as LAError {
        switch error.code {
        case .userCancel, .systemCancel, .appCancel:
            throw BiometryError.cancelled
        case .biometryNotAvailable:
            throw BiometryError.notAvailable
        case .biometryNotEnrolled:
            throw BiometryError.notEnrolled
        case .biometryLockout:
            throw BiometryError.lockout  // Too many failed attempts
        case .userFallback:
            throw BiometryError.fallbackRequested
        default:
            throw BiometryError.unknown(error)
        }
    }
}
```

**LAPolicy options on macOS:**
| Policy | Touch ID | Apple Watch | Passcode Fallback |
|---|---|---|---|
| `.deviceOwnerAuthenticationWithBiometrics` | ✓ | ✓ (watchOS companion) | No |
| `.deviceOwnerAuthenticationWithBiometricsOrCompanion` | ✓ | ✓ | No |
| `.deviceOwnerAuthenticationWithCompanion` | ✗ | ✓ | No |
| `.deviceOwnerAuthentication` | ✓ | ✓ | Yes (automatic) |
**Companion policy migration (macOS 15+ / Xcode 26 SDK):**
The "companion" nomenclature replaces "watch". Adopt these for forward compatibility:
| Old (deprecated) | New (macOS 15+) |
|---|---|
| `.deviceOwnerAuthenticationWithWatch` | `.deviceOwnerAuthenticationWithCompanion` |
| `.deviceOwnerAuthenticationWithBiometricsOrWatch` | `.deviceOwnerAuthenticationWithBiometricsOrCompanion` |
**Apple Watch unlock:**
Apple Watch unlock requires no additional code — handled automatically by the system when any biometric policy is evaluated. The user's unlocked, authenticated, wrist-detected Apple Watch satisfies the policy. Requirements: paired Apple Watch, same Apple ID with 2FA, Wi-Fi and Bluetooth on. Your app gets no indication of whether Touch ID or Apple Watch satisfied the policy.
**`.biometryCurrentSet` vs `.biometryAny`:**
When using `SecAccessControlCreateFlags.biometryCurrentSet`, the keychain item is invalidated (cannot be decrypted) if the user adds or removes a fingerprint. `.biometryAny` does NOT invalidate on enrollment change — the item remains usable with any enrolled biometric. Use `biometryCurrentSet` for higher-security scenarios where you want to detect enrollment changes (e.g., someone adding your finger while you sleep).
## SecureEnclave

The SecureEnclave is a hardware security module that stores private keys and performs cryptographic operations without exposing key material to the application processor. It requires Apple Silicon — `kSecAttrTokenIDSecureEnclave` returns an error on Intel Macs.

**Generating a key pair in the SecureEnclave:**

```swift
import Security

func generateSecureEnclaveKey() throws -> SecKey {
    var error: Unmanaged<CFError>?

    let attributes: [String: Any] = [
        kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,
        kSecAttrKeySizeInBits as String: 256,
        kSecAttrTokenID as String: kSecAttrTokenIDSecureEnclave,
        kSecPrivateKeyAttrs as String: [
            kSecAttrIsPermanent as String: true,
            kSecAttrAccessControl as String: SecAccessControlCreateWithFlags(
                nil,
                kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
                [.biometryCurrentSet, .privateKeyUsage],
                &error
            )!,
            kSecUseOperationPrompt as String: "Sign with your private key"
        ]
    ]

    guard let privateKey = SecKeyCreateRandomKey(attributes as CFDictionary, &error) else {
        throw SecureEnclaveError.keyGeneration(error?.takeRetainedValue())
    }

    return privateKey
}
```

**Signing with a SecureEnclave key:**

```swift
func sign(data: Data, with privateKey: SecKey) throws -> Data {
    guard SecKeyIsAlgorithmSupported(privateKey, .sign, .ecdsaSignatureMessageX962SHA256) else {
        throw SecureEnclaveError.algorithmNotSupported
    }

    var error: Unmanaged<CFError>?
    guard let signature = SecKeyCreateSignature(
        privateKey,
        .ecdsaSignatureMessageX962SHA256,
        data as CFData,
        &error
    ) else {
        throw SecureEnclaveError.signing(error?.takeRetainedValue())
    }

    return signature as Data
}
```

**Key points:**
 The authentication prompt (from `kSecUseOperationPrompt`) fires every time a cryptographic operation is performed if `.biometryCurrentSet` is set.
 You cannot export SecureEnclave keys. Do not use `kSecAttrExtractable: true` on them.
 On Apple Silicon, the Secure Enclave uses Sealed Key Protection (SKP) — keys are bound to both the device UID and the sepOS hash, making them tied to the authentic hardware and OS state. Boot integrity (Boot ROM → sepOS verification + SCIP) prevents tampered SEP code from executing.
 The macOS keychain on Apple Silicon uses SE defense-in-depth: both the metadata table key (for search operations) AND per-item secret keys require Secure Enclave round-trips.
 Converting between `SecureEnclave.P256.Signing.PrivateKey` (CryptoKit) and `SecKey` is not supported by Apple's APIs — pick one abstraction and stick with it. `SecKeyCreateWithData(_:_:_:)` with `kSecAttrTokenIDSecureEnclave` creates a new key, it does not import.
CryptoKit provides a Swift-native, safe interface for cryptographic operations. It covers symmetric encryption, key derivation, hashing, and (as of macOS 14+) HPKE for post-quantum key exchange.

**AES-GCM symmetric encryption:**

```swift
import CryptoKit

func seal(message: String, using key: SymmetricKey) throws -> (ciphertext: Data, nonce: Data, tag: Data) {
    let data = message.data(using: .utf8)!

    // AES-GCM with automatic nonce generation
    let sealedBox = try AES.GCM.seal(data, using: key)

    return (
        ciphertext: sealedBox.ciphertext,
        nonce: sealedBox.nonce.withUnsafeBytes { Data($0) },
        tag: sealedBox.tag
    )
}

func open(ciphertext: Data, nonce: Data, tag: Data, using key: SymmetricKey) throws -> String {
    let nonce = try AES.GCM.Nonce(data: nonce)
    let sealedBox = try AES.GCM.SealedBox(nonce: nonce, ciphertext: ciphertext, tag: tag)
    let plaintext = try AES.GCM.open(sealedBox, using: key)

    return String(data: plaintext, encoding: .utf8)!
}
```

**Key derivation with HKDF:**

```swift
func deriveKey(from sharedSecret: SymmetricKey, salt: Data, info: Data) -> SymmetricKey {
    // HKDF-SHA256 derivation: input key material → derived key
    let derived = HKDF<SHA256>.deriveKey(
        inputKeyMaterial: sharedSecret,
        salt: salt,
        info: info,
        outputByteCount: 32  // 256-bit key for AES.GCM
    )
    return derived
}
```

**Hashing:**

```swift
let data = "message".data(using: .utf8)!
let sha256 = SHA256.hash(data: data)       // 32 bytes
let sha384 = SHA384.hash(data: data)       // 48 bytes
let sha512 = SHA512.hash(data: data)       // 64 bytes
```

**Curve25519 key agreement (Diffie-Hellman):**

```swift
// Alice
let alicePrivateKey = Curve25519.KeyAgreement.PrivateKey()
let alicePublicKey = alicePrivateKey.publicKey

// Bob
let bobPrivateKey = Curve25519.KeyAgreement.PrivateKey()
let bobPublicKey = bobPrivateKey.publicKey

// Shared secret (computed independently by both sides)
let aliceShared = try alicePrivateKey.sharedSecretFromKeyAgreement(with: bobPublicKey)
let bobShared = try bobPrivateKey.sharedSecretFromKeyAgreement(with: alicePublicKey)
// aliceShared == bobShared
```

**HPKE — Hybrid Public Key Encryption (post-quantum, macOS 14+/iOS 26+):**
HPKE combines classical (X25519) and post-quantum (ML-KEM-768 via the X-Wing construction) key agreement in a single operation. The shared secret is only secure if BOTH primitives are broken. Introduced at WWDC 25, formally verified in corecrypto, with Apple open-sourcing the production PQ code (May 2026).
Available ciphersuites in `HPKE.Ciphersuite`:
| Ciphersuite | KEM | KDF | AEAD | PQ |
|---|---|---|---|---|
| `.XWingMLKEM768X25519_SHA256_AES_GCM_256` | X-Wing (ML-KEM-768 + X25519) | SHA-256 | AES-GCM-256 | **Yes** |
| `.P256_SHA256_AES_GCM_256` | P-256 | SHA-256 | AES-GCM-256 | No |
| `.P384_SHA384_AES_GCM_256` | P-384 | SHA-384 | AES-GCM-256 | No |
| `.Curve25519_SHA256_ChaChaPoly` | X25519 | SHA-256 | ChaCha20-Poly1305 | No |
```swift
import CryptoKit
// Post-quantum HPKE with X-Wing (ML-KEM-768 + X25519)
func hpkeSealPQ(plaintext: Data, to publicKey: Curve25519.KeyAgreement.PublicKey) throws -> Data {
    let sender = HPKE.Sender<Curve25519.KeyAgreement, AES.GCM, SHA256>(
        recipientKey: publicKey,
        ciphersuite: HPKE.Ciphersuite.XWingMLKEM768X25519_SHA256_AES_GCM_256
    )
    return try sender.seal(plaintext)
}
// Recipient opens with their private key
func hpkeOpenPQ(ciphertext: Data, using privateKey: Curve25519.KeyAgreement.PrivateKey) throws -> Data {
    let recipient = HPKE.Recipient<Curve25519.KeyAgreement, AES.GCM, SHA256>(
        ciphersuite: HPKE.Ciphersuite.XWingMLKEM768X25519_SHA256_AES_GCM_256,
        recipientKey: privateKey,
        encapsulatedKey: encapsulatedKeyData // received alongside ciphertext
    )
    return try recipient.open(ciphertext)
}
```
**Standalone post-quantum primitives (iOS 26+/macOS 26+):**
Beyond HPKE, CryptoKit exposes `MLKEM` and `ML-DSA` directly for custom protocols:
```swift
// ML-KEM key encapsulation (encryption key exchange)
let kemPrivateKey = try MLKEM768.PrivateKey()
let (ciphertext, sharedSecret) = try kemPrivateKey.publicKey.encapsulate()
// Recipient decapsulates
let recoveredSecret = try kemPrivateKey.decapsulate(ciphertext)
// sharedSecret == recoveredSecret
// Secure Enclave-backed ML-KEM key (hardware-bound, biometric-gated)
let seKEMKey = try SecureEnclave.MLKEM768.PrivateKey()
let (seCiphertext, seSharedSecret) = try seKEMKey.publicKey.encapsulate()
// ML-DSA digital signatures (quantum-safe, FIPS 204)
let dsaPrivateKey = try MLDSA65.PrivateKey()
let signature = try dsaPrivateKey.signature(for: dataToSign)
let isValid = dsaPrivateKey.publicKey.isValidSignature(signature, for: dataToSign)
```

## Hardened Runtime Entitlements

Hardened Runtime restricts your app's runtime to prevent code injection and tampering. Entitlements opt _out_ of specific protections. Only enable what you need — every opt-out weakens security and can cause notarization rejection.

| Entitlement | Type | When You Need It | Notarization |
|---|---|---|---|
| `com.apple.security.cs.allow-jit` | Boolean | JavaScriptCore, WebKit, LuaJIT | Accepted |
| `com.apple.security.cs.allow-unsigned-executable-memory` | Boolean | Some interpreters, plugin hosts (broader than `allow-jit`) | Accepted |
| `com.apple.security.cs.allow-dyld-environment-variables` | Boolean | `DYLD_LIBRARY_PATH` etc. | **REJECTED** in notarized builds |
| `com.apple.security.cs.debugger` | Boolean | Allow `lldb`/`gdb` to attach | **REJECTED** in notarized builds |
| `com.apple.security.cs.disable-library-validation` | Boolean | Third-party plugins, audio unit extensions, camera drivers | Accepted (but broad) |
| `com.apple.security.cs.disable-executable-page-protection` | Boolean | Specialized runtime compilers (extremely rare) | Almost always rejected |
| `com.apple.security.get-task-allow` | Boolean | Set by Xcode for debug builds | **Always REJECTED** for distribution |
| `com.apple.security.automation.apple-events` | Boolean | Send Apple Events to other processes | Accepted |

A typical production entitlements file for a sandboxed app with no special needs:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.app-sandbox</key>
    <true/>
    <key>com.apple.security.network.client</key>
    <true/>
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
    <key>com.apple.security.files.bookmarks.app-scope</key>
    <true/>
    <key>com.apple.security.temporary-exception.apple-events</key>
    <array>
        <string>com.apple.systemevents</string>
    </array>
</dict>
</plist>
```

## App Sandbox

App Sandbox (`com.apple.security.app-sandbox: true`) is required for Mac App Store distribution and recommended for all macOS apps. Once sandboxed, your app can only access resources explicitly granted by entitlements.

**File access:**
- Without entitlement → no file access (except `NSTemporaryDirectory()` and container)
- `com.apple.security.files.user-selected.read-write` → user-picked files via `NSOpenPanel`/`NSSavePanel`
- `com.apple.security.files.bookmarks.app-scope` → persist file access via security-scoped bookmarks
- `com.apple.security.temporary-exception.files.home-relative-path.read-write` → specific home-relative paths (weakens sandbox, not accepted on MAS)

**Network:**
- `com.apple.security.network.client` → outbound requests (URLSession, WebSocket, etc.)
- `com.apple.security.network.server` → incoming connections (local HTTP server, Bonjour)

**Key principle:** Always request the minimum set of entitlements. If your app never listens for incoming connections, don't enable `network.server`. Temporary exceptions broaden attack surface and may cause MAS rejection.

For full entitlement reference tables and notarization workflow, see `macos-security-entitlements`.

## AuthenticationServices

AuthenticationServices (`com.apple.AuthenticationServices.Framework`) brings passkeys and Sign in with Apple to macOS.

**Sign in with Apple:**

```swift
import AuthenticationServices

class SignInManager: NSObject, ASAuthorizationControllerDelegate,
                     ASAuthorizationControllerPresentationContextProviding {

    func startSignIn() {
        let request = ASAuthorizationAppleIDProvider().createRequest()
        request.requestedScopes = [.fullName, .email]

        let controller = ASAuthorizationController(authorizationRequests: [request])
        controller.delegate = self
        controller.presentationContextProvider = self
        controller.performRequests()
    }

    // Delegate
    func authorizationController(
        controller: ASAuthorizationController,
        didCompleteWithAuthorization authorization: ASAuthorization
    ) {
        guard let credential = authorization.credential as? ASAuthorizationAppleIDCredential else { return }
        let userID = credential.user
        let email = credential.email
        let identityToken = credential.identityToken  // JWT to verify server-side
    }

    func authorizationController(
        controller: ASAuthorizationController,
        didCompleteWithError error: Error
    ) {
        // Handle error
    }

    // Presentation context
    func presentationAnchor(for controller: ASAuthorizationController) -> ASPresentationAnchor {
        NSApplication.shared.windows.first!
    }
}
```

**Passkeys on macOS 26+:**

Passkeys use the WebAuthn protocol. On macOS, they're anchored to iCloud Keychain and synced across devices.

```swift
func registerPasskey() {
    let publicKeyCredentialProvider = ASAuthorizationPlatformPublicKeyCredentialProvider(
        relyingPartyIdentifier: "example.com"
    )

    let challenge = Data(...) // Server-generated challenge
    let userID = Data(...)    // Server-generated user ID

    let registrationRequest = publicKeyCredentialProvider.createCredentialRegistrationRequest(
        challenge: challenge,
        name: "User Name",
        userID: userID
    )

    // Optional: exclude already-registered credentials
    // registrationRequest.excludedCredentials = [...]

    let authController = ASAuthorizationController(authorizationRequests: [registrationRequest])
    authController.delegate = self
    authController.presentationContextProvider = self
    authController.performRequests()
}
```

 Associated Domains must use HTTPS

**WebView strategies (macOS 15+):**
+
+ **ASWebAuthenticationSession (System WebView)** — recommended for federated sign-in flows. Runs in a top-level browser-like context, enabling full WebPlatform features including passkeys for third-party domains (IdPs). Use this when signing in via Google, Microsoft, etc.
+ **WKWebView (Embedded WebView)** — only supports passkeys for the linked RP domain (your own service). Cannot handle cross-domain federated sign-in with passkeys. Use only when sign-in is managed by your own non-federated passkeys service.

**WWDC25 passkey improvements (macOS 26+):**
+ New streamlined account creation API for faster passkey onboarding
+ Automatic passkey upgrades from passwords (system-coordinated, no extra code needed beyond ASAuthorizationController)
+ performAutoFillAssistedRequests() for conditional mediation UI in text fields
+ ASAuthorizationWebBrowserPublicKeyCredentialManager for browser-app passkey management
+ isUserVerifyingPlatformAuthenticatorAvailable() and isConditionalMediationAvailable() JavaScript APIs (iOS 16.4+/macOS 13.3+) via WKWebView

**Adoption context (2026):** Passkey support on macOS is near-universal — Safari reports ~99% synced passkey coverage, Chrome ~86%. Passkeys in Safari use iCloud Keychain; Chrome uses Google Password Manager with hybrid transport (QR) for cross-platform bridging. No cross-manager sync exists on the same Mac.

## Constraints & Gotchas

- **Keychain accessibility levels:** `kSecAttrAccessibleWhenPasscodeSetThisDeviceOnly` items are deleted when the user removes their passcode. Never use it for data you can't re-create.
- **SecureEnclave requires Apple Silicon.** It is not available on Intel Macs. `SecKeyCreateRandomKey` with `kSecAttrTokenIDSecureEnclave` returns `nil` on Intel. Always feature-detect:
  ```swift
  if SecureEnclave.isAvailable { /* use SE */ }
  ```
- **TCC (Transparency, Consent, and Control):** Accessing the camera, microphone, contacts, calendar, full disk access, or screen recording requires both a purpose string (`NSCameraUsageDescription` etc. in Info.plist) AND explicit user consent via the system TCC prompt. TCC state resets when the app bundle identifier changes.
- **`kSecAttrAccessControl` replaces `kSecAttrAccessible`.** Setting `kSecAttrAccessControl` on a keychain item implicitly sets its accessibility. Don't set both — `kSecAttrAccessControl` wins.
- **`biometryCurrentSet` invalidation:** If all biometrics are removed and re-added, items with `.biometryCurrentSet` become permanently inaccessible. Plan for data recovery.
- **Passkeys require associated domains.** Without the `com.apple.developer.associated-domains` entitlement and a valid AASA file, passkey registration/authentication fails silently.
- **CryptoKit key serialization:** `SymmetricKey(withUnsafeBytes:)` exposes raw key material. Never log or persist raw key bytes. Use `SymmetricKey(size:)` to generate random keys.
- **LAContext reuse:** Each `LAContext` instance can only be evaluated once. Create a new instance for each authentication request.

## Common Mistakes

- **Inconsistent `kSecUseDataProtectionKeychain`:** Adding an item with `kSecUseDataProtectionKeychain: true` and then querying without it returns `errSecItemNotFound`. Always include this attribute or never include it — be consistent across all operations (add, query, update, delete).
- **Not handling LAError codes:** LAError has many specific codes (`userCancel`, `biometryLockout`, `biometryNotEnrolled`, etc.). A single generic handler leaves users confused. Map each code to appropriate UX.
- **Using `.biometryAny` when `.biometryCurrentSet` is more appropriate:** `.biometryAny` means adding a new fingerprint does NOT invalidate the keychain item. For high-security tokens, prefer `.biometryCurrentSet`.
- **Storing secrets in UserDefaults or plist files:** These are not encrypted on disk. Use Keychain for anything sensitive (passwords, API keys, encryption keys).
- **Forgetting to call `SecItemDelete` before `SecItemAdd` (update):** Attempting to add a duplicate item returns `errSecDuplicateItem`. Either delete-then-add or use `SecItemUpdate` with an update dictionary.
- **Enabling `get-task-allow` in Release builds:** Xcode sometimes propagates debug entitlements. Use separate `.entitlements` files for Debug and Release configurations, or audit the archived entitlements with `codesign --display --entitlements -`.
- **Assuming SecureEnclave availability without checking:** Always call `SecureEnclave.isAvailable` before generating SE keys. Crashes or nil returns on Intel Macs.

## Review Checklist

- [ ] Keychain items use consistent `kSecUseDataProtectionKeychain` across all operations
- [ ] Biometric-protected keychain items use `kSecAttrAccessControl` (not `kSecAttrAccessible`)
- [ ] `SecureEnclave.isAvailability` checked before SE key generation
- [ ] No secrets stored in UserDefaults, plist files, or hardcoded in source
- [ ] `get-task-allow` not present in Release entitlements
- [ ] `com.apple.security.cs.debugger` not present in Release entitlements
- [ ] LAError codes are individually handled with appropriate UX
- [ ] Minimum necessary entitlements enabled (no unnecessary sandbox exceptions)
- [ ] Associated domains configured for passkey usage
- [ ] `kSecUseOperationPrompt` set for biometric keychain items (the user sees this)
- [ ] Post-quantum ciphersuite (XWingMLKEM768X25519) used for sensitive data that must resist quantum attacks

## References

- [references/security-patterns.md](references/security-patterns.md) — Keychain helper tool with App Groups sharing, Touch ID with Apple Watch unlock, SecureEnclave key generation and signing, CryptoKit AES-GCM + HKDF, passkey integration pattern
- [Sharing access to keychain items](https://developer.apple.com/documentation/security/sharing-access-to-keychain-items-among-a-collection-of-apps) — App-group keychain sharing
- [kSecUseDataProtectionKeychain](https://developer.apple.com/documentation/security/ksecusedataprotectionkeychain) — Data Protection Keychain reference
- [The Secure Enclave](https://support.apple.com/guide/security/the-secure-enclave-sec59b0b31ff/web) — Apple Silicon SE architecture (SKP, SCIP, boot integrity)
- [Keychain data protection](https://support.apple.com/guide/security/keychain-data-protection-secb0694df1a/web) — Keychain encryption model on Apple Silicon
- [Quantum-secure cryptography](https://support.apple.com/guide/security/quantum-secure-cryptography-apple-devices-secc7c82e533/web) — PQ cryptography on Apple platforms
- [HPKE.Ciphersuite](https://developer.apple.com/documentation/cryptokit/hpke/ciphersuite) — Post-quantum and classical ciphersuites
- [Get ahead with quantum-secure cryptography (WWDC25)](https://developer.apple.com/videos/play/wwdc2025/314/) — WWDC session on CryptoKit PQ APIs
- [What's new in passkeys (WWDC25)](https://developer.apple.com/videos/play/wwdc25/279/) — Passkey improvements
- [Supporting passkeys](https://developer.apple.com/documentation/authenticationservices/supporting-passkeys) — Passkey setup guide
- [LocalAuthentication policies (macOS)](https://github.com/dotnet/macios/wiki/LocalAuthentication-macOS-xcode26.0-b1) — Xcode 26 / macOS 15 companion policy migration
- [App Groups keychain forum](https://developer.apple.com/forums/thread/721701) — iOS-style app group IDs as keychain access groups on macOS
