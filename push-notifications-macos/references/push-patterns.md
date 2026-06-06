# Push Notifications Patterns

Complete reference for local and remote notification implementation on macOS 26+. Covers full local notification with category and action, remote notification registration and handling, UNUserNotificationCenterDelegate implementation, Focus mode-aware notification scheduling, and custom notification content extension.

## Complete Local Notification with Category and Action

The end-to-end flow for scheduling a local notification that has interactive actions:

```swift
import UserNotifications

// MARK: - 1. Define Actions and Category

func registerNotificationCategories() {
    // Reply with text input
    let replyAction = UNTextInputNotificationAction(
        identifier: "REPLY_ACTION",
        title: "Reply",
        options: [],
        textInputButtonTitle: "Send",
        textInputPlaceholder: "Type your reply..."
    )

    // Mark as done — requires device unlock
    let markDoneAction = UNNotificationAction(
        identifier: "DONE_ACTION",
        title: "Mark Done",
        options: [.authenticationRequired]
    )

    // Delete — destructive action shown in red
    let deleteAction = UNNotificationAction(
        identifier: "DELETE_ACTION",
        title: "Delete",
        options: [.destructive]
    )

    // Category that groups these actions
    let taskCategory = UNNotificationCategory(
        identifier: "TASK_CATEGORY",
        actions: [replyAction, markDoneAction, deleteAction],
        intentIdentifiers: [],
        options: [.customDismissAction, .allowInCarPlay]
    )

    UNUserNotificationCenter.current().setNotificationCategories([taskCategory])
}

// MARK: - 2. Create and Schedule Notification

func scheduleTaskReminder(task: Task) async throws {
    let content = UNMutableNotificationContent()
    content.title = "Task Due: \(task.title)"
    content.body = task.description
    content.categoryIdentifier = "TASK_CATEGORY"
    content.sound = .default
    content.badge = task.pendingCount as NSNumber
    content.threadIdentifier = "tasks-\(task.projectID)"  // groups by project
    content.interruptionLevel = task.isUrgent ? .timeSensitive : .active
    content.relevanceScore = task.isUrgent ? 1.0 : 0.5

    // Custom payload — available in didReceive response
    content.userInfo = [
        "taskID": task.id,
        "projectID": task.projectID,
        "priority": task.priority.rawValue
    ]

    // Attach an image if available
    if let attachmentURL = task.attachmentURL {
        let attachment = try await downloadImageAttachment(at: attachmentURL)
        content.attachments = [attachment]
    }

    // Calendar trigger — fire at the task's due date
    let trigger = UNCalendarNotificationTrigger(
        dateMatching: Calendar.current.dateComponents([.year, .month, .day, .hour, .minute], from: task.dueDate),
        repeats: false
    )

    let request = UNNotificationRequest(
        identifier: "task-\(task.id)",
        content: content,
        trigger: trigger
    )

    try await UNUserNotificationCenter.current().add(request)
}

// Helper: download an image and create a UNNotificationAttachment
func downloadImageAttachment(at url: URL) throws -> UNNotificationNotificationAttachment {
    let (data, _) = try await URLSession.shared.data(from: url)
    let tempURL = FileManager.default.temporaryDirectory.appendingPathComponent(UUID().uuidString + ".png")
    try data.write(to: tempURL)
    return try UNNotificationAttachment(identifier: "image", url: tempURL, options: [UNNotificationAttachmentOptionsTypeHintKey: UTType.png.identifier])
}

// MARK: - 3. Handle Actions in the Delegate

extension NotificationCoordinator: UNUserNotificationCenterDelegate {
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse,
        withCompletionHandler completionHandler: @escaping () -> Void
    ) {
        defer { completionHandler() }

        let userInfo = response.notification.request.content.userInfo
        guard let taskID = userInfo["taskID"] as? String else { return }

        switch response.actionIdentifier {
        case UNNotificationDefaultActionIdentifier:
            // User tapped the notification body — navigate to the task
            NavigationCoordinator.shared.showTask(id: taskID)

        case "REPLY_ACTION":
            if let textResponse = response as? UNTextInputNotificationResponse {
                let message = textResponse.userText
                Task { await TaskService.shared.addComment(to: taskID, text: message) }
            }

        case "DONE_ACTION":
            Task { await TaskService.shared.markComplete(taskID: taskID) }

        case "DELETE_ACTION":
            Task { await TaskService.shared.deleteTask(taskID: taskID) }

        case UNNotificationDismissActionIdentifier:
            // Log dismissal for analytics
            Analytics.track("task_notification_dismissed", properties: ["taskID": taskID])

        default:
            break
        }
    }
}
```

