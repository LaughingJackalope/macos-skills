# CloudKit Patterns — Deep Dive

## CKSyncEngine Complete Setup

`CKSyncEngine` is the modern, event-driven replacement for manual `CKFetchDatabaseChangesOperation` / `CKFetchRecordZoneChangesOperation` loops. It is a system type (no protocol, no mock) introduced in macOS 14 / iOS 17.

### Configuration

```swift
import CloudKit

final class SyncCoordinator {
    private let container: CKContainer
    private let database: CKDatabase
    private var engine: CKSyncEngine?
    private let zoneID = CKRecordZone.ID(zoneName: "MyZone", ownerName: CKCurrentUserDefaultName)

    init(container: CKContainer = .default()) {
        self.container = container
        self.database = container.privateCloudDatabase
    }

    func start() {
        let configuration = CKSyncEngine.Configuration(
            database: database,
            stateSerialization: loadState(),
            delegate: self
        )
        engine = CKSyncEngine(configuration)
    }

    private func loadState() -> CKSyncEngine.State.Serialization? {
        guard let data = UserDefaults.standard.data(forKey: "syncState"),
              let state = try? NSKeyedUnarchiver.unarchivedObject(ofClass: CKSyncEngine.State.Serialization.self, from: data) else {
            return nil
        }
        return state
    }

    private func saveState(_ state: CKSyncEngine.State) {
        guard let data = try? NSKeyedArchiver.archivedData(withRootObject: state.serialization, requiringSecureCoding: true) else { return }
        UserDefaults.standard.set(data, forKey: "syncState")
    }
}
```

### Handling Events

```swift
extension SyncCoordinator: CKSyncEngineDelegate {
    func handleEvent(_ event: CKSyncEngine.Event, syncEngine: CKSyncEngine) async -> CKSyncEngine.Event.Result? {
        switch event {
        case .stateUpdate(let stateUpdate):
            switch stateUpdate.stateUpdateEvent {
            case .initialization:
                // First launch — create zone and subscription if needed.
                await createZoneIfNeeded()
                await createSubscriptionIfNeeded()
            case .zoneNotFound:
                // Zone was deleted remotely — recreate.
                await createZoneIfNeeded()
            }
            saveState(syncEngine.state)
            return nil

        case .fetchedDatabaseChanges(let fetchedDatabaseChanges):
            for modification in fetchedDatabaseChanges.modifications {
                // New or updated zone — fetch its records.
                await fetchZoneChanges(zoneID: modification.zoneID)
            }
            for deletion in fetchedDatabaseChanges.deletions {
                // Zone deleted — remove local data.
                await deleteLocalData(zoneID: deletion.zoneID)
            }
            return nil

        case .fetchedRecordZoneChanges(let fetchedRecordZoneChanges):
            for result in fetchedRecordZoneChanges.modifications {
                switch result {
                case .success(let record):
                    await saveToLocalStore(record)
                case .failure(let error):
                    handleError(error)
                }
            }
            for result in fetchedRecordZoneChanges.deletions {
                switch result {
                case .success(let recordID):
                    await deleteFromLocalStore(recordID: recordID)
                case .failure(let error):
                    handleError(error)
                }
            }
            return nil

        case .sentRecordZoneChanges(let sentRecordZoneChanges):
            for result in sentRecordZoneChanges.savedRecords {
                // Server accepted — update local metadata.
            }
            for result in sentRecordZoneChanges.failedRecordSaves {
                switch result {
                case .failure(let error):
                    if let ckError = error as? CKError,
                       ckError.code == .serverRecordChanged {
                        // Server version won — re-fetch and merge.
                        await handleConflict(ckError)
                    }
                default: break
                }
            }
            return nil

        case .didFetchChanges, .didFetchRecordZoneChanges:
            saveState(syncEngine.state)
            return nil

        default:
            return nil
        }
    }

    func nextFetchChangesOperation(_ syncEngine: CKSyncEngine) -> CKFetchChangesOperation? {
        let configuration = CKFetchChangesOperation.Configuration(
            previousServerChangeToken: syncEngine.state.serverChangeToken
        )
        return CKFetchChangesOperation(configuration: configuration)
    }
}
```

