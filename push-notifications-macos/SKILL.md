---
name: push-notifications-macos
description: "Push notifications on macOS — UNUserNotificationCenter authorization (different prompt behavior), UNNotificationRequest, UNUserNotificationCenterDelegate, notification categories/actions, APNs registration via NSApplicationDelegate, foreground delivery differences, Focus modes impact. Use when implementing push or local notifications in macOS apps."
sources: https://developer.apple.com/documentation/usernotifications, https://developer.apple.com/documentation/appkit/nsapplicationdelegate
researched: 2026-06-06
platform: macOS 26+
---

# Push Notifications on macOS

Push and local notifications on macOS 26+ with Swift 6.3. Covers UNUserNotificationCenter authorization, local notification scheduling with UNMutableNotificationRequest, remote notification registration via NSApplicationDelegate, UNUserNotificationCenterDelegate for foreground delivery and action handling, notification categories and actions, Focus mode filtering with interruption levels, and custom content extensions. For AppKit fundamentals, see `appkit-foundations`. For system integration patterns, see `macos-system-integration`.

See [references/push-patterns.md](references/push-patterns.md) for complete local notification with category and action, remote notification registration and handling, UNUserNotificationCenterDelegate implementation, Focus mode-aware notification scheduling, and custom notification content extension reference.

## Contents

