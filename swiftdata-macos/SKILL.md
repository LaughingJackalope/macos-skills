---
name: swiftdata-macos
description: "SwiftData on macOS — @Model, @Query, ModelContainer/ModelContext, CloudKit sync, migrations, #Predicate, FetchDescriptor, SortDescriptor, performance with large datasets, shared ModelContainer across windows, document-scoped containers. Use when persisting data with SwiftData in macOS apps."
sources: https://developer.apple.com/documentation/swiftdata, https://developer.apple.com/videos/play/wwdc2023/10187/
researched: 2026-06-06
platform: macOS 26+
---

# SwiftData on macOS

SwiftData patterns for macOS 26+ with Swift 6.3. Covers `@Model` definition, `ModelContainer`/`ModelContext` configuration, `@Query` with `#Predicate`, CloudKit sync via `NSPersistentCloudKitContainer`, schema migrations, batch performance operations, shared containers across windows, document-scoped containers, and `NSPersistentHistoryTracking` for XPC.

For general SwiftUI patterns, see the `swiftui-macos-patterns` skill. For AppKit interop (e.g., using SwiftData models in `NSDocument`), see the `swiftui-appkit-interop` skill.

## Contents

- [Model Definition](#model-definition)
- [ModelContainer & ModelContext](#modelcontainer--modelcontext)
- [Querying Data](#querying-data)
- [CloudKit Sync](#cloudkit-sync)
- [Migrations](#migrations)
- [Performance](#performance)
- [Multi-Window macOS](#multi-window-macos)
- [Document-Scoped Containers](#document-scoped-containers)
- [NSPersistentHistoryTracking for XPC](#nspersistenthistorytracking-for-xpc)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Model Definition

Use the `@Macro` on a Swift class to declare a persistent model. SwiftData synthesizes storage, observation, and `Codable` support.

```swift
import SwiftData
import Foundation

@Model
final class Project {
    @Attribute(.unique) var identifier: String
    var name: String
    var createdAt: Date
    @Attribute(.externalStorage) var thumbnailData: Data?
    var tags: [String]

    @Relationship(deleteRule: .cascade, inverse: \TaskItem.project)
    var tasks: [TaskItem] = []

    init(identifier: String = UUID().uuidString, name: String) {
        self.identifier = identifier
        self.name = name
        self.createdAt = .now
        self.tags = []
    }
}

@Model
final class TaskItem {
    var title: String
    var isCompleted: Bool
    var project: Project?

    init(title: String) {
        self.title = title
        self.isCompleted = false
    }
}
```

**Key attributes:**

| Attribute | Purpose |
|---|---|
| `@Attribute(.unique)` | Enforces uniqueness constraint. Fails on `modelContext.save()` if duplicate. |
| `@Attribute(.externalStorage)` | Stores large `Data` or `String` values outside the SQLite store. Use for images, blobs. |
| `@Attribute(.transformable)` | Uses `NSSecureCoding` / `Codable` for custom types. The type must conform. |

**Indexes:** Use `@Index` for query performance on frequently-predicated properties. On macOS 26+, SwiftData supports composite indexes:
```swift
@Index<MyModel>(\.$firstName, \.$lastName)
```

**Relationships:** Three delete rules:
- `.cascade` — deleting the parent deletes all children. Use for composition (Project → Tasks).
- `.nullify` — deleting the parent sets the child's reference to `nil`. Use for optional associations.
- `.deny` — prevents deletion if children exist. Useful for referential integrity enforcement.

## ModelContainer & ModelContext

The `ModelContainer` owns the SQLite store and schema. The `ModelContext` is the unit-of-work — fetch, insert, delete, then `save()`.

```swift
import SwiftData

// Schema-driven container
let schema = Schema([Project.self, TaskItem.self])
let configuration = ModelConfiguration(
    schema: schema,
    isStoredInMemoryOnly: false,
    cloudKitDatabase: .automatic
)
let container = try ModelContainer(for: schema, configurations: [configuration])

// In SwiftUI, attach to the scene:
@main
struct MyApp: App {
    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(container)
    }
}
```

**In-memory container for testing and previews:**
```swift
let testConfig = ModelConfiguration(
    schema: Schema([Project.self]),
    isStoredInMemoryOnly: true)
let testContainer = try ModelContainer(for: Schema([Project.self]), configurations: [testConfig])

// In previews:
#Preview {
    ContentView()
        .modelContainer(for: Project.self, inMemory: true)
}
```

**Programmatic context access:**
```swift
let context = ModelContext(container)
let projects = try context.fetch(FetchDescriptor<Project>())
context.insert(newProject)
try context.save()
```

## Querying Data

### @Query

The primary SwiftUI integration point. Fetches and observes a collection automatically.

```swift
struct ProjectListView: View {
    @Query(
        filter: #Predicate<Project> { $0.name.contains("alpha") },
        sort: \.createdAt,
        order: .reverse
    )
    var projects: [Project]

    var body: some View {
        List(projects) { project in
            Text(project.name)
        }
    }
}
```

### #Predicate with macOS 26 improvements

On macOS 26+, `#Predicate` supports string operations, collection predicates, and arithmetic:

```swift
// Compound predicates
#Predicate<Project> { project in
    project.name.localizedStandardContains("urgent")
    && !project.tasks.isEmpty
    && project.tasks.allSatisfy { $0.isCompleted == false }
}

// Date predicates
#Predicate<TaskItem> { task in
    task.project?.createdAt > Calendar.current.date(byAdding: .month, value: -1, to: .now)!
}
```

> **Note:** `#Predicate` macros generate `NSPredicate` at compile time. Not all Swift expressions are supported. Use the macro directly — do not build `NSPredicate` format strings for SwiftData models.

### SortDescriptor

```swift
@Query(sort: [
    SortDescriptor(\Project.createdAt, order: .reverse),
    SortDescriptor(\Project.name)
])
var projects: [Project]
```

### Manual FetchDescriptor

For imperative code outside SwiftUI views:

```swift
func fetchIncompleteTasks() throws -> [TaskItem] {
    let descriptor = FetchDescriptor<TaskItem>(
        predicate: #Predicate { $0.isCompleted == false },
        sortBy: [SortDescriptor(\.title)],
        fetchLimit: 100
    )
    return try modelContext.fetch(descriptor)
}
```

### SectionedQuery

For sectioned lists (e.g., grouped by first letter or date):

```swift
struct GroupedProjectList: View {
    @Query<Project>(\.$createdAt) var projects: [Project]

    var body: some View {
        List {
            SectionedQuery<DateTimeInterval, Project>(
                \.$createdAt,
                sectionedBy: .init(component: .day, calendar: .current)
            ) { section in
                Section(section.id.formatted(date: .abbreviated, time: .omitted)) {
                    ForEach(section.items) { project in
                        ProjectRow(project: project)
                    }
                }
            }
        }
    }
}
```

## CloudKit Sync

SwiftData integrates with CloudKit via `NSPersistentCloudKitContainer` under the hood. Configure the container with a CloudKit-aware `ModelConfiguration`:

```swift
let configuration = ModelConfiguration(
    schema: Schema([Project.self, TaskItem.self]),
    cloudKitDatabase: .automatic  // .automatic or .private(identifier:.public(identifier:))
)

let container = try ModelContainer(
    for: Schema([Project.self, TaskItem.self]),
    configurations: [configuration]
)
```

**CloudKit database scopes:**

| Scope | Use |
|---|---|
| `.automatic` | Uses the default private database. Matches `NSPersistentCloudKitContainer` default. |
| .`private("com.example.private")` | Private database per user. |
| `.public("com.example.shared")` | Public database, shared across users. |

**Sync conflict resolution:** SwiftData defaults to "server wins" on merge conflicts. For custom conflict handling, access the underlying `NSPersistentCloudKitContainer` via `NSPersistentStoreDescription`:

```swift
// For custom conflict resolution, you need to access the NSPersistentStoreCoordinator
// This requires dropping below SwiftData's abstraction — see NSPersistentCloudKitContainer docs
```

- **macOS 26 (Tahoe) CloudKit sync fixes:** macOS 26.4.1 fixed CloudKit data syncing issues introduced in 26.4. If targeting 26.4+, test sync behavior carefully.
- **CloudKit.framework linking:** Unlike iOS, macOS projects do not always auto-link CloudKit.framework. If sync works in Xcode Debug but fails in TestFlight/Release, explicitly add CloudKit.framework to Build Phases > Link Binary With Libraries.
- **CloudKit sharing URL fixes:** macOS 26 resolved an issue where CloudKit sharing URLs failed to launch third-party apps.
- **Sync status monitoring:** SwiftData does not expose a direct API to observe sync progress. Use Core Data's `NSPersistentStoreRemoteChange` notifications to approximate sync state. No dedicated SwiftData sync progress API exists.
- **Opt-out of automatic sync:** To disable CloudKit sync, set `cloudKitDatabase: .none` on `ModelConfiguration`. Useful when using multiple CloudKit containers or existing production schemas.

## Migrations

### Lightweight Migrations

- Adding new optional properties
- Adding new models
- Adding relationships with nullify delete rule
- **Property renames** with `@Attribute(originalName: "oldName")` — SwiftData preserves the old column name in the store. Without `originalName`, SwiftData treats the old property as deleted and the new one as a fresh column, losing data.
For lightweight migrations, update your `Schema` version:

```swift
enum V1_1: VersionedSchema {
    static var models: [any PersistentModel.Type] { [Project.self, TaskItem.self] }
    static var versionIdentifier: String { "v1.1" }
}

static var migrateV1toV1_1: MigrationStage {
    .lightweight(fromVersion: V1, toVersion: V1_1)
}

static var schema: VersionedSchema {
    V1_1.self
}
```

### Custom Migration Stages

For non-additive changes (renaming, removing, transforming data), use `custom`:

```swift
enum V1: VersionedSchema {
    static var models: [any PersistentModel.Type] { [Project.self, OldTask.self] }
    static var versionIdentifier: String { "v1" }
}

enum V2: VersionedSchema {
    static var models: [any.PersistentModel.Type] { [Project.self, TaskItem.self] }
    static var versionIdentifier: String { "v2" }
}

static var migrateV1toV2: MigrationStage {
    .custom(
        fromVersion: V1_1.self,
        toVersion: V2.self,
        willMigrate: { context in
            // Read old models, transform data
            let oldTasks = try context.fetch(FetchDescriptor<OldTask>())
            for oldTask in oldTasks {
                let newTask = TaskItem(title: oldTask.title)
                context.insert(newTask)
                context.delete(oldTask)
            }
            try context.save()
        },
        didMigrate: nil
    )
}
```
```
**Migration types (macOS 26 / SwiftData 2025-2026):**
1. **Automatic lightweight** — Most additive changes (add/delete entities, attributes, relationships, rename with `originalName`). No version ladder needed.
2. **Declared lightweight** — Explicit `MigrationStage.lightweight` transitions in a `SchemaMigrationPlan`. SwiftData infers the mapping.
3. **Custom** — `MigrationStage.custom(willMigrate:didMigrate:)` for data transformations (split/derive/merge columns).
**SchemaMigrationPlan structure (WWDC25 session 291):**
```swift
enum AppMigrationPlan: SchemaMigrationPlan {
    static var schemas: [any VersionedSchema.Type] {
        [V1.self, V2.self]
    }
    static var stages: [MigrationStage] {
        [migrateV1toV2]
    }
}
// Attach to container:
let container = try ModelContainer(
    for: AppMigrationPlan.self,
    configurations: [ModelConfiguration(schema: AppMigrationPlan.schemas.last!)]
)
```
**When a version ladder is NOT needed:** If all changes are handled by automatic lightweight migration (additive only), you do not need to define `VersionedSchema` versions at all. The ladder is only necessary when using custom migration stages for data transformation.
**Downgrade migrations:** SwiftData supports bidirectional migrations — define separate `UpgradeMigrationPlan` and `DowngradeMigrationPlan` enums with inverse `MigrationStage.custom` stages to copy data back to old schema shapes. This enables safe rollback between versions.

### Migration Testing

Always test migrations with a real SQLite file from the previous version:
1. Build and run the old version, create data
2. Copy the SQLite file from `~/Library/Application Support/` or the container URL
3. Check in the file as a test fixture
4. Build the new version against the old SQLite file in tests

## Performance

### Batch Operations

For large datasets, avoid loading every object. Use batch methods on `ModelContext`:

```swift
// Batch delete
let descriptor = FetchDescriptor<TaskItem>(
    predicate: #Predicate { $0.isCompleted == true }
)
try modelContext.delete(model: TaskItem.self, where: descriptor.predicate)

// Batch insert (more efficient than individual insert + save)
let newItems = (0..<10_000).map { TaskItem(title: "Task \($0)") }
for item in newItems {
    modelContext.insert(item)
}
try modelContext.save()  // Single transaction for all inserts
```

### Lazy Fetching and Faulting

SwiftData faults objects automatically — relationship arrays are not loaded until accessed. Control fetch behavior with `FetchDescriptor`:

```swift
var descriptor = FetchDescriptor<TaskItem>()
descriptor.fetchLimit = 50        // Limit results
descriptor.fetchOffset = 100      // Pagination offset
descriptor.propertiesToFetch = [\.title, \.isCompleted]  // Partial faulting (macOS 26+)
let tasks = try modelContext.fetch(descriptor)
```

### 10k+ Row Queries

Strategy for large datasets:
1. Use `fetchLimit` and `fetchOffset` for pagination
2. Apply `#Predicate` to filter at the SQLite level — never load all rows and filter in Swift
3. Use `delete(model:where:)` for batch deletes
4. For counts, prefer `modelContext.count(FetchDescriptor<T>())` over fetching and counting
5. Consider `NSPersistentHistoryTracking` for incremental updates instead of full refetches

## Multi-Window macOS

A single shared `ModelContainer` is the standard pattern. All windows observe the same data because the container is attached at the scene level:

```swift
@main
struct MyApp: App {
    let container: ModelContainer

    init() {
        container = try! ModelContainer(for: [Project.self, TaskItem.self])
    }

    var body: some Scene {
        WindowGroup {
            ContentView()
        }
        .modelContainer(container)

        WindowGroup("Inspector", id: "inspector") {
            InspectorView()
        }
        .modelContainer(container)
    }
}
```

**Passing model objects between windows:**

Model objects in SwiftData conform to `PersistentModel` and carry a stable `PersistentIdentifier`. Pass the identifier between windows, then resolve on the other side:

```swift
// Window A: pass the ID
@Environment(\.openWindow) var openWindow

func openInspector(for project: Project) {
    openWindow(id: "inspector", value: project.persistentModelID)
}

// Window B: resolve from ID
struct InspectorView: View {
    @Environment(\.modelContext) var modelContext
    let projectID: PersistentIdentifier?

    var project: Project? {
        projectID.flatMap { modelContext.model(for: $0) as? Project }
    }
}
```

## Document-Scoped Containers

For document-based apps where each document has its own data store, create a per-document `ModelContainer` with a unique `StoreConfiguration`:

```swift
struct DocumentView: View {
    let documentURL: URL

    private var container: ModelContainer {
        let storeURL = documentURL.appendingPathComponent("store.sqlite")
        let config = ModelConfiguration(
            schema: Schema([Project.self, TaskItem.self]),
            url: storeURL
        )
        // URL is different per document = separate store
        return try! ModelContainer(for: [Project.self, TaskItem.self], configurations: [config])
    }

    var body: some View {
        ContentView()
            .modelContainer(container)
    }
}
```

**Key rules:**
- Each document gets a unique `url` in `ModelConfiguration`
- One container per document — never share a container between documents
- The container must live as long as the document is open

- **macOS 26 update:** On macOS 26+, prefer `.modelContainer(sharedModelContainer)` over `.modelContext(ModelContext(shared))`. The `.modelContext` approach has a known persistence bug on macOS 26. Use `WindowGroup(for: PersistentModel.ID.self)` to open windows with typed model IDs, resolving via `context.model(for: id)`.
- A single `ModelContainer` can be shared across `WindowGroup` and `MenuBarExtra` scenes.
## NSPersistentHistoryTracking for XPC

When multiple processes access the same SQLite store (e.g., via XPC), use `NSPersistentHistoryTracking` to propagate changes:

```swift
// In the store description, enable history tracking
let description = NSPersistentStoreDescription(url: storeURL)
description.setOption(true as NSNumber,
    forKey: NSPersistentHistoryTrackingKey)
description.setOption(true as NSNumber,
    forKey: NSPersistentStoreRemoteChangeNotificationPostOptionKey)

// Each process observes remote changes
NotificationCenter.default.addObserver(
    forName: .NSPersistentStoreRemoteChange,
    object: container.persistentStoreCoordinator,
    queue: .main
) { _ in
    // Merge history into local context
    self.mergePersistentHistory()
}
```

SwiftData does not expose `NSPersistentHistoryTracking` directly — you access it through the `NSPersistentStoreDescription` on the underlying coordinator. This is an advanced pattern for multi-process app extensions or XPC services.
**SwiftData History (WWDC24):** SwiftData now has a native `History` API that supersedes the Core Data `NSPersistentHistoryTracking` pattern for most use cases. History consists of `Transactions` (grouped changes) and `Changes` (insert/update/delete), with a `HistoryToken` to bookmark positions. For custom stores, conform to `HistoryProviding`. The history API is `ModelActor`-based, providing safe concurrency for transaction processing. Use `HistoryDescriptor` and `fetchHistory(after:)` to observe changes. Note: history fetch filtering must be applied in memory (the fetch request returns nil; you filter transactions after fetching).
**Cross-process refresh limitation (macOS 26):** When another process (XPC service, MCP server) writes to the shared SwiftData store, `@Query` views in the main app may not auto-refresh. `NSPersistentStoreRemoteChange` notifications may not trigger a UI update. A manual `ModelContext` reset or app restart may be required. This is a known gap — there is no public API to force `@Query` to re-evaluate from disk for cross-process writes.
## Constraints & Gotchas

- **Uniqueness constraints:** `@Attribute(.unique)` violations throw on `save()`. Wrap in `do/catch` and handle duplicates explicitly.
- **CloudKit sandbox vs production:** Debug builds use the sandbox. Release builds use production. Never mix data between them. There is no migration between sandbox and production CloudKit environments.
- **Predicate limitations:** `#Predicate` does not support all Swift expressions. Operations like custom closures, `reduce`, or compound `if-let` chains within predicates will fail at compile time. Stick to supported operators and key paths.
- **Relationship ownership:** Only one side of a relationship needs `@Relationship`. The inverse is inferred. Add `@Relationship` on both sides only when you need explicit control over delete rules.
- **Migration order:** Custom migration `willMigrate` runs before `didMigrate`. The schema changes between these two callbacks. Use `willMigrate` to read old data, `didMigrate` to validate.
- **Thread confinement:** `ModelContext` is bound to the actor that created it. On macOS 26+ with Swift 6.3, use `@ModelActor` for background work and `DispatchQueue.main` or `@MainActor` for UI context.
- **CloudKit initial sync:** First CloudKit sync can take minutes for large datasets. Show a progress indicator.

## Common Mistakes

1. **Creating ModelContainer inside a SwiftUI view body** — the container gets recreated on every render. Store it at the app/scene level or in a singleton.
2. **Using `NSPredicate` format strings instead of `#Predicate`** — SwiftData models require the macro. Format strings bypass type checking and fail at runtime.
3. **Saving inside a `@Query` update loop** — modifying a model while SwiftUI is iterating the query results causes undefined behavior. Batch changes and save once.
4. **Heavy model initialization in `@Model` init** — SwiftData calls init during fault synthesis. Keep inits lightweight; use computed properties for expensive derived values.
5. **Ignoring migration errors** — a failed migration crashes the app on launch. Always test migrations with real data from previous versions.
6. **Using `@Relationship` on both sides without matching inverse** — SwiftData infers the inverse. Declaring it twice with conflicting rules causes undefined behavior.
7. **Accessing relationship arrays from a background context** — faults trigger SQLite I/O. Always fault on the correct actor.
8. **Assuming CloudKit sync is immediate** — sync is eventual. Never depend on immediate cross-device data availability.

## Review Checklist

- [ ] `@Model` classes use value types for simple properties and `@Attribute` modifiers where needed (`.unique`, `.externalStorage`, `.transformable`)
- [ ] Relationships have explicit delete rules (`.cascade`, `.nullify`, `.deny`) — not relying on default
- [ ] `ModelContainer` is created once at app/scene level, not inside views
- [ ] `#Predicate` macro is used for all `@Query` filtering — no `NSPredicate` format strings
- [ ] CloudKit configuration matches entitlement container identifiers
- [ ] Test builds use CloudKit sandbox; release builds use production — no cross-contamination
- [ ] Migrations are versioned with `VersionedSchema` and tested against real SQLite fixtures
- [ ] Batch operations (`delete(model:where:)`) used for large dataset mutations
- [ ] Multi-window apps share a single container, passing `PersistentIdentifier` between windows
- [ ] Document-scoped containers use unique `ModelConfiguration` URLs per document
- [ ] `@ModelActor` used for background context access
- [ ] Uniqueness constraint violations are caught and handled on `save()`

## References

- [references/swiftdata-patterns.md](references/swiftdata-patterns.md) — Deep-dive patterns: complete model with relationships, multi-window shared container, CloudKit conflict handling, document-scoped containers, batch performance operations
- WWDC 2023 Session 10187: "Meet SwiftData" — https://developer.apple.com/videos/play/wwdc2023/10187/
- SwiftData documentation: https://developer.apple.com/documentation/swiftdata
- WWDC 2024 Session 10075: "Track model changes with SwiftData history" — https://developer.apple.com/videos/play/wwdc2024/10075/
- WWDC 2025 Session 291: "SwiftData: Dive into inheritance and schema migration" — https://developer.apple.com/videos/play/wwdc2025/291/
- SwiftData Cookbook (Hacking with Swift): https://www.hackingwithswift.com/quick-start/swiftdata