### Push Notification Integration

When a push arrives while the app is running, feed it to the engine:

```swift
// In AppDelegate
func application(_ application: NSApplication,
                 didReceiveRemoteNotification userInfo: [AnyHashable: Any],
                 fetchCompletionHandler completionHandler: @escaping (NSApplication.FetchResult) -> Void) {
    let notification = CKNotification(fromRemoteNotificationDictionary: userInfo)
    Task {
        await syncCoordinator.engine?.handleNotification(notification)
        completionHandler(.newData)
    }
}
```

## NSPersistentCloudKitContainer with Conflict Handling

### Setup

```swift
import CoreData

final class PersistenceController {
    static let shared = PersistenceController()
    let container: NSPersistentCloudKitContainer

    init() {
        container = NSPersistentCloudKitContainer(name: "Model")
        container.loadPersistentStores { _, error in
            if let error = error {
                fatalError("Core Data load failed: \(error)")
            }
        }
        container.viewContext.automaticallyMergesChangesFromParent = true
        container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
    }
}
```

### Custom Merge Policy

For field-level conflict resolution (e.g., last-writer-wins per field):

```swift
final class FieldLevelMergePolicy: NSMergePolicy {
    init() {
        super.init(mergeType: .mergeByPropertyObjectTrumpMergePolicyType)
    }

    override func resolve(constraintConflicts list: [NSConstraintConflict]) throws {
        for conflict in list {
            guard let databaseObject = conflict.databaseObject else {
                try super.resolve(constraintConflicts: [conflict])
                continue
            }
            // Custom logic: compare modification dates, merge fields.
            for (key, conflictingValue) in conflict.conflictingValues {
                let databaseValue = databaseObject.value(forKey: key)
                // Prefer newer value based on a timestamp field.
                if let localDate = conflictingValue as? Date,
                   let serverDate = databaseValue as? Date,
                   localDate > serverDate {
                    databaseObject.setValue(conflictingValue, forKey: key)
                }
            }
        }
    }
}
```

### Observing Remote Changes

```swift
NotificationCenter.default.addObserver(
    forName: .NSPersistentStoreRemoteChange,
    object: container.persistentStoreCoordinator,
    queue: .main
) { notification in
    // Trigger UI refresh — NSFetchedResultsController will update automatically
    // if automaticallyMergesChangesFromParent is true.
}
```

## CKQuerySubscription with Push Notification Handling

### Creating a Subscription

```swift
func createSubscription() async throws {
    let subscription = CKQuerySubscription(
        recordType: "Task",
        predicate: NSPredicate(format: "isCompleted == %@", NSNumber(value: false)),
        subscriptionID: "active-tasks",
        options: [.firesOnRecordCreation, .firesOnRecordUpdate, .firesOnRecordDeletion]
    )

    let notificationInfo = CKSubscription.NotificationInfo()
    notificationInfo.shouldSendContentAvailable = true // Silent push
    notificationInfo.desiredKeys = ["title", "dueDate"] // Only these fields in push payload
    subscription.notificationInfo = notificationInfo

    let operation = CKModifySubscriptionsOperation(
        subscriptionsToSave: [subscription],
        subscriptionIDsToDelete: nil
    )
    operation.qualityOfService = .utility

    try await withCheckedThrowingContinuation { (continuation: CheckedContinuation<Void, Error>) in
        operation.modifySubscriptionsResultBlock = { result in
            switch result {
            case .success:
                continuation.resume()
            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
        database.add(operation)
    }
}
```

### Handling the Push

```swift
func application(_ application: NSApplication,
                 didReceiveRemoteNotification userInfo: [AnyHashable: Any],
                 fetchCompletionHandler completionHandler: @escaping (NSApplication.FetchResult) -> Void) {
    let notification = CKNotification(fromRemoteNotificationDictionary: userInfo)

    guard notification.notificationType == .query,
          let queryNotification = notification as? CKQueryNotification else {
        completionHandler(.noData)
        return
    }

    // Fetch the changed record.
    let recordID = queryNotification.recordID!
    database.fetch(withRecordID: recordID) { record, error in
        if let record = record {
            // Update local store.
            self.updateLocalStore(with: record)
        }
        completionHandler(error == nil ? .newData : .failed)
    }
}
```

