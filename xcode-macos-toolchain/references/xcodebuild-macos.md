# xcodebuild macOS Reference

Complete reference for xcodebuild flags, export options plist templates, and CI/CD pipeline patterns specific to macOS targets. Targets macOS 26+ with Swift 6.3.

## Contents

- [xcodebuild Flag Reference](#xcodebuild-flag-reference)
- [Export Options Plist Templates](#export-options-plist-templates)
- [CI/CD Pipeline Examples](#cicd-pipeline-examples)
- [Troubleshooting](#troubleshooting)

## xcodebuild Flag Reference

### Global Options

| Flag | Description |
|---|---|
| `-project <path>` | Path to `.xcodeproj`. Required if multiple projects are in the working directory. |
| `-workspace <path>` | Path to `.xcworkspace`. Required when using CocoaPods or SPM with an workspace. |
| `-scheme <name>` | The scheme to build. Required for all actions. |
| `-target <name>` | Build a specific target instead of a scheme. |
| `-configuration <name>` | `Debug` or `Release`. Default: `Debug`. |
| `-destination <spec>` | Where to build/run/test. See [Destination Syntax](#destination-syntax). |
| `-derivedDataPath <path>` | Override the derived data directory. |
| `-xcconfig <path>` | Path to an `.xcconfig` file with build settings overrides. |
| `-quiet` | Print only warnings and errors (Xcode 16+). |
| `-verbose` | Print all build commands as they execute. |
| `-resultBundlePath <path>` | Write a `.xcresult` bundle with build/test results. |
| `-enableCodeCoverage <BOOL>` | Enable code coverage during test runs. |
| `-skipPackageUpdates` | Skip SPM package resolution (use cached packages). |
| `-disableAutomaticPackageResolution` | Same as `-skipPackageUpdates`. |
| `-clonedSourcePackagesDirPath <path>` | Override SPM clone directory. |
| `-packageCachePath <path>` | Override SPM package cache directory. |

### Actions

| Action | Description |
|---|---|
| `build` | Compile and link without running. |
| `build-for-testing` | Build a `.xctestrun` file for later test execution. |
| `test` | Build and run tests. |
| `test-without-building` | Run tests from a previously built `.xctestrun`. |
| `archive` | Create an `.xcarchive` bundle. |
| `-exportArchive` | Export a distributable artifact from an `.xcarchive`. |
| `-exportLocalizations` | Export localization catalogs. |
| `-importLocalizations` | Import localization catalogs. |
| `clean` | Remove build products. |
| `-showdestinations` | List all valid destinations for a scheme. |
| `-showBuildSettings` | Print all resolved build settings for a scheme/destination. |
| `-list` | List schemes, targets, and configurations in the project. |

### Destination Specifiers

The `-destination` parameter accepts a comma-separated list of key-value pairs:

```
-destination "key=value,key=value,..."
```

**macOS-specific keys:**

| Key | Values | Description |
|---|---|---|
| `platform` | `macOS`, `macOS,variant=Mac Catalyst` | Target platform |
| `arch` | `arm64`, `x86_64` | CPU architecture |
| `id` | Device UUID | Target a specific device by identifier |
| `name` | Device name | Target a specific device by name |

**Examples:**

```bash
# Native macOS, any architecture
-destination "platform=macOS"

# Native macOS, Apple Silicon only
-destination "platform=macOS,arch=arm64"

# Native macOS, Intel only
-destination "platform=macOS,arch=x86_64"

# Mac Catalyst
-destination "platform=macOS,variant=Mac Catalyst"

# Specific device by UUID
-destination "id=00008112-001234567890ABCD"

# Multiple destinations (build for both)
-destination "platform=macOS,arch=arm64" -destination "platform=macOS,arch=x86_64"
```

### Build Settings Overrides

Pass build settings on the command line with `SETTING=VALUE`:

```bash
xcodebuild archive \
  -scheme MyApp \
  -destination "platform=macOS" \
  CODE_SIGN_IDENTITY="Developer ID Application" \
  PROVISIONING_PROFILE_SPECIFIER="" \
  ENABLE_HARDENED_RUNTIME=YES
```

Command-line overrides take precedence over `.xcconfig` files and target build settings.

## Export Options Plist Templates

### Developer ID Distribution

For direct distribution outside the Mac App Store (download from website, GitHub releases, etc.):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key>
  <string>developer-id</string>
  <key>teamID</key>
  <string>YOUR_TEAM_ID</string>
  <key>signingStyle</key>
  <string>automatic</string>
  <key>stripSwiftSymbols</key>
  <true/>
  <key>uploadSymbols</key>
  <true/>
</dict>
</plist>
```

**Post-export workflow:** Notarize the exported `.app`, then staple it. Optionally wrap in a DMG:

```bash
# Create DMG
hdiutil create -volname "MyApp" -srcfolder /tmp/MyApp-export/MyApp.app \
  -ov -format UDZO /tmp/MyApp.dmg

# Notarize the DMG (or the .app)
xcrun notarytool submit /tmp/MyApp.dmg \
  --keychain-profile "AC_PASSWORD" --wait

# Staple
xcrun stapler staple /tmp/MyApp.dmg
xcrun stapler staple /tmp/MyApp-export/MyApp.app
```

### Mac App Store Distribution

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key>
  <string>app-store</string>
  <key>teamID</key>
  <string>YOUR_TEAM_ID</string>
  <key>signingStyle</key>
  <string>automatic</string>
  <key>stripSwiftSymbols</key>
  <true/>
  <key>uploadSymbols</key>
  <true/>
  <key>destination</key>
  <string>upload</string>
</dict>
</plist>
```

**With `destination=upload`**, the export step uploads the build to App Store Connect. You can also use `destination=export` to write locally and upload separately with `xcrun altool` or `xcrun notarytool`.

### Development Distribution

For local testing of the archive/export pipeline:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key>
  <string>development</string>
  <key>teamID</key>
  <string>YOUR_TEAM_ID</string>
  <key>signingStyle</key>
  <string>automatic</string>
</dict>
</plist>
```

### Manual Signing

When you need explicit control over provisioning profiles:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>method</key>
  <string>developer-id</string>
  <key>teamID</key>
  <string>YOUR_TEAM_ID</string>
  <key>signingStyle</key>
  <string>manual</string>
  <key>provisioningProfiles</key>
  <dict>
    <key>com.example.MyApp</key>
    <string>MyApp Developer ID Profile</string>
  </dict>
  <key>signingCertificate</key>
  <string>Developer ID Application: Your Name (TEAM_ID)</string>
</dict>
</plist>
```

The `provisioningProfiles` dictionary maps bundle IDs to provisioning profile names (as shown in Xcode's Accounts preferences).

### Complete Plist Key Reference

| Key | Type | Required | Description |
|---|---|---|---|
| `method` | String | Yes | `app-store`, `developer-id`, `development`, `mac-application` |
| `teamID` | String | Yes | Apple Developer Team ID (10 characters) |
| `signingStyle` | String | No | `automatic` (default) or `manual` |
| `signingCertificate` | String | Manual only | Exact certificate name in keychain |
| `provisioningProfiles` | Dict | Manual only | Bundle ID → profile name mapping |
| `destination` | String | No | `upload` (submit to ASC) or `export` (write locally) |
| `stripSwiftSymbols` | Boolean | No | Strip Swift symbols (default: true) |
| `uploadSymbols` | Boolean | No | Upload dSYMs to ASC (default: true for app-store) |
| `compileBitcode` | Boolean | No | Compile bitcode (default: true for app-store, unused for macOS) |
| `installerSigningCertificate` | String | No | Certificate for `.pkg` installer signing |
| `iCloudContainerEnvironment` | String | No | `Development` or `Production` |

## CI/CD Pipeline Examples

### GitHub Actions: Full Release Pipeline

```yaml
name: Release

on:
  push:
    tags: ['v*']

jobs:
  build:
    runs-on: macos-15
    steps:
      - uses: actions/checkout@v4

      - name: Select Xcode
        run: sudo xcode-select -s /Applications/Xcode_16.4.app

      - name: Show destinations
        run: xcodebuild -showdestinations -scheme MyApp

      - name: Build and Archive
        run: |
          xcodebuild archive \
            -scheme MyApp \
            -destination "platform=macOS" \
            -archivePath ${{ runner.temp }}/MyApp.xcarchive \
            -configuration Release \
            CODE_SIGN_IDENTITY="Developer ID Application" \
            CODE_SIGN_ENTITLEMENTS="MyApp/MyApp.entitlements" \
            ENABLE_HARDENED_RUNTIME=YES

      - name: Export
        run: |
          xcodebuild -exportArchive \
            -archivePath ${{ runner.temp }}/MyApp.xcarchive \
            -exportPath ${{ runner.temp }}/export \
            -exportOptionsPlist ExportOptions.plist

      - name: Notarize
        env:
          APPLE_ID: ${{ secrets.APPLE_ID }}
          TEAM_ID: ${{ secrets.TEAM_ID }}
          APP_PASSWORD: ${{ secrets.APP_PASSWORD }}
        run: |
          # Store credentials
          xcrun notarytool store-credentials "AC_PASSWORD" \
            --apple-id "$APPLE_ID" \
            --team-id "$TEAM_ID" \
            --password "$APP_PASSWORD"

          # Zip the app
          ditto -c -k --keepParent \
            "${{ runner.temp }}/export/MyApp.app" \
            ${{ runner.temp }}/MyApp.zip

          # Submit
          xcrun notarytool submit ${{ runner.temp }}/MyApp.zip \
            --keychain-profile "AC_PASSWORD" \
            --wait

      - name: Staple
        run: |
          xcrun stapler staple "${{ runner.temp }}/export/MyApp.app"

      - name: Create DMG
        run: |
          hdiutil create \
            -volname "MyApp" \
            -srcfolder "${{ runner.temp }}/export/MyApp.app" \
            -ov -format UDZO \
            ${{ runner.temp }}/MyApp.dmg
          xcrun stapler staple ${{ runner.temp }}/MyApp.dmg

      - name: Upload Release Asset
        uses: softprops/action-gh-release@v2
        with:
          files: ${{ runner.temp }}/MyApp.dmg
```

### GitHub Actions: Mac App Store Submission

```yaml
name: App Store

on:
  push:
    tags: ['release-*']

jobs:
  build:
    runs-on: macos-15
    steps:
      - uses: actions/checkout@v4

      - name: Archive
        run: |
          xcodebuild archive \
            -scheme MyApp \
            -destination "platform=macOS" \
            -archivePath ${{ runner.temp }}/MyApp.xcarchive \
            -configuration Release

      - name: Export and Upload
        env:
          API_KEY: ${{ secrets.APP_STORE_CONNECT_API_KEY }}
        run: |
          xcodebuild -exportArchive \
            -archivePath ${{ runner.temp }}/MyApp.xcarchive \
            -exportPath ${{ runner.temp }}/export \
            -exportOptionsPlist ExportOptions-AppStore.plist
```

With `ExportOptions-AppStore.plist` containing `destination=upload`.

### GitHub Actions: Test Matrix (Universal Binary)

```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: macos-15
    strategy:
      matrix:
        arch: [arm64, x86_64]
    steps:
      - uses: actions/checkout@v4

      - name: Run Tests
        run: |
          xcodebuild test \
            -scheme MyApp \
            -destination "platform=macOS,arch=${{ matrix.arch }}" \
            -resultBundlePath ${{ runner.temp }}/TestResults.xcresult

      - name: Upload Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results-${{ matrix.arch }}
          path: ${{ runner.temp }}/TestResults.xcresult
```

### Makefile for Local Development

```makefile
SCHEME = MyApp
DEST = platform=macOS
ARCHIVE_PATH = /tmp/$(SCHEME).xcarchive
EXPORT_PATH = /tmp/$(SCHEME)-export

.PHONY: build archive export notarize staple test clean

build:
	xcodebuild build \
		-scheme $(SCHEME) \
		-destination "$(DEST)" \
		-configuration Debug

test:
	xcodebuild test \
		-scheme $(SCHEME) \
		-destination "$(DEST)"

archive:
	xcodebuild archive \
		-scheme $(SCHEME) \
		-destination "$(DEST)" \
		-archivePath $(ARCHIVE_PATH) \
		-configuration Release

export: archive
	xcodebuild -exportArchive \
		-archivePath $(ARCHIVE_PATH) \
		-exportPath $(EXPORT_PATH) \
		-exportOptionsPlist ExportOptions.plist

notarize: export
	ditto -c -k --keepParent \
		$(EXPORT_PATH)/$(SCHEME).app \
		/tmp/$(SCHEME).zip
	xcrun notarytool submit /tmp/$(SCHEME).zip \
		--keychain-profile "AC_PASSWORD" \
		--wait

staple: notarize
	xcrun stapler staple $(EXPORT_PATH)/$(SCHEME).app

clean:
	rm -rf $(ARCHIVE_PATH) $(EXPORT_PATH) /tmp/$(SCHEME).zip
```

## Troubleshooting

### "No matching destination found"

**Cause:** The destination specifier doesn't match any available destination for the scheme.

**Fix:** Run `xcodebuild -showdestinations -scheme <name>` to list valid destinations. Ensure the scheme has a macOS target (not just iOS).

### "Code signing failed: No matching provisioning profile"

**Cause:** The `CODE_SIGN_IDENTITY` doesn't match any provisioning profile for the bundle ID.

**Fix:** Check that the provisioning profile is installed (`~/Library/MobileDevice/Provisioning Profiles/`) and that the bundle ID matches exactly. For automatic signing, set `signingStyle=automatic` in the export options plist.

### "The archive contains nothing that can be signed"

**Cause:** The scheme's build targets don't produce a signed product, or the archive path already exists from a previous run.

**Fix:** Delete the old archive (`rm -rf /tmp/MyApp.xcarchive`) and re-archive. Ensure the scheme has `Archive` enabled in its scheme settings (Edit Scheme → Archive → Build Configuration).

### "Notarization failed: Invalid"

**Cause:** The binary was not built with the hardened runtime, or it contains disallowed entitlements.

**Fix:** Check the notarization log for specific issues:

```bash
xcrun notarytool log <submission-id> \
  --keychain-profile "AC_PASSWORD"
```

Common issues:
- Missing `ENABLE_HARDENED_RUNTIME=YES`
- Unsigned libraries or frameworks inside the app bundle
- Frameworks built without `SKIP_INSTALL=NO` (embedded frameworks must be signed)

### "Unsupported arch" when running on Apple Silicon

**Cause:** The binary only contains `x86_64` and Rosetta 2 is not installed.

**Fix:** Install Rosetta 2 (`softwareupdate --install-rosetta`) or build a universal binary with `ARCHS = arm64 x86_64`.

### "xcodebuild: error: Scheme MyApp is not currently configured for the archive action"

**Cause:** The scheme doesn't have archive configuration, or you're targeting the wrong scheme.

**Fix:** In Xcode, go to Product → Scheme → Edit Scheme → Archive, and ensure the correct build configuration is selected. Or pass `-scheme` with the correct scheme name from `xcodebuild -list`.

### Slow builds in CI

**Causes and fixes:**

| Cause | Fix |
|---|---|
| SPM package resolution on every build | Use `-skipPackageUpdates` and cache `~/Library/Developer/Xcode/DerivedData/SourcePackages/` |
| No derived data cache | Set `-derivedDataPath` to a consistent location and cache it |
| Building all archs in Debug | Set `ONLY_ACTIVE_ARCH=YES` for Debug builds |
| Indexing during build | Set `COMPILER_INDEX_STORE_ENABLE=NO` for Release builds |

### "Test bundle could not be loaded"

**Cause:** The test target's host application is not built, or the test bundle is not signed.

**Fix:** Ensure the test target has a valid host application. For macOS unit tests, the host is typically the app itself. Check that `TEST_HOST` is set correctly in the test target's build settings.
