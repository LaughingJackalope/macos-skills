# SwiftData Patterns — Reference

Complete patterns for SwiftData on macOS 26+. Covers model definition with relationships, predicate composition, migrations, shared containers, CloudKit sync, document-scoped stores, and batch performance operations. Swift 6.3 / macOS 26+.

---

## 1. Complete Model with Relationships

Full three-model hierarchy with cascading relationships and computed properties:

```swift
import SwiftData
import Foundation

@Model
final class Workspace {
    @Attribute(.unique) var slug: String
    var name: String
    var colorHex: String
    var createdAt: Date
    var sortOrder: Int

    @Relationship(deleteRule: .cascade, inverse: \Project.workspace)
    var projects: [Project] = []

    init(slug: String = UUID().uuidString, name: String, colorHex: String = "#007AFF") {
        self.slug = slug
        self.name = name
        self.colorHex = colorHex
        self.createdAt = .now
        self.sortOrder = 0
    }

    var projectCount: Int { projects.count }
    var allTasks: [TaskItem] { projects.flatMap(\.tasks) }
}

@Model
final class Project {
    @Attribute(.unique) var identifier: String
    var name: String
    var notes: String
    var priority: Int
    var dueDate: Date?
    var workspace: Workspace?

    @Relationship(deleteRule: .cascade, inverse: \TaskItem.project)
    var tasks: [TaskItem] = []

    init(identifier: String = UUID().uuidString, name: String) {
        self.identifier = identifier
        self.name = name
        self.notes = ""
        self.priority = 0
    }

    var completedTasks: [TaskItem] { tasks.filter(\.isCompleted) }
    var isFullyCompleted: Bool { !tasks.isEmpty && tasks.allSatisfy(\.isCompleted) }
}

@Model
final class TaskItem {
    var title: String
    var isCompleted: Bool
    var completedAt: Date?
    var project: Project?
    @Attribute(.externalStorage) var attachmentData: Data?

    init(title: String) {
        self.title = title
        self.isCompleted = false
    }

    func markComplete() {
        isCompleted = true
        completedAt = .now
    }
}
```

**Predicate composition across relationships:**

```swift
// Tasks in a specific workspace
#Predicate<TaskItem> { task in
    task.project?.workspace?.slug == "main"
    && task.isCompleted == false
}

// Projects with overdue tasks
#Predicate<Project> { project in
    project.dueDate != nil
    && project.dueDate! < Date()
    && project.tasks.contains { $0.isCompleted == false }
}

// Search across model and relationship properties
func searchPredicate(query: String) -> Predicate<TaskItem> {
    #Predicate<TaskItem> { task in
        task.title.localizedStandardContains(query)
        || (task.project?.name.localizedStandardContains(query) ?? false)
    }
}
```

**Migration adding a field to an existing model:**

```swift
// Version 2: adding `sortOrder` to Workspace
enum SchemaV1: VersionedSchema {
    static var versionIdentifier: String { "v1" }
    static var models: [any PersistentModel.Type] {
        [Workspace.self, Project.self, TaskItem.self]
    }
}

enum SchemaV2: VersionedSchema {
    static var versionIdentifier: String { "v2" }
    static var models: [any PersistentModel.Type] {
        [Workspace.self, Project.self, TaskItem.self]
    }
}

let migrationPlan = MigrationStage.custom(
    fromVersion: SchemaV1.self,
    toVersion: SchemaV2.self,
    willMigrate: { context in
        // Provide default value for new `sortOrder` property
        let workspaces = try context.fetch(FetchDescriptor<Workspace>())
        for (index, workspace) in workspaces.enumerated() {
            workspace.sortOrder = index
        }
        try context.save()
    },
    didMigrate: nil
)

let container = try ModelContainer(
    for: SchemaV2.self,
    migrationPlan: migrationPlan,
    configurations: [ModelConfiguration(schema: SchemaV2.schema)]
)
```

