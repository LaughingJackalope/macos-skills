# Entitlements & Notarization Reference

Complete entitlement reference tables, notarization troubleshooting guide, SMJobBless setup walkthrough, and SecureEnclave code patterns for macOS 26+.

## Contents

- [Complete Entitlement Reference](#complete-entitlement-reference)
- [Notarization Troubleshooting Guide](#notarization-troubleshooting-guide)
- [SMJobBless Full Setup Walkthrough](#smjobbless-full-setup-walkthrough)
- [SecureEnclave Patterns](#secureenclave-patterns)

## Complete Entitlement Reference

### Sandbox Entitlements

| Entitlement Key | Type | Description |
|---|---|---|
| `com.apple.security.app-sandbox` | Boolean | Enables App Sandbox. Must be `true` for any other sandbox entitlements to take effect. |
| `com.apple.security.network.client` | Boolean | Outbound network connections. |
| `com.apple.security.network.server` | Boolean | Incoming network connections. |
| `com.apple.security.files.user-selected.read-only` | Boolean | Read access to files the user selects in `NSOpenPanel`. |
| `com.apple.security.files.user-selected.read-write` | Boolean | Read/write access to files the user selects. |
| `com.apple.security.files.downloads.read-write` | Boolean | Read/write access to `~/Downloads`. |
| `com.apple.security.files.bookmarks.app-scope` | Boolean | Allows creating and resolving security-scoped bookmarks. |
| `com.apple.security.temporary-exception.files.home-relative-path.read-write` | Array | Specific home-relative paths for read/write. Use sparingly — each weakens the sandbox. |
| `com.apple.security.temporary-exception.files.absolute-path.read-write` | Array | Absolute paths for read/write. Even weaker — avoid in notarized apps. |
| `com.apple.security.device.camera` | Boolean | Camera access. Requires `NSCameraUsageDescription` in Info.plist. |
| `com.apple.security.device.microphone` | Boolean | Microphone access. Requires `NSMicrophoneUsageDescription`. |
| `com.apple.security.device.usb` | Boolean | USB device access. |
| `com.apple.security.print` | Boolean | Access to printer functionality. |
| `com.apple.security.automation.apple-events` | Boolean | Send Apple Events to other apps. Requires `NSAppleEventsUsageDescription`. |

### Hardened Runtime Entitlements

These opt out of specific hardened runtime protections. Only enable what you need.

| Entitlement Key | Type | Description | Notarization Impact |
|---|---|---|---|
| `com.apple.security.cs.allow-jit` | Boolean | Allows `mmap` with `PROT_EXEC` for JIT compilation. | Accepted — common for WebKit, JS engines. |
| `com.apple.security.cs.allow-unsigned-executable-memory` | Boolean | Allocates executable memory without code signing. | Accepted — broader than `allow-jit`. |
| `com.apple.security.cs.allow-dyld-environment-variables` | Boolean | `DYLD_*` env vars are respected (e.g., `DYLD_LIBRARY_PATH`). | **Rejected** in notarized builds. Remove before submission. |
| `com.apple.security.cs.debugger` | Boolean | Allows debuggers to attach. | **Rejected** in notarized builds. Debug-only. |
| `com.apple.security.cs.disable-library-validation` | Boolean | Load code signed by other teams. Needed for some plugins, audio units, camera drivers. | Accepted — but broadens attack surface. |
| `com.apple.security.cs.disable-executable-page-protection` | Boolean | Disables W^X. | Almost always rejected. Extremely rare legitimate use. |
| `com.apple.security.automation.apple-events` | Boolean | Send Apple Events to other processes. | Accepted. Requires purpose string. |
| `com.apple.security.get-task-allow` | Boolean | `true` allows debuggers (set by Xcode for debug builds). | **Always rejected** for notarized distribution builds. |

### iCloud Entitlements

| Entitlement Key | Type | Description |
|---|---|---|
| `com.apple.developer.icloud-container-identifiers` | Array | iCloud container identifiers (e.g., `iCloud.com.example.MyApp`). |
| `com.apple.developer.ubiquity-kvstore-identifier` | String | Ubiquitous key-value store identifier. Format: `TEAMID.bundle.id`. |
| `com.apple.developer.icloud-services` | Array | iCloud services: `CloudKit`, `CloudDocuments`, `KeyValueStore`. |

### Push Notifications

| Entitlement Key | Type | Description |
|---|---|---|
| `aps-environment` | String | `development` or `production`. Set automatically by the provisioning profile. |

### Associated Domains

| Entitlement Key | Type | Description |
|---|---|---|
| `com.apple.developer.associated-domains` | Array | Domains for universal links and shared web credentials. |

## Notarization Troubleshooting Guide

### Common Rejection Reasons

**"The binary is not signed with a valid signing certificate"**

```bash
# Verify the signing identity
codesign -dv --verbose=4 /path/to/MyApp.app 2>&1 | grep "Authority="
# Should show: Authority=Developer ID Application: ...
# NOT "Authority=ad hoc" or "Authority=3rd Party Mac Developer"
```

Fix: Re-sign with a Developer ID Application certificate:
```bash
codesign --force --deep --sign "Developer ID Application: Name (TEAMID)" \
    --options runtime --timestamp /path/to/MyApp.app
```

**"The signature does not include a secure timestamp"**

```bash
codesign --verify --verbose /path/to/MyApp.app
# Look for: "signed with fulltime timestamp"
```

Fix: Add `--timestamp` to your codesign invocation.

**"Unable to validate your app. The archive is not notarizable"**

Nested code is unsigned. Check every executable:
```bash
# Find all Mach-O binaries in the app
find /path/to/MyApp.app -type f -perm +111 | while read f; do
    file "$f" | grep -q "Mach-O" && codesign --verify "$f" 2>&1
done
```

**"Invalid code signing entitlements"**

The notary rejects builds with `com.apple.security.get-task-allow` set to `true`:
```bash
# Check for get-task-allow
codesign --display --entitlements - /path/to/MyApp.app 2>&1 | grep get-task-allow
```

Fix: Ensure your Release build configuration doesn't include the debug entitlement file:
```bash
# In your Release .entitlements file, make sure there's no:
# <key>com.apple.security.get-task-allow</key><true/>
```

### Polling Notarization Status

When not using `--wait`, track the submission:

```bash
# Submit without --wait
xcrun notarytool submit MyApp.zip --keychain-profile "AC_PROFILE"
# Output: Successfully uploaded submissions/abc123-def456

# Check status
xcrun notarytool info abc123-def456 --keychain-profile "AC_PROFILE"
# Status: In Progress / Accepted / Invalid / Rejected

# Fetch full log after completion
xcrun notarytool log abc123-def456 --keychain-profile "AC_PROFILE" notary_log.json

# View issues
cat notary_log.json | python3 -m json.tool | grep -A5 '"severity": "error"'
```

### Stapling Failures

**"The staple and validate action failed!"**

This usually means the notarization ticket hasn't propagated yet. Wait a few minutes and try again:
```bash
xcrun stapler staple MyApp.app
```

If it persists, re-download the ticket explicitly:
```bash
# Fetch the latest notarization info
xcrun notarytool info SUBMISSION_ID --keychain-profile "AC_PROFILE"

# Once status is "Accepted", retry stapling
xcrun stapler staple MyApp.app
```

**"Could not validate the installer"**

The staple is on the `.app` but you're distributing a `.dmg` or `.pkg`. The staple must be on the outermost artifact:
```bash
# For DMG distribution: staple the DMG, not the .app inside it
xcrun stapler staple MyApp.dmg

# For PKG distribution: staple the PKG
xcrun stapler staple MyApp.pkg
```

### DMG and PKG Notarization

For disk images, create a read-only or compressed DMG:
```bash
hdiutil create -volname "MyApp" -srcfolder MyApp.app \
    -ov -format UDZO MyApp.dmg

xcrun notarytool submit MyApp.dmg --keychain-profile "AC_PROFILE" --wait
xcrun stapler staple MyApp.dmg
```

For installer packages:
```bash
# Build the package (requires pkgbuild or productbuild in your build script)
pkgbuild --component MyApp.app \
    --install-location /Applications \
    --sign "Developer ID Installer: Name (TEAMID)" \
    MyApp.pkg

xcrun notarytool submit MyApp.pkg --keychain-profile "AC_PROFILE" --wait
xcrun stapler staple MyApp.pkg
```

Note: PKGs require a **Developer ID Installer** certificate (separate from Developer ID Application). Both are available in the Apple Developer portal.

## SMJobBless Full Setup Walkthrough

### Project Structure

```
MyApp/
  MyApp/                    ← Main app target
    Info.plist              ← App Info.plist
    MyApp.entitlements      ← App entitlements
    main.swift
  MyPrivilegeHelper/        ← Helper tool target (Cocoa App, stripped)
    Info.plist              ← Helper Info.plist with SMAuthorizedClients
    HelperProtocol.swift    ← Shared XPC protocol
    Helper.swift            ← Helper implementation
    main.swift              ← Helper entry point
  Shared/
    HelperProtocol.swift     ← Protocol shared between app and helper
```

### Step 1: Create the Helper Target

1. In Xcode, add a new target: **Cocoa App** named "MyPrivilegeHelper".
2. Delete the Main.storyboard, window controller, and AppDelegate UI code.
3. Replace `main.swift`:

```swift
// MyPrivilegeHelper/main.swift
import Foundation

class HelperDelegate: NSObject, NSXPCListenerDelegate {
    func listener(_ listener: NSXPCListener,
                  shouldAcceptNewConnection newConnection: NSXPCConnection) -> Bool {
        newConnection.exportedInterface = NSXPCInterface(with: HelperProtocol.self)
        newConnection.exportedObject = Helper()
        newConnection.resume()
        return true
    }
}

let delegate = HelperDelegate()
let listener = NSXPCListener(machServiceName: "TEAMID.com.example.MyPrivilegeHelper")
listener.delegate = delegate
listener.resume()
RunLoop.main.run()
```

### Step 2: Define the Shared Protocol

```swift
// Shared/HelperProtocol.swift
import Foundation

@objc protocol HelperProtocol {
    func installSystemComponent(at path: String,
                               reply: @escaping (Bool, String?) -> Void)
    func getSystemVersion(reply: @escaping (String) -> Void)
}
```

### Step 3: Implement the Helper

```swift
// MyPrivilegeHelper/Helper.swift
import Foundation

class Helper: NSObject, HelperProtocol {
    func installSystemComponent(at path: String,
                               reply: @escaping (Bool, String?) -> Void) {
        // Running as root — perform privileged operation
        let process = Process()
        process.launchPath = "/usr/bin/installer"
        process.arguments = ["-pkg", path, "-target", "/"]

        let pipe = Pipe()
        process.standardOutput = pipe
        process.standardError = pipe

        do {
            try process.run()
            process.waitUntilExit()

            let data = pipe.fileHandleForReading.readDataToEndOfFile()
            let output = String(data: data, encoding: .utf8)

            reply(process.terminationStatus == 0, output)
        } catch {
            reply(false, error.localizedDescription)
        }
    }

    func getSystemVersion(reply: @escaping (String) -> Void) {
        let version = ProcessInfo.processInfo.operatingSystemVersion
        reply("\(version.majorVersion).\(version.minorVersion).\(version.patchVersion)")
    }
}
```

### Step 4: Configure Helper Info.plist

```xml
<!-- MyPrivilegeHelper/Info.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
    <key>CFBundleIdentifier</key>
    <string>TEAMID.com.example.MyPrivilegeHelper</string>
    <key>CFBundleName</key>
    <string>MyPrivilegeHelper</string>
    <key>SMAuthorizedClients</key>
    <array>
        <string>anchor apple generic and identifier "TEAMID.com.example.MyApp" and
            (certificate leaf[field.1.2.840.113635.100.6.1.9] /* exists */ or
             certificate 1[field.1.2.840.113635.100.6.2.6] /* exists */ and
             certificate leaf[field.1.2.840.113635.100.6.1.13] /* exists */ and
             certificate leaf[subject.OU] = TEAMID)</string>
    </array>
</dict>
</plist>
```

The `SMAuthorizedClients` entries are code-signing requirement strings. The long format ensures only your team's properly signed app can connect. You can also use the simpler form `TEAMID.com.example.MyApp` but the requirement-string format is more secure.

### Step 5: Copy the Helper into the Main App

Add a Copy Files build phase to the main app target:
- Destination: Wrapper
- Subpath: `Contents/Library/LoginItems`
- Copy the built `MyPrivilegeHelper.app` product.

### Step 6: Install and Connect from the Main App

```swift
// MyApp/HelperManager.swift
import Foundation
import ServiceManagement

final class HelperManager: ObservableObject {
    private var connection: NSXPCConnection?

    var isHelperInstalled: Bool {
        guard let helperURL = bundledHelperURL else { return false }
        return FileManager.default.fileExists(atPath: helperURL.path)
    }

    private var bundledHelperURL: URL? {
        Bundle.main.bundleURL
            .appendingPathComponent("Contents/Library/LoginItems/MyPrivilegeHelper.app")
    }

    func installHelper() async throws {
        guard let helperURL = bundledHelperURL else {
            throw HelperError.helperNotFound
        }

        return try await withCheckedThrowingContinuation { continuation in
            let auth = SFAuthorization.authorization() as! AuthorizationRef

            var authRef: AuthorizationRef?
            let status = AuthorizationCreate(nil, nil, [.preAuthorize], &authRef)

            guard status == errAuthorizationSuccess, let authRef = authRef else {
                continuation.resume(throwing: HelperError.authorizationFailed)
                return
            }

            var error: Unmanaged<CFError>?
            let success = SMJobBless(
                kSMDomainSystemLaunchd,
                "TEAMID.com.example.MyPrivilegeHelper" as CFString,
                authRef,
                &error
            )

            AuthorizationFree(authRef, [.destroyRights])

            if success {
                continuation.resume()
            } else {
                let error = error?.takeRetainedValue()
                continuation.resume(throwing: error ?? HelperError.blessFailed)
            }
        }
    }

    func connect() {
        guard connection == nil else { return }
        let conn = NSXPCConnection(
            machServiceName: "TEAMID.com.example.MyPrivilegeHelper",
            options: []
        )
        conn.remoteObjectInterface = NSXPCInterface(with: HelperProtocol.self)
        conn.invalidationHandler = { [weak self] in
            self?.connection = nil
        }
        conn.resume()
        connection = conn
    }

    func installPkg(at url: URL) async throws -> String {
        connect()
        guard let helper = connection?.remoteObjectProxyWithErrorHandler({ error in
            print("XPC error: \(error)")
        }) as? HelperProtocol else {
            throw HelperError.notConnected
        }

        return try await withCheckedThrowingContinuation { continuation in
            helper.installSystemComponent(at: path.path) { success, output in
                if success {
                    continuation.resume(returning: output ?? "OK")
                } else {
                    continuation.resume(throwing: HelperError.installFailed(output))
                }
            }
        }
    }
}

enum HelperError: Error {
    case helperNotFound
    case authorizationFailed
    case blessFailed
    case notConnected
    case installFailed(String?)
}
```

### Build Settings for Helper Target

Key build settings for the helper:
- **Skip Install**: `NO` (the helper installs itself via SMJobBless, not via `make install`)
- **Installation Directory**: `/Library/PrivilegedHelperTools` (SMJobBless handles this)
- **Code Signing Entitlements**: A separate entitlements file with minimal capabilities — the helper should not be sandboxed (privileged helpers cannot be sandboxed)
- **Hardened Runtime**: Enabled, but the helper doesn't need sandbox entitlements

## SecureEnclave Patterns

### Pattern 1: Sign-In with SecureEnclave Key + Biometric Gate

```swift
import CryptoKit
import LocalAuthentication
import Security

actor SecureEnclaveManager {
    static let shared = SecureEnclaveManager()
    private let keyLabel = "com.example.MyApp.userSigningKey"

    /// Generate a new SecureEnclave key, protected by biometrics.
    func generateKey(biometryRequired: Bool = true) throws -> SecKey {
        var flags: SecAccessControlCreateFlags = .privateKeyUsage
        if biometryRequired {
            flags.insert(.biometryCurrentSet)
        } else {
            flags.insert(.devicePasscode)
        }

        guard let accessControl = SecAccessControlCreateWithFlags(
            nil,
            kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
            flags,
            nil
        ) else {
            throw SecureEnclaveError.accessControlCreationFailed
        }

        let attributes: [String: Any] = [
            kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,
            kSecAttrKeySizeInBits as String: 256,
            kSecAttrTokenID as String: kSecAttrTokenIDSecureEnclave,
            kSecPrivateKeyAttrs as String: [
                kSecAttrIsPermanent as String: true,
                kSecAttrAccessControl as String: accessControl,
                kSecAttrLabel as String: keyLabel
            ]
        ]

        var error: Unmanaged<CFError>?
        guard let key = SecKeyCreateRandomKey(attributes as CFDictionary, &error) else {
            throw error?.takeRetainedValue() ?? SecureEnclaveError.keyGenerationFailed
        }
        return key
    }

    /// Retrieve the existing key. System prompts for biometrics automatically.
    func fetchKey() throws -> SecKey {
        let query: [String: Any] = [
            kSecClass as String: kSecClassKey,
            kSecAttrLabel as String: keyLabel,
            kSecReturnRef as String: true,
            kSecMatchLimit as String: kSecMatchLimitOne
        ]

        var item: CFTypeRef?
        let status = SecItemCopyMatching(query as CFDictionary, &item)

        switch status {
        case errSecSuccess:
            return (item as! SecKey)
        case errSecItemNotFound:
            return try generateKey()
        default:
            throw SecureEnclaveError.keychainError(status)
        }
    }

    /// Sign data using the SecureEnclave key.
    func sign(data: Data) throws -> Data {
        let key = try fetchKey()

        guard SecKeyIsAlgorithmSupported(key, .messageSignature, .ecdsaSignatureMessageX962SHA256) else {
            throw SecureEnclaveError.unsupportedAlgorithm
        }

        var error: Unmanaged<CFError>?
        guard let signature = SecKeyCreateSignature(
            key,
            .ecdsaSignatureMessageX962SHA256,
            data as CFData,
            &error
        ) else {
            throw error?.takeRetainedValue() ?? SecureEnclaveError.signingFailed
        }
        return signature as Data
    }

    /// Get the public key for verification by a server.
    func publicKeyData() throws -> Data {
        let key = try fetchKey()
        guard let publicKey = SecKeyCopyPublicKey(key) else {
            throw SecureEnclaveError.noPublicKey
        }

        var error: Unmanaged<CFError>?
        guard let data = SecKeyCopyExternalRepresentation(publicKey, &error) else {
            throw error?.takeRetainedValue() ?? SecureEnclaveError.publicKeyExportFailed
        }
        return data as Data
    }

    /// Delete the key (e.g., on user logout).
    func deleteKey() throws {
        let query: [String: Any] = [
            kSecClass as String: kSecClassKey,
            kSecAttrLabel as String: keyLabel
        ]
        let status = SecItemDelete(query as CFDictionary)
        guard status == errSecSuccess || status == errSecItemNotFound else {
            throw SecureEnclaveError.keychainError(status)
        }
    }
}

enum SecureEnclaveError: Error {
    case accessControlCreationFailed
    case keyGenerationFailed
    case keychainError(OSStatus)
    case unsupportedAlgorithm
    case signingFailed
    case noPublicKey
    case publicKeyExportFailed
}
```

### Pattern 2: SecureEnclave Key with Context-Based Authentication

Use `LAContext` to provide a custom authentication UI before the key operation:

```swift
func signWithCustomPrompt(data: Data, reason: String) async throws -> Data {
    let context = LAContext()

    // Evaluate policy first — this shows the custom prompt
    let success = try await context.evaluatePolicy(
        .deviceOwnerAuthenticationWithBiometrics,
        localizedReason: reason
    )
    guard success else {
        throw SecureEnclaveError.userCancelled
    }

    // The context can be passed to key operations on iOS, but on macOS
    // the access control flags on the key itself trigger the prompt.
    // Use the LAContext evaluation as a pre-flight check.
    return try await Task.detached(priority: .userInitiated) {
        try await SecureEnclaveManager.shared.sign(data: data)
    }.value
}
```

### Pattern 3: Verifying a SecureEnclave Signature

```swift
func verify(signature: Data, data: Data, publicKeyData: Data) throws -> Bool {
    let attributes: [String: Any] = [
        kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,
        kSecAttrKeyClass as String: kSecAttrKeyClassPublic,
        kSecAttrKeySizeInBits as String: 256
    ]

    var error: Unmanaged<CFError>?
    guard let publicKey = SecKeyCreateWithData(
        publicKeyData as CFData,
        attributes as CFDictionary,
        &error
    ) else {
        throw error?.takeRetainedValue() ?? SecureEnclaveError.invalidPublicKey
    }

    var verifyError: Unmanaged<CFError>?
    let valid = SecKeyVerifySignature(
        publicKey,
        .ecdsaSignatureMessageX962SHA256,
        data as CFData,
        signature as CFData,
        &verifyError
    )
    return valid
}
```

### Access Control Flag Reference

| Flag | Behavior |
|---|---|
| `.biometryAny` | Any enrolled biometrics can unlock. Key survives fingerprint/face changes. |
| `.biometryCurrentSet` | Key is invalidated if biometrics change. More secure — recommended default. |
| `.devicePasscode` | Device passcode can be used instead of biometrics. |
| `.userPresence` | Any user authentication (biometric or passcode). Broader than biometrics-only. |
| `.privateKeyUsage` | Required for SecureEnclave key signing operations. Must always be present. |

### Accessibility Constant Reference

| Constant | Behavior |
|---|---|
| `kSecAttrAccessibleWhenUnlocked` | Key accessible when device is unlocked. |
| `kSecAttrAccessibleWhenUnlockedThisDeviceOnly` | Same, but not included in backups. **Recommended for SecureEnclave keys.** |
| `kSecAttrAccessibleAfterFirstUnlock` | Accessible after first unlock following boot. |
| `kSecAttrAccessibleAfterFirstUnlockThisDeviceOnly` | Same, not included in backups. |
