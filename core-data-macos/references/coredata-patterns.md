# Core Data Patterns for macOS

Deep reference for production Core Data stack patterns on macOS 26+. Covers complete container setup with lightweight migration, `NSFetchedResultsController` for `NSTableView`, batch update/delete with refresh, staged migration with custom entity mapping, and `NSPersistentCloudKitContainer` with conflict resolution.

## Complete NSPersistentContainer Setup with Lightweight Migration

```swift
import CoreData
import CloudKit

enum StoreMode {
    case cloudKit
    case local
    case memory
}

final class PersistenceController {
    static let shared = PersistenceController()

    let container: NSPersistentContainer
    private let inMemory: Bool

    init(mode: StoreMode = .local) {
        self.inMemory = (mode == .memory)

        switch mode {
        case .cloudKit:
            container = NSPersistentCloudKitContainer(name: "Model")
        case .local, .memory:
            container = NSPersistentContainer(name: "Model")
        }

        // Configure store description
        guard let description = container.persistentStoreDescriptions.first else {
            fatalError("No store description found")
        }

        if inMemory {
            description.url = URL(fileURLWithPath: "/dev/null")
        }

        // Lightweight migration defaults (explicitly set for clarity)
        description.shouldMigrateStoreAutomatically = true
        description.shouldInferMappingModelAutomatically = true

        // History tracking for multi-process (always enable, even without CloudKit)
        description.setOption(true, forKey: NSPersistentHistoryTrackingKey)
        description.setOption(true, forKey: NSPersistentStoreRemoteChangeNotificationPostOptionKey)

        // Load
        container.loadPersistentStores { desc, error in
            if let error = error as NSError? {
                // Production: present recovery options or silently rebuild
                fatalError("Core Data load failed: \(error), \(error.userInfo)")
            }
        }

        // View context configuration
        let viewContext = container.viewContext
        viewContext.automaticallyMergesChangesFromParent = true
        viewContext.mergePolicy = NSMergePolicy.mergeByPropertyObjectTrump
        viewContext.transactionAuthor = "com.example.myapp.main"
    }

    // Background context factory
    func newBackgroundContext() -> NSManagedObjectContext {
        let context = container.newBackgroundContext()
        context.mergePolicy = NSMergePolicy.mergeByPropertyObjectTrump
        context.transactionAuthor = "com.example.myapp.background"
        return context
    }

    // Background task helper
    func performBackground<T>(_ block: @escaping (NSManagedObjectContext) throws -> T) async throws -> T {
        try await container.performBackgroundTask(block)
    }

    func save() {
        let context = container.viewContext
        guard context.hasChanges else { return }
        try? context.save()
    }
}
```

**Key decisions:**
- `automaticallyMergesChangesFromParent = true` means saves to background contexts propagate to the view context without manual notification observation.
- `mergeByPropertyObjectTrump` is chosen over the default `error` policy to handle concurrent writes from CloudKit or XPC services.

## NSFetchedResultsController for Efficient NSTableView Data Source

`NSFetchedResultsController` is the canonical way to drive an `NSTableView`. It monitors the context, caches results, and notifies the delegate about inserts/deletes/moves.