- [Authorization](#authorization)
- [Local Notifications](#local-notifications)
- [Remote Notifications](#remote-notifications)
- [UNUserNotificationCenterDelegate](#usernotificationcenterdelegate)
- [Notification Categories & Actions](#notification-categories--actions)
- [Foreground Delivery](#foreground-delivery)
- [Focus Modes](#focus-modes)
- [Notification Content Extension](#notification-content-extension)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Authorization

Request notification authorization early in your app's lifecycle — ideally in your app delegate's `applicationDidFinishLaunching` or in an early view controller. The user sees a system prompt the first time you request authorization.

```swift
import UserNotifications

func requestNotificationAuthorization() async throws -> Bool {
    let center = UNUserNotificationCenter.current()
    let granted = try await center.requestAuthorization(options: [.alert, .badge, .sound, .provisional])
    return granted
}
```

**`UNAuthorizationOptions` flags:**

| Option | Effect |
|--------|--------|
| `.alert` | Show alert banners |
| `.badge` | Update app icon badge |
| `.sound` | Play notification sounds |
| `.provisional` | **iOS only** — On macOS this flag is ignored |
| `.carPlay` | CarPlay — not applicable on macOS |
| `.criticalAlert` | Bypass Do Not Disturb and Focus modes (requires Apple entitlement) |
| `.providesAppNotificationSettings` | Link to in-app notification settings |

**macOS prompt behavior — key difference from iOS:**

On macOS, `requestAuthorization(options:)` shows the system prompt **only once per app install**. After the user responds (Allow or Deny), subsequent calls return the current authorization status without showing the prompt again. The user must change notification permissions in System Settings > Notifications. This differs from iOS where the prompt can reappear in certain edge cases (e.g., app reinstall with different bundle ID).

There is **no provisional/quiet authorization on macOS** — setting `.provisional` in the options is silently ignored. All authorization requests are explicit.

**Checking current authorization status:**

```swift
func checkNotificationSettings() async -> UNAuthorizationStatus {
    let center = UNUserNotificationCenter.current()
    let settings = await center.notificationSettings()
    return settings.authorizationStatus
}

// Usage:
let status = await checkNotificationSettings()
switch status {
case .authorized:
    print("Fully authorized — alerts, badge, sound allowed")
case .denied:
    print("Denied — direct user to System Settings > Notifications")
case .notDetermined:
    print("Not yet requested — call requestAuthorization")
case .provisional:
    print("Provisional — unlikely on macOS, handled as authorized")
case .ephemeral:
    print("Ephemeral — typically for App Clip experiences")
@unknown default:
    print("Unknown status")
}
```

**The `UNNotificationSettings` object** provides granular access to each delivery mechanism:

```swift
let settings = await center.notificationSettings()

// Per-capability status
let alertsEnabled = (settings.alertSetting == .enabled)
let badgesEnabled = (settings.badgeSetting == .enabled)
let soundsEnabled = (settings.soundSetting == .enabled)
let lockScreenEnabled = (settings.lockScreenSetting == .enabled)      // N/A on macOS
let carPlayEnabled = (settings.carPlaySetting == .enabled)              // N/A on macOS

// Notification center display
let notificationCenter = (settings.notificationCenterSetting == .enabled)

// Banner style (macOS)
let alertStyle: UNAlertStyle = settings.alertStyle  // .banner, .alert, .none
```

## Local Notifications

Local notifications are scheduled by your app and delivered by the notification center without a server.

### Building a Notification

```swift
// 1. Content
let content = UNMutableNotificationContent()
content.title = "Reminder"
content.subtitle = "Task due soon"
content.body = "Your report is due in 15 minutes"
content.badge = 1
content.sound = .default
content.categoryIdentifier = "TASK_REMINDER"
content.userInfo = ["taskID": "task-123", "projectID": "proj-456"]

// 2. Trigger
// Time-based
let timeTrigger = UNTimeIntervalNotificationTrigger(
    timeInterval: 60 * 15,  // 15 minutes
    repeats: false
)

// Calendar-based
var dateComponents = DateComponents()
dateComponents.hour = 9
dateComponents.minute = 0
dateComponents.weekday = 2  // Monday
let calendarTrigger = UNCalendarNotificationTrigger(
    dateMatching: dateComponents,
    repeats: true
)

// Location-based (requires Core Location authorization)
// let region = CLCircularRegion(center: coordinate, radius: 100, identifier: "office")
// let locationTrigger = UNLocationNotificationTrigger(region: region, repeats: false)

// 3. Request
let request = UNNotificationRequest(
    identifier: "task-reminder-\(UUID().uuidString)",
    content: content,
    trigger: timeTrigger
)

// 4. Schedule
try await UNUserNotificationCenter.current().add(request)
```

**Trigger types:**

| Trigger | Class | Repeats |
|---------|-------|---------|
| Time interval | `UNTimeIntervalNotificationTrigger` | Yes, but minimum 60s between repeats |
| Calendar | `UNCalendarNotificationTrigger` | Yes, with date components |
| Location | `UNLocationNotificationTrigger` | Yes, withCLCircularRegion |

**Reusing identifiers:** If you schedule a notification with an identifier that already exists, the new request replaces the old one. Use this for updating pending reminders.

### Managing Pending Notifications

```swift
// List all pending
let pending = await center.pendingNotificationRequests()
print("Pending: \(pending.count)")

// Cancel specific
center.removePendingNotificationRequests(withIdentifiers: ["task-reminder-abc"])

// Cancel all
center.removePendingNotificationRequests(withIdentifiers: pending.map(\.identifier))

// List delivered notifications (still in Notification Center)
let delivered = await center.deliveredNotifications()

// Remove delivered
center.removeDeliveredNotifications(withIdentifiers: ["task-reminder-abc"])
```

## Remote Notifications

Remote (push) notifications require APNs (Apple Push Notification service) registration through NSApplicationDelegate.

### Enabling Push Notifications

1. Enable the **Push Notifications** capability in your target's Signing & Capabilities.
2. For a standard APNs connection, no additional entitlements needed. For Notification Service Extension, create a separate target.

### Registration via NSApplicationDelegate

```swift
import Cocoa
import UserNotifications

final class AppDelegate: NSObject, NSApplicationDelegate {
    func applicationDidFinishLaunching(_ notification: Notification) {
        // 1. Set the notification center delegate
        UNUserNotificationCenter.current().delegate = self

        // 2. Register for remote notifications
        NSApp.registerForRemoteNotifications()
    }

    // MARK: - NSApplicationDelegate push methods

    /// Called when APNs registration succeeds.
    func application(
        _ application: NSApplication,
        didRegisterForRemoteNotificationWithDeviceToken deviceToken: Data
    ) {
        let token = deviceToken.map { String(format: "%02.2hhx", $0) }.joined()
        print("APNs token: \(token)")
        // Send this token to your push server
        Task {
            await PushService.shared.registerDeviceToken(token)
        }
    }

    /// Called when APNs registration fails.
    func application(
        _ application: NSApplication,
        didFailToRegisterForRemoteNotificationWithError error: Error
    ) {
        print("Failed to register for remote notifications: \(error)")
        // Common cause: Missing Push Notifications entitlement, or running in a context
        // where push is unavailable (e.g., certain debug configurations)
    }
}
```

**Remote notification payload format:**

```json
{
    "aps": {
        "alert": {
            "title": "New Message",
            "subtitle": "From Alice",
            "body": "Hey, are you free for lunch?"
        },
        "badge": 3,
        "sound": "default",
        "category": "MESSAGE_CATEGORY",
        "mutable-content": 1,
        "interruption-level": "time-sensitive"
    },
    "messageID": "msg-789",
    "conversationID": "conv-456"
}
```

**Receiving remote notifications while the app is running:**

```swift
extension AppDelegate: UNUserNotificationCenterDelegate {
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        willPresent notification: UNNotification,
        withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void
    ) {
        // Called when a notification arrives while the app is in the foreground
        // On macOS, the system still shows the banner unless you call completionHandler([])
        let userInfo = notification.request.content.userInfo
        if let messageID = userInfo["messageID"] as? String {
            // Update UI in-place instead of showing a banner
            handleIncomingMessage(id: messageID)
            completionHandler([])  // suppress banner
        } else {
            completionHandler([.banner, .sound])
        }
    }
}
```

## UNUserNotificationCenterDelegate

The delegate handles foreground notification delivery and user interaction with notifications. Set it early — before `applicationDidFinishLaunching` returns.

```swift
final class NotificationDelegate: NSObject, UNUserNotificationCenterDelegate {
    // MARK: - Foreground Delivery

    /// Called when a notification is delivered while the app is in the foreground.
    /// On macOS, the system will still display the banner unless you call
    /// completionHandler with an empty array.
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        willPresent notification: UNNotification,
        withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void
    ) {
        let category = notification.request.content.categoryIdentifier

        switch category {
        case "CHAT_MESSAGE":
            // Suppress banner — we'll update the chat UI directly
            completionHandler([])
        case "TASK_REMINDER":
            // Show banner but no sound (user is already working)
            completionHandler([.banner])
        default:
            // Default: show everything
            completionHandler([.banner, .sound, .badge])
        }
    }

    // MARK: - Action Response

    /// Called when the user clicks a notification action or the notification body.
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse,
        withCompletionHandler completionHandler: @escaping () -> Void
    ) {
        let userInfo = response.notification.request.content.userInfo
        let actionIdentifier = response.actionIdentifier

        switch actionIdentifier {
        case UNNotificationDefaultActionIdentifier:
            // User tapped the notification body
            handleNotificationTap(userInfo: userInfo)

        case "REPLY_ACTION":
            // User replied via text input action
            if let textResponse = response as? UNTextInputNotificationResponse {
                let replyText = textResponse.userText
                handleReply(text: replyText, userInfo: userInfo)
            }

        case "MARK_DONE_ACTION":
            handleMarkDone(userInfo: userInfo)

        case "DELETE_ACTION":
            handleDelete(userInfo: userInfo)

        case UNNotificationDismissActionIdentifier:
            // User dismissed the notification
            handleDismiss(userInfo: userInfo)

        default:
            break
        }

        // Always call completionHandler when done
        completionHandler()
    }
}
```

**Key rules for the delegate:**

- Always call `completionHandler` — failing to call it can cause the notification center to hang.
- `willPresent` is called for both local and remote notifications when the app is foreground.
- `didReceive` is called after any user interaction (tap, action, dismiss).
- Set the delegate **before** requesting authorization, or you may miss the initial foreground delivery callback.

## Notification Categories & Actions

Categories group notification types and define the actions available when a user interacts with a notification.

### Defining Actions

```swift
// Simple action — taps launch the app (or bring it to foreground)
let replyAction = UNNotificationAction(
    identifier: "REPLY_ACTION",
    title: "Reply",
    options: [.foreground]  // brings app to foreground when tapped
)

// Destructive action — shown in red
let deleteAction = UNNotificationAction(
    identifier: "DELETE_ACTION",
    title: "Delete",
    options: [.destructive]  // displayed in red in the notification UI
)

// Authentication-required action — requires device unlock
let markDoneAction = UNNotificationAction(
    identifier: "MARK_DONE_ACTION",
    title: "Mark Done",
    options: [.authenticationRequired]  // won't fire unless device is unlocked
)

// Text input action — presents a text field in the notification
let textInputAction = UNTextInputNotificationAction(
    identifier: "QUICK_REPLY_ACTION",
    title: "Quick Reply",
    options: [],
    textInputButtonTitle: "Send",
    textInputPlaceholder: "Type a message..."
)
```

**Action options:**

| Option | Effect |
|--------|--------|
| `.foreground` | Brings the app to foreground when action is tapped |
| `.destructive` | Renders the action title in red |
| `.authenticationRequired` | Only fires if the device is unlocked at tap time |

You can combine options: `[.foreground, .destructive]`

### Defining Categories

```swift
// A category with multiple actions
// Actions are shown in order — first action is the primary/leftmost one
let messageCategory = UNNotificationCategory(
    identifier: "MESSAGE_CATEGORY",
    actions: [replyAction, textInputAction, deleteAction],
    intentIdentifiers: [],  // Leave empty unless you have SiriKit intents
    options: [.customDismissAction]  // triggers didReceive with dismiss action identifier
)

// A category with authentication-required actions
let sensitiveCategory = UNNotificationCategory(
    identifier: "SENSITIVE_CATEGORY",
    actions: [markDoneAction, deleteAction],
    intentIdentifiers: [],
    options: [.allowInCarPlay]  // N/A on macOS, ignored
)

// Register categories
UNUserNotificationCenter.current().setNotificationCategories([
    messageCategory,
    sensitiveCategory
])
```

**Assigning a category to a notification:**

```swift
let content = UNMutableNotificationContent()
content.title = "New Message"
content.body = "Alice: Are you coming to the meeting?"
content.categoryIdentifier = "MESSAGE_CATEGORY"
content.userInfo = ["messageID": "msg-123"]
```

### How Actions Appear

On macOS, actions appear when the user right-clicks or force-clicks a notification banner/alert, or by hovering and clicking the options button. The first action is the primary action. On alert-style notifications, actions appear as buttons.

## Foreground Delivery

**Critical difference from iOS:** On macOS, notifications are delivered to the app **and** displayed by the system even when the app is in the foreground. On iOS, foreground notifications are silently delivered to the delegate without any system UI. On macOS, you must explicitly suppress the banner by calling `completionHandler([])` in `willPresent` if you don't want it shown.

```swift
func userNotificationCenter(
    _ center: UNUserNotificationCenter,
    willPresent notification: UNNotification,
    withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void
) {
    // macOS: The banner WILL show unless you suppress it
    // Option 1: Suppress entirely (update your UI directly)
    updateInAppUI(with: notification.request.content)
    completionHandler([])

    // Option 2: Show banner but suppress sound
    // completionHandler([.banner])

    // Option 3: Show everything (default behavior)
    // completionHandler([.banner, .sound, .badge])
}
```

**Presentation options (macOS):**

| Option | Effect |
|--------|--------|
| `.banner` | Show the notification banner |
| `.sound` | Play the notification sound |
| `.badge` | Update the app icon badge |
| `.list` | Display in the Notification Center list (macOS 12+) |

Note: `.alert` style and `.criticalAlert` are macOS-specific considerations — the system notification settings determine whether the user sees banners or alerts.

## Focus Modes

macOS Focus modes (Do Not Disturb, Work, Sleep, custom) filter notifications. Your app has limited but important influence over this.

### Interruption Levels

Set the interruption level in your notification content to indicate urgency:

```swift
let content = UNMutableNotificationContent()
content.title = "Urgent"
content.body = "Server is down!"
content.interruptionLevel = .timeSensitive  // or .passive, .active, .critical
```

| Level | Behavior |
|-------|----------|
| `.passive` | Silently delivered, no banner, no sound. Appears in Notification Center only. |
| `.active` | Default. Shows banner, plays sound (if enabled). |
| `.timeSensitive` | Can break through Focus modes the user has allowed time-sensitive notifications for. |
| `.critical` | Bypasses all Focus modes and Do Not Disturb. Requires a special entitlement from Apple. |

### Checking Focus Status

```swift
import Intents

func checkFocusStatus() async -> Bool {
    let center = INFocusStatusCenter.default
    // authorizationStatus must be .authorized
    let isFocused = center.focusStatus.isFocused
    return isFocused ?? false  // nil if undetermined
}

// Usage: adapt notification behavior
func scheduleNotification(content: UNMutableNotificationContent) async {
    let isFocused = await checkFocusStatus()

    if isFocused && content.interruptionLevel != .timeSensitive && content.interruptionLevel != .critical {
        // User is in Focus mode — lower the interruption level so the notification
        // doesn't get silently dropped
        content.interruptionLevel = .passive
    }

    let request = UNNotificationRequest(
        identifier: UUID().uuidString,
        content: content,
        trigger: nil
    )
    try? await UNUserNotificationCenter.current().add(request)
}
```

**`INFocusStatusCenter` authorization:**

```swift
// Request access (one-time)
let status = await INFocusStatusCenter.default.requestAuthorization()
// Returns .notDetermined, .restricted, .denied, or .authorized
```

### Focus Filter (iOS concept — not applicable)

Focus Filter API (custom app behavior per Focus mode) is **iOS only**. On macOS, you can only influence delivery via interruption levels. There is no API to provide different notification content based on the active Focus mode.

## Notification Content Extension

A Notification Content Extension provides custom UI for your notifications. This is a separate app extension target.

### Setup

1. Add a **Notification Content Extension** target to your Xcode project.
2. The template provides a `NotificationViewController` subclass.
3. The extension's `Info.plist` declares which categories it handles.

### Info.plist Configuration

In the extension's `Info.plist`:

```xml
<key>NSExtension</key>
<dict>
    <key>NSExtensionAttributes</key>
    <dict>
        <key>UNNotificationExtensionCategory</key>
        <array>
            <string>CUSTOM_UI_CATEGORY</string>
        </array>
        <key>UNNotificationExtensionInitialContentSizeRatio</key>
        <real>0.5</real>
        <key>UNNotificationExtensionDefaultContentHidden</key>
        <false/>
        <key>UNNotificationExtensionUserInteractionEnabled</key>
        <true/>
    </dict>
    <key>NSExtensionPrincipalClass</key>
    <string>$(PRODUCT_MODULE_NAME).NotificationViewController</string>
    <key>NSExtensionPointIdentifier</key>
    <string>com.apple.usernotifications.content-extension</string>
</dict>
```

### View Controller Implementation

```swift
import UIKit  // Content extensions use UIKit, even on macOS
import UserNotifications
import UserNotificationsUI

final class NotificationViewController: UIViewController, UNNotificationContentExtension {
    @IBOutlet var label: UILabel!
    @IBOutlet var imageView: UIImageView!

    func didReceive(_ notification: UNNotification) {
        label.text = notification.request.content.body

        // Access custom payload data
        let userInfo = notification.request.content.userInfo
        if let imageURL = userInfo["imageURL"] as? String,
           let url = URL(string: imageURL) {
            loadImage(from: url)
        }
    }

    func didReceive(
        _ response: UNNotificationResponse,
        completionHandler completion: @escaping (UNNotificationContentExtensionResponseOption) -> Void
    ) {
        // Handle actions within the custom UI
        if response.actionIdentifier == "INLINE_ACTION" {
            // Process action within the extension
            completion(.dismissAndForwardAction)  // forward to app
        } else {
            completion(.doNotDismiss)
        }
    }

    private func loadImage(from url: URL) {
        URLSession.shared.dataTask(with: url) { [weak self] data, _, _ in
            if let data = data, let image = UIImage(data: data) {
                DispatchQueue.main.async {
                    self?.imageView.image = image
                }
            }
        }.resume()
    }
}
```

**Content extension gotchas:**
- Content extensions use UIKit, not AppKit, even in macOS targets.
- The extension runs in a separate process with limited memory (~24MB). Keep it lean.
- `UNNotificationExtensionUserInteractionEnabled` must be `true` for the user to interact with your custom UI.
- `UNNotificationExtensionDefaultContentHidden` set to `true` hides the standard notification UI, showing only your custom view.

## Constraints & Gotchas

- **No provisional authorization on macOS.** The `.provisional` option is ignored. Notifications are either authorized or not — there is no "deliver quietly" tier.

- **macOS shows foreground banners by default.** Unlike iOS, you must explicitly suppress them in `willPresent` by calling `completionHandler([])`.

- **No Focus Filter API.** macOS does not support the iOS Focus Filter feature. You can only set interruption levels and check focus status.

- **`UNTimeIntervalNotificationTrigger` minimum repeat interval is 60 seconds.** Attempting to set a repeat interval below 60s raises an error.

- **Device token is app-specific, not install-specific.** The token can change after OS updates, restoring from backup, or changing the APNs environment. Always re-register on app launch and update your server.

- **Notification authorization prompt appears only once.** After the user responds, they must change permissions in System Settings > Notifications. Provide an in-app button that opens the settings pane:

```swift
if let url = URL(string: "x-apple.systempreferences:com.apple.Notifications-Settings.extension") {
    NSWorkspace.shared.open(url)
}
```

- **Sandbox entitlement:** Non-sandboxed macOS apps can always register for push. Sandboxed apps need the `com.apple.developer.aps-environment` entitlement (added automatically when you enable Push Notifications capability).

- **Test builds and APNs:** Development builds use the APNs sandbox environment (`development`), while TestFlight and App Store builds use production. Device tokens are environment-specific — a sandbox token won't work with the production APNs endpoint.

- **Badge count persistence:** The badge count is part of the notification payload. If you set `badge` to 0, the badge is cleared. If you don't include the `badge` key, the badge count is unchanged. There is no "increment" behavior — always set the absolute value.

- **Notification groups:** On macOS 12+, notifications can be grouped using `threadIdentifier` in the content. Notifications with the same `threadIdentifier` are grouped together in the Notification Center list.

- **Relevance scores:** You can provide `relevanceScore` (0.0 to 1.0) to influence notification ordering in the Summary. Higher relevance = higher placement.

```swift
content.relevanceScore = 0.8
```

## Common Mistakes

1. **Not setting `UNUserNotificationCenter.current().delegate` early enough.** If you set it after requesting authorization, you may miss `willPresent` callbacks. Set it in `applicationDidFinishLaunching` before registering for remote notifications.

2. **Forgetting to call `completionHandler` in `willPresent` or `didReceive`.** This causes the notification center to hang and can result in notifications being dropped silently.

3. **Expecting provisional authorization on macOS.** The `.provisional` option exists in `UNAuthorizationOptions` but has no effect on macOS. All notifications require explicit user permission or the user adjusts settings post-denial.

4. **Not suppressing foreground banners on macOS.** Because macOS shows banners by default (unlike iOS), failing to call `completionHandler([])` results in duplicate UI — your in-app update plus the system banner.

5. **Hardcoding the APNs token.** The token changes across installs, OS updates, and environment switches. Always call `registerForRemoteNotifications()` at launch and handle updates in the delegate.

6. **Using the same notification identifier for different notifications.** Replacing a pending notification may be intentional, but if you mean to create two distinct notifications, use unique identifiers.

7. **Repeating triggers below 60 seconds.** The system enforces a minimum repeat interval of 60 seconds for `UNTimeIntervalNotificationTrigger`. Attempting to set a shorter interval crashes.

8. **Not handling the dismiss action.** If you set `.customDismissAction` on a category, implement the dismiss case in `didReceive`. Otherwise you lose tracking of dismissed notifications.

9. **Including `mutable-content: 1` in your push payload without a Notification Service Extension.** This tells the system to call your service extension before display. Without one, the notification displays normally, but it's wasted payload data.

## Review Checklist

- [ ] Authorization requested with appropriate options (`.alert`, `.badge`, `.sound` — no `.provisional` on macOS)
- [ ] Authorization status checked before scheduling notifications
- [ ] `UNUserNotificationCenter.current().delegate` set in `applicationDidFinishLaunching`
- [ ] `NSApp.registerForRemoteNotifications()` called after delegate is set
- [ ] Device token forwarded to server in `didRegisterForRemoteNotificationWithDeviceToken`
- [ ] `willPresent` handles foreground delivery (suppresses or shows banner appropriately)
- [ ] `didReceive` handles all action identifiers including default tap and dismiss
- [ ] `completionHandler` called in both `willPresent` and `didReceive`
- [ ] Categories registered via `setNotificationCategories` before scheduling notifications with those categories
- [ ] Notification content includes `categoryIdentifier` to match registered categories
- [ ] Interruption level set appropriately (`.timeSensitive` for important, `.passive` for low-priority)
- [ ] Focus status checked before scheduling time-sensitive notifications
- [ ] Notification Content Extension Info.plist configured with correct category identifiers
- [ ] Timer triggers respect 60-second minimum repeat interval
- [ ] Badge count set explicitly (absolute value, not increment)
- [ ] Unique `identifier` used for each distinct notification request
- [ ] In-app UI provided to redirect denied users to System Settings > Notifications

## References

- [UNUserNotificationCenter | Apple Developer Documentation](https://developer.apple.com/documentation/usernotifications/unusernotificationcenter)
- [UNUserNotificationCenterDelegate | Apple Developer Documentation](https://developer.apple.com/documentation/usernotifications/unusernotificationcenterdelegate)
- [UNNotificationRequest | Apple Developer Documentation](https://developer.apple.com/documentation/usernotifications/unnotificationrequest)
- [UNMutableNotificationContent | Apple Developer Documentation](https://developer.apple.com/documentation/usernotifications/unmutablenotificationcontent)
- [UNNotificationCategory | Apple Developer Documentation](https://developer.apple.com/documentation/usernotifications/unnotificationcategory)
- [UNNotificationAction | Apple Developer Documentation](https://developer.apple.com/documentation/usernotifications/unnotificationaction)
- [UNTextInputNotificationAction | Apple Developer Documentation](https://developer.apple.com/documentation/usernotifications/untextinputnotificationaction)
- [UNNotificationContentExtension | Apple Developer Documentation](https://developer.apple.com/documentation/usernotificationsui/unnotificationcontentextension)
- [INFocusStatusCenter | Apple Developer Documentation](https://developer.apple.com/documentation/intents/infocusstatuscenter)
- [NSApplicationDelegate push notification methods](https://developer.apple.com/documentation/appkit/nsapplicationdelegate)
- [UserNotifications Framework | Apple Developer Documentation](https://developer.apple.com/documentation/usernotifications)

## Research Addendum — 2026 Updates

### UNUserNotificationCenter Authorization (macOS 26)

- **Main-thread requirement (macOS 26)**: `UNUserNotificationCenter.requestAuthorization` and its completion handler must run on the main thread. Violation causes a `libdispatch` crash (`EXC_BREAKPOINT`). This was relaxed in earlier macOS versions but is now strictly enforced.
  ```swift
  // Correct: dispatch to main if called from background
  await MainActor.run {
      let granted = try await UNUserNotificationCenter.current()
          .requestAuthorization(options: [.alert, .badge, .sound])
  }
 ```
- **macOS prompt = once per install**: The system shows the authorization prompt only once. After that, users must change permissions in System Settings > Notifications.
- **`.provisional` is silently ignored on macOS** — all authorization is explicit. There is no "deliver quietly" tier.
- **Required options**: Only `.alert`, `.badge`, `.sound`, `.criticalAlert`, and `.providesAppNotificationSettings` are meaningful on macOS. Others (`.carPlay`, `.provisional`) are ignored.

### APNs Registration via NSApplicationDelegate (2026)

- **`NSApp.registerForRemoteNotifications()`** remains the registration call. The delegate methods `application(_:didRegisterForRemoteNotificationWithDeviceToken:)` and `application(_:didFailToRegisterForRemoteNotificationWithError:)` are unchanged.
- **Token lifecycle**: Device tokens are app-specific and can change after OS updates, device restores, or APNs environment changes. Always re-register at app launch.
- **Sandbox entitlement**: Sandboxed apps need `com.apple.developer.aps-environment` (auto-added with Push Notifications capability capability). Non-sandboxed apps don't need it.
- **Environment**: Development builds → sandbox APNs. TestFlight/App Store → production. Tokens are NOT interchangeable across environments.
- **Test builds need the entitlement** even when running outside Xcode. The push Notifications capability in Signing & Capabilities handles this.

### Notification Categories & Actions (2026)

- **Always register a category**: Even for notifications with no actions, register a category (e.g., `"default"`). macOS may not properly route banner clicks back to your process without a registered category.
- **Action limits**: Up to 4 actions per category. Banners show max 2; alerts show all.
- **`UNNotificationActionIcon`** (iOS 15+/macOS 12+): Add SF Symbol icons to action buttons for visual distinction.
- **Communication notifications**: For messaging apps, use `INSendMessageIntent` with the Communication Notifications capability. Shows sender avatar and enables inline reply.
- **`relevanceScore`** (0.0–1.0): Influence notification ordering in Notification Center and Summary. Higher = higher placement.
- **`threadIdentifier`**: Groups related notifications together in Notification Center. Essential for conversation-style notifications.

### Focus Mode & Interruption Levels (2026)

- **Four tiers**: `.passive` (silent, no banner), `.active` (default), `.timeSensitive` (breaks through Focus), `.critical` (bypasses everything, requires Apple entitlement).
- **`INFocusStatusCenter`**: Check focus status before scheduling. Requires `INFocusStatusCenter.requestAuthorization()`.
- **Focus adaptations**: Lower interruption level to `.passive` when user is in a Focus you don't have time-sensitive permission to bypass.
- **No Focus Filter on macOS**: The iOS Focus Filter API is unavailable. You can only read focus status and set interruption levels.
- **`relevanceScore` alternative**: For time-sensitive game or fitness notifications, consider `relevanceScore` as a secondary signal alongside interruption level.