## Remote Notification Registration and Handling

Complete remote notification setup with token management:

```swift
import Cocoa
import UserNotifications

final class AppDelegate: NSObject, NSApplicationDelegate {

    func applicationDidFinishLaunching(_ notification: Notification) {
        // Set delegate BEFORE requesting authorization or registering
        UNUserNotificationCenter.current().delegate = NotificationCoordinator.shared

        Task {
            // Request authorization first
            let granted = try await requestAuthorization()

            if granted {
                // Then register for remote notifications
                NSApp.registerForRemoteNotifications()
            }
        }
    }

    private func requestAuthorization() async throws -> Bool {
        let center = UNUserNotificationCenter.current()

        // Check current status first — don't re-prompt if already determined
        let settings = await center.notificationSettings()
        guard settings.authorizationStatus == .notDetermined else {
            return settings.authorizationStatus == .authorized
        }

        return try await center.requestAuthorization(options: [.alert, .badge, .sound])
    }

    // MARK: - APNs Registration

    func application(
        _ application: NSApplication,
        didRegisterForRemoteNotificationWithDeviceToken deviceToken: Data
    ) {
        let tokenString = tokenToString(deviceToken)
        print("Registered with APNs. Token: \(tokenString)")

        // Send to your server — always, since tokens can change
        Task {
            await PushDeviceRegistry.shared.updateToken(tokenString)
        }
    }

    func application(
        _ application: NSApplication,
        didFailToRegisterForRemoteNotificationWithError error: Error
    ) {
        print("APNs registration failed: \(error.localizedDescription)")

        // Common causes:
        // - Missing Push Notifications capability
        // - App not code-signed with a valid provisioning profile
        // - Network unavailable for APNs connection
    }

    // MARK: - Remote Notification Reception

    // When the app is in the background or not running, the system handles
    // notification delivery. When the user taps the notification, the app
    // launches and you can check launchOptions:
    func application(_ application: NSApplication, didReceiveRemoteNotification userInfo: [AnyHashable: Any]) {
        // This fires when a remote notification is received while the app is running
        // in the foreground, OR when the app was in the background and the notification
        // was delivered.
        handleRemoteNotificationPayload(userInfo)
    }

    private func tokenToString(_ token: Data) -> String {
        token.map { String(format: "%02.2hhx", $0) }.joined()
    }

    private func handleRemoteNotificationPayload(_ userInfo: [AnyHashable: Any]) {
        guard let aps = userInfo["aps"] as? [String: Any] else { return }

        // Handle badge update
        if let badge = aps["badge"] as? Int {
            NSApp.dockTile.badgeLabel = String(badge)
        }

        // Handle custom payload
        if let messageID = userInfo["messageID"] as? String {
            NotificationCenter.default.post(name: .remoteMessageReceived, object: nil, userInfo: ["messageID": messageID])
        }
    }
}
```

### Push Notification Server Payload Reference

```json
{
    "aps": {
        "alert": {
            "title": "Ship It!",
            "subtitle": "Build #42 complete",
            "body": "Your app has been submitted to the App Store."
        },
        "badge": 1,
        "sound": "Glass.aiff",
        "category": "BUILD_STATUS",
        "threadIdentifier": "build-42",
        "interruption-level": "active",
        "relevance-score": 0.9,
        "mutable-content": 1
    },
    "buildID": "build-42",
    "result": "success"
}
```

## UNUserNotificationCenterDelegate Implementation

A production-ready delegate that handles all cases:

```swift
import UserNotifications
import Combine

final class NotificationCoordinator: NSObject, UNUserNotificationCenterDelegate {
    static let shared = NotificationCoordinator()

    // Combine publishers for reactive UI updates
    let messageReceived = PassthroughSubject<IncomingMessage, Never>()
    let notificationTapped = PassthroughSubject<[String: Any], Never>()

    private override init() {
        super.init()
    }

    // MARK: - Foreground Delivery

    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        willPresent notification: UNNotification,
        withCompletionHandler completionHandler: @escaping (UNNotificationPresentationOptions) -> Void
    ) {
        let content = notification.request.content
        let category = content.categoryIdentifier

        // Log for analytics
        Logger.notifications.info("Will present notification: \(notification.request.identifier) category: \(category)")

        switch category {
        case "CHAT_MESSAGE":
            // We handle chat updates in-app — suppress the banner
            messageReceived.trySend(IncomingMessage(dictionary: content.userInfo))
            completionHandler([])

        case "TASK_REMINDER":
            // Show banner but no sound (likely already looking at the app)
            completionHandler([.banner, .badge])

        case "SYSTEM_ALERT":
            // Critical alert — always show
            if #available(macOS 12.0, *) {
                completionHandler([.banner, .sound, .badge, .list])
            } else {
                completionHandler([.banner, .sound, .badge])
            }

        default:
            // Default behavior — show everything
            completionHandler([.banner, .sound, .badge])
        }
    }

    // MARK: - User Interaction

    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse,
        withCompletionHandler completionHandler: @escaping () -> Void
    ) {
        defer { completionHandler() }

        let content = response.notification.request.content
        let userInfo = content.userInfo
        let action = response.actionIdentifier

        Logger.notifications.info("Action: \(action) for \(response.notification.request.identifier)")

        switch action {
        case UNNotificationDefaultActionIdentifier:
            // Main notification body tap — navigate to relevant content
            notificationTapped.send(userInfo)
            navigateToContent(userInfo: userInfo)

        case "SNOOZE_ACTION":
            // Reschedule the same notification for later
            reschedule(notification: response.notification.request, delay: 60 * 10)  // 10 min

        case "REPLY_ACTION":
            guard let textResponse = response as? UNTextInputNotificationResponse else { return }
            let replyText = textResponse.userText
            sendReply(text: replyText, context: userInfo)

        case "JOIN_ACTION":
            // Foreground action — app comes to foreground
            if let meetingID = userInfo["meetingID"] as? String {
                MeetingManager.shared.join(meetingID: meetingID)
            }

        case UNNotificationDismissActionIdentifier:
            // User swiped/dismissed — track for analytics
            Analytics.track("notification_dismissed", properties: [
                "category": content.categoryIdentifier,
                "identifier": response.notification.request.identifier
            ])

        default:
            Logger.notifications.warning("Unhandled action: \(action)")
        }
    }

    // MARK: - Helpers

    private func reschedule(notification: UNNotificationRequest, delay: TimeInterval) {
        let newContent = notification.content.mutableCopy() as! UNMutableNotificationContent
        newContent.body = "⏰ Reminder: \(newContent.body)"
        newContent.interruptionLevel = .timeSensitive

        let trigger = UNTimeIntervalNotificationTrigger(timeInterval: delay, repeats: false)
        let request = UNNotificationRequest(
            identifier: notification.identifier + "-snoozed",
            content: newContent,
            trigger: trigger
        )

        UNUserNotificationCenter.current().add(request) { error in
            if let error {
                Logger.notifications.error("Reschedule failed: \(error.localizedDescription)")
            }
        }
    }

    private func navigateToContent(userInfo: [AnyHashable: Any]) {
        if let screen = userInfo["screen"] as? String {
            DispatchQueue.main.async {
                NavigationCoordinator.shared.navigate(to: screen, parameters: userInfo)
            }
        }
    }

    private func sendReply(text: String, context: [AnyHashable: Any]) {
        guard let conversationID = context["conversationID"] as? String else { return }
        Task {
            await MessagingService.shared.send(message: text, to: conversationID)
        }
    }
}
```

## Focus Mode-Aware Notification Scheduling

Complete pattern for adapting notification behavior to Focus mode state:

```swift
import Intents
import UserNotifications

final class FocusAwareNotificationScheduler {

    struct Context {
        let isFocusEnabled: Bool
        let isCriticalAllowed: Bool
        let isTimeSensitiveAllowed: Bool
    }

    private let center = UNUserNotificationCenter.current()

    /// Check current Focus status and return a context for scheduling decisions.
    func currentFocusContext() async -> Context {
        let focusCenter = INFocusStatusCenter.default

        guard focusCenter.authorizationStatus == .authorized else {
            return Context(isFocusEnabled: false, isCriticalAllowed: true, isTimeSensitiveAllowed: true)
        }

        let isFocused = focusCenter.focusStatus.isFocused ?? false

        // When a Focus is active, time-sensitive notifications still deliver if the user
        // has allowed them in their Focus settings. We can't query the user's Focus
        // configuration from within the app — we trust the system's interruption level
        // filtering and set the level appropriately.
        return Context(
            isFocusEnabled: isFocused,
            isCriticalAllowed: true,  // Critical bypasses all Focus (with entitlement)
            isTimeSensitiveAllowed: true  // System handles per-Focus allowance
        )
    }

    /// Schedule a notification with Focus-aware interruption level.
    func schedule(content: UNMutableNotificationContent, trigger: UNNotificationTrigger?) async throws {
        let context = await currentFocusContext()

        // Adapt interruption level based on Focus state
        if context.isFocusEnabled {
            switch content.interruptionLevel {
            case .critical:
                // Critical alerts bypass Focus — no adjustment needed
                break
            case .timeSensitive:
                // Time-sensitive may or may not bypass depending on user's Focus settings.
                // Keep as-is — the system will filter appropriately.
                break
            case .active:
                // Downgrade to passive to respect the user's Focus
                // The notification will still appear in Notification Center
                content.interruptionLevel = .passive
            case .passive:
                break
            @unknown default:
                break
            }
        }

        let request = UNNotificationRequest(
            identifier: "focus-aware-\(UUID().uuidString)",
            content: content,
            trigger: trigger
        )

        try await center.add(request)
    }

    /// Schedule a time-sensitive notification only if it will actually reach the user.
    /// Falls back to a delayed notification if Focus mode would suppress it.
    func scheduleUrgent(content: UNMutableNotificationContent, trigger: UNNotificationTrigger?) async throws {
        let context = await currentFocusContext()

        if context.isFocusEnabled && content.interruptionLevel == .active {
            // The notification would be downgraded to passive.
            // Option 1: Keep as passive (user sees it later)
            // Option 2: Add a relevance score to boost it in the summary
            content.relevanceScore = 1.0

            // Option 3: Add a note in the body
            let originalBody = content.body
            content.body = "🔔 (Will notify when Focus ends) \(originalBody)"

            // Schedule a silent "reminder" that fires when Focus status might change
            // (Approach: schedule a passive notification for now)
        }

        try await schedule(content: content, trigger: trigger)
    }

    /// Request Focus status authorization. Call once during onboarding.
    func requestFocusAccess() async -> INFocusStatusAuthorizationStatus {
        await INFocusStatusCenter.default.requestAuthorization()
    }
}

// MARK: - Monitoring Focus Changes

extension FocusAwareNotificationScheduler {
    /// Observe Focus status changes to adapt notification scheduling in real time.
    func startMonitoring() -> NSObjectProtocol {
        return DistributedNotificationCenter.default.addObserver(
            forName: Notification.Name("com.apple.focusmode.changed"),
            object: nil,
            queue: .main
        ) { [weak self] _ in
            guard let self else { return }
            Task {
                let context = await self.currentFocusContext()
                if context.isFocusEnabled {
                    Logger.notifications.info("Focus mode active — subsequent notifications will be passive")
                } else {
                    Logger.notifications.info("Focus mode inactive — notifications will use requested interruption level")
                }
            }
        }
    }
}
```

### INFocusStatusCenter Usage Notes

```swift
import Intents

// One-time authorization request
let status = await INFocusStatusCenter.default.requestAuthorization()

// Subsequent queries (no prompt after initial authorization)
let isFocused = INFocusStatusCenter.default.focusStatus.isFocused

// Important: isFocused returns Bool?
// - true: a Focus mode is currently active
// - false: no Focus mode is active
// - nil: authorization status is not .authorized, or status is undetermined

// You cannot determine WHICH Focus mode is active (Work, Sleep, custom, etc.)
// You cannot query the user's Focus notification allowlist
// You can only check whether Focus is on or off
```

## Custom Notification Content Extension

Complete pattern for a Notification Content Extension with custom UI:

### Xcode Setup

1. File → New → Target → **Notification Content Extension**
2. Name it (e.g., "RichNotificationContent")
3. Xcode generates `NotificationViewController.swift` and `Info.plist`

### Info.plist (Extension Target)

```
NSExtension:
  NSExtensionAttributes:
    UNNotificationExtensionCategory: ["RICH_TASK_CATEGORY"]
    UNNotificationExtensionInitialContentSizeRatio: 0.75
    UNNotificationExtensionDefaultContentHidden: false
    UNNotificationExtensionUserInteractionEnabled: true
    UNNotificationExtensionDefaultContentSizeCategory: UICTContentSizeCategoryM
  NSExtensionPrincipalClass: $(PRODUCT_MODULE_NAME).NotificationViewController
  NSExtensionPointIdentifier: com.apple.usernotifications.content-extension
```

### NotificationViewController (UIKit — even on macOS)

