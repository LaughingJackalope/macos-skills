---
name: cloudkit-macos
description: "CloudKit on macOS — CKContainer/CKDatabase/CKRecord/CKQuery/CKSubscription, CKSyncEngine, NSPersistentCloudKitContainer, push-based sync, CloudKit entitlement + sandbox, iCloud Drive document sync, multi-window conflict resolution. Use when syncing data via CloudKit on macOS."
sources: https://developer.apple.com/documentation/cloudkit, https://developer.apple.com/icloud/cloudkit/
researched: 2026-06-06
platform: macOS 26+
---

# CloudKit on macOS

CloudKit provides schema-managed cloud storage for structured records, assets, and subscriptions. On macOS it powers Core Data sync (NSPersistentCloudKitContainer), CKSyncEngine event-based sync, and iCloud Drive document coordination. Every CloudKit-backed app needs the iCloud capability enabled with CloudKit service selected in the application target's Signing & Capabilities editor.

## Contents

- CKContainer & CKDatabase
- CKRecord
- CKQuery
- CKSubscription & Push
- CKSyncEngine
- NSPersistentCloudKitContainer
- iCloud Drive Document Sync
- Multi-Window Editing
- CloudKit Entitlement & Sandbox
- Constraints & Gotchas
- Common Mistakes
- Review Checklist
- References

## CKContainer & CKDatabase

`CKContainer` is the entry point. Access it through `CKContainer.default()` for the app's own `iCloud.<team-prefix>.<bundle-id>` container, or `CKContainer(identifier:)` for a shared container. Each container exposes three databases:

- **Public database** (`publicCloudDatabase` or `CKContainer.default().publicCloudDatabase`) — readable by all users, writable by the app team. Quotas count against the container, not the individual user.
- **Private database** (`privateCloudDatabase`) — scoped to the current iCloud account. Data is encrypted in transit and at rest; Apple cannot read private database records.
- **Shared database** (`sharedCloudDatabase`, macOS 14+) — entries created when a user accepts a share invitation.

Account availability is checked via `accountStatus(completionHandler:)`. Always test for `.available`, `.noAccount`, `.restricted`, and `.couldNotDetermine`. The UI must degrade gracefully — do not block the user interface based on iCloud availability for features that can work offline.

`CKDatabase` operations (`save(_:)`, `delete(withRecordID:)`, `fetch(withRecordID:)`) are asynchronous and return `CKRecord` values or network errors. Use `CKModifyRecordsOperation` for batch operations — individual `save()` calls in a loop trigger excessive round-trips.

## CKRecord

`CKRecord` is a dictionary-like container keyed by `CKRecord.FieldKey` (a typealias for `String`). Supported field types:

- `String`, `NSNumber`, `Int`, `Double`, `Bool`, `Date`
- `CKRecord.Reference` — references are directional; "back-references" to parents require an explicit field (you can enable `CKReferenceAction.deleteSelf` so deleting a parent cascades)
- `CKAsset` — backed by an `NSURL` file on disk; CloudKit deletes the record when the asset is deallocated if the record is the only owner
- `CLLocation` and `CLLocationCoordinate2D`
- `Data` blobs (up to 1 MB per record)

**CKRecord.ID** uniquely identifies a record within its zone. It requires a `recordName` (auto-generated UUIDs when you pass an empty string) and a `zoneID` (`CKRecordZone.ID`). Custom zones (`CKRecordZone(zoneName:)`) let you perform atomic multi-record operations and push subscriptions targeted to specific record subsets.

System fields (e.g., `creationDate`, `modificationDate`, `creatorUserRecordID`) are set by the server. Never assign them yourself — the server will quietly overwrite or reject your value. Access them via `record.creationDate` and similar properties. Custom record types are declared in the CloudKit Dashboard schema editor; you must deploy the schema to Production before release.

## CKQuery

`CKQuery(recordType:predicate:)` is backed by `NSPredicate`. Any `NSPredicate` that can be translated to a CloudKit predicate works; compound predicates use `NSCompoundPredicate(andPredicateWithSubpredicates:)` or `orPredicateWithSubpredicates:`. Supported operators include equality, inequality, `CONTAINS`, `IN`, `ANY...IN`, and aggregate expressions (`@count`). Location-predicate queries also work (`distanceToLocation:near:` on `CLLocation`-keyed fields).

