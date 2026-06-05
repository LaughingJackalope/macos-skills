---
name: macos-security-entitlements
description: "Configure macOS sandbox, hardened runtime, code signing, notarization, SMJobBless privileged helpers, SecureEnclave key storage, and LAContext biometric authentication. Use when enabling app capabilities, signing and notarizing a macOS app, setting up XPC helper tools for elevated operations, storing keys in the Secure Enclave, or integrating Touch ID / Apple Watch unlock."
sources: https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution, https://help.apple.com/xcode/mac/current/en.lproj/devf87a2ac8f.html, https://developer.apple.com/documentation/bundleresources/entitlements, https://developer.apple.com/library/archive/technotes/tn2415/_index.html
researched: 2026-06-05
platform: macOS 26+
---

# macOS Security & Entitlements

Configure sandbox, hardened runtime, code signing, notarization, privileged helpers, SecureEnclave, and biometric authentication for macOS 26+ with Swift 6.3. This skill covers the full chain from entitlements configuration to distributing notarized software.

See [references/entitlements-notarization.md](references/entitlements-notarization.md) for a complete entitlement reference table, notarization troubleshooting guide, SMJobBless setup walkthrough, and SecureEnclave code patterns.

## Contents

- [Sandbox Entitlements](#sandbox-entitlements)
- [Hardened Runtime](#hardened-runtime)
- [Code Signing for Local Dev](#code-signing-for-local-dev)
- [Notarization Workflow](#notarization-workflow)
- [SMJobBless / Privileged Helper Tools](#smjobbless--privileged-helper-tools)
- [SecureEnclave on Apple Silicon](#secureenclave-on-apple-silicon)
- [LAContext on macOS](#lacontext-on-macos)
- [SIP Awareness](#sip-awareness)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Sandbox Entitlements

The App Sandbox restricts your app's access to system resources. Enable it via the `com.apple.security.app-sandbox` entitlement (Boolean `true`). Once sandboxed, you must declare specific entitlements for every resource your app touches.

**File Access:**

| Entitlement | Value | Use |
|---|---|---|
| `com.apple.security.files.user-selected.read-write` | `true` | Read/write files the user picks in an `NSOpenPanel`/`NSSavePanel` |
| `com.apple.security.files.user-selected.read-only` | `true` | Read-only access to user-selected files |
| `com.apple.security.files.downloads.read-write` | `true` | Access the user's Downloads folder |
| `com.apple.security.files.bookmarks.app-scope` | `true` | Persist file access across launches via security-scoped bookmarks |
| `com.apple.security.temporary-exception.files.home-relative-path.read-write` | `[String]` | Specific home-relative paths (avoid if possible — weakens sandbox) |

Security-scoped bookmarks are the correct way to persist file access. Without the `bookmarks.app-scope` entitlement, your app loses access to files between launches.

```swift
// Starting a security-scoped access session
if let bookmarkData = UserDefaults.standard.data(forKey: "projectBookmark") {
    var isStale = false
    do {
        let url = try NSURL(
            resolvingBookmarkData: bookmarkData,
            options: .withSecurityScope,
            relativeTo: nil,
            bookmarkDataIsStale: &isStale
        )
        if url.startAccessingSecurityScopedResource() {
            defer { url.stopAccessingSecurityScopedResource() }
            // Read/write the file at `url`
        }
        if isStale {
            // Re-create and store the bookmark
        }
    } catch {
        // Handle stale or invalid bookmark
    }
}
```

**Network:**

| Entitlement | Value | Use |
|---|---|---|
| `com.apple.security.network.client` | `true` | Outbound connections (HTTP clients, WebSocket, etc.) |
| `com.apple.security.network.server` | `true` | Incoming connections (local server, Bonjour, etc.) |

Both are independent — an app that only makes outbound requests needs only `client`. A local-only server (e.g., XPC over Mach) may not need either.

**Hardware:**

| Entitlement | Value | Use |
|---|---|---|
| `com.apple.security.device.camera` | `true` | Camera access |
| `com.apple.security.device.microphone` | `true` | Microphone access |
| `com.apple.security.device.usb` | `true` | USB device access |
| `com.apple.security.print` | `true` | Printing |

**Other common sandbox entitlements:**

- `com.apple.security.automation.apple-events` — `true` to send Apple Events to other apps (requires `NSAppleEventsUsageDescription` in Info.plist)
- `com.apple.security.cs.disable-library-validation` — `true` to load third-party plugins not signed by the same team (needed for some camera drivers, audio units)
- `com.apple.security.cs.allow-unsigned-executable-memory` — `true` for JIT compilation (JavaScriptCore, Lua interpreters)

## Hardened Runtime

Hardened Runtime is mandatory for notarization. It restricts your app to prevent code injection, tampering, and certain runtime exploits. Enable it in Xcode's Signing & Capabilities tab, or add `--options runtime` to your `codesign` invocation.

**Key restrictions and the entitlements that relax them:**

| Entitlement | When You Need It |
|---|---|
| `com.apple.security.cs.allow-jit` | Your app uses JIT compilation (JavaScriptCore, LuaJIT, WebKit). Without this, `mmap` with `PROT_EXEC` fails. |
| `com.apple.security.cs.allow-unsigned-executable-memory` | Your app allocates executable memory without code signing (some interpreters, plugin hosts). Superseded by `allow-jit` for most JIT use cases. |
| `com.apple.security.cs.debugger` | `true` to allow `lldb` or `gdb` to attach. Never enable for distribution builds — notarization rejects it. |
| `com.apple.security.cs.disable-library-validation` | Load plugins/frameworks signed by a different team. Also required for some third-party audio units and camera drivers. |
| `com.apple.security.cs.allow-dyld-environment-variables` | `true` to allow `DYLD_*` environment variables (e.g., `DYLD_LIBRARY_PATH`, `DYLD_INSERT_LIBRARIES`). Only for development — never in notarized builds. |
| `com.apple.security.automation.apple-events` | Send Apple Events to other apps. Requires the target app's bundle ID in the `NSAppleEventsUsageDescription` purpose string. |
| `com.apple.security.cs.disable-executable-page-protection` | Disables W^X enforcement on executable pages. Extremely rare — only for specialized runtime compilers. Not accepted for notarization in most cases. |

**Notarization impact:** The notary service checks for the `com.apple.security.get-task-allow` entitlement (set to `true` when debugging via Xcode). If present, notarization fails. Always remove it before submitting. Similarly, `allow-dyld-environment-variables` and `debugger` entitlements will cause rejection.

**Minimal hardened runtime for a typical app:**

```xml
<!-- MyApp.entitlements -->
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
</dict>
</plist>
```

## Code Signing for Local Dev

Three signing identity types matter for macOS development:

**Ad-hoc signing (`-`):** No identity required. The binary is signed with a placeholder. Use for quick local testing of sandbox and entitlements. Cannot be notarized, cannot use certain capabilities (Push Notifications, iCloud).

```bash
# Ad-hoc sign
codesign --force --deep --sign - /path/to/MyApp.app
```

**Developer ID Application certificate:** For distribution outside the Mac App Store. Required for notarization. Available to Apple Developer Program members. The certificate name looks like `Developer ID Application: Your Name (TEAM_ID)`.

```bash
# Sign with Developer ID
codesign --force --deep --sign "Developer ID Application: Your Name (TEAM_ID)" \
    --options runtime --timestamp /path/to/MyApp.app
```

**Mac App Store certificate:** For App Store distribution. The notary service is not needed — App Store submission includes equivalent checks. Certificate name: `3rd Party Mac Developer Application: Your Name (TEAM_ID)`.

```bash
# Sign for Mac App Store
codesign --force --deep --sign "3rd Party Mac Developer Application: Your Name (TEAM_ID)" \
    --entitlements MyApp.entitlements --timestamp /path/to/MyApp.app
```

**Key flags:**
- `--options runtime` — enables hardened runtime (required for notarization)
- `--timestamp` — includes a secure timestamp (required for notarization)
- `--deep` — signs nested code (frameworks, plugins). Apple recommends signing each component individually instead; use `--deep` only for quick validation.
- `--force` — replaces existing signature

**Verifying a signature:**

```bash
# Check code signature
codesign --verify --deep --strict --verbose=2 /path/to/MyApp.app

# Display embedded entitlements
codesign --display --entitlements - /path/to/MyApp.app

# Check notarization stapling
spctl --assess --type execute --verbose=4 /path/to/MyApp.app
```

## Notarization Workflow

Notarization is required for all Developer ID-distributed software on macOS 10.15+. The `notarytool` CLI (Xcode 14+) replaced the deprecated `altool`.

**Step 1: Prepare the archive.**

Zip the `.app` bundle (notarization works on `.app`, `.dmg`, `.pkg`, or `.zip`):

```bash
ditto -c -k --keepParent MyApp.app MyApp.zip
```

**Step 2: Submit to the notary service.**

Store your Apple ID credentials in the keychain (one-time setup):

```bash
xcrun notarytool store-credentials "AC_PASSWORD" \
    --apple-id "your@email.com" \
    --team-id "TEAMID123" \
    --password "abcd-efgh-ijkl-mnop"
```

The app-specific password is generated at appleid.apple.com. The profile name (`AC_PASSWORD`) is a local label — pick anything descriptive.

Submit:

```bash
xcrun notarytool submit MyApp.zip \
    --keychain-profile "AC_PASSWORD" \
    --wait
```

`--wait` blocks until notarization completes (typically 1–15 minutes). Without it, you get a `submission-id` and must poll with `notarytool log`.

**Step 3: Staple the ticket.**

Stapling attaches the notary ticket to the app so Gatekeeper can verify it offline:

```bash
xcrun stapler staple MyApp.app
```

Verify stapling:

```bash
spctl --assess --type execute --verbose=4 MyApp.app
# Should show: "accepted" and "source=Notarized Developer ID"
```

**Step 4: Validate before distribution.**

```bash
# Full validation chain
codesign --verify --deep --strict MyApp.app
spctl --assess --type execute MyApp.app
xcrun stapler validate MyApp.app
```

**Troubleshooting notarization failures:**

```bash
# Fetch the notarization log (use submission-id from the submit output)
xcrun notarytool log <submission-id> --keychain-profile "AC_PASSWORD" developer_log.json
```

Common rejection reasons:
- `get-task-allow` entitlement present → remove it from release builds
- Missing secure timestamp → add `--timestamp` to `codesign`
- Unsigned nested binaries → sign all executables, dylibs, and frameworks
- Hardened Runtime not enabled → add `--options runtime`
- `altool` used instead of `notarytool` → switch to `notarytool` (required since Nov 2023)

## SMJobBless / Privileged Helper Tools

SMJobBless is the supported mechanism for installing a privileged helper tool that runs as root. The helper is an XPC service bundled inside your app, installed via `SMJobBless` and launched by `launchd`.

**Architecture:**

```
MyApp.app/
  Contents/
    Library/
      LoginItems/
        MyHelper.app/          ← Helper bundle (app target, not a plain binary)
        Contents/
          Info.plist           ← SMAuthorizedClients array
          MacOS/
            MyHelper
    MacOS/
      MyApp
    XPC Services/              ← (alternative location for XPC services)
```

**Setup steps:**

1. **Create a helper target** in Xcode (Cocoa App template, then strip the UI). The helper must be a proper `.app` bundle.

2. **Configure the helper's Info.plist** with `SMAuthorizedClients` — an array of bundle identifiers allowed to connect:

```xml
<!-- Helper's Info.plist -->
<key>SMAuthorizedClients</key>
<array>
    <string>TEAMID.com.example.MyApp</string>
</array>
```

3. **Configure the app's Info.plist** with `SMLoginItemEnabled` if using the LoginItems approach, or use `SMJobBless` directly.

4. **Install the helper:**

```swift
import ServiceManagement

func installHelper() -> Bool
    let helperBundleID = "TEAMID.com.example.MyHelper"
    var authRef: AuthorizationRef?
    let status = AuthorizationCreate(nil, nil, [.preAuthorize], &authRef)
    guard status == errAuthorizationSuccess, let auth = authRef else {
        return false
    }
    defer { AuthorizationFree(auth, [.destroyRights]) }

    var error: Unmanaged<CFError>?
    let success = SMJobBless(
        kSMDomainSystemLaunchd,
        helperBundleID as CFString,
        auth,
        &error
    )
    if !success, let error = error?.takeRetainedValue() {
        print("SMJobBless failed: \(error)")
    }
    return success
}
```

5. **Communicate via XPC** using `NSXPCConnection`:

```swift
// In the main app
let connection = NSXPCConnection(machServiceName: "TEAMID.com.example.MyHelper", options: [])
connection.remoteObjectInterface = NSXPCInterface(with: HelperProtocol.self)
connection.resume()

let helper = connection.remoteObjectProxyWithErrorHandler { error in
    print("XPC error: \(error)")
} as? HelperProtocol

helper?.performPrivilegedTask { result in
    print("Helper returned: \(result)")
}
```

**Critical rules:**
- The helper and the app must be signed with the same team ID
- The helper's `SMAuthorizedClients` must contain the app's bundle ID
- The helper must be built as part of the same Xcode project (or at least signed with the same provisioning profile)
- Never use `AuthorizationExecuteWithPrivileges` — it's deprecated and not accepted for notarization
- The helper tool itself must also be notarized as part of the app bundle

## SecureEnclave on Apple Silicon

The Secure Enclave is a hardware-isolated coprocessor on Apple Silicon Macs. Private keys generated inside it never leave the enclave — the CPU can only request signing/decryption operations.

**Generating a SecureEnclave key:**

```swift
import CryptoKit
import Security
import LocalAuthentication

func generateSecureEnclaveKey() throws -> SecKey {
    let accessControl = SecAccessControlCreateWithFlags(
        nil,
        kSecAttrAccessibleWhenUnlockedThisDeviceOnly,
        [.privateKeyUsage, .biometryCurrentSet],  // Invalidated if biometrics change
        nil
    )!

    let attributes: [String: Any] = [
        kSecAttrKeyType as String: kSecAttrKeyTypeECSECPrimeRandom,
        kSecAttrKeySizeInBits as String: 256,
        kSecAttrTokenID as String: kSecAttrTokenIDSecureEnclave,
        kSecPrivateKeyAttrs as String: [
            kSecAttrIsPermanent as String: true,
            kSecAttrAccessControl as String: accessControl,
            kSecAttrLabel as String: "com.example.MyApp.signingKey"
        ]
    ]

    var error: Unmanaged<CFError>?
    guard let privateKey = SecKeyCreateRandomKey(attributes as CFDictionary, &error) else {
        throw error!.takeRetainedValue() as Error
    }
    return privateKey
}
```

**Using the key for signing:**

```swift
func sign(data: Data, using key: SecKey) throws -> Data {
    var error: Unmanaged<CFError>?
    guard let signature = SecKeyCreateSignature(
        key,
        .ecdsaSignatureMessageX962SHA256,
        data as CFData,
        &error
    ) else {
        throw error!.takeRetainedValue() as Error
    }
    return signature as Data
}
```

**Key points:**
- `kSecAttrTokenIDSecureEnclave` routes key generation to the Secure Enclave
- `.biometryCurrentSet` access control requires biometric authentication (Touch ID) to use the key and invalidates it if fingerprints are added/removed
- Use `.devicePasscode` instead if you want passcode fallback
- Omit both for keys that don't require user presence
- Secure Enclave keys are 256-bit EC (P-256) — no RSA support
- Keys persist in the Secure Enclave across app reinstalls (they're tied to the device, not the app)

## LAContext on macOS

`LAContext` evaluates biometric authentication policies on macOS. The behavior differs from iOS in important ways.

**Supported policies on macOS:**

| Policy | Behavior |
|---|---|
| `.deviceOwnerAuthenticationWithBiometrics` | Touch ID only. Fails if no fingerprints enrolled. |
| `.deviceOwnerAuthenticationWithBiometricsOrWatch` | Touch ID or Apple Watch unlock. |
| `.deviceOwnerAuthenticationWithWatch` | Apple Watch only (no Touch ID fallback). |
| `.deviceOwnerAuthentication` | Biometrics or device passcode fallback. |

**Evaluating a policy:**

```swift
import LocalAuthentication

func authenticateUser(reason: String, completion: @escaping (Bool, Error?) -> Void) {
    let context = LAContext()
    context.localizedFallbackTitle = "Use Password"  // shown after 2 failed attempts

    var error: NSError?
    guard context.canEvaluatePolicy(.deviceOwnerAuthenticationWithBiometrics, error: &error) else {
        completion(false, error)
        return
    }

    context.evaluatePolicy(
        .deviceOwnerAuthenticationWithBiometrics,
        localizedReason: reason
    ) { success, error in
        DispatchQueue.main.async {
            completion(success, error)
        }
    }
}
```

**macOS-specific differences from iOS:**
- No Face ID on macOS — `.deviceOwnerAuthenticationWithBiometrics` maps to Touch ID
- Apple Watch unlock works on macOS 26+ when the watch is on the wrist and unlocked
- `context.biometryType` returns `.touchID` or `.none` (no `.faceID`)
- `localizedCancelTitle` is ignored on macOS — the system controls the cancel button
- `interactionNotAllowed` is not used on macOS
- The Touch ID prompt is a system modal dialog, not an in-app overlay like on iOS

**Combining with SecureEnclave:**

The typical pattern is: use `LAContext` to authenticate the user, then use a `.biometryCurrentSet`-protected SecureEnclave key. The key's access control handles the biometric gate — you don't need to call `LAContext` before every key operation. The system prompts automatically when you attempt to use the key.

```swift
// Query for the key — system prompts for Touch ID automatically
func findKey() throws -> SecKey {
    let query: [String: Any] = [
        kSecClass as String: kSecClassKey,
        kSecAttrLabel as String: "com.example.MyApp.signingKey",
        kSecReturnRef as String: true
    ]

    var item: CFTypeRef?
    let status = SecItemCopyMatching(query as CFDictionary, &item)
    guard status == errSecSuccess, let key = item else {
        throw KeyError.notFound
    }
    return key as! SecKey
}
```

## SIP Awareness

System Integrity Protection (SIP) restricts what even root can do. Agents and daemons must not assume write access to SIP-protected paths.

**Protected paths (even root cannot modify):**
- `/System` (except `/System/Library/User Template`)
- `/usr` (except `/usr/local`)
- `/bin`, `/sbin`
- `/var` (partially — `/var/db/dslocal` is protected)
- Pre-installed Apple apps in `/Applications` (e.g., Safari, Mail)

**What SIP blocks:**
- Loading kernel extensions (kexts) not signed by Apple — use System Extensions (`SystemExtensions` framework) instead
- Attaching debuggers to Apple system processes (unless `csrutil debug` is enabled in recovery OS)
- Modifying NVRAM variables from user space (some are restricted)
- Writing to protected directories even with `sudo`

**Practical implications for agents:**
- Never write to `/System`, `/usr/bin`, or `/bin` — use `/usr/local` or `~/Library` instead
- Never attempt to load kexts — use `SystemExtensions` or `DriverKit`
- Don't assume you can modify files in other apps' containers
- `tccutil` can reset privacy permissions, but the TCC database itself is SIP-protected
- Test with SIP enabled — disabling it for development creates a false sense of capability

**Checking SIP status:**

```bash
csrutil status
# "System Integrity Protection status: enabled."
```

## Constraints & Gotchas

- **Entitlement format must be valid XML ASCII.** The notary service rejects malformed plist. Use Xcode's property list editor or `plutil -lint` to validate.
- **Plug-in entitlements are inherited.** If your app hosts third-party plugins, the host's entitlements must include everything the plugins need (JIT, unsigned memory, library validation disabled). Plan this before notarizing.
- **Hardened Runtime + `DYLD_INSERT_LIBRARIES` = rejection.** If you use `allow-dyld-environment-variables` for development, create a separate debug entitlement file. Never ship it.
- **Ad-hoc signed apps cannot use Push Notifications, iCloud, or associated domains.** These require a valid signing identity tied to a provisioning profile.
- **Notarization is not App Review.** The notary service only checks for code signing issues and known malware. It does not evaluate app content or behavior.
- **Stapling is per-artifact.** If you distribute both a `.dmg` and a `.zip`, each needs its own staple. Stapling the `.app` inside the archive is not sufficient — staple the distribution artifact itself.
- **Secure Enclave keys are device-bound.** They do not sync via iCloud Keychain. If the user gets a new Mac, the keys are gone. Plan for key recovery or re-enrollment.

## Common Mistakes

- **Forgetting `--options runtime`** when codesigning for notarization. The notary rejects binaries without hardened runtime.
- **Using `altool` instead of `notarytool`.** `altool` was deprecated November 2023. Always use `xcrun notarytool`.
- **Including `get-task-allow` in release builds.** Xcode sets this for debug builds. Ensure your release configuration doesn't inherit it.
- **Signing only the outer app bundle.** Nested frameworks, XPC services, and helper tools must each be signed. Use `--deep` for quick validation, but prefer per-component signing for production.
- **Mismatched bundle IDs between `SMAuthorizedClients` and the app.** The helper's `SMAuthorizedClients` must contain the exact bundle ID of the main app, including the team ID prefix.
- **Using `.biometryAny` instead of `.biometryCurrentSet`.** `.biometryAny` doesn't invalidate the key when fingerprints change — a security downgrade. Prefer `.biometryCurrentSet`.
- **Assuming Secure Enclave keys are backed up.** They're hardware-bound to the single device. If the Secure Enclave is wiped or the logic board replaced, keys are gone.
- **Not testing on a notarized-and-stapled build.** Gatekeeper behavior differs between stapled, online-notarized, and unsigned builds. Always test the final artifact.

## Review Checklist

- [ ] Sandbox enabled with minimum necessary entitlements
- [ ] Hardened Runtime enabled (`--options runtime`)
- [ ] No `get-task-allow` in release builds
- [ ] No `allow-dyld-environment-variables` in release builds
- [ ] All nested code signed (frameworks, XPC services, helpers)
- [ ] Secure timestamp included (`--timestamp`)
- [ ] Notarized via `notarytool` (not `altool`)
- [ ] Ticket stapled to distribution artifact
- [ ] `spctl --assess` returns "accepted" with "Notarized Developer ID"
- [ ] SMJobBless helper: `SMAuthorizedClients` matches app bundle ID
- [ ] Secure Enclave keys use `.biometryCurrentSet` (not `.biometryAny`)
- [ ] LAContext policy matches macOS capabilities (no Face ID assumptions)
- [ ] No writes to SIP-protected paths
- [ ] Entitlement files pass `plutil -lint`

## References

- [Notarizing macOS software before distribution](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution)
- [Customizing the notarization workflow](https://developer.apple.com/documentation/security/customizing-the-notarization-workflow)
- [Resolving common notarization issues](https://developer.apple.com/documentation/security/resolving-common-notarization-issues)
- [Enable hardened runtime](https://help.apple.com/xcode/mac/current/#/devf87a2ac8f)
- [Entitlements documentation](https://developer.apple.com/documentation/bundleresources/entitlements)
- [Technical Note TN2415: Entitlements Troubleshooting](https://developer.apple.com/library/archive/technotes/tn2415/_index.html)
- [SMJobBless reference](https://developer.apple.com/documentation/servicemanagement/smjobbless)
- [SecKey Secure Enclave](https://developer.apple.com/documentation/security/certificate_key_and_trust_services/keys/storing_keys_in_the_secure_enclave)
- [LAContext](https://developer.apple.com/documentation/localauthentication/lacontext)
