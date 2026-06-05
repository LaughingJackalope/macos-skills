# ServiceManagement: Launch Agents and Launch Daemons

Complete reference for `SMAppService` patterns on macOS 13+, LaunchAgent/LaunchDaemon plist format, and migration from the deprecated `SMLoginItemSetEnabled` API.

## SMAppService Overview

`SMAppService` is the unified replacement for `SMLoginItemSetEnabled`, `LSSharedFileList`, and manual `launchctl` usage. It provides async/await registration, proper error handling, and status reporting.

### Service Type Selection

| Type | API | Use Case | GUI? | Runs At |
|------|-----|----------|------|---------|
| Login Item | `SMAppService.loginItem(identifier:)` | Helper app that needs a UI (menu bar extra, dock icon) | Yes | User login |
| Agent | `SMAppService.agent(plistName:)` | Background user process (XPC service, sync engine) | No | User login |
| Daemon | `SMAppService.daemon(plistName:)` | System-level background process | No | System boot |

**Rule of thumb:** If the helper needs a window or menu bar, use `.loginItem`. If it's purely background, use `.agent`. Use `.daemon` only for system-wide services that must run before any user logs in.

## Login Item Setup

### Bundle Structure

```
MyApp.app/
  Contents/
    Library/
      LoginItems/
        MyHelper.app/
          Contents/
            Info.plist          # LSBackgroundOnly = YES or LSUIElement = YES
            MacOS/
              MyHelper
            Resources/
              ...
```

The helper app's `Info.plist` must include:
- `LSBackgroundOnly = YES` (no dock icon, no menu bar) — or —
- `LSUIElement = YES` (no dock icon, but can have a menu bar)

### Registration with Error Handling

```swift
import ServiceManagement

@MainActor
final class LoginItemController: ObservableObject {
    let service = SMAppService.loginItem(identifier: "com.example.MyApp.Helper")

    @Published var status: SMAppService.Status = .notRegistered
    @Published var errorMessage: String?

    init() {
        status = service.status
    }

    func register() async {
        do {
            try await service.register()
            status = service.status

            if status == .requiresApproval {
                promptForApproval()
            }
        } catch {
            errorMessage = "Registration failed: \(error.localizedDescription)"
        }
    }

    func unregister() async {
        do {
            try await service.unregister()
            status = service.status
        } catch {
            errorMessage = "Unregistration failed: \(error.localizedDescription)"
        }
    }

    private func promptForApproval() {
        guard let url = URL(string: "x-apple.systempreferences:com.apple.LoginItems-Settings.extension") else { return }
        NSWorkspace.shared.open(url)
    }
}
```

### SwiftUI Toggle View

```swift
struct LoginItemToggle: View {
    @StateObject private var controller = LoginItemController()

    var body: some View {
        Toggle("Open at Login", isOn: Binding(
            get: { controller.status == .enabled },
            set: { _ in
                Task {
                    await controller.toggle()
                }
            }
        ))
        .disabled(controller.status == .requiresApproval)
        .help(controller.status == .requiresApproval
              ? "Approval required in System Settings > General > Login Items"
              : "")
    }
}
```

## LaunchAgent Plist Format

Place the plist in your app's bundle and copy it to `~/Library/LaunchAgents/` at runtime. The plist name must match the name passed to `SMAppService.agent(plistName:)`.

### Minimal Agent Plist

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.MyApp.Agent</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Applications/MyApp.app/Contents/Library/LaunchAgents/MyAgent</string>
        <string>--background</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <false/>

    <key>StandardOutPath</key>
    <string>/tmp/com.example.MyApp.Agent.stdout.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/com.example.MyApp.Agent.stderr.log</string>
</dict>
</plist>
```

### Agent with Throttling and Environment

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.MyApp.SyncAgent</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Applications/MyApp.app/Contents/Library/LaunchAgents/SyncAgent</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>ThrottleInterval</key>
    <integer>30</integer>

    <key>WorkingDirectory</key>
    <string>/Applications/MyApp.app/Contents/Library/LaunchAgents</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>MY_APP_BUNDLE_PATH</key>
        <string>/Applications/MyApp.app</string>
        <key>OS_ACTIVITY_MODE</key>
        <string>disable</string>
    </dict>

    <key>ProcessType</key>
    <string>Background</string>

    <key>Nice</key>
    <integer>10</integer>
</dict>
</plist>
```

### Key Plist Keys Reference

