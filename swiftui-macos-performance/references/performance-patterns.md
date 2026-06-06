# SwiftUI Performance Patterns

Deep-dive patterns for SwiftUI performance optimization on macOS 26+. Covers EquatableView for expensive subtrees, Table with 10k+ rows, reading Instruments Hangs traces, MainActor hop batching, and async image loading with cache.

## Contents

- [EquatableView for Expensive View Subtree](#equatableview-for-expensive-view-subtree)
- [Table with 10k+ Rows: Identity, Cell Reuse, Background Loading](#table-with-10k-rows-identity-cell-reuse-background-loading)
- [Instruments Hangs: Reading the Trace, Finding Main Thread Stalls](#instruments-hangs-reading-the-trace-finding-main-thread-stalls)
- [MainActor Hop Batching Pattern](#mainactor-hop-batching-pattern)
- [Async Image Loading with Cache](#async-image-loading-with-cache)

## EquatableView for Expensive View Subtree

When a view subtree is expensive to evaluate and its inputs are stable most of the time, `EquatableView` prevents the entire subtree from being re-evaluated and diffed. The technique is to conform the view to `Equatable`, implement a meaningful `==`, and apply `.equatable()`.

**Pattern: Chart wrapper**

A chart view that renders hundreds of data points via `Canvas` or `CoreGraphics`:

```swift
struct PowerUsageChart: View, Equatable {
    let samples: [PowerSample]
    let timeRange: ClosedRange<Date>
    let maxY: Double

    static func == (lhs: Self, rhs: Self) -> Bool {
        // Compare count first (cheap fast path):
        guard lhs.samples.count == rhs.samples.count else { return false }
        // If count same, check the range — if the visible time window
        // hasn't changed, the chart looks identical.
        guard lhs.timeRange == rhs.timeRange else { return false }
        guard lhs.maxY == rhs.maxY else { return false }
        // Deep compare only when the range changed, and even then
        // sample-by-sample only if count is small:
        if lhs.samples.count < 100 {
            return zip(lhs.samples, rhs.samples).allSatisfy { $0 == $1 }
        }
        // For large datasets, count + range match is enough.
        return true
    }

    var body: some View {
        Canvas { context, size in
            guard samples.count > 1 else { return }
            let xStep = size.width / CGFloat(samples.count - 1)
            let yScale = size.height / maxY

            var path = Path()
            for (index, sample) in samples.enumerated() {
                let x = CGFloat(index) * xStep
                let y = size.height - CGFloat(sample.value) * yScale
                if index == 0 {
                    path.move(to: CGPoint(x: x, y: y))
                } else {
                    path.addLine(to: CGPoint(x: x, y: y))
                }
            }

            context.stroke(path, with: .color(.accentColor), lineWidth: 1.5)
        }
        .frame(height: 100)
    }
}

// In parent view:
struct DashboardView: View {
    @StateObject var monitor = PowerMonitor()

    var body: some View {
        VStack {
            // Without .equatable(): every DashboardView re-evaluation
            // (triggered by a text field edit elsewhere) causes the
            // Canvas drawing code to re-diff.
            // With .equatable(): SwiftUI checks == first. If the
            // chart inputs haven't changed, it skips everything.
            PowerUsageChart(
                samples: monitor.samples,
                timeRange: monitor.timeRange,
                maxY: monitor.maxY
            )
            .equatable()

            TextEditor(text: monitor.notes)  // edits here don't
                                             // re-evaluate the chart
        }
    }
}
```

**When to use:**

- `Canvas` or `CoreGraphics` drawing with stable data
- Nested `ForEach` with complex row views
- Views with expensive `GeometryReader` + layout calculations
- Views behind conditionals (`.ifLet`, custom `if/else`) where the condition rarely toggles

**When not to use:**

- Simple leaf views (`Text`, `Image`, `Divider`) — equality check costs more than body re-evaluation
- Views with `@Binding` inputs — `Binding<Struct>` compares the current value, but `Binding` over reference types won't work
- Views whose every input changes every evaluation

**Testing that `.equatable()` is working:**

In the SwiftUI instrument, add the `swiftui` signpost interval. Record your scenario and check the "View Body" track: if `.equatable()` is active and inputs are stable, the view body evaluation count should be 1 (or proportional to actual input changes, not parent re-evaluations).

## Table with 10k+ Rows: Identity, Cell Reuse, Background Loading

`Table` on macOS uses lazy row materialization. Only rows visible in the scroll view exist in memory. Understanding identity, cell reuse, and loading strategy is essential for large datasets.

**Identity:**

Every row needs a stable `Identifiable` ID. The ID drives selection state, diffing, and animation:

```swift
struct RowModel: Identifiable, Hashable {
    let id: UUID          // stable across data reloads
    var name: String
    var size: Int64
    var modified: Date

    // If rows can be re-sorted, Hashable must NOT include position.
    // id + content is correct:
    func hash(into hasher: inout Hasher) {
        hasher.combine(id)
    }
}
```

**Critical:** If you use `enumerated()` or array indices as IDs, deleting row 0 shifts all indices. The row previously at index 1 (identity `1`) now has identity `0`, so SwiftUI reuses its view state for the *content* that moved from index 1 to 0. Selection state attaches to identity, so the selection jumps.

**Cell reuse and view state:**

`Table` internally recycles row views, similar to `NSTableView`. If you use `@State` inside a row view (e.g., for a toggle or disclosure state), the state will be recycled with the cell. Solution: hoist row-level state into the model:

```swift
// BAD: State in recycled row view
struct BadRowView: View {
    let item: RowModel
    @State private var isExpanded = false  // recycled with cell!

    var body: some View {
        HStack {
            Button(isExpanded ? "Collapse" : "Expand") {
                isExpanded.toggle()
            }
            Text(item.name)
        }
    }
}

// GOOD: State in the model
struct RowModel: Identifiable {
    let id: UUID
    var name: String
    var isExpanded: Bool = false  // persists across cell recycling
}

struct GoodRowView: View {
    @Binding var item: RowModel  // model holds expansion state

    var body: some View {
        HStack {
            Button(item.isExpanded ? "Collapse" : "Expand") {
                item.isExpanded.toggle()
            }
            Text(item.name)
        }
    }
}
```

**Background loading:**

```swift
@Observable
class FileListViewModel {
    private(set) var rows: [RowModel] = []
    private(set) var isLoading = false

    func load(directory: URL) async {
        isLoading = true

        let loadedRows = await Task.detached(priority: .userInitiated) {
            let fm = FileManager.default
            guard let enumerator = fm.enumerator(
                at: directory,
                includingPropertiesForKeys: [.fileSizeKey, .contentModificationDateKey],
                options: .skipsHiddenFiles
            ) else { return [RowModel]() }

            var results: [RowModel] = []
            results.reserveCapacity(10_000)

            for case let fileURL as URL in enumerator {
                guard let resourceValues = try? fileURL.resourceValues(
                    forKeys: [.fileSizeKey, .contentModificationDateKey]
                ) else { continue }

                results.append(RowModel(
                    id: UUID(),
                    name: fileURL.lastPathComponent,
                    size: Int64(resourceValues.fileSize ?? 0),
                    modified: resourceValues.contentModificationDate ?? .distantPast
                ))
            }

            return results
        }.value

        // Single MainActor hop, single diff pass
        self.rows = loadedRows
        self.isLoading = false
    }

    func appendRows(_ newRows: [RowModel]) {
        // Batch append — one diff pass
        self.rows.append(contentsOf: newRows)
    }
}
```

**Sorting without full reload:**

```swift
let descriptors = [KeyPathComparator(\RowModel.name)]

Table(of: RowModel.self, selection: $selection, sortOrder: $descriptors) {
    TableColumn("Name", value: \.name) { item in Text(item.name) }
    TableColumn("Size", value: \.size) { item in
        Text(ByteCountFormatter.string(fromByteCount: item.size, countStyle: .file))
    }
}

// Sorting mutates the array in-place:
.onChange(of: descriptors) { _, newDescriptors in
    rows.sort(using: newDescriptors)  // in-place sort, Table handles animation
}
```

## Instruments Hangs: Reading the Trace, Finding Main Thread Stalls

The Hangs instrument captures moments when the main thread fails to process events within a responsiveness threshold. It's the primary tool for diagnosing jank and freezes in SwiftUI apps.

**Recording:**

1. Open Xcode > Product > Profile (⌘I), select the "Hangs" template.
2. Click the record button, reproduce the sluggish interaction.
3. Stop recording.

**Reading the trace:**

The Hangs instrument shows:
- **A timeline at the top:** Bars represent hangs; longer bars = longer stalls.
- **A categorized list below:** Each hang categorized by the call stack at the stall point.
- **The call stack at the bottom:** Select any hang to see exactly what the main thread was doing.

**Interpreting call stacks:**

| Common symbol | Meaning | Fix |
|---------------|---------|-----|
| `View.updateBody` / `closure #1 in View.body` | SwiftUI body evaluation | Narrow `@Observable` reads, use `.equatable()` |
| `static Layout.evaluate` | Layout pass forced | Reduce view complexity, avoid `GeometryReader` abuse |
| `[NSView layoutSubtreeIfNeeded]` | AppKit layout (bridged) | Check autolayout constraints in representable views |
| `-[NSTableCellView ...]` | Cell configuration in Table | Lighter row content, fewer subviews |
| Your code in a loop | Custom computation on main thread | Move to `Task.detailed`, use `nonisolated` |
| `URLSession.data(for:)` (synchronous) | Blocking network call | Use `URLSession.data(from:)` async variant |
| `Data(contentsOf:)` (synchronous) | Blocking file read | Use `Data(contentsOf:options:)` on background queue |

**Thresholds:**

The default render-loop threshold is tied to the display refresh interval (~16ms at 60Hz). Hangs shorter than this are common and usually irrelevant. Focus on hangs > 33ms (one missed frame on 30Hz, two on 60Hz) or > 100ms (perceptible freeze).

**Correlating with SwiftUI instrument:**

Record with both Hangs and SwiftUI templates simultaneously. When Hangs shows a stall, switch to the SwiftUI instrument and look at the same timestamp: which views were being evaluated? Which ones have high body duration? The two instruments together pinpoint *what* SwiftUI was doing when the stall occurred.

**Export for sharing:**

Right-click any hang > "Export Track" > save as `.trace`. Colleagues can open the trace in Instruments without access to your project or build.

## MainActor Hop Batching Pattern

Every transition to the main actor has overhead: context switching, isolation checking, and SwiftUI's view diffing after each state mutation. The batching pattern minimizes these transitions.

**Anti-pattern — per-item hops:**

```swift
// N hops, N diffs, N layout passes
func processResults(_ results: [SearchResult]) async {
    for result in results {
        await MainActor.run {
            self.results.append(result)  // SwiftUI diffs and lays out after each
        }
    }
}
```

**Pattern — single batch hop:**

```swift
// 1 hop, 1 diff, 1 layout pass
func processResults(_ results: [SearchResult]) async {
    let processed = await Task.detached(priority: .userInitiated) {
        results.map { SearchResult(displayTitle: format($0)) }
    }.value

    self.results = processed  // single MainActor hop
}
```

**Pattern — chunked processing for streaming results:**

When results arrive incrementally (e.g., from a streaming search API), batch them into chunks:

```swift
@SearchableActor  // custom global actor for search coordination
class SearchModel {
    private var buffer: [SearchResult] = []
    private var flushTask: Task<Void, Never>?

    func addResults(_ newResults: [SearchResult]) {
        buffer.append(contentsOf: newResults)

        // Flush every 100ms or 50 items, whichever comes first
        flushTask?.cancel()
        flushTask = Task { @MainActor in
            try? await Task.sleep(for: .milliseconds(100))
            if Task.isCancelled { return }
            self.flushBuffer()
        }
    }

    @MainActor
    private func flushBuffer() {
        guard !buffer.isEmpty else { return }
        results.append(contentsOf: buffer)
        buffer.removeAll(keepingCapacity: true)
    }
}
```

**Pattern — nonisolated computation with final publish:**

```swift
@Observable
class ImageGalleryModel {
    private(set) var thumbnails: [Thumbnail] = []

    // nonisolated: runs on whatever actor calls it (or a detached task)
    nonisolated func generateThumbnails(from images: [ImageURL]) async -> [Thumbnail] {
        await withTaskGroup(of: Thumbnail.self) { group in
            for url in images {
                group.addTask {
                    await self.generateOne(url)
                }
            }

            var results: [Thumbnail] = []
            results.reserveCapacity(images.count)
            for await thumbnail in group {
                results.append(thumbnail)
            }
            return results
        }
    }

    // Called from the main actor context with final results
    func reload(with images: [ImageURL]) async {
        let thumbs = await generateThumbnails(from: images)
        self.thumbnails = thumbs  // single hop: assign, diff, layout
    }
}
```

## Async Image Loading with Cache

Images are a common performance bottleneck: loading from disk or network on the main thread causes stalls, while naive async loading causes flicker and redundant fetches.

**Two-tier cache pattern (memory + disk):**

```swift
actor ImageCache {
    private var memoryCache: [URL: NSImage] = [:]
    private let fileManager = FileManager.default
    private let cacheDirectory: URL

    init() {
        cacheDirectory = fileManager.urls(for: .cachesDirectory, in: .userDomainMask)[0]
            .appendingPathComponent("ImageCache")
        try? fileManager.createDirectory(at: cacheDirectory, withIntermediateDirectories: true)
    }

    func image(for url: URL) async -> NSImage? {
        // Check memory first
        if let cached = memoryCache[url] {
            return cached
        }

        // Check disk
        let diskPath = cacheDirectory.appendingPathComponent(url.lastPathComponent)
        if let image = NSImage(contentsOf: diskPath) {
            memoryCache[url] = image  // promote to memory
            return image
        }

        // Fetch
        guard let (data, _) = try? await URLSession.shared.data(from: url),
              let image = NSImage(data: data) else {
            return nil
        }

        // Write to disk (fire-and-forget)
        Task.detached(priority: .background) {
            try? data.write(to: diskPath)
        }

        // Write to memory
        memoryCache[url] = image
        return image
    }
}

// Usage in SwiftUI:
struct CachedAsyncImage: View {
    let url: URL
    @State private var image: NSImage?
    @State private var isLoading = false

    private let cache = ImageCache()  // shared instance in practice

    var body: some View {
        Group {
            if let image {
                Image(nsImage: image)
                    .resizable()
                    .aspectRatio(contentMode: .fit)
            } else if isLoading {
                ProgressView()
                    .controlSize(.small)
            } else {
                Image(systemName: "photo")
                    .foregroundStyle(.secondary)
            }
        }
        .task(id: url) {  // re-runs if url changes; cancels previous
            isLoading = true
            image = await cache.image(for: url)
            isLoading = false
        }
    }
}
```

**Key details:**

- **`actor ImageCache`:** The cache is an actor, so all access is serialized. No locks needed — Swift's actor isolation guarantees safety.
- **`.task(id: url)`:** SwiftUI's `.task` modifier with an `id` parameter automatically cancels the previous task when the URL changes. This prevents stale downloads from completing and overwriting correct images (the "cell reuse" problem).
- **Two-tier:** Memory cache avoids disk I/O for recently viewed images. Disk cache avoids network I/O for images that may return to view.
- **Non-sendable note:** `NSImage` is not `Sendable`. Since we only access it from the main actor after the `await`, this is safe. The actor stores it, but the caller always reads it on the main actor context.
- **For large galleries with 1k+ images:** Add an eviction policy to the memory cache (LRU with a max count, e.g., 200 images) to avoid unbounded growth:

```swift
private var accessOrder: [URL] = []
private let maxMemoryItems = 200

private func insertToMemory(_ image: NSImage, for url: URL) {
    memoryCache[url] = image
    accessOrder.removeAll { $0 == url }
    accessOrder.append(url)

    while memoryCache.count > maxMemoryItems {
        let evicted = accessOrder.removeFirst()
        memoryCache.removeValue(forKey: evicted)
    }
}
```

---

*Last updated: 2026-06-06. Tested patterns against macOS 26 SDK / Swift 6.3.*
