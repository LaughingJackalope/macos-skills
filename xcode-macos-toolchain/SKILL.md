---
name: xcode-macos-toolchain
description: "Build, archive, export, notarize, and unit test macOS apps with xcodebuild. Covers destination syntax for native macOS, Mac Catalyst, and universal binaries; archive/export workflows for Developer ID, Mac App Store, and development distribution; notarization with notarytool and stapling; macOS-specific build settings; and CI/CD pipeline patterns. Use when building macOS targets, configuring distribution pipelines, or debugging xcodebuild failures on macOS."
sources: https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution, https://keith.github.io/xcode-man-pages/xcodebuild.1.html, https://github.com/rudrankriyam/app-store-connect-cli-skills/blob/main/skills/asc-notarization/SKILL.md, https://terminalmac.com/en/blog/articles/remote-mac-release-runbook-xcode-notary-ssh-m4-tmux/remote-mac-release-runbook-xcode-notary-ssh-m4-tmux.html, https://developer.apple.com/documentation/security/customizing-the-notarization-workflow, https://keith.github.io/xcode-man-pages/notarytool.1.html, https://developer.apple.com/library/archive/technotes/tn2339/_index.html
researched: 2026-06-06
platform: macOS 26+
---

# Xcode macOS Toolchain

Build, archive, export, notarize, and test macOS apps with `xcodebuild`. Targets macOS 26+ with Swift 6.3. This skill covers the macOS-specific toolchain — for iOS/simulator xcodebuild, see the `ios-simulator` skill in swift-ios-skills. For hardened runtime and entitlement details, see `macos-security-entitlements`.

See [references/xcodebuild-macos.md](references/xcodebuild-macos.md) for complete xcodebuild flag reference, export options plist templates, and CI/CD pipeline examples.

## Contents

