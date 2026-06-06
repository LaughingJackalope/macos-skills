# Security Patterns Reference

Deep-dive code patterns for macOS security: keychain helper tool with App Groups sharing, Touch ID with Apple Watch unlock fallback, SecureEnclave key generation and signing, CryptoKit AES-GCM + HKDF + quantum-secure HPKE/ML-KEM/ML-DSA, and passkey integration. Updated for macOS 26 (WWDC25).

## Contents

- [Keychain Helper Tool with App Group Sharing](#keychain-helper-tool-with-app-group-sharing)
- [Quantum-Secure Ciphersuites](#quantum-secure-ciphersuites)
- [Apple Watch companion policy migration notes](#apple-watch-companion-policy-migration-notes)
- [Touch ID with Apple Watch Unlock Fallback](#touch-id-with-apple-watch-unlock-fallback)
- [SecureEnclave Key Generation and Signing](#secureenclave-key-generation-and-signing)
- [CryptoKit: AES-GCM + HKDF Complete Example](#cryptokit-aes-gcm--hkdf-complete-example)
- [Passkey Integration Pattern](#passkey-integration-pattern)

## Keychain Helper Tool with App Group Sharing

This pattern shares keychain items between a sandboxed main app and a sandboxed privileged helper tool using App Groups.

**Configuration:**

1. Both targets get the App Groups capability with the same group ID.
2. The helper is installed via `SMJobBless` (see `macos-security-entitlements` skill).
3. Both the app and helper use `kSecAttrAccessGroup` matching the App Group.

**Shared keychain helper (used by both app and helper):**

```swift
import Foundation
import Security

enum SharedKeychain {
    static let accessGroup = "TEAMID.com.example.MyAppSuite"
    static let service = "com.example.MyApp"

    static func store(key: String, data: Data) throws {
        // Delete existing first to enable "upsert" semantics
        try? delete(key: key)

        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecAttrService as String: service,
            kSecAttrAccessGroup as String: accessGroup,
            kSecValueData as String: data,
            kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly,
            kSecUseDataProtectionKeychain as String: true
        ]

        let status = SecItemAdd(query as CFDictionary, nil)
        guard status == errSecSuccess else {
            throw KeychainError.unexpectedStatus(status)
        }
    }

    static func load(key: String) throws -> Data {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecAttrService as String: service,
            kSecAttrAccessGroup as String: accessGroup,
            kReturnData as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne,
            kSecUseDataProtectionKeychain as String: true
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)
        guard status == errSecSuccess, let data = result as? Data else {
            throw KeychainError.unexpectedStatus(status)
        }
        return data
    }

    static func delete(key: String) throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: key,
            kSecAttrService as String: service,
            kSecAttrAccessGroup as String: accessGroup,
            kSecUseDataProtectionKeychain as String: true
        ]

        let status = SecItemDelete(query as CFDictionary)
        // errSecItemNotFound is acceptable — item didn't exist
        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw KeychainError.unexpectedStatus(status)
        }
    }
}
```

**Usage from the main app (reads API tokens written by the helper):**

```swift
// The helper, running as root, writes an API token after privileged setup
// The app reads it non-privately

func readAPIToken() -> String? {
    guard let data = try? SharedKeychain.load(key: "api-token"),
          let token = String(data: data, encoding: .utf8)
    else { return nil }
    return token
}
```

**Usage from the helper tool (writes tokens the app can read):**

```swift
// Running as root — generate token, store in shared keychain
func setupAPIToken() throws {
    let token = UUID().uuidString
    guard let data = token.data(using: .utf8) else { return }
    try SharedKeychain.store(key: "api-token", data: data)
    // Main app can now read this via SharedKeychain.load(key: "api-token")
}
```

**Entitlements** (both app and helper `.entitlements`):

```xml
<key>com.apple.security.application-groups</key>
<array>
    <string>TEAMID.com.example.MyAppSuite</string>
</array>
```

## Touch ID with Apple Watch Unlock Fallback

This pattern tries Touch ID first, falling back to Apple Watch unlock, and finally to device passcode.

```swift
import LocalAuthentication

enum AuthenticationResult {
    case success(method: String)
    case cancelled
    case notAvailable
    case lockout
    case error(Error)
}

class BiometricAuthenticator {

    /// Authenticates with biometric, preferring Touch ID but accepting Apple Watch.
    /// Falls back to device passcode if biometrics fail.
    func authenticate(reason: String) async -> AuthenticationResult {
        let context = LAContext()
        context.localizedFallbackTitle = "Use Password"
        context.localizedCancelTitle = "Cancel"

        // Check what biometrics are available
        var error: NSError?
        let canEvaluate = context.canEvaluatePolicy(
            .deviceOwnerAuthenticationWithBiometrics,
            error: &error
        )

        if !canEvaluate {
            return .notAvailable
        }

        // Use deviceOwnerAuthentication which includes passcode fallback.
        // The system will try Touch ID first, then Apple Watch, then passcode
        // if .deviceOwnerAuthWithBiometrics fails.
        do {
            let success = try await context.evaluatePolicy(
                .deviceOwnerAuthentication,
                localizedReason: reason
            )
            return success ? .success(method: "biometric_or_passcode") : .cancelled
        } catch let laError as LAError {
            switch laError.code {
            case .userCancel, .systemCancel, .appCancel:
                return .cancelled
            case .biometryNotEnrolled:
                return .notAvailable
            case .biometryLockout:
                return .lockout
            case .userFallback:
                // User tapped "Use Password" — retry with passcode-only policy
                return await authenticateWithPasscode(reason: reason)
            default:
                return .error(laError)
            }
        } catch {
            return .error(error)
        }
    }

    /// Passcode-only authentication (after biometric fallback)
    private func authenticateWithPasscode(reason: String) async -> AuthenticationResult {
        let context = LAContext()
        context.localizedCancelTitle = "Cancel"

        do {
            let success = try await context.evaluatePolicy(
                .deviceOwnerAuthentication,  // On macOS, this includes passcode
                localizedReason: reason
            )
            return success ? .success(method: "passcode") : .cancelled
        } catch {
            if let laError = error as? LAError,
               [.userCancel, .systemCancel, .appCancel].contains(laError.code) {
                return .cancelled
            }
            return .error(error)
        }
    }

    /// Checks biometric type availability (informational only)
    var biometricType: BiometricType {
        let context = LAContext()
        var error: NSError?
        guard context.canEvaluatePolicy(
            .deviceOwnerAuthenticationWithBiometrics,
            error: &error
        ) else {
            return .none
        }
        switch context.biometryType {
        case .touchID:
            return .touchID
        case .faceID:
            return .faceID
        default:
            return .none
        }
    }
}

enum BiometricType {
    case none
    case touchID
    case faceID
}
```

**SwiftUI integration:**

```swift
struct ProtectedView: View {
    @State private var isUnlocked = false
    @State private var errorMessage: String?
    private let authenticator = BiometricAuthenticator()

    var body: some View {
        Group {
            if isUnlocked {
                SecretContentView()
            } else {
                VStack {
                    Text("Authentication Required")
                        .font(.headline)

                    Button("Unlock") {
                        Task {
                            let result = await authenticator.authenticate(
                                reason: "Access your encrypted data"
                            )
                            switch result {
                            case .success(let method):
                                isUnlocked = true
                                print("Unlocked via \(method)")
                            case .cancelled:
                                break
                            case .notAvailable:
                                errorMessage = "Biometrics not available"
                            case .lockout:
                                errorMessage = "Biometrics locked out. Use password."
                            case .error(let error):
                                errorMessage = error.localizedDescription
                            }
                        }
                    }
                    .keyboardShortcut(.defaultAction)

                    if let error = errorMessage {
                        Text(error)
                            .foregroundColor(.red)
                            .font(.caption)
                    }
                }
                .frame(maxWidth: .infinity, maxHeight: .infinity)
            }
        }
    }
}
```

## SecureEnclave Key Generation and Signing

Complete pattern for generating an ECDSA key in the SecureEnclave, persisting it by reference in the Keychain, and using it for signing operations with biometric gating.

```swift
import Foundation
import Security
import CryptoKit

enum SecureEnclaveKeyManager {
    static let keyTag = "com.example.MyApp.signing-key".data(using: .utf8)!

    /// Generate or retrieve the SecureEnclave signing key.
    /// The key is stored in the Keychain and requires Touch ID to use.
    static func getOrCreateSigningKey() throws -> SecKey {
        // Try to retrieve existing key
        if let existing = try? retrieveKey() {
            return existing
        }

        // Generate new key
        let key = try generateKey()
        return key
    }

    private static func generateKey() throws -> SecKey {
        guard SecureEnclave.isAvailable else {
            throw SecureEnclaveError.notAvailable
        }

        var error: Unmanaged<CFError>?

        guard let accessControl = SecAccessControlCreateWithFlags(
            nil,
            kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
            [.biometryCurrentSet, .privateKeyUsage],
            &error
        ) else {
            throw SecureEnclaveError.accessControl(error?.takeRetainedValue())
        }

        let attributes: [String: Any] = [
            kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,
            kSecAttrKeySizeInBits as String: 256,
            kSecAttrTokenID as String: kSecAttrTokenIDSecureEnclave,
            kSecPrivateKeyAttrs: [
                kSecAttrIsPermanent as String: true,
                kSecAttrApplicationTag as String: keyTag,
                kSecAttrAccessControl as String: accessControl,
                kSecUseOperationPrompt as String: "Authenticate to sign data"
            ]
        ]

        guard let privateKey = SecKeyCreateRandomKey(attributes as CFDictionary, &error) else {
            throw SecureEnclaveError.generation(error?.takeRetainedValue())
        }

        return privateKey
    }

    private static func retrieveKey() throws -> SecKey {
        let query: [String: Any] = [
            kSecClass as String: kSecClassKey,
            kSecAttrApplicationTag as String: keyTag,
            kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,
            kSecReturnRef as String: true
        ]

        var result: AnyObject?
        let status = SecItemCopyMatching(query as CFDictionary, &result)

        guard status == errSecSuccess, let key = result else {
            throw SecureEnclaveError.keyNotFound(status)
        }

        return (key as! SecKey)
    }

    /// Sign data using the SecureEnclave key. Triggers Touch ID prompt.
    static func sign(data: Data) throws -> Data {
        let privateKey = try getOrCreateSigningKey()

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

    /// Delete the SecureEnclave key (for key rotation)
    static func deleteKey() throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassKey,
            kSecAttrApplicationTag as String: keyTag
        ]
        let status = SecItemDelete(query as CFDictionary)
        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw SecureEnclaveError.deletion(status)
        }
    }
}

enum SecureEnclaveError: Error {
    case notAvailable
    case accessControl(Unmanaged<CFError>?)
    case generation(Unmanaged<CFError>?)
    case keyNotFound(OSStatus)
    case algorithmNotSupported
    case signing(Unmanaged<CFError>?)
    case deletion(OSStatus)
}
```

**Usage in async context:**

```swift
func signRequest(payload: Data) async throws -> Data {
    // This triggers Touch ID / Apple Watch prompt automatically
    return try await Task.detached(priority: .userInitiated) {
        try SecureEnclaveKeyManager.sign(data: payload)
    }.value
}
```

## CryptoKit: AES-GCM + HKDF Complete Example

This pattern shows the full lifecycle: key derivation from a password via HKDF, AES-GCM authenticated encryption, and chained secure erasure of sensitive material.

```swift
import CryptoKit
import Foundation

struct CryptoVault {

    /// Encrypts plaintext using AES-256-GCM with a key derived from a password
    /// and HKDF. Returns all components needed for decryption.
    static func encrypt(
        plaintext: Data,
        password: String,
        salt: Data? = nil
    ) throws -> EncryptedBlob {
        // Generate or use provided salt
        let actualSalt = salt ?? Data((0..<32).map { _ in UInt8.random(in: 0...255) })

        // Derive a 256-bit key from the password using HKDF
        let passwordData = Data(password.utf8)
        let passwordKey = SymmetricKey(data: passwordData)
        let derivedKey = HKDF<SHA256>.deriveKey(
            inputKeyMaterial: passwordKey,
            salt: actualSalt,
            info: Data("com.example.MyApp.encryption".utf8),
            outputByteCount: 32
        )

        // AES-GCM seal (automatically generates random nonce)
        let sealedBox = try AES.GCM.seal(plaintext, using: derivedKey, nonce: nil)

        // Zero out password material
        // (Swift value types — not perfect, but good practice)

        return EncryptedBlob(
            ciphertext: sealedBox.ciphertext,
            nonce: Data(sealedBox.nonce),
            tag: sealedBox.tag,
            salt: actualSalt
        )
    }

    /// Decrypts an EncryptedBlob using the same password and HKDF derivation path.
    static func decrypt(blob: EncryptedBlob, password: String) throws -> Data {
        let passwordData = Data(password.utf8)
        let passwordKey = SymmetricKey(data: passwordData)
        let derivedKey = HKDF<SHA256>.deriveKey(
            inputKeyMaterial: passwordKey,
            salt: blob.salt,
            info: Data("com.example.MyApp.encryption".utf8),
            outputByteCount: 32
        )

        let nonce = try AES.GCM.Nonce(data: blob.nonce)
        let sealedBox = try AES.GCM.SealedBox(
            nonce: nonce,
            ciphertext: blob.ciphertext,
            tag: blob.tag
        )

        return try AES.GCM.open(sealedBox, using: derivedKey)
    }

    /// Two-layer encryption: encrypt with ephemeral key, then encrypt that key
    /// with a password-derived key. Enables key rotation without re-encrypting
    /// the full payload.
    static func encryptWithEnvelope(
        plaintext: Data,
        password: String
    ) throws -> EnvelopeEncryptedBlob {
        // Layer 2 (outer): derive password key
        let salt = Data((0..<32).map { _ in UInt8.random(in: 0...255) })
        let passwordData = Data(password.utf8)
        let passwordKey = SymmetricKey(data: passwordData)
        let wrappingKey = HKDF<SHA256>.deriveKey(
            inputKeyMaterial: passwordKey,
            salt: salt,
            info: Data("envelope-wrap".utf8),
            outputByteCount: 32
        )

        // Layer 1 (inner): generate random content key
        let contentKey = SymmetricKey(size: .bits256)
        let sealedBox = try AES.GCM.seal(plaintext, using: contentKey)

        // Wrap content key with password-derived key
        let contentKeyRaw = contentKey.withUnsafeBytes { Data($0) }
        let contentKeyData = Data(contentKeyRaw)  // Copy for wrapping
        let wrappedKey = try AES.GCM.seal(contentKeyData, using: wrappingKey)

        return EnvelopeEncryptedBlob(
            wrappedKey: WrappedKey(
                ciphertext: wrappedKey.ciphertext,
                nonce: Data(wrappedKey.nonce),
                tag: wrappedKey.tag
            ),
            ciphertext: sealedBox.ciphertext,
            nonce: Data(sealedBox.nonce),
            tag: sealedBox.tag,
            salt: salt
        )
    }

    static func decryptEnvelope(
        blob: EnvelopeEncryptedBlob,
        password: String
    ) throws -> Data {
        // Unwrap content key
        let passwordData = Data(password.utf8)
        let passwordKey = SymmetricKey(data: passwordData)
        let wrappingKey = HKDF<SHA256>.deriveKey(
            inputKeyMaterial: passwordKey,
            salt: blob.salt,
            info: Data("envelope-wrap".utf8),
            outputByteCount: 32
        )

        let wrapNonce = try AES.GCM.Nonce(data: blob.wrappedKey.nonce)
        let wrappedBox = try AES.GCM.SealedBox(
            nonce: wrapNonce,
            ciphertext: blob.wrappedKey.ciphertext,
            tag: blob.wrappedKey.tag
        )
        let contentKeyData = try AES.GCM.open(wrappedBox, using: wrappingKey)
        let contentKey = SymmetricKey(data: contentKeyData)

        // Decrypt content
        let contentNonce = try AES.GCM.Nonce(data: blob.nonce)
        let contentBox = try AES.GCM.SealedBox(
            nonce: contentNonce,
            ciphertext: blob.ciphertext,
            tag: blob.tag
        )
        return try AES.GCM.open(contentBox, using: contentKey)
    }
}

/// Simple encrypted blob (single-layer AES-GCM with HKDF)
struct EncryptedBlob: Codable {
    let ciphertext: Data
    let nonce: Data       // 12 bytes for AES-GCM
    let tag: Data          // 16 bytes for AES-GCM
    let salt: Data         // 32 bytes for HKDF
}

/// Envelope-encrypted blob (two-layer AES-GCM with key wrapping)
struct EnvelopeEncryptedBlob: Codable {
    let wrappedKey: WrappedKey
    let ciphertext: Data
    let nonce: Data
    let tag: Data
    let salt: Data
}

struct WrappedKey: Codable {
    let ciphertext: Data
    let nonce: Data
    let tag: Data
}
```

**Usage:**

```swift
// Simple encryption
let blob = try CryptoVault.encrypt(
    plaintext: sensitiveData,
    password: userPassword
)

// Serialize to disk or Keychain
let encoded = try JSONEncoder().encode(blob)

// Decrypt later
let loaded = try JSONDecoder().decode(EncryptedBlob.self, from: encoded)
let decrypted = try CryptoVault.decrypt(blob: loaded, password: userPassword)
```

## Passkey Integration Pattern

Complete passkey registration and authentication flow on macOS 26+ with server coordination.

```swift
import AuthenticationServices

class PasskeyManager: NSObject {
    let relyingPartyIdentifier: String
    let authenticationDomain: String

    init(relyingPartyID: String, domain: String) {
        self.relyingPartyIdentifier = relyingPartyID
        self.authenticationDomain = domain
        super.init()
    }

    // MARK: - Registration

    /// Begin passkey registration. The challenge and userID come from your server.
    func beginRegistration(challenge: Data, userID: Data, userName: String) {
        let provider = ASAuthorizationPlatformPublicKeyCredentialProvider(
            relyingPartyIdentifier: relyingPartyIdentifier
        )

        let registrationRequest = provider.createCredentialRegistrationRequest(
            challenge: challenge,
            name: userName,
            userID: userID
        )

        // On macOS, the authenticator is iCloud Keychain
        // The user sees a system prompt to create the passkey

        let controller = ASAuthorizationController(
            authorizationRequests: [registrationRequest]
        )
        controller.delegate = registrationDelegate
        controller.presentationContextProvider = self
        controller.performRequests()
    }

    // MARK: - Authentication

    /// Begin passkey authentication. The challenge comes from your server.
    func beginAuthentication(challenge: Data, allowedCredentials: [Data] = []) {
        let provider = ASAuthorizationPlatformPublicKeyCredentialProvider(
            relyingPartyIdentifier: relyingPartyIdentifier
        )

        let assertionRequest = provider.createCredentialAssertionRequest(
            challenge: challenge
        )

        // Optional: restrict to specific credentials (e.g., from server)
        if !allowedCredentials.isEmpty {
            assertionRequest.allowedCredentials = allowedCredentials.map { credID in
                ASAuthorizationPlatformPublicKeyCredentialDescriptor(
                    credentialID: credID
                )
            }
        }

        let controller = ASAuthorizationController(
            authorizationRequests: [assertionRequest]
        )
        controller.delegate = authenticationDelegate
        controller.presentationContextProvider = self
        controller.performRequests()
    }

    // MARK: - Delegates

    private lazy var registrationDelegate = RegistrationDelegate()
    private lazy var authenticationDelegate = AuthenticationDelegate()
}

// MARK: - Registration Delegate

class RegistrationDelegate: NSObject, ASAuthorizationControllerDelegate {

    func authorizationController(
        controller: ASAuthorizationController,
        didCompleteWithAuthorization authorization: ASAuthorization
    ) {
        guard let credential = authorization.credential
            as? ASAuthorizationPlatformPublicKeyCredentialRegistration
        else {
            return
        }

        let attestationObject = credential.rawAttestationObject   // Send to server
        let clientDataJSON = credential.rawClientDataJSON         // Send to server
        let credentialID = credential.credentialID               // Send to server

        // POST to your server:
        // POST /api/passkeys/register
        // { credentialID, attestationObject, clientDataJSON }
    }

    func authorizationController(
        controller: ASAuthorizationController,
        didCompleteWithError error: Error
    ) {
        if let asError = error as? ASAuthorizationError {
            switch asError.code {
            case .canceled:
                print("User cancelled passkey registration")
            case .failed:
                print("Passkey registration failed")
            case .invalidResponse:
                print("Invalid response from authenticator")
            case .notHandled:
                print("Request not handled by authenticator")
            default:
                print("Unknown error: \(asError)")
            }
        }
    }
}

// MARK: - Authentication Delegate

class AuthenticationDelegate: NSObject, ASAuthorizationControllerDelegate {

    func authorizationController(
        controller: ASAuthorizationController,
        didCompleteWithAuthorization authorization: ASAuthorization
    ) {
        guard let credential = authorization.credential
            as? ASAuthorizationPlatformPublicKeyCredentialAssertion
        else {
            return
        }

        let signature = credential.signature           // Send to server
        let authenticatorData = credential.authenticatorData  // Send to server
        let clientDataJSON = credential.rawClientDataJSON     // Send to server
        let credentialID = credential.credentialID           // Send to server
        let userID = credential.userID                       // Send to server

        // POST to your server:
        // POST /api/passkeys/verify
        // { credentialID, signature, authenticatorData, clientDataJSON, userID }
    }

    func authorizationController(
        controller: ASAuthorizationController,
        didCompleteWithError error: Error
    ) {
        if let asError = error as? ASAuthorizationError {
            switch asError.code {
            case .canceled:
                print("User cancelled passkey authentication")
            default:
                print("Authentication error: \(asError)")
            }
        }
    }
}

// MARK: - Presentation Context

extension PasskeyManager: ASAuthorizationControllerPresentationContextProviding {
    func presentationAnchor(for controller: ASAuthorizationController) -> ASPresentationAnchor {
        // Return the key window on macOS
        NSApplication.shared.windows.first ?? ASPresentationAnchor()
    }
}
```

**Required setup for passkeys:**

1. **Entitlement** (in your `.entitlements` file):
   ```xml
   <key>com.apple.developer.associated-domains</key>
   <array>
       <string>webcredentials:example.com</string>
       <!-- Also add applinks if using universal links: -->
       <string>applinks:example.com</string>
   </array>
   ```

2. **Apple App Site Association file** (hosted at `https://example.com/.well-known/apple-app-site-association`):
   ```json
   {
     "webcredentials": {
       "apps": ["TEAMID.com.example.MyApp"]
     },
     "applinks": {
       "details": [
         {
           "appID": "TEAMID.com.example.MyApp",
           "paths": ["*"]
         }
       ]
     }
   }
   ```
   Note: No file extension. Served with `Content-Type: application/json`. Must be HTTPS with a valid certificate.

3. **Server responsibilities:**
   - Generate random challenges (32+ bytes) per request and store them server-side
   - Verify attestation objects during registration (using your preferred WebAuthn library)
   - Verify signatures during authentication against stored public keys
   - Bind challenges to the user session to prevent replay attacks

## Quantum-Secure Ciphersuites

As of WWDC25 / iOS 26, CryptoKit exposes post-quantum ciphersuites with formal verification in corecrypto. Apple open-sourced the production PQ crypto code (May 2026), including FIPS 203 (ML-KEM) and FIPS 204 (ML-DSA).

Available `HPKE.Ciphersuite` options:

| Ciphersuite constant | KEM | PQ? |
|---|---|---|
| `.XWingMLKEM768X25519_SHA256_AES_GCM_256` | X-Wing (ML-KEM-768 + X25519) | **Yes** |
| `.P256_SHA256_AES_GCM_256` | NIST P-256 | No |
| `.P384_SHA384_AES_GCM_256` | NIST P-384 | No |
| `.Curve25519_SHA256_ChaChaPoly` | X25519 | No |

Simple round-trip test pattern:

```swift
import CryptoKit

func roundTripPQ(plaintext: Data) throws -> Data {
    // Generate an ephemeral keypair
    let recipientKey = Curve25519.KeyAgreement.PrivateKey()

    // Seal with post-quantum HPKE
    let sender = HPKE.Sender<Curve25519.KeyAgreement, AES.GCM, SHA256>(
        recipientKey: recipientKey.publicKey,
        ciphersuite: HPKE.Ciphersuite.XWingMLKEM768X25519_SHA256_AES_GCM_256
    )
    let ciphertext = try sender.seal(plaintext)

    // Open with recipient private key
    let recipient = HPKE.Recipient<Curve25519.KeyAgreement, AES.GCM, SHA256>(
        ciphersuite: HPKE.Ciphersuite.XWingMLKEM768X25519_SHA256_AES_GCM_256,
        recipientKey: recipientKey,
        encapsulatedKey: sender.encapsulatedKey
    )
    return try recipient.open(ciphertext)
}
```

Standalone PQ primitives for custom protocols:

```swift
// ML-KEM key encapsulation (FIPS 203)
let kemKey = try MLKEM768.PrivateKey()
let (encapsulatedKey, sharedSecret) = try kemKey.publicKey.encapsulate()
let recoveredSecret = try kemKey.decapsulate(encapsulatedKey)
assert(sharedSecret == recoveredSecret)

// Secure Enclave ML-KEM (hardware-bound + biometric-gated)
let seKem = try SecureEnclave.MLKEM768.PrivateKey()

// ML-DSA signatures (FIPS 204)
let dsaKey = try MLDSA65.PrivateKey()
let signature = try dsaKey.signature(for: Data("message".utf8))
_ = dsaKey.publicKey.isValidSignature(signature, for: Data("message".utf8))
```

## Apple Watch companion policy migration notes

The Xcode 26 SDK (macOS 15+) soft-deprecates "watch" LAPoliciy names in favor of "companion":

| Deprecated | Preferred (macOS 15+) |
|---|---|
| `.deviceOwnerAuthenticationWithWatch` | `.deviceOwnerAuthenticationWithCompanion` |
| `.deviceOwnerAuthenticationWithBiometricsOrWatch` | `.deviceOwnerAuthenticationWithBiometricsOrCompanion` |

Both old and new names work at runtime, but the new "companion" nomenclature covers Apple Watch and any future companion device. Adopt the new names in new code.