---

## 2. Shared ModelContainer with Multi-Window Pattern

The standard approach: create the container once at the app level and pass it to all scenes. Windows communicate via `PersistentIdentifier`.

```swift
@main
struct ProjectApp: App {
    let container: ModelContainer

    init() {
        let schema = Schema([Workspace.self, Project.self, TaskItem.self])
        let config = ModelConfiguration(
            schema: schema,
            isStoredInMemoryOnly: false
        )
        container = try! ModelContainer(for: schema, configurations: [config])
    }

    var body: some Scene {
        // Main window
        WindowGroup {
            SidebarView()
        }
        .modelContainer(container)

        // Detail window — receives a project ID
        WindowGroup("Project", id: "project-detail") { $projectID in
            if let projectID {
                ProjectDetailView(projectID: projectID)
            }
        }
        .modelContainer(container)

        // Inspector panel (macOS 26+ window_scene hybrid)
        WindowGroup("Inspector", id: "inspector") { $inspectorProjectID in
            if let inspectorProjectID {
                InspectorView(projectID: inspectorProjectID)
            }
        }
        .modelContainer(container)
        .defaultSize(width: 320, height: 600)

        // Settings
        Settings {
            SettingsView()
        }
        .modelContainer(container)
    }
}

// Passing a model ID to open a window
struct SidebarView: View {
    @Environment(\.openWindow) var openWindow
    @Query(sort: \Project.name) var projects: [Project]

    var body: some View {
        List(projects) { project in
            Text(project.name)
                .contextMenu {
                    Button("Open in New Window") {
                        openWindow(id: "project-detail", value: project.persistentModelID)
                    }
                    Button("Show Inspector") {
                        openWindow(id: "inspector", value: project.persistentModelID)
                    }
                }
        }
    }
}

// Resolving the model from a PersistentIdentifier in the destination window
struct ProjectDetailView: View {
    @Environment(\.modelContext) var modelContext
    let projectID: PersistentIdentifier?

    private var project: Project? {
        guard let projectID else { return nil }
        return modelContext.model(for: projectID) as? Project
    }

    var body: some View {
        if let project {
            ProjectEditor(project: project)
        } else {
            ContentUnavailableView("Project not found",
                systemImage: "questionmark.folder")
        }
    }
}
```

**Key points:**
- The container is created once. Every `.modelContainer(container)` in the scene graph uses the same instance.
- `PersistentIdentifier` is stable across windows and resizing. Use it to pass references, not the model object itself.
- When a model is deleted in one window, `@Query` in other windows updates automatically — all windows share the same context observation.

---

## 3. CloudKit Sync Configuration and Conflict Handling