```swift
// Note: Content extensions use UIKit/AppKit bridged APIs, not AppKit directly.
// The view controller lifecycle is standard: viewDidLoad → viewWillAppear → etc.

final class NotificationViewController: UIViewController, UNNotificationContentExtension {
    // UI Elements
    private let titleLabel = UILabel()
    private let bodyLabel = UILabel()
    private let progressView = UIProgressView(progressViewStyle: .bar)
    private let actionButton = UIButton(type: .system)
    private let thumbnailView = UIImageView()

    private var taskID: String?

    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
    }

    private func setupUI() {
        view.backgroundColor = .systemBackground

        titleLabel.font = .preferredFont(forTextStyle: .headline)
        titleLabel.numberOfLines = 2
        titleLabel.translatesAutoresizingMaskIntoConstraints = false

        bodyLabel.font = .preferredFont(forTextStyle: .body)
        bodyLabel.numberOfLines = 0
        bodyLabel.textColor = .secondaryLabel
        bodyLabel.translatesAutoresizingMaskIntoConstraints = false

        progressView.progressTintColor = .systemBlue
        progressView.translatesAutoresizingMaskIntoConstraints = false

        actionButton.setTitle("Mark Complete", for: .normal)
        actionButton.titleLabel?.font = .preferredFont(forTextStyle: .callout)
        actionButton.backgroundColor = .systemBlue
        actionButton.setTitleColor(.white, for: .normal)
        actionButton.layer.cornerRadius = 8
        actionButton.translatesAutoresizingMaskIntoConstraints = false
        actionButton.addTarget(self, action: #selector(handleActionTap), for: .touchUpInside)

        thumbnailView.contentMode = .scaleAspectFit
        thumbnailView.translatesAutoresizingMaskIntoConstraints = false

        let stack = UIStackView(arrangedSubviews: [titleLabel, bodyLabel, progressView, actionButton])
        stack.axis = .vertical
        stack.spacing = 12
        stack.translatesAutoresizingMaskIntoConstraints = false

        view.addSubview(stack)
        NSLayoutConstraint.activate([
            stack.topAnchor.constraint(equalTo: view.topAnchor, constant: 16),
            stack.leadingAnchor.constraint(equalTo: view.leadingAnchor, constant: 16),
            stack.trailingAnchor.constraint(equalTo: view.trailingAnchor, constant: -16),
            stack.bottomAnchor.constraint(equalTo: view.bottomAnchor, constant: -16),
            actionButton.heightAnchor.constraint(equalToConstant: 44),
            progressView.heightAnchor.constraint(equalToConstant: 4)
        ])
    }

    // Called when the notification is delivered
    func didReceive(_ notification: UNNotification) {
        let content = notification.request.content

        titleLabel.text = content.title
        bodyLabel.text = content.body

        // Load custom data
        let userInfo = content.userInfo
        taskID = userInfo["taskID"] as? String

        if let progressValue = userInfo["progress"] as? Float {
            progressView.progress = progressValue
        } else {
            progressView.isHidden = true
        }

        // Load attachment image if present
        if let attachment = content.attachments.first {
            if attachment.url.startAccessingSecurityScopedResource() {
                defer { attachment.url.stopAccessingSecurityScopedResource() }
                thumbnailView.image = UIImage(contentsOfFile: attachment.url.path)
            }
        }
    }

    // Called when the user interacts with an action in the custom UI
    func didReceive(
        _ response: UNNotificationResponse,
        completionHandler completion: @escaping (UNNotificationContentExtensionResponseOption) -> Void
    ) {
        if response.actionIdentifier == "COMPLETE_ACTION" {
            // Animate the button
            actionButton.setTitle("✓ Done", for: .normal)
            actionButton.backgroundColor = .systemGreen

            // Forward the action to the main app for processing
            completion(.dismissAndForwardAction)
        } else if response.actionIdentifier == "SNOOZE_ACTION" {
            // Handle within the extension — no need to forward
            completion(.doNotDismiss)
        } else {
            completion(.dismissAndForwardAction)
        }
    }

    @objc private func handleActionTap() {
        // Trigger a notification update
        // This can be used to reflect user interaction in the UI
    }
}
```

### Important Content Extension Constraints

1. **Memory limit:** ~24MB on macOS. Loading large images or complex view hierarchies will cause the extension to be killed.
2. **No networking directly:** While `URLSession` technically works, avoid heavy network calls. Better: have the main app download the image and include it as an attachment.
3. **Size management:** Use `preferredContentSize` to set the vertical size. The system uses the content size ratio from Info.plist as the initial value, and you can update it:

```swift
override func viewDidLayoutSubviews() {
    super.viewDidLayoutSubviews()
    preferredContentSize = CGSize(width: view.bounds.width, height: systemLayoutSizeFitting(UIView.layoutFittingCompressedSize).height)
}
```

4. **Deeplinking from the content extension:** You cannot directly open your main app. Use `NSExtensionContext` to request the containing app to open a URL:

```swift
extensionContext?.open(URL(string: "myapp://task/\(taskID ?? "")")!)
```

However, the more reliable approach is to use `completion(.dismissAndForwardAction)` and handle the task in the main app's `didReceive` delegate method.