- [Destination Syntax](#destination-syntax)
- [Building](#building)
- [Archiving](#archiving)
- [Exporting](#exporting)
- [Notarization](#notarization)
- [Unit Testing on macOS](#unit-testing-on-macos)
- [Key Build Settings](#key-build-settings)
- [Mac Catalyst](#mac-catalyst)
- [Universal Binaries](#universal-binaries)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Destination Syntax

Every `xcodebuild` action that runs or tests code requires a `-destination` parameter. For macOS targets there are three distinct platform selectors:

```bash
# Native macOS (current machine architecture)
-destination "platform=macOS"

# Native macOS, explicit architecture
-destination "platform=macOS,arch=arm64"
-destination "platform=macOS,arch=x86_64"

# Mac Catalyst (UIKit-on-Mac)
-destination "platform=macOS,variant=Mac Catalyst"
```

**When to use each:**

| Destination | Use when |
|---|---|
| `platform=macOS` | Building or testing a native AppKit/SwiftUI macOS target |
| `platform=macOS,arch=arm64` | Explicitly targeting Apple Silicon (e.g., in a universal binary build matrix) |
| `platform=macOS,variant=Mac Catalyst` | Building the Mac Catalyst variant of an iOS target |

**Multiple destinations** are comma-separated in a single flag:

```bash
-destination "platform=macOS,arch=arm64" -destination "platform=macOS,arch=x86_64"
```

**Finding available destinations:**

```bash
xcodebuild -showdestinations -scheme MyApp
```

This lists all valid destination specifiers for the scheme, including connected devices and simulators. Use this to discover the exact string for a specific Mac or configuration.

**Disambiguating macOS vs. Mac Catalyst:** When both native macOS and Mac Catalyst destinations exist, `platform=macOS` alone may match the first one found (which could be Catalyst). To force native macOS, use `variant=macOS` (e.g., `generic/platform=macOS,variant=macOS`). To force Catalyst, use `variant=Mac Catalyst`. Note: `variant=macOS` may not yield a match in all Xcode versions — use `xcodebuild -showdestinations` to verify.

## Building

Basic build invocation for a macOS scheme:

```bash
xcodebuild build \
  -scheme MyApp \
  -destination "platform=macOS" \
  -configuration Release
```

**Build configuration vs. Xcode:** `xcodebuild` defaults to `Debug`. Always pass `-configuration Release` for distribution builds. The configuration determines which build settings variant is used (e.g., `CODE_SIGN_IDENTITY[config=Release]`).

**Derived data and build location:**

```bash
xcodebuild build \
  -scheme MyApp \
  -destination "platform=macOS" \
  -derivedDataPath /tmp/MyApp-build
```

Use `-derivedDataPath` in CI to isolate builds and enable caching. The default derived data location is `~/Library/Developer/Xcode/DerivedData/` with a hash suffix.

**Silencing output:** Use `-quiet` (Xcode 16+) to reduce log noise in CI. For older Xcode versions, pipe through `xcpretty`:

```bash
xcodebuild build -scheme MyApp -destination "platform=macOS" | xcpretty
```

## Archiving

Archiving produces an `.xcarchive` bundle containing the built app, dSYMs, and linkage information. This is the required input for export and distribution.

```bash
xcodebuild archive \
  -scheme MyApp \
  -destination "platform=macOS" \
  -archivePath /tmp/MyApp.xcarchive \
  -configuration Release
```

**Archive path:** Always pass an explicit `-archivePath`. Without it, Xcode places the archive in `~/Library/Developer/Xcode/Archives/` with a date stamp — unpredictable in CI.

**What's inside an xcarchive:**

```
MyApp.xcarchive/
  Products/Applications/MyApp.app
  dSYMs/MyApp.app.dSYM
  Info.plist
  BCSymbolMaps/
```

**Archiving for Mac Catalyst:**

```bash
xcodebuild archive \
  -scheme MyApp \
  -destination "platform=macOS,variant=Mac Catalyst" \
  -archivePath /tmp/MyApp-Catalyst.xcarchive
```

**Code signing during archive:** The archive step signs the app using the build settings `CODE_SIGN_IDENTITY` and `CODE_SIGN_ENTITLEMENTS`. For distribution, set these to your Developer ID or Mac App Store certificate. See [Key Build Settings](#key-build-settings) for details.

## Exporting

Export takes an `.xcarchive` and produces a distributable artifact (`.app`, `.pkg`, or `.dmg`). It requires an export options plist.

```bash
xcodebuild -exportArchive \
  -archivePath /tmp/MyApp.xcarchive \
  -exportPath /tmp/MyApp-export \
  -exportOptionsPlist ExportOptions.plist
```

### Export Options Plist

The plist keys differ by distribution method. Here are the three common cases:

**Developer ID (direct distribution outside the Mac App Store):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key>
  <string>developer-id</string>
  <key>teamID</key>
  <string>ABCD123456</string>
  <key>signingStyle</key>
  <string>automatic</string>
  <key>uploadSymbols</key>
  <true/>
</dict>
</plist>
```

**Mac App Store:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key>
  <string>app-store</string>
  <key>teamID</key>
  <string>ABCD123456</string>
  <key>signingStyle</key>
  <string>automatic</string>
  <key>uploadSymbols</key>
  <true/>
  <key>destination</key>
  <string>upload</string>
</dict>
</plist>
```

**Development (local testing of the archive):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key>
  <string>development</string>
  <key>teamID</key>
  <string>ABCD123456</string>
  <key>signingStyle</key>
  <string>automatic</string>
</dict>
</plist>
```

**Key plist keys:**

| Key | Values | Description |
|---|---|---|
| `method` | `app-store`, `developer-id`, `development`, `mac-application` | Distribution channel |
| `teamID` | Your Apple Developer Team ID | Required for all methods |
| `signingStyle` | `automatic`, `manual` | Automatic uses Xcode's signing; manual requires explicit provisioning profiles |
| `provisioningProfiles` | Dict mapping bundle ID to profile name | Required when `signingStyle` is `manual` |
| `destination` | `upload`, `export` | `upload` submits to App Store Connect; `export` writes locally |
| `uploadSymbols` | Boolean | Upload dSYM crash symbol files |
| `stripSwiftSymbols` | Boolean | Strip Swift symbols from the distributed binary (default: true) |

**Export failures:** If `-exportArchive` fails with a signing error, check that the archive was signed with a certificate matching the export method. Developer ID archives require a "Developer ID Application" certificate; Mac App Store archives require a "3rd Party Mac Developer Application" certificate. You cannot re-sign an archive — you must re-archive with the correct certificate.

**PKG export for App Store Connect:** When exporting a PKG for App Store Connect upload, use `asc builds upload --pkg` with explicit `--version` and `--build-number` flags (these are not auto-extracted from PKGs as they are from IPAs).

## Notarization

Notarization is Apple's automated malware scan. Required for Developer ID distribution on macOS 10.15+. A notarized app passes Gatekeeper without warning.

### Submit

```bash
xcrun notarytool submit /tmp/MyApp.zip \
  --apple-id "dev@example.com" \
  --team-id "ABCD123456" \
  --password "xxxx-xxxx-xxxx-xxxx" \
  --wait
```

**`--wait`** blocks until notarization completes (typically 1–5 minutes). Without it, the command returns immediately and you must poll for status.

**App-specific password:** The `--password` is an app-specific password generated at [appleid.apple.com](https://appleid.apple.com). It is not your Apple ID password. Format: `xxxx-xxxx-xxxx-xxxx` (four groups of four characters).

**What to submit:** Notarize a ZIP of the `.app` (not the `.xcarchive`):

```bash
ditto -c -k --keepParent /path/to/MyApp.app /tmp/MyApp.zip
xcrun notarytool submit /tmp/MyApp.zip --apple-id ... --team-id --password ... --wait
```

**Notarization creates tickets** for the top-level file and any nested items (e.g., a disk image containing an installer and an app bundle). You cannot staple to a ZIP directly — staple each item inside the archive separately, then recreate the ZIP if needed.

**Webhook notifications:** Use `--webhook <URL>` to receive a callback when notarization completes, instead of polling:

```bash
xcrun notarytool submit /tmp/MyApp.zip \
  --keychain-profile "AC_PASSWORD" \
  --wait \
  --webhook "https://ci.example.com/notary-callback"
```

### Check Status (without --wait)

```bash
# Get the submission ID from the submit output, then:
xcrun notarytool log <submission-id> \
  --apple-id "dev@example.com" \
  --team-id "ABCD123456" \
  --password "xxxx-xxxx-xxxx-xxxx"
```

The log output includes a `status` field (`Accepted`, `Invalid`, `In Progress`) and a JSON log URL with detailed failure reasons.

### Staple

After successful notarization, staple the ticket to the app so Gatekeeper can verify it offline:

```bash
xcrun stapler staple /path/to/MyApp.app
```

**Verify stapling:**

```bash
spctl --assess --type execute --verbose /path/to/MyApp.app
# Should output: "accepted" with source "Notarized Developer ID"
```

**Staple the DMG too** if you distribute via disk image:

```bash
xcrun stapler staple /path/to/MyApp.dmg
```

**DMG notarization workflow (macOS 26+):**

```bash
# Sign the DMG itself
codesign --force --timestamp --sign "Developer ID Application: ..." MyApp.dmg

# Notarize the DMG directly (not the app inside it)
xcrun notarytool submit MyApp.dmg --keychain-profile "AC_PASSWORD" --wait

# Staple the DMG
xcrun stapler staple MyApp.dmg
```

**macOS 26.2+ TCC caveat:** Avoid mounting DMGs at `/Volumes` in scripts due to TCC restrictions. Use `/private/tmp` for mounting in CI/release scripts.

### Keychain Profile (CI)

For CI, avoid passing credentials on the command line. Store them in the keychain:

```bash
xcrun notarytool store-credentials "AC_PASSWORD" \
  --apple-id "dev@example.com" \
  --team-id "ABCD123456" \
  --password "xxxx-xxxx-xxxx-xxxx"
```

Then submit using the profile name:

```bash
xcrun notarytool submit /tmp/MyApp.zip --keychain-profile "AC_PASSWORD" --wait
```

The profile is stored in your login keychain and persists across sessions. On CI machines, create a temporary keychain and store the profile there.

## Unit Testing on macOS

### Destination-Based Testing

```bash
# Run tests on the current Mac (native)
xcodebuild test \
  -scheme MyApp \
  -destination "platform=macOS"
```

This launches the test runner on the host Mac. Use this for macOS-specific tests (AppKit, AppKit integration, macOS-only logic).

### Testing by Device ID

```bash
# Find the Mac's device identifier
xcodebuild -showdestinations -scheme MyApp

# Use the identifier directly
xcodebuild test \
  -scheme MyApp \
  -destination "id=<device-uuid>"
```

Using `id=` is useful when you have multiple build destinations or need to target a specific Mac in a remote build farm.

### Mac Catalyst Testing

```bash
xcodebuild test \
  -scheme MyApp \
  -destination "platform=macOS,variant=Mac Catalyst"
```

This runs the iOS test suite under Mac Catalyst. Tests that depend on iOS-only APIs (e.g., `UIKit` without Catalyst availability) may fail — use `#if targetEnvironment(macCatalyst)` to gate test code.

### Test-Only Build (no run)

```bash
xcodebuild build-for-testing \
  -scheme MyApp \
  -destination "platform=macOS"

xcodebuild test-without-building \
  -scheme MyApp \
  -destination "platform=macOS" \
  -xctrun /tmp/MyApp-tests.xctestrun
```

Separating build and test is useful in CI: build once, test on multiple destinations.

## Key Build Settings

These settings control macOS-specific behavior. Set them in your target's build settings or in an `.xcconfig` file.

| Setting | Typical Value | Purpose |
|---|---|---|
| `MACOSX_DEPLOYMENT_TARGET` | `15.0` or `26.0` | Minimum macOS version the app supports |
| `CODE_SIGN_IDENTITY` | `Apple Development`, `Developer ID Application`, `3rd Party Mac Developer Application` | Code signing certificate |
| `ENABLE_HARDENED_RUNTIME` | `YES` | Enables the hardened runtime (required for notarization) |
| `CODE_SIGN_ENTITLEMENTS` | `MyApp/MyApp.entitlements` | Path to the entitlements file |
| `PRODUCT_BUNDLE_IDENTIFIER` | `com.example.MyApp` | Unique bundle ID |
| `SWIFT_VERSION` | `6.0` | Swift language version |
| `ARCHS` | `arm64`, `x86_64`, or `$(ARCHS_STANDARD)` | Target architectures |
| `ONLY_ACTIVE_ARCH` | `YES` (Debug), `NO` (Release) | Build only for active arch or all archs |

**`MACOSX_DEPLOYMENT_TARGET`** controls API availability. Setting it to `26.0` means the app will not launch on macOS 15.x. Use availability checks (`#available(macOS 26, *)`) for APIs introduced after your deployment target.

**`CODE_SIGN_IDENTITY` values:**

| Value | Use |
|---|---|
| `Apple Development` | Local development and debugging |
| `Developer ID Application` | Direct distribution (outside Mac App Store) |
| `3rd Party Mac Developer Application` | Mac App Store distribution |
| `-` (ad hoc) | Unsigned (not recommended for distribution) |

**`ENABLE_HARDENED_RUNTIME`** must be `YES` for notarization. The hardened runtime restricts certain operations (JIT, writable-executable memory, etc.). If your app needs exceptions, add them to the entitlements file:

```xml
<key>com.apple.security.cs.allow-jit</key>
<true/>
<key>com.apple.security.cs.allow-unsigned-executable-memory</key>
<true/>
```

## Mac Catalyst

Mac Catalyst runs iOS apps on macOS. Use it when you have an iOS app and want a macOS version without rewriting in AppKit.

### When to Use Mac Catalyst

- Your app is UIKit-based and you want a macOS presence quickly
- You want to share business logic and UI between iOS and macOS
- Your app doesn't need deep macOS-specific features (menu bar extras, Touch Bar, AppKit integrations)

### When Not to Use Mac Catalyst

- The app needs AppKit features (NSPopover, NSSplitView, NSDocument)
- You need a native macOS menu bar structure
- The app uses Metal directly with macOS-specific GPU features
- You need fine-grained control over window management

### Capability Gaps

Mac Catalyst does not support all iOS APIs. Key gaps:

| API | Catalyst Support |
|---|---|
| UIKit | Partial — some types are unavailable |
| ARKit | Not supported |
| CallKit | Not supported |
| HealthKit | Not supported |
| HomeKit | Not supported |
| MessageUI | Not supported |
| Core NFC | Not supported |
| App Clip | Not supported |
| PushKit (VoIP) | Not supported |

Check `WK_INTERFACE_CONTROLLER` and `TARGET_OS_MACCATALYST` at compile time.

### Conditional Compilation

```swift
#if targetEnvironment(macCatalyst)
// Mac Catalyst-specific code
// e.g., adjust layout for pointer interaction
#elseif os(iOS)
// iOS-specific code
#endif
```

**In Swift packages and frameworks**, use `#if targetEnvironment(macCatalyst)` to exclude unsupported APIs. In Objective-C, use `TARGET_OS_MACCATALYST`.

### Catalyst-Specific Build Settings

```bash
# In .xcconfig:
SUPPORTS_MACCATALYST = YES
DERIVE_MACCATALYST_PRODUCT_BUNDLE_IDENTIFIER = YES
```

`DERIVE_MACCATALYST_PRODUCT_BUNDLE_IDENTIFIER` automatically appends `.MacCatalyst` to your iOS bundle ID for the Catalyst variant. Set it to `NO` if you want a custom bundle ID.

## Universal Binaries

A universal binary (fat binary) contains code for both `arm64` (Apple Silicon) and `x86_64` (Intel).

### Build Settings

```bash
# In .xcconfig:
ARCHS = arm64 x86_64
ONLY_ACTIVE_ARCH = NO
```

**`ARCHS`** lists all architectures to build. `$(ARCHS_STANDARD)` expands to `arm64 x86_64`.

**`ONLY_ACTIVE_ARCH`** controls whether all archs are built:

| Configuration | `ONLY_ACTIVE_ARCH` | Behavior |
|---|---|---|
| Debug | `YES` | Build only for the current machine's arch (faster builds) |
| Release | `NO` | Build all archs in `ARCHS` |

### Verifying Architectures

```bash
# Check which architectures are in a binary
lipo -archs /path/to/MyApp.app/Contents/MacOS/MyApp
# Output: arm64 x86_64

# Detailed info
lipo -info /path/to/MyApp.app/Contents/MacOS/MyApp
```

### Lipo Post-Processing

If you build separate single-arch archives and need to combine them:

```bash
# Extract x86_64 from one archive
lipo /path/to/Intel.xcarchive/Products/Applications/MyApp.app/Contents/MacOS/MyApp \
  -thin x86_64 -output /tmp/MyApp-x86_64

# Extract arm64 from another archive
lipo /path/to/ARM.xcarchive/Products/Applications/MyApp.app/Contents/MacOS/MyApp \
  -thin arm64 -output /tmp/MyApp-arm64

# Combine into universal binary
lipo -create /tmp/MyApp-x86_64 /tmp/MyApp-arm64 \
  -output /path/to/MyApp.app/Contents/MacOS/MyApp
```

**When to use lipo:** Only when you have separate build machines (e.g., an Intel CI runner and an ARM CI runner). If your build machine supports both archs (Xcode can cross-compile), use `ARCHS = arm64 x86_64` with `ONLY_ACTIVE_ARCH = NO` instead — it's simpler and less error-prone.

### Thin Binaries for Distribution

If you want to ship separate per-architecture downloads:

```bash
lipo /path/to/MyApp -thin arm64 -output /tmp/MyApp-arm64
lipo /path/to/MyApp -thin x86_64 -output /tmp/MyApp-x86_64
```

Then notarize and staple each variant separately.

## Common Mistakes

**Wrong destination for Mac Catalyst.** Using `platform=macOS` for a Catalyst scheme builds the native macOS target, not the Catalyst variant. Always use `platform=macOS,variant=Mac Catalyst` for Catalyst.

**Forgetting `-configuration Release` for distribution.** `xcodebuild` defaults to `Debug`. Debug builds are not suitable for distribution — they're not optimized, may include debug symbols, and use the wrong signing identity.

**Exporting with the wrong method.** A Developer ID-signed archive cannot be exported with `method=app-store`. The signing identity used during archive must match the export method. Re-archive with the correct certificate.

**Notarizing the `.xcarchive` instead of the `.app`.** Notarytool requires a ZIP of the `.app` bundle, not the `.xcarchive`. Use `ditto -c -k --keepParent` to create the ZIP.

**Stapling before notarization completes.** The staple ticket comes from the notarization service. If you staple before the status is `Accepted`, the staple is invalid. Always use `--wait` or poll with `notarytool log` until status is `Accepted`.

**Setting `ONLY_ACTIVE_ARCH=YES` in Release.** This produces a single-architecture binary. Apple expects universal binaries for Mac App Store and Developer ID distribution. Set `ONLY_ACTIVE_ARCH=NO` for Release builds.

**Missing `ENABLE_HARDENED_RUNTIME`.** Notarization fails without the hardened runtime. Set `ENABLE_HARDENED_RUNTIME=YES` and ensure your entitlements file includes any necessary exceptions.

**Using `platform=macOS` in CI without a connected Mac.** `platform=macOS` targets the build host. In CI, this means the build machine must be a Mac. There is no macOS simulator — you cannot use `platform=macOS Simulator` like you can with iOS.

**Notarizing a ZIP and then stapling the ZIP.** You cannot staple to a ZIP directly. Staple the `.app` or `.DMG` inside the archive, then recreate the ZIP if needed.

**DMG mounting at `/Volumes` on macOS 26.2+.** TCI restrictions prevent mounting at `/Volumes` in scripts. Use `/private/tmp` for mounting in CI/release scripts.

## Review Checklist

- [ ] `-destination` matches the target type (`platform=macOS` for native, `platform=macOS,variant=Mac Catalyst` for Catalyst)
- [ ] `-configuration Release` is set for all archive and export invocations
- [ ] `-archivePath` is explicit and deterministic (not relying on Xcode's default archive folder)
- [ ] Export options plist `method` matches the archive's signing identity
- [ ] `ENABLE_HARDENED_RUNTIME=YES` is set for distribution builds
- [ ] `CODE_SIGN_ENTITLEMENTS` points to a valid entitlements file
- [ ] `ONLY_ACTIVE_ARCH=NO` for Release builds targeting multiple architectures
- [ ] Notarization uses `ditto -c -k --keepParent` to ZIP the `.app` (not the `.xcarchive`)
- [ ] `xcrun stapler staple` runs only after `notarytool` reports `Accepted`
- [ ] CI uses `--keychain-profile` instead of inline credentials for notarytool
- [ ] `MACOSX_DEPLOYMENT_TARGET` is set explicitly (not relying on Xcode's default)
- [ ] DMG notarized and stapled separately if distributing via disk image
- [ ] `lipo -archs` confirms universal binary contains both `arm64` and `x86_64`

## References

- [references/xcodebuild-macos.md](references/xcodebuild-macos.md) — Complete xcodebuild flag reference, export options plist templates, and CI/CD pipeline examples
- [Apple: Notarizing macOS Software](https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution)
- [Apple: Customizing the Notarization Workflow](https://developer.apple.com/documentation/security/customizing-the-notarization-workflow)
- [Keith's xcodebuild man page](https://keith.github.io/xcode-man-pages/xcodebuild.1.html)
- [Keith's notarytool man page](https://keith.github.io/xcode-man-pages/notarytool.1.html)
- [Apple TN2339: Building from the Command Line with Xcode FAQ](https://developer.apple.com/library/archive/technotes/tn2339/_index.html)