```swift
import Cocoa
import CoreData

final class ItemListViewController: NSViewController {
    @IBOutlet var tableView: NSTableView!

    let context = PersistenceController.shared.container.viewContext

    lazy var fetchedResultsController: NSFetchedResultsController<Item> = {
        let request: NSFetchRequest<Item> = Item.fetchRequest()
        request.predicate = NSPredicate(format: "project == %@", project)
        request.sortDescriptors = [
            NSSortDescriptor(keyPath: \Item.sortIndex, ascending: true),
            NSSortDescriptor(keyPath: \Item.createdAt, ascending: false)
        ]
        request.fetchBatchSize = 50

        let controller = NSFetchedResultsController(
            fetchRequest: request,
            managedObjectContext: context,
            sectionNameKeyPath: nil,
            cacheName: nil
        )
        controller.delegate = self
        return controller!
    }()

    private let project: Project // injected

    override func viewDidLoad() {
        super.viewDidLoad()

        tableView.dataSource = self
        tableView.delegate = self

        do {
            try fetchedResultsController.performFetch()
            tableView.reloadData()
        } catch {
            print("Fetch failed: \(error)")
        }
    }

    func changePredicate(for priority: Int?) {
        if let priority {
            fetchedResultsController.fetchRequest.predicate = NSPredicate(
                format: "project == %@ AND priority == %d", project, priority
            )
        } else {
            fetchedResultsController.fetchRequest.predicate = NSPredicate(
                format: "project == %@", project
            )
        }

        // Cache must be deleted when changing fetch request parameters
        NSFetchedResultsController<Item>.deleteCache(withName: nil)

        try? fetchedResultsController.performFetch()
        tableView.reloadData()
    }
}

// MARK: - NSTableViewDataSource

extension ItemListViewController: NSTableViewDataSource {
    func numberOfRows(in tableView: NSTableView) -> Int {
        fetchedResultsController.sections?.first?.numberOfObjects ?? 0
    }
}

// MARK: - NSTableViewDelegate

extension ItemListViewController: NSTableViewDelegate {
    func tableView(
        _ tableView: NSTableView,
        viewFor tableColumn: NSTableColumn?,
        row: Int
    ) -> NSView? {
        guard let columnId = tableColumn?.identifier else { return nil }
        let item = fetchedResultsController.object(at: IndexPath(item: row, section: 0))

        let cellId = NSUserInterfaceItemIdentifier(columnId.rawValue)
        let cell = tableView.makeView(withIdentifier: cellId, owner: nil) as? NSTableCellView
        cell?.textField?.stringValue = item.name
        return cell
    }
}

// MARK: - NSFetchedResultsControllerDelegate

extension ItemListViewController: NSFetchedResultsControllerDelegate {
    func controllerWillChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        tableView.beginUpdates()
    }

    func controller(
        _ controller: NSFetchedResultsController<NSFetchRequestResult>,
        didChange anObject: Any,
        at indexPath: IndexPath?,
        for type: NSFetchedResultsChangeType,
        newIndexPath: IndexPath?
    ) {
        switch type {
        case .insert:
            if let newIndexPath {
                tableView.insertRows(at: IndexSet(integer: newIndexPath.item), withAnimation: .effectFade)
            }
        case .delete:
            if let indexPath {
                tableView.removeRows(at: IndexSet(integer: indexPath.item), withAnimation: .effectFade)
            }
        case .update:
            if let indexPath {
                tableView.reloadData(forRowIndexes: IndexSet(integer: indexPath.item),
                                     columnIndexes: IndexSet(0..<tableView.numberOfColumns))
            }
        case .move:
            if let indexPath, let newIndexPath {
                tableView.removeRows(at: IndexSet(integer: indexPath.item), withAnimation: .effectFade)
                tableView.insertRows(at: IndexSet(integer: newIndexPath.item), withAnimation: .effectFade)
            }
        @unknown default:
            break
        }
    }

    func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
        tableView.endUpdates()
    }
}
```

**Gotchas:**
- Always wrap changes in `beginUpdates()`/`endUpdates()`. Table view insertions and deletions must be balanced.
- `.update` receives the `indexPath` of the row to refresh — reload all columns for that row.
- If you delete the cache, call `performFetch()` before `reloadData()`.

## Batch Update/Delete with Refresh

Batch operations bypass the context. After execution, affected objects may remain as stale faults. Merge and refresh.

