---
name: core-data-macos
description: "Core Data on macOS — NSPersistentContainer, NSManagedObject, fetch requests, NSFetchedResultsController, batch operations, history tracking, staged migration, NSPersistentCloudKitContainer, multi-process via NSPersistentHistoryTracking. Use when persisting data with Core Data in macOS apps."
sources: https://developer.apple.com/documentation/coredata, https://developer.apple.com/documentation/coredata/migrating_to_icloud_kit
researched: 2026-06-06
platform: macOS 26+
---

# Core Data on macOS

Core Data persistence stack for macOS 26+ with Swift 6.3. Covers `NSPersistentContainer` stack setup, `NSManagedObject` modeling, fetching with `NSFetchedResultsController` for table views, batch operations for bulk changes, staged model migrations, CloudKit sync via `NSPersistentCloudKitContainer`, multi-process history tracking, and memory management for large object graphs.

See [references/coredata-patterns.md](references/coredata-patterns.md) for complete code patterns: container setup with lightweight migration, `NSFetchedResultsController` table data source, batch update/delete with refresh, staged migration with custom mapping, and CloudKit container with conflict resolution.

## Contents

- [Stack Setup](#stack-setup--nspersistentcontainer-persistentstoredescription-lightweight-migration)
- [Data Model](#data-model--nsobject-subclassing-relationships-delete-rules)
- [Fetching](#fetching--nsfetchrequest-nspredicate-nssortdescriptor)
- [Batch Operations](#batch-operations--nsbatchupdaterequest-nsbatchdeleterequest)
- [Staged Migration](#staged-migration--nsmigrationstage-custom-nentitymapping)
- [NSPersistentCloudKitContainer](#nspersistentcloudkitcontainer)
- [Multi-Process Access](#multi-process-access--nspersistenthistorytracking)
- [Memory Management](#memory-management--faulting-refreshallobjects)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Stack Setup — NSPersistentContainer, NSPersistentStoreDescription, Lightweight Migration

`NSPersistentContainer` (available macOS 10.12+) wraps the entire Core Data stack — `NSManagedObjectModel`, `NSPersistentStoreCoordinator`, and managed object contexts — into a single initializer. Use it as the root of all Core Data access.

```swift
final class PersistenceController {
    static let shared = PersistenceController()

    let container: NSPersistentContainer

    init(inMemory: Bool = false) {
        container = NSPersistentContainer(name: "Model")

        if inMemory {
            container.persistentStoreDescriptions.first?.url = URL(fileURLWithPath: "/dev/null")
        }

        container.loadPersistentStores { description, error in
            if let error = error as NSError? {
                fatalError("Core Data load failed: \(error), \(error.userInfo)")
            }
        }

        container.viewContext.automaticallyMergesChangesFromParent = true
        container.viewContext.mergePolicy = NSMergePolicy.mergeByPropertyObjectTrump
    }
}
```
```
**Best practices for macOS (2025 research):**
- Avoid global `PersistenceController.shared` singleton. Inject the container via SwiftUI environment for testability.
- Load stores asynchronously on a background queue to avoid blocking the UI during large first-time loads or migrations.
- Don't `fatalError` on all store load failures. Classify failures (corruption vs migration) and provide recovery paths (user-facing options, store rebuild).
- Extract a repository layer between Core Data and views/view models to make storage swappable for testing.
- `performAndWait` vs `perform`: Use `performAndWait` when you need the result on the calling thread; use `perform` for fire-and-forget background work.

**Lightweight migration** is enabled by default on the default `NSPersistentStoreDescription` (which uses `NSSQLiteStoreType` and sets `shouldMigrateStoreAutomatically` and `shouldInferMappingModelAutomatically` to `true`). For explicit control:

```swift
let description = container.persistentStoreDescriptions.first
description?.shouldMigrateStoreAutomatically = true
description?.shouldInferMappingModelAutomatically = true
```

**In-memory store** for testing: set the store URL to `/dev/null` on the `NSPersistentStoreDescription`. The view context stays fully functional; data is discarded when the container deallocates.

**Background work:**
```swift
container.performBackgroundTask { context in
    // Do work, then save
    try? context.save()
}
```

## Data Model — NSManagedObject Subclassing, Relationships, Delete Rules

Use `NSManagedObject` subclasses for type-safe access. In Xcode's model editor, set Codegen to "Manual/None" and generate Swift files, then add your custom logic.

**Relationships and delete rules:**

| Delete Rule | Behavior |
|---|---|
| `.nullify` | Removes the reference to the deleted object. Default. |
| `.cascade` | Deletes all related objects. Dangerous — can cascade through the entire graph. |
| `.deny` | Prevents deletion if any related objects exist. |
| `.noAction` | Does nothing to the relationship. You must maintain integrity manually. |

**Fetched properties** define fetch requests stored in the model. They're evaluated lazily and re-fetched each time accessed. Useful for "all items matching criteria" without maintaining a separate property.

**Validation:** Override `validateForInsert()`, `validateForUpdate()`, and `validateForDelete()` on your `NSManagedObject` subclass. Core Data calls these automatically before save. Throw `NSError` for validation failures.

**`@NSManaged` properties** must not provide initial values — Core Data provides them. Declared as `var` with a type annotation matching the model.

## Fetching — NSFetchRequest, NSPredicate, NSSortDescriptor

Every fetch requires at minimum an entity and sort descriptors (for deterministic ordering). Set `fetchBatchSize` on large result sets to avoid materializing all objects at once.

```swift
let request: NSFetchRequest<Item> = Item.fetchRequest()
request.predicate = NSPredicate(format: "isCompleted == NO AND dueDate > %@", Date() as NSDate)
request.sortDescriptors = [
    NSSortDescriptor(keyPath: \Item.priority, ascending: false),
    NSSortDescriptor(keyPath: \Item.dueDate, ascending: true)
]
request.fetchBatchSize = 50
request.relationshipKeyPathsForPrefetching = ["tags"] // eager-fetch relationships
```

**Batch size**: Core Data fetches in batches of `fetchBatchSize`. Objects that scroll out of scope are turned back into faults. For table views with hundreds of rows, a batch size of 50–100 is typical.

**Key-path sorting** (`NSSortDescriptor(keyPath:)`) is type-safe and preferred over string-based keys. Use `#keyPath(Entity.attribute)` only when key-path expressions aren't viable.

## Batch Operations — NSBatchUpdateRequest, NSBatchDeleteRequest

Batch operations bypass the managed object context and execute directly against the persistent store. Orders of magnitude faster than fetching-and-suid.

```swift
// Batch update: set all expired items to completed
let batchUpdate = NSBatchUpdateRequest(entityName: "Item")
batchUpdate.propertiesToUpdate = ["isCompleted": true]
batchUpdate.predicate = NSPredicate(format: "dueDate < %@", Date() as NSDate)
batchUpdate.resultType = .updatedObjectIDsResult

let result = try context.execute(batchUpdate) as? NSBatchUpdateResult
let ids = result?.result as? [NSManagedObjectID] ?? []

// Merge into the view context
NSManagedObjectContext.mergeChanges(
    fromContextSave: [NSDeletedObjectsKey: ids],
    into: [viewContext]
)
```

```swift
// Batch delete
let fetch = NSFetchRequest<NSFetchRequestResult>(entityName: "Item")
fetch.predicate = NSPredicate(format: "isArchived == YES")

let batchDelete = NSBatchDeleteRequest(fetchRequest: fetch)
batchDelete.resultType = .resultTypeObjectIDs

let result = try context.execute(batchDelete) as? NSBatchDeleteResult
let ids = result?.result as? [NSManagedObjectID] ?? []

// Merge deletions into the view context
NSManagedObjectContext.mergeChanges(
    fromContextSave: [NSDeletedObjectsKey: ids],
    into: [viewContext]
)
// Also refresh to evict stale in-memory state
viewContext.refreshAllObjects()
```

- Batch operations block the context during execution (like a fetch) and bypass the context entirely — this is why in-memory state becomes stale.
- For batch updates, merge using `NSUpdatedObjectsKey`. For batch deletes, merge with `NSDeletedObjectsKey` and call `refreshAllObjects()` to evict stale objects.
- `refreshAllObjects()` re-faults all registered objects, discarding unsaved changes. Use targeted `refresh(_:mergeChanges:)` to preserve unsaved edits on other objects.

## NSFetchedResultsController (macOS Optimization)

`NSFetchedResultsController` monitors its `NSManagedObjectContext` for changes and drives animated table view updates. It is purpose-built for binding Core Data to `NSTableView` on macOS.

Conform to `NSTableViewDataSource`, `NSTableViewDelegate`, and `NSFetchedResultsControllerDelegate`. The data source returns counts from the controller's `sections`; the delegate wraps changes in `tableView.beginUpdates()`/`endUpdates()`.

Changes arrive as `.insert`, `.delete`, `.update`, and `.move`. For `.update`, reload the affected row's visible columns (there are no sections in most macOS tables; use `sectionNameKeyPath: nil`). After a batch of changes, optionally call `tableView.reloadData()` to resolve rendering artifacts.

Set `cacheName: nil` unless you need the cache — and remember to call `deleteCache(withName:)` before changing the fetch request predicate or sort descriptors.
**NSFetchedResultsController with SwiftUI (2025 research):** For simple SwiftUI lists, `@FetchRequest` is sufficient. For complex, multi-section, or highly dynamic lists on macOS, wrap NSFetchedResultsController in an `@Observable` view model and expose data to SwiftUI. This gives you section support, cache management, and fine-grained change notifications that `@FetchRequest` lacks. `fetchBatchSize` primarily benefits table/collection views, not SwiftUI `@FetchRequest`.
## Staged Migration — NSMigrationStage, Custom NSEntityMapping

When a model change exceeds lightweight migration (e.g., making a non-optional attribute required without a default, splitting/merging entities, complex transformations), use **staged migrations** (WWDC23, available macOS 10.14+).

Staged migrations break the upgrade into a sequence of `NSMigrationStage` objects handled by `NSStagedMigrationManager`:

```swift
// Stage 1: lightweight migration (add optional attribute)
let lightweightStage = NSLightweightMigrationStage(["v2-model-checksum"])
lightweightStage.label = "V1→V2: Add location attribute"

// Stage 2: custom migration (transform data, split entity)
let customStage = NSCustomMigrationStage(
    migratingFrom: v2ModelReference,
    to: v3ModelReference
)
customStage.label = "V2→V3: Denormalize location into separate entity"

let manager = NSStagedMigrationManager([lightweightStage, customStage])
description.setOption(manager, forKey: NSPersistentStoreStagedMigrationManagerOptionKey)
```

**Custom entity mappings** via `NSEntityMigrationPolicy`:

```swift
final class ItemMigrationPolicy: NSEntityMigrationPolicy {
    override func createDestinationInstances(
        forSource sInstance: NSManagedObject,
        in mapping: NSEntityMapping,
        manager: NSMigrationManager
    ) throws {
        try super.createDestinationInstances(forSource: sInstance, in: mapping, manager: manager)

        // Access the destination object
        guard let dest = manager.destinationContext
            .registeredObjects
            .first(where: { $0.entity.name == mapping.destinationEntityName })
        else { return }

        // Custom transformation logic
        if let legacyTag = sInstance.value(forKey: "tag") as? String {
            dest.setValue(legacyTag.uppercased(), forKey: "normalizedTag")
        }
    }
}
```

Assign the policy class name in the mapping model: set `entityMigrationPolicyClassName` to `YourModule.ItemMigrationPolicy`.

**Testing migrations:** Fabricate a store at the old model version, then attempt to load it with the new staged migration manager. Use `NSMappingModel.inferredMappingModel(forSourceModel:destinationModel:)` to verify lightweight steps before committing to staged migration.
```
**Deferred lightweight migration (macOS compatible back to Big Sur):**
Set `NSPersistentStoreDeferredLightweightMigrationOptionKey` to `true` on the store description. This defers cleanup tasks (index updates, column drops) while keeping the latest schema active. Call `NSPersistentStore.finishDeferredLightweightMigration()` later to complete deferred work when resources are available.
**NSCustomMigrationStage handlers:**
`NSCustomMigrationStage` supports `willMigrateHandler` and `didMigrateHandler` closures. Use `willMigrate` to prepare data (e.g., fill nil values when an optional attribute becomes non-optional). Use `didMigrate` for cleanup. Access the migrating store through the migration manager's container within handlers.
**Multi-version staged migrations:**
You can chain more than two versions (V1 → V2 → V3 → V4) in a single `NSStagedMigrationManager`. Each stage is applied sequentially. All lightweight versions must be covered by at least one `NSLightweightMigrationStage`. This is the programmatic equivalent of Xcode's model editor migration but keeps version control diffs clean.

## NSPersistentCloudKitContainer

`NSPersistentCloudKitContainer` extends `NSPersistentContainer` with automatic CloudKit sync. Initialize with the same name — the container handles schema initialization and change tracking.

```swift
let container = NSPersistentCloudKitContainer(name: "Model")
container.loadPersistentStores { _, error in
    if let error { /* handle */ }
}
container.viewContext.automaticallyMergesChangesFromParent = true
container.viewContext.mergePolicy = NSMergePolicy.mergeByPropertyObjectTrump
```

**Conflict resolution:** `mergeByPropertyObjectTrump` resolves conflicts per-property, favoring the in-memory version over the store version. This is the recommended policy for CloudKit sync — local edits win over remote changes for individual properties, and remote-only changes are still adopted. For last-writer-wins semantics, use `overwrite`. For store-authoritative, use `mergeByPropertyStoreTrump`.

**Toggling CloudKit:** To disable CloudKit sync without code structure changes:
```swift
container.persistentStoreDescriptions.forEach {
    $0.cloudKitContainerOptions = nil
}
```
Keep `NSPersistentHistoryTrackingKey` enabled when disabling CloudKit — the container can reconcile accumulated history if CloudKit is re-enabled later.

**Subscriptions:** CloudKit subscriptions are managed automatically by the container. Call `CKContainer.default().accountStatus` before relying on sync. Subscription delivery requires the app to be running or woken via `CKSubscription.NotificationInfo`.
```
**macOS Tahoe 26 changes:**
- All NSPersistentStoreUbiquitous option keys are removed in the macOS 26 SDK. Build errors occur if present on macOS 26; warnings on earlier SDKs.
- Multi-store configurations use NSPersistentCloudKitContainerOptions per NSPersistentStoreDescription, each targeting a different CloudKit container.
- Event monitoring: Use NSPersistentCloudKitContainer.Event and .eventChangedNotification for progress indication.
- Permission checks: canUpdateRecord, canDeleteRecord, canModifyManagedObjects exposed by the container.

## Multi-Process Access — NSPersistentHistoryTracking

When multiple processes (app + XPC service + widget) share the same store, enable **persistent history tracking**:

```swift
let description = container.persistentStoreDescriptions.first
description?.setOption(true, forKey: NSPersistentHistoryTrackingKey)
description?.setOption(true, forKey: NSPersistentStoreRemoteChangeNotificationPostOptionKey)
```

Record a **transaction author** per context to identify change sources:
```swift
context.transactionAuthor = "com.example.myapp.main"
```

Consume history in other processes:

```swift
// Fetch history after the last known token
let request = NSPersistentHistoryChangeRequest.fetchHistory(after: lastToken)
let result = try backgroundContext.execute(request) as? NSPersistentHistoryResult
let transactions = result?.result as? [NSPersistentHistoryTransaction] ?? []

for transaction in transactions {
    let changes = transaction.changes ?? []
    let objectIDs = changes.map { $0.changedObjectID }

    // Merge into the local view context
    viewContext.perform {
        viewContext.mergeChanges(
            fromContextSave: [NSUpdatedObjectsKey: objectIDs],
            into: [viewContext]
        )
    }
}

// Advance the token
if let newToken = transactions.last?.token {
    lastToken = newToken
    // Persist the token for next launch
}
```

**XPC-friendly pattern:** Run all store access inside an `NSFileCoordinator` block. Share the store URL via XPC; each process creates its own `NSPersistentContainer` pointing at the same file. History tracking ensures eventual consistency.

## Memory Management — Faulting, refreshAllObjects(), Batch Faulting

Core Data's **faulting** mechanism keeps placeholder objects for unfired properties, reducing memory consumption. Accessing any property of a fault fires it (loads the full object from the store).

### When to intervene

| Scenario | Action |
|---|---|
| After batch delete/update | `refreshAllObjects()` to evict stale in-memory state |
| Scrolled-out table rows | Set `fetchBatchSize` on fetch request — Core Data refaults automatically |
| Long-running context growing | `reset()` to clear everything, or `refresh(_:mergeChanges:)` on specific objects |
| Large relationship traversal | Use `setReturnsObjectsAsFaults(true)` on fetch request, fire individual faults on demand |
| Memory warning | Call `refreshAllObjects()` on the view context after saving/refreshing |

### Batch faulting
```swift
// Fetch object IDs only, then batch-fire specific objects
let request = NSFetchRequest<NSManagedObjectID>(entityName: "Item")
request.resultType = .managedObjectIDResultType
request.predicate = NSPredicate(format: "project == %@", project)
let ids = try context.fetch(request)

// Fire in batches of 50
for batch in ids.chunked(into: 50) {
    for objectID in batch {
        let object = try context.existingObject(with: objectID)
        _ = object.value(forKey: "name") // fire the fault
    }
}
```

**`refreshAllObjects()`** re-faults every registered object, discarding unsaved changes. Use it after merging batch operation results. It does not reset the context — unsaved inserts remain registered.

## Constraints & Gotchas

- **CloudKit merge policies:** `mergeByPropertyObjectTrump` is the only policy that properly handles CloudKit-conflict resolution at the per-property level. The default `error` policy causes save failures when concurrent processes modify different properties of the same object.
- **Batch operations and relationships:** Batch deletes execute at the SQL level and bypass relationship delete rules. If an entity has a `.deny` delete rule, the batch delete may fail with a constraint violation. Remove deny rules or handle referential integrity before batch operations.
- **History tracking cost:** Persistent history tracking adds a write transaction for every store write. For write-heavy workloads (>1000 transactions/sec), this creates measurable overhead. Profile with Instruments (Core Data template).
- **NSFetchedResultsController cache:** If you use a cache, you must call `deleteCache(withName:)` before changing `fetchRequest`, `predicate`, or `sortDescriptors`. Otherwise, the controller silently returns stale results.
- **Multiple `NSPersistentStoreDescription` objects:** A single container can manage multiple stores (e.g., one in-memory, one SQLite). Assign objects to stores via `assign(_:to:)` on the context.
- **Deny rules and batch operations:** Deny delete rules are enforced at the context level, not the store level — batch operations bypass them but may cause SQLite constraint failures if the store has inferred constraints.

## Common Mistakes

- Forgetting to set `automaticallyMergesChangesFromParent = true` after background saves, leading to stale UI.
- Calling `refreshAllObjects()` before merging batch results — the merge picks up nil objects.
- Using `NSBatchDeleteRequest` without merging objectIDs back into the view context — deleted objects remain visible.
- Not deleting the `NSFetchedResultsController` cache before changing sort descriptors.
- Relying on the default `mergePolicy` (`error`) with CloudKit — conflicts cause save failures.
- Saving background contexts on the main thread (deadlock risk with `viewContext.save()`).
- Not setting a `transactionAuthor` on shared-store contexts, making history tracking unreadable.
- Operating on `NSManagedObject` instances across thread boundaries. Use `objectID` for cross-context reference.

## Review Checklist

- [ ] `NSPersistentContainer` uses lightweight migration (enabled by default; verify `shouldMigrateStoreAutomatically` and `shouldInferMappingModelAutomatically`)
- [ ] View context has `automaticallyMergesChangesFromParent = true`
- [ ] View context has appropriate `mergePolicy` (especially `mergeByPropertyObjectTrump` for CloudKit)
- [ ] Batch operations merge results back into the affected context(s) via `NSManagedObjectContext.mergeChanges(fromContextSave:into:)`
- [ ] `NSFetchedResultsController` cache is deleted before predicate/sort changes
- [ ] Multi-process stacks enable both `NSPersistentHistoryTrackingKey` and `NSPersistentStoreRemoteChangeNotificationPostOptionKey`
- [ ] Staged migrations are tested against real stores at every prior model version
- [ ] `refreshAllObjects()` (or targeted `refresh`) called after batch operations complete
- [ ] No `NSManagedObject` instances passed across thread/queue boundaries — only `NSManagedObjectID`
- [ ] Background context saves happen on the context's own queue (use `perform` or `performAndWait`)

## References

- [references/coredata-patterns.md](references/coredata-patterns.md) — Complete code patterns for container setup, `NSFetchedResultsController`, batch operations, staged migration, and CloudKit sync.
- WWDC 2023 Session 10186: "What's new in Core Data" — https://developer.apple.com/videos/play/wwdc2023/10186/
- WWDC 2019 Session 202: "Using Core Data with CloudKit" — https://developer.apple.com/videos/play/wwdc2019/202/
- Core Data Batch Programming Guide: https://developer.apple.com/library/archive/featuredarticles/CoreData_Batch_Guide/
