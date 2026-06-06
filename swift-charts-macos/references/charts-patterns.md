# Swift Charts Patterns — macOS

## Interactive Bar Chart with Selection

```swift
struct SalesData: Identifiable, PlottableValue {
    let id: String
    let quarter: String
    let revenue: Double
}

class ChartModel: ObservableObject {
    @Published var selection: String?
    let data: [SalesData] = [
        SalesData(id: "Q1-2025", quarter: "Q1 2025", revenue: 84000),
        SalesData(id: "Q2-2025", quarter: "Q2 2025", revenue: 92000),
        SalesData(id: "Q3-2025", quarter: "Q3 2025", revenue: 105000),
        SalesData(id: "Q4-2025", quarter: "Q4 2025", revenue: 118000),
    ]
}

struct SalesChartView: View {
    @StateObject private var model = ChartModel()

    var body: some View {
        VStack(alignment: .leading) {
            Chart(model.data) { item in
                BarMark(
                    x: .value("Quarter", item.quarter),
                    y: .value("Revenue", item.revenue)
                )
                .foregroundStyle(
                    model.selection == item.id
                        ? Color.accentColor
                        : Color.accentColor.opacity(0.3)
                )
                .cornerRadius(4)
            }
            .chartOverlay { proxy in
                Rectangle()
                    .fill(Color.clear)
                    .contentShape(Rectangle())
                    .onTapGesture { location in
                        // Hit-test via proxy position for X
                        guard let plotArea = proxy.plotAreaRect else { return }
                        // Map screen X to data-space; iterate data to find nearest
                        guard let _ = proxy.value(atX: location.x, as: String.self) else {
                            return
                        }
                        // In practice, iterate model.data and pick nearest quarter
                    }
            }
            .chartYAxis {
                AxisMarks(position: .leading) { value in
                    AxisValueLabel()
                    AxisGridLine()
                }
            }
            .chartXAxis {
                AxisMarks { value in
                    AxisValueLabel()
                }
            }
            .frame(height: 220)
        }
    }
}
```

Key patterns:
- `Identifiable` + selection via `id` enables focused state without binding arrays
- Overlay with `Rectangle().fill(.clear)` + `.contentShape(Rectangle())` ensures tappability over Bars
- Use `.foregroundStyle` conditional to drive selection highlight visually

## Time Series Line Chart with Scrollable Axes

```swift
struct MetricPoint: Identifiable, PlottableValue {
    let id: String
    let timestamp: Date
    let cpu: Double
}

struct CPUChartView: View {
    @State private var scrollPosition: Date = Calendar.current.date(byAdding: .day, value: -30, to: Date()) ?? Date()
    @State private var visibleDomain: ClosedRange<Date> = {
        let now = Date()
        let start = Calendar.current.date(byAdding: .day, value: -180, to: now)!
        return start...now
    }()

    let fullRange: ClosedRange<Date>
    let data: [MetricPoint]

    var body: some View {
        Chart {   // data passed in via Chart initializer
            ForEach(data) { point in
                LineMark(
                    x: .value("Time", point.timestamp),
                    y: .value("CPU %", point.cpu)
                )
                .interpolationMethod(.catmullRom)
                .foregroundStyle(Color.accentColor)

                AreaMark(
                    x: .value("Time", point.timestamp),
                    y: .value("CPU %", point.cpu)
                )
                .interpolationMethod(.catmullRom)
                .foregroundStyle(Color.accentColor.opacity(0.1))
            }

            RuleMark(y: .value("Threshold", 80))
                .foregroundStyle(.red)
                .lineStyle(StrokeStyle(lineWidth: 1, dash: [4, 4]))
        }
        .chartScrollableAxes(.horizontal)
        .chartXScale(domain: visibleDomain)
        .chartScrollPosition(x: $scrollPosition)
        .chartScrollTargetBehavior(.paging(model: .calendar(unit: .day)))
        .chartXAxis {
            AxisMarks(values: .stride(by: .month)) { value in
                AxisGridLine()
                AxisTick()
                AxisValueLabel(format: .dateTime.month(.abbreviated))
            }
        }
        .chartYAxis {
            AxisMarks(position: .leading) { value in
                AxisValueLabel {
                    if let d = value.as(Double.self) {
                        Text("\(Int(d))%")
                    }
                }
                AxisGridLine()
            }
        }
        .frame(height: 250)
    }
}
```