```swift
import SwiftData
import CloudKit

class CloudKitContainerFactory {
    static func makeContainer() throws -> ModelContainer {
        let schema = Schema([Workspace.self, Project.self, TaskItem.self])

        // Private CloudKit database
        let privateConfig = ModelConfiguration(
            schema: schema,
            url: URL.documentsDirectory.appendingPathComponent("private.sqlite"),
            cloudKitDatabase: .private("iCloud.com.example.ProjectApp")
        )

        return try ModelContainer(for: schema, configurations: [privateConfig])
    }
}

// Conflict handling: SwiftData uses NSPersistentCloudKitContainer underneath.
// For custom merge policies, access the persistent store coordinator:
class ConflictResolver {
    func configureStore(for container: ModelContainer) {
        guard let coordinator = container.persistentStoreCoordinator.persistentStores.first else {
            return
        }
        // Merge policy: prefer in-memory changes over store (client wins)
        // Note: this affects the entire store
        container.persistentStoreCoordinator.managedObjectContext?
            .mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
    }
}

// Checking CloudKit account status
class CloudKitStatusObserver: ObservableObject {
    @Published var accountStatus: CKAccountStatus = .couldNotDetermine
    @Published var isSyncing = false

    init() {
        Task {
            let status = try await CKContainer(identifier: "iCloud.com.example.ProjectApp")
                .accountStatus()
            await MainActor.run {
                self.accountStatus = status
            }
        }
    }

    var isAvailable: Bool {
        accountStatus == .available
    }
}

struct CloudKitStatusView: View {
    @StateObject private var observer = CloudKitStatusObserver()

    var body: some View {
        HStack {
            switch observer.accountStatus {
            case .available:
                Label("CloudKit Active", systemImage: "icloud.fill")
                    .foregroundStyle(.green)
            case .noAccount:
                Label("Sign in to iCloud", systemImage: "iclash.slash")
                    .foregroundStyle(.orange)
            case .restricted:
                Label("iCloud Restricted", systemImage: "lock.icloud")
                    .foregroundStyle(.red)
            case .couldNotDetermine:
                Label("Checking…", systemImage: "icloud")
                    .foregroundStyle(.secondary)
            case .temporarilyUnavailable:
                Label("iCloud Temporarily Unavailable", systemImage: "exclamationmark.icloud")
                    .foregroundStyle(.orange)
            @unknown default:
                EmptyView()
            }
            if observer.isSyncing {
                ProgressView()
                    .controlSize(.small)
            }
        }
    }
}
```

**Testing CloudKit sync:**
1. Run on device A, create data
2. Wait for sync indicator to settle
3. Run on device B, verify data appears
4. Disconnect device B, make changes on device A
5. Reconnect device B, verify eventual consistency

---

## 4. Document-Scoped ModelContainer

Each document window gets its own SQLite store and `ModelContainer`. This is essential for document-based SwiftData apps.

```swift
import SwiftData
import UniformTypeIdentifiers

extension UTType {
    static var projectStore = UTType(exportedAs: "com.example.projectstore")
}

struct ProjectDocumentView: View {
    let documentURL: URL
    @State private var container: ModelContainer?

    var body: some View {
        Group {
            if let container {
                ContentView()
                    .modelContainer(container)
            } else {
                ProgressView("Loading document…")
            }
        }
        .task {
            container = await makeDocumentContainer(url: documentURL)
        }
    }

    private func makeDocumentContainer(url: URL) async -> ModelContainer {
        let storeURL = url.appendingPathComponent("data.sqlite")
        let config = ModelConfiguration(
            schema: Schema([Project.self, TaskItem.self]),
            url: storeURL,
            cloudKitDatabase: .none  // Document stores typically don't sync via CloudKit
        )
        return try! ModelContainer(for: [Project.self, TaskItem.self], configurations: [config])
    }
}

// With NSDocument integration via AppKit interop
final class Document: NSDocument {
    private var container: ModelContainer?

    override func makeWindowControllers() {
        guard let container else { return }
        let hostingController = NSHostingController(
            rootView: DocumentContentView()
                .modelContainer(container)
        )
        let window = NSWindow(contentViewController: hostingController)
        window.setContentSize(NSSize(width: 800, height: 600))
        let wc = NSWindowController(window: window)
        addWindowController(wc)
    }

    override func read(from url: URL, ofType typeName: String) throws {
        let storeURL = url.appendingPathComponent("data.sqlite")
        let config = ModelConfiguration(
            schema: Schema([Project.self]),
            url: storeURL
        )
        container = try ModelContainer(for: [Project.self], configurations: [config])
    }

    override func data(ofType typeName: String) throws -> Data {
        // SwiftData auto-saves to SQLite. Return the directory package data.
        guard let fileURL = fileURL else { throw DocumentError.noFileURL }
        let storeData = try Data(contentsOf: fileURL.appendingPathComponent("data.sqlite"))
        return storeData
    }
}
```

---

## 5. Performance: Batch Delete, Faulting, NSFetchRequest Batch Size

### Batch Delete