For performance, use `desiredKeys` to retrieve only the fields you need — `CKQueryOperation(recordType:...)` is preferred over bare `CKQuery` because it lets you set `desiredKeys`, `resultsLimit`, and a cursor for paging. Without `desiredKeys`, every fetch returns the full record, including any large asset fields.

## CKSubscription & Push

Push notifications drive CloudKit sync on macOS. Two subscription types:

- **CKQuerySubscription** — fires when records matching a predicate are added, modified, or deleted. Use `firesOnRecordCreation`, `firesOnRecordUpdate`, and `firesOnRecordDeletion` options to choose what triggers the push. `notificationInfo.title` and `alertBody` set the notification payload; silent pushes set `shouldContentAvailable = true`.
- **macOS silent push reliability (2025-2026):** Silent pushes via CKQuerySubscription with `shouldContentAvailable = true` have been reported unreliable on macOS (Sonoma through macOS 26.3). Pushes may arrive with long delays or not at all on macOS while working on iOS. Killing the push daemon (apsd) temporarily resolves the last push. This appears to be a macOS-specific bug. Always implement CKFetchDatabaseChangesOperation on app launch as a fallback to catch missed pushes. File feedback reports with sysdiagnose.
- **CKDatabaseSubscription** — fires for any changes in the containing zone (custom zones only).

Save subscriptions with `CKModifySubscriptionsOperation`. To retrieve existing subscriptions (and avoid duplicates), use `CKFetchSubscriptionsOperation` on app launch. Subscriptions are per-container, per-user; after a reinstall, all subscriptions must be recreated.

In your AppDelegate:

```swift
func application(_ application: NSApplication,
                 didReceiveRemoteNotification userInfo: [AnyHashable : Any],
                 fetchCompletionHandler completionHandler: @escaping (NSApplication.FetchResult) -> Void) {
    let notification = CKNotification(fromRemoteNotificationDictionary: userInfo)
    // CKQuerySubscription fires arrive here; dispatch to sync engine.
    completionHandler(.noData)
}
```

## CKSyncEngine

`CKSyncEngine` (macOS 14+) is a system type — no shared protocol or mock. It takes a `CKSyncEngineConfiguration` with a list of state zones and an operation group. The engine emits `.stateUpdate`, `.accountChange`, `.fetchedDatabaseChanges`, `.fetchedRecordZoneChanges`, `.sentRecordZoneChanges`, `.sentDatabaseChanges`, `.willFetchChanges`, `.willFetchRecordZoneChanges`, `.didFetchRecordZoneChanges`, `.didFetchChanges`, `.willSend`, and `.didSend` events.

You maintain `syncState` (serialized between launches via `NSKeyedArchiver`), appending event metadata after each batch completes. The engine never handles merges automatically — on `.fetchedRecordZoneChanges` you apply changes to local storage and on `.sentRecordZoneChanges` you handle server-side conflicts.