```swift
func markAllExpiredItemsAsCompleted() async throws {
    let context = PersistenceController.shared.newBackgroundContext()

    try await context.perform {
        let batchUpdate = NSBatchUpdateRequest(entityName: "Item")
        batchUpdate.propertiesToUpdate = [
            "isCompleted": true,
            "modifiedAt": Date()
        ]
        batchUpdate.predicate = NSPredicate(
            format: "dueDate < %@ AND isCompleted == NO",
            Date() as NSDate
        )
        batchUpdate.resultType = .updatedObjectIDsResult

        let result = try context.execute(batchUpdate) as! NSBatchUpdateResult
        let ids = result.result as? [NSManagedObjectID] ?? []

        // Merge into every context that might hold stale copies
        await MainActor.run {
            NSManagedObjectContext.mergeChanges(
                fromContextSave: [NSUpdatedObjectsKey: ids],
                into: [PersistenceController.shared.container.viewContext]
            )
            // Force re-fault to pick up store-level changes
            PersistenceController.shared.container.viewContext.refreshAllObjects()
        }
    }
}

func deleteAllArchived() async throws {
    let context = PersistenceController.shared.newBackgroundContext()

    try await context.perform {
        let fetch = NSFetchRequest<NSFetchRequestResult>(entityName: "Item")
        fetch.predicate = NSPredicate(format: "isArchived == YES")

        let batchDelete = NSBatchDeleteRequest(fetchRequest: fetch)
        batchDelete.resultType = .resultTypeObjectIDs

        let result = try context.execute(batchDelete) as! NSBatchDeleteResult
        let ids = result.result as? [NSManagedObjectID] ?? []

        await MainActor.run {
            NSManagedObjectContext.mergeChanges(
                fromContextSave: [NSDeletedObjectsKey: ids],
                into: [PersistenceController.shared.container.viewContext]
            )
            PersistenceController.shared.container.viewContext.refreshAllObjects()
        }
    }
}
```

**Why both merge and refresh?** `mergeChanges` updates the context's knowledge of the store state. `refreshAllObjects()` re-faults registered objects, ensuring that any object whose properties were changed at the SQL level picks up the new values. Without refresh, property access returns the old in-memory value.

## Staged Migration with Custom Mapping

When lightweight migration cannot infer the mapping, use `NSStagedMigrationManager`. This example migrates from V3 to V4 by splitting an embedded `location` attribute into a separate `Location` entity.

```swift
func configureStagedMigration(for description: NSPersistentStoreDescription) throws {
    // Load model references for the intermediate versions
    guard
        let v3Model = NSManagedObjectModel.mergedModel(from: nil, forStoreMetadata: ["NSStoreModelVersionHashes": v3Metadata]),
        let v4Model = NSManagedObjectModel.makeModel(version: "V4") // your loader
    else {
        fatalError("Cannot load model versions")
    }

    let v3Ref = NSManagedObjectModelReference(model: v3Model, versionIdentifiers: ["V3"])
    let v4Ref = NSManagedObjectModelReference(model: v4Model, versionIdentifiers: ["V4"])

    // Stage 1: custom migration (split location attribute into Location entity)
    let v3toV4 = NSCustomMigrationStage(migratingFrom: v3Ref, to: v4Ref)
    v3toV4.label = "V3→V4: Split location into Location entity"

    let manager = NSStagedMigrationManager([v3toV4])
    description.setOption(manager, forKey: NSPersistentStoreStagedMigrationManagerOptionKey)
}
```

**Custom entity migration policy:**

```swift
final class ItemToItemV4MigrationPolicy: NSEntityMigrationPolicy {
    override func createDestinationInstances(
        forSource sourceItem: NSManagedObject,
        in mapping: NSEntityMapping,
        manager: NSMigrationManager
    ) throws {
        try super.createDestinationInstances(forSource: sourceItem, in: mapping, manager: manager)

        // Find the destination Item that was just created
        guard let destItem = manager.destinationContext.registeredObjects
            .compactMap({ $0 as? NSManagedObject })
            .first(where: { $0.entity.name == "Item" && manager.sourceInstances(for: mapping, destinationInstances: [$0]).contains(sourceItem) })
        else { return }

        // Create the Location entity and wire up the relationship
        let locationEntity = NSEntityDescription.insertNewObject(
            forEntityName: "Location",
            into: manager.destinationContext
        )

        if let lat = sourceItem.value(forKey: "latitude") as? Double,
           let lon = sourceItem.value(forKey: "longitude") as? Double {
            locationEntity.setValue(lat, forKey: "latitude")
            locationEntity.setValue(lon, forKey: "longitude")
        }

        destItem.setValue(locationEntity, forKey: "location")
    }
}
```

**Testing the migration:**