Key patterns:
- `AreaMark` must precede `LineMark` so the line renders on top of the fill
- `.chartScrollTargetBehavior(.paging(model:))` snaps to calendar-aligned units
- Temporal domain uses `ClosedRange<Date>` for the scrollable window
- `chartScrollPosition` binding drives visible offset programmatically
- Downsample data externally before feeding Chart: one point per pixel column at target resolution

## Custom Annotation Pattern

```swift
struct AnnotationChartView: View {
    let data: [DataPoint]
    let annotatedDate: Date
    let annotatedValue: Double
    let note: String

    var body: some View {
        Chart(data) { point in
            LineMark(x: .value("Date", point.date),
                     y: .value("Value", point.value))
                .interpolationMethod(.catmullRom)
        }
        .chartOverlay { proxy in
            GeometryReader { geo in
                guard let plotArea = proxy.plotAreaRect,
                      let xPos = proxy.position(forX: annotatedDate),
                      let yPos = proxy.position(forY: annotatedValue) else {
                    return EmptyView()
                }

                let annotationX = xPos
                let annotationY = yPos - 20   // offset above the point

                VStack(spacing: 2) {
                    Text(note)
                        .font(.caption2.bold())
                        .padding(.horizontal, 6)
                        .padding(.vertical, 2)
                        .background(.regularMaterial, in: RoundedRectangle(cornerRadius: 3))
                    Rectangle()
                        .fill(Color.secondary)
                        .frame(width: 1, height: 14)
                }
                .position(x: annotationX, y: annotationY)
                .clipped()
            }
        }
        .frame(height: 200)
        .clipped()   // prevents annotation from escaping frame
    }
}
```

Key patterns:
- `proxy.position(forX:)` / `position(forY:)` are the only safe pixel-map from data space
- Always guard against nil; proxy returns nil during layout passes
- `.clipped()` on the containing frame prevents annotation overflow
- Use `.opaque` background on annotation badges over light/dark material — `Color.primary` text on `.regularMaterial` background adapts automatically
- For multiple annotations, iterate over `[Annotation]` and render each inside a single `chartOverlay`

## Print-Ready Chart Rendering

```swift
struct PrintableChart {
    let data: [DataPoint]

    var chartView: some View {
        Chart(data) { point in
            BarMark(x: .value("Day", point.day),
                    y: .value("Sales", point.sales))
                .foregroundStyle(Color.accentColor)
        }
        .animation(nil, value: data)   // disable animation for rendering
        .chartPlotStyle { plotArea in
            plotArea.background(.background.opacity(0.01))   // non-transparent for print
        }
    }

    func renderToData(scale: CGFloat = 3.0) -> Data? {
        let renderer = ImageRenderer(content: chartView)
        renderer.scale = scale

        // macOS: convert to PNG data
        guard let nsImage = renderer.nsImage,
              let tiffData = nsImage.tiffRepresentation,
              let bitmap = NSImage(data: tiffData),
              let cgImage = bitmap.cgImage(forProposedRect: nil, context: nil, hints: nil) else {
            return nil
        }

        let rep = NSBitmapImageRep(cgImage: cgImage)
        return rep.representation(using: .png, properties: [:])
    }

    func exportToPDF(size: NSSize = NSSize(width: 576, height: 756)) -> Data? {
        let renderer = ImageRenderer(content: chartView)
        renderer.scale = 3.0

        var box = CGRect(origin: .zero, size: size)
        return renderer.cgImage?.pdfData(in: &box)
    }
}

// Usage:
let chart = PrintableChart(data: myData)
let pngData = chart.renderToData(scale: 3.0)
try? pngData?.write(to: URL(fileURLWithPath: "/tmp/chart.png"))
```