Push notifications from CKSubscription arrive on the system queue; call `CKSyncEngine.notifications(didReceive:)` to re-lay into the engine when the app is already running. When the app launches cold, the engine automatically fetches new pushes, but only if you've provided persisted sync state.
```
**CKSyncEngine 2025 research notes:**
- CKSyncEngine manages periodic syncs automatically (battery, network, iCloud sign-in). Force immediate syncs via `fetchChanges(_:)` to pull or `sendChanges(_:)` to push.
- State management is critical: every `stateUpdate` must be persisted to disk. The engine serializes its entire state into a checkpoint, enabling clean restarts after crashes.
- Do not run multiple CKSyncEngine instances for a single private database — they conflict and misroute sync events.
- When a sync item fails, retry ordering is constrained. Requeue items by rebuilding the queue (placing retried items at the front is not directly supported).
- CloudKit's long replay of deletions (tombstones) can be costly for busy feeds — hours-long restores for new devices. For new installs, consider fetching initial data manually before initializing CKSyncEngine.
- Account change events (signedIn, signedOut, switchAccounts) supersede CloudKit's account-change notifications. Reinitialize and clear old state when the user signs out or switches accounts.

## NSPersistentCloudKitContainer

`NSPersistentCloudKitContainer` wraps `NSPersistentContainer` and auto-generates the CloudKit schema from the Core Data model (attribute types must map to CloudKit types). After adding a container, CloudKit auto-sync works without additional code. Setup steps:

1. Add iCloud + CloudKit capability.
2. Create a `NSPersistentCloudKitContainer(name:)`.
3. Call `loadPersistentStores`, which creates the schema on first run.
4. Set `container.viewContext.automaticallyMergesChangesFromParent = true`.

Conflict resolution uses the merge policy on the view context:
- `NSMergeByPropertyObjectTrumpMergePolicy` (default) — in-memory values win over stores.
- `NSMergeByPropertyStoreTrumpMergePolicy` — store values win.
- `NSOverwriteMergePolicy` / `NSRollbackMergePolicy` — all-or-nothing.
- Custom `mergePolicy` using `NSEntityMigrationPolicy` for field-level merge.
```
**Merge policy selection guide:**
- `NSMergeByPropertyObjectTrumpMergePolicy` — Use when local user edits are authoritative. Per-property, in-memory wins.
- `NSMergeByPropertyStoreTrumpMergePolicy` — Use when CloudKit/server data is authoritative. Per-property, store wins.
- `NSOverwriteMergePolicy` — All-or-nothing: in-memory overwrites entire store state, even without conflicts. Use with caution.
- `NSRollbackMergePolicy` — All-or-nothing: store wins everywhere, discarding in-memory changes even without conflicts.
- `NSErrorMergePolicy` — Save fails on conflict. Safe for explicit conflict handling (prompt user, log, retry).
- Best practice: Model data as relationships rather than flat attributes to reduce collisions. Order related objects (e.g., by date) for deterministic merges.
**Swift 6 actor isolation and CloudKit sync (2025-2026):** Swift 6's actor isolation can conflict with NSPersistentCloudKitContainer. CloudKit callbacks arrive on background threads while Core Data and UI code may run on MainActor. A single `@MainActor` annotation on the Core Data save method that dispatches to the correct queue can resolve the issue. Ensure `perform`/`performAndWait` is used when accessing contexts from CloudKit callbacks.

Container-wide helpers: `requestReference(for:)` produces `CKRecordZone.Reference` for sharing; `acceptShare(movers:)` handles share acceptance.

## iCloud Drive Document Sync

For documents not stored in Core Data, use `NSFileCoordinator` + `NSFilePresenter`. CloudKit's coordination daemon (`bird`) manages conflict versions:

```swift
let file = CloudDocument(fileURL: url)
file.open { success in
    // file.contents is loaded; Bird resolves server wins or client wins.
}
```