```swift
extension ModelContext {
    /// Delete all models matching the predicate.
    func batchDelete<T: PersistentModel>(
        _ type: T.Type,
        where predicate: Predicate<T>
    ) throws {
        let descriptor = FetchDescriptor<T>(predicate: predicate)
        let items = try fetch(descriptor)
        for item in items {
            delete(item)
        }
        try save()  // Single transaction for entire batch
    }

    /// Delete using SQLite-level batch delete (no object faulting).
    /// Much faster for large sets — doesn't load objects into memory.
    func efficientBatchDelete<T: PersistentModel>(
        _ type: T.Type,
        where predicate: Predicate<T>
    ) throws {
        try delete(model: type, where: predicate, subpredicate: nil)
    }
}

// Usage: Delete all completed tasks older than 30 days
let thirtyDaysAgo = Calendar.current.date(byAdding: .day, value: -30, to: .now)!
let predicate = #Predicate<TaskItem> { task in
    task.isCompleted && (task.completedAt ?? .distantPast) < thirtyDaysAgo
}
try context.efficientBatchDelete(TaskItem.self, where: predicate)
```

### Faulting

SwiftData faults objects — accessing a property triggers only when needed. However, accessing relationships from a background context blocks the calling thread. Control faulting explicitly:

```swift
// Fetching with controlled properties
func fetchTaskTitles() throws -> [String] {
    var descriptor = FetchDescriptor<TaskItem>()
    // Fetch only the title property — no full object fault needed
    // On macOS 26+, use propertiesToFetch:
    // descriptor.propertiesToFetch = [\.title]
    descriptor.fetchBatchSize = 100  // Fault in batches of 100

    let items = try modelContext.fetch(descriptor)
    return items.map(\.title)
}
```

### NSFetchRequest Batch Size (for NSFetchedResultsController interop)

When using SwiftData with Core Data interop (e.g., `NSPersistentCloudKitContainer`):

```swift
func makeBatchedFetchRequest() -> NSFetchRequest<TaskItem> {
    let request = NSFetchRequest<TaskItem>(entityName: "TaskItem")
    request.predicate = NSPredicate(format: "isCompleted == NO")
    request.sortDescriptors = [NSSortDescriptor(key: "title", ascending: true)]
    request.fetchBatchSize = 50
    request.returnsObjectsAsFaults = true  // Fault until property access
    request.includesPropertyValues = false  // Only objectID initially
    return request
}
```

### Counting Without Fetching

```swift
extension ModelContext {
    func count<T: PersistentModel>(
        _ type: T.Type,
        where predicate: Predicate<T>? = nil
    ) throws -> Int {
        var descriptor = FetchDescriptor<T>()
        if let predicate {
            descriptor.predicate = predicate
        }
        return try fetchCount(descriptor)
    }
}

// Usage: Count without loading objects
let pendingCount = try context.count(TaskItem.self,
    where: #Predicate { $0.isCompleted == false })
// VS. the slow approach:
// let pendingCount = try context.fetch(FetchDescriptor<TaskItem>()).filter { !$0.isCompleted }.count
```

### Insert Performance

For bulk inserts, wrap in a single transaction:

```swift
func bulkInsert(count: Int) throws {
    for i in 0..<count {
        let task = TaskItem(title: "Task \(i)")
        modelContext.insert(task)
    }
    // Single write transaction
    try modelContext.save()
}
```

For 10,000+ rows, batch into chunks:

```swift
func bulkInsertChunked(items: [TaskItem], chunkSize: Int = 1000) throws {
    for chunk in items.chunked(into: chunkSize) {
        for item in chunk {
            modelContext.insert(item)
        }
        try modelContext.save()
        // Reset to clear registered objects
        modelContext.reset()
    }
}

extension Array {
    func chunked(into size: Int) -> [[Element]] {
        stride(from: 0, to: count, by: size).map {
            Array(self[$0..<Swift.min($0 + size, count)])
        }
    }
}
```