### Avoiding Duplicate Subscriptions

```swift
func fetchExistingSubscriptions() async throws -> [CKSubscription] {
    try await withCheckedThrowingContinuation { continuation in
        database.fetchAllSubscriptions { subscriptions, error in
            if let error = error {
                continuation.resume(throwing: error)
            } else {
                continuation.resume(returning: subscriptions ?? [])
            }
        }
    }
}
```

## iCloud Drive Document Sync with NSFileCoordinator + CloudKit

### NSFilePresenter Implementation

```swift
final class DocumentPresenter: NSObject, NSFilePresenter {
    let presentedItemURL: URL?
    let presentedItemOperationQueue = OperationQueue.main

    init(url: URL) {
        self.presentedItemURL = url
        super.init()
        NSFileCoordinator.addFilePresenter(self)
    }

    deinit {
        NSFileCoordinator.removeFilePresenter(self)
    }

    func presentedItemDidChange() {
        // Bird (CloudKit daemon) has updated the file on disk.
        reloadDocument()
    }

    func presentedItemDidGain(_ version: NSFileVersion) {
        // New conflict version created.
        handleConflict(version)
    }

    func presentedItemDidLose(_ version: NSFileVersion) {
        // Conflict version resolved.
    }

    func presentedItemDidResolveConflictVersion(_ version: NSFileVersion) {
        // Conflict resolved by Bird or user.
    }

    private func reloadDocument() {
        guard let url = presentedItemURL else { return }
        let coordinator = NSFileCoordinator(filePresenter: self)
        var error: NSError?
        coordinator.coordinate(readingItemAt: url, options: .withoutChanges, error: &error) { url in
            // Read updated contents.
            do {
                let data = try Data(contentsOf: url)
                DispatchQueue.main.async {
                    self.updateUI(with: data)
                }
            } catch {
                print("Read error: \(error)")
            }
        }
    }

    private func handleConflict(_ version: NSFileVersion) {
        guard let url = presentedItemURL else { return }
        let conflicts = NSFileVersion.unresolvedConflictVersions(ofItemAt: url) ?? []
        if !conflicts.isEmpty {
            // Present conflict resolution UI.
            showConflictResolution(conflicts)
        }
    }
}
```

### Coordinated Writing

```swift
func writeDocument(data: Data, to url: URL) {
    let coordinator = NSFileCoordinator(filePresenter: self)
    var error: NSError?
    coordinator.coordinate(writingItemAt: url, options: .forReplacing, error: &error) { url in
        do {
            try data.write(to: url, options: .atomic)
        } catch {
            print("Write error: \(error)")
        }
    }
}
```

### NSDocument Integration

For `NSDocument`-based apps, override `read(from:ofType:)` and `write(to:ofType:)` with coordinated access. `NSDocument` automatically registers as a file presenter when the document is opened. Use `NSFileVersion` APIs for explicit conflict management:

```swift
override func read(from url: URL, ofType typeName: String) throws {
    let coordinator = NSFileCoordinator(filePresenter: self)
    var error: NSError?
    var readError: Error?
    coordinator.coordinate(readingItemAt: url, options: .withoutChanges, error: &error) { url in
        do {
            let data = try Data(contentsOf: url)
            // Parse into document model.
        } catch {
            readError = error
        }
    }
    if let error = readError ?? error {
        throw error
    }
}
```

### Key Points

- Always use `NSFileCoordinator` for reads and writes — direct file access bypasses CloudKit's coordination daemon.
- `NSFilePresenter` callbacks arrive on `presentedItemOperationQueue` (set to `.main` for UI updates).
- `NSFileVersion` provides access to conflict versions; resolve them by calling `removeOtherVersionsOfItemAt:` after merging.
- Bird (the CloudKit daemon) handles background sync; your app only needs to respond to presenter callbacks.