Use `NSFileVersion` for explicit conflict management — `unresolvedConflictVersions(ofItemAt:)` returns arrays for user mediation. When logging document changes, observe `.documentStateChanged` on the `NSFilePresenter` object.
```
**iCloud Drive coordination best practices:**
- All file operations (read, write, move, delete) must go through NSFileCoordinator. Direct low-level file operations bypass presenter notifications and cause data corruption.
- NSDocument automatically conforms to NSFilePresenter and handles coordination. For non-NSDocument files, implement NSFilePresenter manually.
- Register presenters via `NSFileCoordinator.addFilePresenter(_:)` and balance with `removeFilePresenter(_:)`.
- Enable iCloud in Capabilities and add `NSUbiquitousContainers` to Info.plist with container IDs to store documents in the user's iCloud Drive folder.
- Use `NSFileCoordinator(filePresenter: self)` when the presenter initiates the operation to avoid deadlocks.

## Multi-Window Editing

When multiple windows edit the same record or Core Data object, use:

```swift
container.viewContext.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
```

With `automaticallyMergesChangesFromParent = true`, `NSPersistentCloudKitContainer` posts `NSPersistentStoreRemoteChange` notifications on each window's view context, allowing UI updates. For per-UI sync, implement controllerDidChangeContent on your `NSFetchedResultsController`.

## CloudKit Entitlement & Sandbox

Add the `com.apple.developer.icloud-services` entitlement with values `CloudKit` (for CK APIs) or `CloudDocuments` (for NSDocument + CloudKit). Xcode adds these automatically when enabling iCloud + CloudKit capability. The embedded provisioning profile must match — mismatched entitlements cause `CKErrorNotAuthenticated` and silent sandbox fallback.

The CloudKit Console provides a web dashboard at `icloud.developer.apple.com/dashboard`. Use it to:

- Create custom record types and indexes.
- Switch between **Development** and **Production** schemas (schema changes in Development push to Production via the "Deploy Schema" button, but record data stays production-side).
- Monitor daily request and storage quotas.
- Test push subscriptions via a manual "Send Push Notification" from the dashboard.

## Constraints & Gotchas

- **Rate limits**: CloudKit enforces per-container and per-device limits (~40 requests/second). Batch with `CKModifyRecordsOperation` and observe `CKError.requestRateLimited` — retry with the provided `CKError.retryAfterSeconds`.
- **Record size**: Hard cap of 1 MB per record; assets up to 250 MB each. Fields over 1 KB are automatically treated as `CKAsset`-backed on upload; never inline large blobs.
- **Query restrictions**: `BEGINSWITH` and `CONTAIN` are case/diacritic-sensitive. No `LIKE`, no regex; location queries must use single distance predicates with a single location field.
- **Zone custom zones**: Atomic multi-record operations require a custom zone; default zone records are independent (partial failure possible).
- **Subscription limits**: Maximum 200 subscriptions per container. Subscriptions are not shared across users — each user must create their own.
- **Schema changes**: Once a record type is deployed to Production, you cannot delete fields or change field types. Add new fields as optional; deprecate old ones in code.
- **Push reliability**: Silent pushes are not guaranteed delivery. Use `CKFetchDatabaseChangesOperation` on app launch to catch missed pushes.

## Common Mistakes

- Forgetting to call `CKContainer.default().accountStatus()` before any database operation — leads to `CKError.notAuthenticated` on devices without iCloud.
- Saving records in a loop instead of batching with `CKModifyRecordsOperation` — triggers rate limits and poor performance.
- Not handling `CKError.serverRecordChanged` — the server version wins; you must re-fetch and re-apply local changes.
- Using `CKQuery` without `desiredKeys` — fetches full records including large assets, wasting bandwidth.
- Not recreating subscriptions after app reinstall — subscriptions are stored per-install, not per-user.
- Setting system fields (`creationDate`, `modificationDate`) — server silently overwrites them.
- Not deploying schema to Production before App Store submission — the app crashes on launch in production.
- Using `CKRecord.Reference` without `CKReferenceAction.deleteSelf` — orphaned child records accumulate.

## Review Checklist

- [ ] iCloud + CloudKit capability enabled in target Signing & Capabilities.
- [ ] `com.apple.developer.icloud-services` entitlement present in provisioning profile.
- [ ] `CKContainer.default().accountStatus()` checked before first database operation.
- [ ] Batch operations use `CKModifyRecordsOperation` (not per-record `save()` loops).
- [ ] `CKQueryOperation` uses `desiredKeys` to limit fetched fields.
- [ ] Subscriptions recreated on app launch via `CKFetchSubscriptionsOperation`.
- [ ] `CKSyncEngine` state serialized between launches.
- [ ] `NSPersistentCloudKitContainer` merge policy set explicitly.
- [ ] Schema deployed to Production in CloudKit Console.
- [ ] Rate-limit errors handled with `CKError.retryAfterSeconds`.
- [ ] Silent push handling in `application(_:didReceiveRemoteNotification:fetchCompletionHandler:)`.
- [ ] `automaticallyMergesChangesFromParent = true` on view contexts.

## References

- [CloudKit Documentation](https://developer.apple.com/documentation/cloudkit)
- [iCloud + CloudKit Guide](https://developer.apple.com/icloud/cloudkit/)
- WWDC 2023 Session 10188: "Sync to iCloud with CKSyncEngine" — https://developer.apple.com/videos/play/wwdc2023/10188/
- WWDC 2019 Session 202: "Using Core Data with CloudKit" — https://developer.apple.com/videos/play/wwdc2019/202/
- Technical Q&A QA1917: Debugging issues with CloudKit subscriptions — https://developer.apple.com/library/archive/qa/qa1917/
- [CKSyncEngine Documentation](https://developer.apple.com/documentation/cloudkit/cksyncengine)
- [NSPersistentCloudKitContainer Documentation](https://developer.apple.com/documentation/coredata/nspersistentcloudkitcontainer)
- [CloudKit Console](https://icloud.developer.apple.com/dashboard)