```swift
func testV3toV4Migration() throws {
    // Copy a known V3 store into a temp directory
    let testStoreURL = tempDir.appendingPathComponent("test.sqlite")
    try FileManager.default.copyItem(at: v3StoreURL, to: testStoreURL)

    let container = NSPersistentContainer(name: "Model")
    let description = NSPersistentStoreDescription(url: testStoreURL)

    // Apply staged migration config from above
    try configureStagedMigration(for: description)

    container.persistentStoreDescriptions = [description]

    let expectation = expectation(description: "Load with migration")
    container.loadPersistentStores { _, error in
        XCTAssertNil(error, "Migration should succeed")
        expectation.fulfill()
    }
    wait(for: [expectation], timeout: 30)

    // Verify the split occurred
    let request: NSFetchRequest<NSFetchRequestResult> = NSFetchRequest(entityName: "Location")
    let count = try container.viewContext.count(for: request)
    XCTAssertGreaterThan(count, 0, "Locations should be created from migration")
}
```

## NSPersistentCloudKitContainer with Conflict Resolution

```swift
final class CloudKitController {
    let container: NSPersistentCloudKitContainer

    init() {
        container = NSPersistentCloudKitContainer(name: "Model")

        guard let description = container.persistentStoreDescriptions.first else {
            fatalError("No store description")
        }

        // Enable history tracking (required for CloudKit sync)
        description.setOption(true, forKey: NSPersistentHistoryTrackingKey)
        description.setOption(true, forKey: NSPersistentStoreRemoteChangeNotificationPostOptionKey)

        // Optional: customize CloudKit container ID
        let cloudKitOptions = NSPersistentCloudKitContainerOptions(
            containerIdentifier: "iCloud.com.example.MyApp"
        )
        description.cloudKitContainerOptions = cloudKitOptions

        container.loadPersistentStores { desc, error in
            if let error {
                fatalError("CloudKit load failed: \(error)")
            }
            print("Store loaded: \(desc.url?.absoluteString ?? "unknown")")
        }

        let viewContext = container.viewContext
        viewContext.automaticallyMergesChangesFromParent = true
        viewContext.mergePolicy = NSMergePolicy.mergeByPropertyObjectTrump  // key for CloudKit
        viewContext.transactionAuthor = "com.example.myapp.cloudkit"
    }

    // Toggle CloudKit off while preserving history for later re-enable
    func disableCloudKit() {
        // Must be called BEFORE loadPersistentStories on next launch
        container.persistentStoreDescriptions.forEach {
            $0.cloudKitContainerOptions = nil
        }
    }

    func enableCloudKit() {
        container.persistentStoreDescriptions.forEach {
            if $0.cloudKitContainerOptions == nil {
                $0.cloudKitContainerOptions = NSPersistentCloudKitContainerOptions(
                    containerIdentifier: "iCloud.com.example.MyApp"
                )
            }
        }
    }
}
```

**Merge policy comparison for CloudKit:**

| Policy | Behavior on conflict | Recommended for CloudKit |
|---|---|---|
| `error` (default) | Throws on save | No |
| `rollback` | Discards all in-memory changes | No — loses local edits |
| `overwrite` | In-memory wins, all properties | Yes — last-write-wins |
| `mergeByPropertyStoreTrump` | Store wins per-property | No — remote edits always win |
| `mergeByPropertyObjectTrump` | In-memory wins per-property | **Yes** — local edits win, remote-only props adopted |

**Remote change detection:**
```swift
// Observe remote history store changes
NotificationCenter.default.addObserver(
    forName: .NSPersistentStoreRemoteChange,
    object: container.persistentStoreCoordinator,
    queue: .main
) { [weak self] _ in
    self?.processRemoteHistory()
}

private func processRemoteHistory() {
    let request = NSPersistentHistoryChangeRequest.fetchHistory(after: lastHistoryToken)

    do {
        let result = try backgroundContext.execute(request) as? NSPersistentHistoryResult
        let transactions = result?.result as? [NSPersistentHistoryTransaction] ?? []

        // Filter to only transactions from other processes/authors
        let remoteTransactions = transactions.filter {
            $0.author != "com.example.myapp.cloudkit" // exclude our own
        }

        for transaction in remoteTransactions {
            viewContext.perform {
                viewContext.mergeChanges(
                    fromContextSave: [
                        NSUpdatedObjectsKey: (transaction.changes ?? [])
                            .filter { $0.changeType == .update }
                            .map { $0.changedObjectID }
                    ],
                    into: [self.viewContext]
                )
            }
        }

        if let newToken = transactions.last?.token {
            lastHistoryToken = newToken
        }
    } catch {
        print("History processing failed: \(error)")
    }
}
```