Key patterns:
- `.animation(nil)` before `ImageRenderer` prevents partial-frame captures
- `scale: 3.0` matches Retina print density; 1.0 for web export
- `.chartPlotStyle` ensures non-transparent background for PDF embedding
- Avoid `.blur`, `.shadow`, and `.opacity` on marks — they rasterize poorly in PDF
- `NSPrintOperation` wrapping a hosting view also works for native print panels:
  ```swift
  let hosting = NSHostingController(rootView: chart.chartView)
  hosting.view.frame = NSRect(x: 0, y: 0, width: 576, height: 400)
  let printInfo = NSPrintInfo.shared
  printInfo.topMargin = 36
  printInfo.bottomMargin = 36
  printInfo.leftMargin = 36
  printInfo.rightMargin = 36
  let op = NSPrintOperation(view: hosting.view, printInfo: printInfo)
  op.showsPrintPanel = true
  op.run()
  ```

## Accessible Chart with Chart Descriptor

Since `.accessibilityChartDescriptor` is not available on macOS (as of macOS 15), provide an equivalent experience via a parallel accessible representation:

```swift
struct AccessibleChartView: View {
    let data: [DataPoint]
    private var summary: String {
        let values = data.map(\.value)
        let max = values.max() ?? 0
        let min = values.min() ?? 0
        return "Chart showing \(data.count) data points from \(Int(min)) to \(Int(max))"
    }

    var body: some View {
        VStack(alignment: .leading, spacing: 0) {
            // Visual chart
            Chart(data) { point in
                BarMark(x: .value("Date", point.date, unit: .day),
                        y: .value("Value", point.value))
                    .foregroundStyle(Color.accentColor)
            }
            .chartXAxis(.hidden)   // data list provides the X labels
            .chartYAxis(.hidden)
            .accessibilityElement(children: .ignore)
            .accessibilityLabel(summary)
            .accessibilityHint("Data table available below")

            // Accessible data table (revealed automatically by VoiceOver)
            ScrollView {
                VStack(alignment: .leading, spacing: 4) {
                    ForEach(data) { point in
                        HStack(spacing: 12) {
                            Text(point.date, format: .dateTime.month(.abbreviated).day())
                                .monospacedDigit()
                            Spacer()
                            Text("\(point.value, specifier: "%.1f")")
                                .monospacedDigit()
                        }
                        .accessibilityElement(children: .combine)
                        .accessibilityLabel("\(point.date, format: .dateTime.month(.wide).day().year()): \(point.value, specifier: "%.1f") units")
                    }
                }
                .font(.footnote)
                .padding(.horizontal)
            }
            .frame(maxHeight: 120)
        }
    }
}

// For apps requiring more VoiceOver navigation, attach an 
// accessibilityAction(.magicTap) to the Chart to jump to the table:
.chartOverlay { _ in EmptyView() }
.accessibilityAction(.magicTap) {
    // Focus the data table via accessibility notification
    NSAccessibility.post(
        element: dataTableAccessibilityElement,
        notification: NSAccessibilityNotificationName.focusedUIElementChanged
    )
}
```

Key patterns:
- `accessibilityElement(children: .ignore)` suppresses individual mark announcements — the summary becomes the only chart announcement
- Parallel `List` or `Table` satisfies VoiceOver data exploration without custom descriptor
- `.accessibilityAction(.magicTap)` (two-finger double-tap) is the macOS standard for "show data" gesture
- Use `.monospacedDigit()` for number columns so speech synthesis reads digits individually
- If you must build a custom descriptor path, compute `AXDataSeriesDescriptor` via AppKit's `NSAccessibility` protocol with a custom `accessibilityChildrenInNavigationOrder` on the chart view — significantly more code, only needed for strict compliance audits