| Key | Type | Default | Purpose |
|-----|------|---------|---------|
| `Label` | String | Required | Unique identifier (reverse-DNS) |
| `ProgramArguments` | Array | Required | First element is the executable path, rest are arguments |
| `RunAtLoad` | Bool | false | Launch immediately when the plist is loaded |
| `KeepAlive` | Bool/Dict | false | Restart if the process exits. Dict form allows conditional keepalive |
| `ThrottleInterval` | Int | 10 | Minimum seconds between launch attempts |
| `WorkingDirectory` | String | / | Working directory for the process |
| `EnvironmentVariables` | Dict | — | Environment variables passed to the process |
| `ProcessType` | String | Standard | `Background`, `Standard`, `Adaptive`, `Interactive` |
| `Nice` | Int | 0 | Process priority (-20 to 20, lower = higher priority) |
| `StandardOutPath` | String | — | Redirect stdout to a file |
| `StandardErrorPath` | String | — | Redirect stderr to a file |
| `SoftResourceLimits` | Dict | — | Per-process resource limits (NumberOfFiles, Core, etc.) |
| `HardResourceLimits` | Dict | — | Hard resource limits |

### KeepAlive Dictionary Form

```xml
<key>KeepAlive</key>
<dict>
    <key>SuccessfulExit</key>
    <false/>
    <key>NetworkState</key>
    <true/>
    <key>OtherJobEnabled</key>
    <dict>
        <key>com.example.MyApp.MainApp</key>
        <true/>
    </dict>
</dict>
```

This keeps the agent running only when: the exit was unsuccessful, the network is up, and the main app job is enabled.

## LaunchDaemon Plist Format

LaunchDaemons run in system context (root). Place plists in `/Library/LaunchDaemons/`. The format is identical to agents but with additional considerations:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.example.MyApp.Daemon</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Library/PrivilegedHelperTools/com.example.MyApp.Daemon</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>UserName</key>
    <string>root</string>

    <key>GroupName</key>
    <string>wheel</string>

    <key>SessionCreate</key>
    <false/>
</dict>
</plist>
```

**Important:** LaunchDaemons must be installed via `SMJobBless` or a privileged installer. `SMAppService.daemon(plistName:)` handles the `SMJobBless` flow internally, but the calling app must have the `com.apple.security.temporary-exception.sbpl` entitlement or use the ServiceManagement framework's built-in approval flow.

## Migration from SMLoginItemSetEnabled

### Before (Deprecated)

```swift
// Deprecated — do not use in new code
let success = SMLoginItemSetEnabled("com.example.MyApp.Helper" as CFString, true)
if !success {
    print("Failed to enable login item")
}
```

### After (SMAppService)

```swift
// Modern replacement
Task {
    do {
        let service = SMAppService.loginItem(identifier: "com.example.MyApp.Helper")
        try await service.register()

        switch service.status {
        case .enabled:
            print("Login item is active")
        case .requiresApproval:
            print("User must approve in System Settings")
        case .notFound:
            print("Helper bundle not found — check Contents/Library/LoginItems/")
        case .notRegistered:
            print("Not registered (unexpected after register())")
        @unknown default:
            break
        }
    } catch {
        print("Registration error: \(error)")
    }
}
```

### Migration Checklist

1. Replace all `SMLoginItemSetEnabled(identifier, true)` calls with `try await SMAppService.loginItem(identifier: identifier).register()`.
2. Replace all `SMLoginItemSetEnabled(identifier, false)` calls with `try await SMAppService.loginItem(identifier: identifier).unregister()`.
3. Remove the `Deprecated` compiler warning suppression if you had one.
4. Add `.requiresApproval` handling — the old API silently failed in this case.
5. The helper bundle location (`Contents/Library/LoginItems/`) is unchanged.
6. The helper's `Info.plist` requirements (`LSBackgroundOnly` or `LSUIElement`) are unchanged.
7. If you were using `SMCopyAllJobDictionaries` to check login item status, replace with `SMAppService.status`.

## Debugging

### Check Service Status

```swift
func debugServiceStatus(identifier: String) {
    let loginItem = SMAppService.loginItem(identifier: identifier)
    print("Login item status: \(loginItem.status)")

    // For agents/daemons, check the plist name
    let agent = SMAppService.agent(plistName: "\(identifier).Agent.plist")
    print("Agent status: \(agent.status)")
}
```

### Common Issues

| Symptom | Cause | Fix |
|---------|-------|-----|
| `.notFound` after `register()` | Helper bundle not at expected path | Verify `Contents/Library/LoginItems/Helper.app` exists and is signed |
| `.requiresApproval` persists | User hasn't approved in System Settings | Direct user to System Settings > General > Login Items |
| Agent doesn't launch | Plist has errors or wrong executable path | Run `launchctl print gui/$UID/com.example.agent` to diagnose |
| Agent crashes immediately | Code signing issue or missing entitlement | Check Console.app for crash logs, verify code signing |
| `register()` throws | Missing `SMAppService` entitlement | Add `com.apple.developer.service-management` entitlement |

### Console.app Filtering

Use these predicates in Console.app to debug launch agent issues:
- `process:launchd` — all launchd activity
- `subsystem:com.apple.ServiceManagement` — ServiceManagement framework messages
- `category:SMAppService` — SMAppService-specific messages
