---
name: swift-charts-macos
description: "Swift Charts on macOS — all chart types (BarMark, LineMark, PointMark, AreaMark, RectangleMark, RuleMark), ChartProxy, selection, scrollableAxes, chartOverlay, annotations, dark mode colors, print-ready chart rendering, accessibility for charts. Use when creating data visualizations with Swift Charts on macOS."
sources: https://developer.apple.com/documentation/charts, https://developer.apple.com/videos/play/wwdc2022/10136/
researched: 2026-06-06
platform: macOS 26+
---

# Swift Charts on macOS

## Contents

- Chart Types & Core Marks
- Data & Encoding
- Selection & Interaction
- Annotations & Overlay
- macOS-Specific Patterns
- Accessibility
- Styling
- Constraints & Gotchas
- Common Mistakes
- Review Checklist
- References

## Chart Types — BarMark (vertical/horizontal), LineMark, PointMark, AreaMark, RectangleMark, RuleMark (reference lines), Plot (custom)

Swift Charts on macOS reuses the same Chart view and mark types as iOS, but window lifecycle and rendering context differ. Every chart value conforms to `Chart` protocol wrapping a container of `PlottableValue` elements.

- **BarMark** — Use `.bar(width: Ratio or .fixed(.bar))` with `x:` and `y:` encodings. Horizontal bars swap the axes: `x: .value("Count", data.count)`, `y: .value("Category", data.category)`. Stacked bars use `.foregroundStyle(by:)` with a second `.position(by:)` to stack.
- **LineMark** — Connects data points. `.interpolationMethod(.catmullRom)` gives smooth curves; `.linear` for straight segments. Essential for time series.
- **PointMark** — Individual dots. Combine with LineMark for scatter-with-trend visuals. `.symbol(.circle)` or any `SymbolShape`.
- **AreaMark** — Filled region under a LineMark. Use `.opacity(0.3)` so underlying marks stay visible. Layer AreaMark before LineMark for correct z-ordering.
- **RectangleMark** — Arbitrary rectangles; useful as highlight bands. Encode `xStart:`/`xEnd:` and `yStart:`/`yEnd:` for reference regions.
- **RuleMark** — Thin horizontal/vertical reference lines. `y: .value("Target", 42)` draws a horizontal line; swap `x:` for vertical. Ideal for thresholds and averages.
- **Plot (Swift Charts 2+)** — Custom drawing via `Plot { ... }` with SwiftUI `Canvas`. Full control but bypasses proxy annotations; pass data in as a binding.

Every mark supports `.opacity`, `.cornerRadius` (BarMark), `.lineStyle` (LineMark), `.symbolSize`, `.blur`, `.clipShape`.

## Data & Encoding — x/y encoding, color/symbol/angle/size encoding, series with .foregroundStyle(by:)

Encoding maps data to visual channels:

```swift
Chart(data) { element in
    BarMark(
        x: .value("Month", element.month),
        y: .value("Revenue", element.revenue)
    )
    .foregroundStyle(by: .value("Region", element.region))
    .symbol(by: .value("Region", element.region))   // for PointMark / LineMark
    .opacity(0.8)
}
```

Encoding options per mark axis:
- **quantitative** — numeric values (Int, Double, Date)
- **nominal** — categorical strings
- **temporal** — Date, automatically handles axis calendar units
- **color** — `.foregroundStyle(by:)` maps a PlottableValue to the default color palette
- **size** — `.symbolSize(by:)` maps numeric values to symbol area
- **angle** — for pie/donut via `.sector` marks

Key patterns:
```swift
// Time-based x-axis
x: .value("Date", element.timestamp, unit: .day)

// Secondary y-axis alignment
.chartYAxis(.automatic)   // default; add AxisMarks for customization

// Series grouping — omit .foregroundStyle(by:) for same-color; enable for multi-series
```

## Selection — chartSelection with .chartAngleSelection/.chartScrollableAxes selection, single/multi selection

Selection requires a `@State` binding and differs by mark type.

```swift
@State private var selectedAngle: Double?

Chart(data) { element in
    SectorMark(
        angle: .value("Sales", element.sales),
        innerRadius: .ratio(0.5)
    )
}
.chartAngleSelection(value: $selectedAngle)
```

For cartesian marks (BarMark, LineMark), selection is driven by the x-axis value or index with `.chartScrollableAxes`:

```swift
@State private var scrollPosition: Double = 0

Chart(data) { element in
    LineMark(
        x: .value("Year", element.year),
        y: .value("Value", element.value)
    )
}
.chartScrollableAxes(.horizontal)
.chartXScale(domain: 2018...2026)

// Use ChartProxy for hit-testing
.chartOverlay { proxy in
    Rectangle()
        .fill(.clear)
        .onTapGesture { location in
            if let value = proxy.value(atX: location.x, as: Double.self) {
                // value is the plotted x-coordinate
            }
        }
}
```

**Single vs. multi-selection**: Swift Charts does not provide a native multi-selection binding. Store a `Set` of identifiers and drive highlight visuals via `.foregroundStyle` conditional logic.

```swift
var body: some View {
    Chart(data) { element in
        BarMark(...)
            .foregroundStyle(selection.contains(element.id)
                             ? Color.accentColor
                             : Color.gray)
    }
}
```

`ChartProxy.value(atX:as:)` is the primary lookup method; returns the plotted value at a screen position. Use inside `.chartGesture(.selection(value:))` or `.chartOverlay`.

## Interaction — .chartScrollableAxes, .chartGesture, .chartProxy, zooming and panning

Scrollable axes make large datasets navigable. Pair with `.chartXScale` or `.chartYScale` to define the visible window:

```swift
Chart(bigDataset) { point in
    LineMark(x: .value("Time", point.timestamp),
             y: .value("CPU", point.cpu))
}
.chartScrollableAxes(.horizontal)
.chartXScale(domain: startDate...endDate)
.chartScrollPosition(x: $scrollPosition)
```

`.chartScrollTargetBehavior(.paging)` snaps to logical units.

`.chartProxy` enables pixel-to-data-space conversion — essential for overlays and hover tracking:

```swift
.chartOverlay { proxy in
    GeometryReader { geo in
        if let x = proxy.plotAreaRect {
            let pixelX = x.minX
            let date = proxy.value(atX: pixelX, as: Date.self)
            // render annotation at pixelX
        }
    }
}
```

**Zooming** — Use `.chartXScale(domain: scaleDomain)` with a `@State` binding that mutates via drag gesture. Pinch-to-zoom needs an overlay intercepting gestures.

**Panning** — `.chartScrollableAxes` provides panning automatically when paired with `.chartScrollPosition`.

## Annotations — ChartProxy for annotation positioning, RuleMark for reference lines, RectangleMark for highlight regions

Reference lines via RuleMark:

```swift
RuleMark(y: .value("Threshold", 99.5))
    .foregroundStyle(.red)
    .lineStyle(StrokeStyle(lineWidth: 1, dash: [5]))
```

Highlight bands via RectangleMark:

```swift
RectangleMark(
    xStart: .value("Start", range.lowerBound),
    xEnd: .value("End", range.upperBound),
    yStart: .value("Bottom", 0)
)
.foregroundStyle(.blue.opacity(0.08))
```

Custom annotation positioned with ChartProxy:

```swift
.chartOverlay { proxy in
    if let annotationX = proxy.position(forX: targetDate) {
        VStack(alignment: .leading) {
            Text("Release \(version)")
                .font(.caption.bold())
            Text("\(value, specifier: "%.1f")ms")
        }
        .padding(6)
        .background(.regularMaterial, in: RoundedRectangle(cornerRadius: 4))
        .position(x: annotationX, y: plotArea.midY)
    }
}
```

Always guard `position(forX:)` and `value(atX:)` — return `nil` when the coordinate is outside the plot area.

## Chart Overlay — chartOverlay for custom SwiftUI elements on top of chart, using ChartProxy for positioning

`.chartOverlay` is the primary bridge between data space and SwiftUI views:

```swift
.chartOverlay { proxy in
    GeometryReader { geo in
        if proxy.plotAreaRect != nil {
            // Custom drawing here
            // Use proxy.position(forX:)/proxy.position(forY:)
            // and geo.size for layout
        }
    }
}
```

Common uses:
- Floating tooltips on hover
- Custom legend positioning outside default space
- Bracket annotations spanning multiple marks
- Crosshair cursor for precise data reading
- Context menu anchors

The overlay view sits on top of the chart in z-order; it does not clip to the plot area. Clip manually with `.clipSelectionEnabled(true)` to bound hit-testing.

## macOS-Specific — chart in NSWindow/panel context, dark mode color adaptation, print-ready rendering

**Window Context** — Charts in macOS often live in non-standard containers: sidebars, panels, inspector views. Prefer `GeometryReader` wrappers for responsive sizing instead of fixed frames.

```swift
Chart(data) { ... }
    .frame(minHeight: 200)
```

**Dark Mode** — Swift Charts uses automatic color adaptation. Define semantic colors for best results:

```swift
.foregroundStyle(Color.accentColor)          // auto-adapts
.foregroundStyle(.blue)                       // static — avoid
.foregroundStyle(.primary)                    // foreground auto-adapt
.foregroundStyle(.secondary)                  // de-emphasized auto-adapt
```

Use `Color(nsColor:)` when pulling from AppKit-bridged palettes. Test with `NSApp.effectiveAppearance.name` in `preferredColorScheme(_:)`.

**Print-Ready Rendering** — For `NSPrintOperation`, convert the chart to PDF/PNG via `ImageRenderer`:

```swift
let renderer = ImageRenderer(content: chartView)
renderer.scale = 3.0    // Retina-quality output
if let png = renderer.nsImage {
    // Write to NSPasteboard or save file
}
```

Best practices for print:
- Use vector-safe colors (not opacity-heavy gradients)
- Increase `.symbolSize` for printed dots
- Disable animations before `ImageRenderer` call: `chart.animation(nil, value: ())`
- Prefer `.opacity` over `.blur` for print-compatible rendering
- Render at `scale: 3.0` minimum for Retina print

## Accessibility — accessibilityLabel for data points, accessibilityValue, .accessibilityChartDescriptor

Native chart accessibility via ChartUIAccessibilityElement:

```swift
.chartAccessibilityElementChildren { element in
    ZStack(alignment: .leading) {
        accessibilityLabel(Text("Month: \(element.month)"))
        accessibilityValue(Text("Value: \(element.value, specifier: "%.0f") units"))
    }
}
```

For full VoiceOver chart browsing, `.accessibilityChartDescriptor` (iOS 16+) is not yet available on macOS. Instead, provide a parallel data `List` or `Table` alongside the chart as a data alternative:

```swift
HStack {
    Chart(data) { ... }   // Visual
    Table(data) { ... }   // VoiceOver alternative
}
```

Or use `accessibilityLabel` on the `Chart` itself for a summary:

```swift
Chart(data) { ... }
    .accessibilityLabel("Bar chart showing revenue per quarter")
    .accessibilityHint("Swipe to explore data points")
```

## Styling — .foregroundStyle, .symbol, .symbolSize, .interpolationMethod, mark combinations

Mark styling matrix:

| Property | Applies To | Example |
|---|---|---|
| `.foregroundStyle` | all marks | `.foregroundStyle(by: .value("Team", t))` |
| `.symbol` | PointMark, LineMark | `.symbol(.diamond)` |
| `.symbolSize` | PointMark, LineMark | `.symbolSize(by: .value("Count", c))` |
| `.lineStyle` | LineMark, RuleMark | `.lineStyle(StrokeStyle(lineWidth: 2, dash: [5,3]))` |
| `.interpolationMethod` | LineMark | `.interpolationMethod(.monotone)` |
| `.cornerRadius` | BarMark | `.cornerRadius(4)` |
| `.opacity` | all marks | `0.6` |

Composite pattern — trend + individual points:

```swift
Chart(data) { element in
    AreaMark(x: .value("X", element.x),
             y: .value("Y", element.y))
        .interpolationMethod(.catmullRom)
        .foregroundStyle(.blue.opacity(0.2))

    LineMark(x: .value("X", element.x),
             y: .value("Y", element.y))
        .interpolationMethod(.catmullRom)

    PointMark(x: .value("X", element.x),
              y: .value("Y", element.y))
        .symbolSize(40)
}
```

Use `.symbol(by:)` and `.foregroundStyle(by:)` together for categorical series encoding.

## Constraints & Gotchas — large dataset performance, dark mode, selection state management

1. **Large datasets** — Charts re-renders the full mark set on every data change. Pre-aggregate or downsample to <1000 marks for smooth scrolling. For time series >10k points, provide a pre-reduced sample for the visible window.
2. **State reset** — `selection` state resets when providing a new `id()` to the `Chart`. Pass stable data IDs or use `.animation(nil, value: selection)`.
3. **Dark mode flicker** — Avoid `Color.clear` as foreground; use `Color.accentColor.opacity(0)` instead.
4. **Axis labels overlapping** — Set `.chartXAxis(.visible)` and provide `AxisMarks` with desired stride; the default may produce unreadable labels for dense data.
5. **Animation drops frames** — Linear datasets animate smoothly; categorical datasets with heterogeneous types may stutter. Disable with `.animation(.none)` for programmatic updates.
6. **Proxy before layout** — `chartOverlay` receives nil proxy values during initial layout pass. Always unwrap with `if let`.
7. **ScrollView conflict** — Do not embed Chart inside a `ScrollView`; use `.chartScrollableAxes` instead.
8. **Date encoding density** — Temporal axes use automatic calendar units. Override via `unit: .day` or `unit: .month` for predictable spacing.
9. **ImageRenderer lifetime** — `ImageRenderer` captures a static snapshot; it does not update live. Re-render before every print or export.
10. **Pie chart performance** — Sector marks with many slices (>30) render slowly. Prefer BarMark for high-cardinality categorical data.

## Common Mistakes

- **Using numeric literals as colors** — `Color(red: 0.1, green: 0.8, blue: 0.2)` ignores dark mode. Prefer semantic colors.
- **Encoding a non-Plottable type** — Only `String`, `Int`, `Double`, `Date`, and types conforming to `Plottable` work in `.value`. Convert or make custom types conform.
- **Placing AreaMark after LineMark** — fills over the line path; always declare AreaMark first.
- **Missing `@State` for selection** — `.chartAngleSelection(value: $x)` requires a binding, not a value.
- **Ignoring `plotAreaRect`** — overlay positioning fails when `plotAreaRect` is nil; the chart hasn't laid out yet.
- **Using `ScrollView` around Chart** — breaks gesture handling; use `.chartScrollableAxes`.
- **Heavy shadow on marks** — shadows don't render in print; remove before `ImageRenderer`.
- **Not setting `chartPlotStyle`** — plot background may default to transparent; set explicitly for print.

## Review Checklist

- [ ] Axis titles and units labeled
- [ ] Semantic (adaptive) colors used, not static hex
- [ ] Accessibility summary provided via `accessibilityLabel`
- [ ] Selection state captured in `@State`
- [ ] `chartOverlay` guards proxy with `if let`
- [ ] Print rendering uses `ImageRenderer` at `scale >= 3.0`
- [ ] Large datasets pre-aggregated
- [ ] No `ScrollView` wrapping `Chart`
- [ ] `.chartScrollableAxes` ranges set before scroll position
- [ ] Dark mode tested via `preferredColorScheme` toggle

## References

- [Apple Swift Charts Documentation](https://developer.apple.com/documentation/charts)
- [WWDC 2022 Visualize data with Swift Charts](https://developer.apple.com/videos/play/wwdc2022/10136/)
- [WWDC 2022 Design App Experiences with Swift Charts](https://developer.apple.com/videos/play/wwdc2022/10137/)
- [Swift Charts Tutorials](https://developer.apple.com/tutorials/charts)

## Research Addendum — 2026 Updates

### macOS 26 Selection & Scrollable Enhancements

- **`.chartXSelection(value:)` / `.chartYSelection(value:)`** — New in iOS 17+/macOS 14+, these bind user drag gestures to selection values. Use for precise data point selection via drag rather than only tap.
- **`.chartAngleSelection(value:)`** — For SectorMark (pie/donut), binds rotation angle to selection. Works the same on macOS as iOS.
- **`chartXVisibleDomain(length:)`** — Controls the visible window size when `.chartScrollableAxes(.horizontal)` is enabled. Pair with `chartScrollPosition(x:)` for programmatic control.
- **`.chartScrollTargetBehavior(.paging)` / `.alignedToEachUnit(Calendar.Component)`** — Enables snapping to logical units (e.g., day, month) on scroll.
- **Known issue on macOS**: Horizontal scrollbar mouse interaction is inconsistent; trackpad two-finger scrolling works reliably. Mouse-driven scrollbar dragging may not respond on some hardware. Radar this if hit.

### Dark Mode & Accessibility (AXChart)

- **AXChart protocol** — Implement `accessibilityChartDescriptor` with an `AXChartDescriptorRepresentable` struct to provide full VoiceOver chart browsing, including audio graphs and a Chart Detail modal where users listen to data points at their own pace.
- **Swift Charts auto-builds accessibility tree** — Per data point, use `.value("Label", ...)` encoding so VoiceOver reads descriptions naturally.
- **`AXChartDescriptor` construction** — Provide title, summary, x/y axes (`AXNumericAxisDescriptor` or `AXCategoricalAxisDescriptor`), and data series. VoiceOver reads axis titles then enumerates data points.
- **High contrast design** — Pair `.foregroundStyle(by:)` with `.symbol(by:)` so color-blind users can distinguish series. Test with Color Filters in Accessibility Inspector.
- **Dark mode** — Use semantic colors (`Color.accentColor`, `.primary`, `.secondary`). Swift Charts auto-adapts to `preferredColorScheme`. Avoid static `Color(red:green:blue:)` which ignores dark mode.
- **Sonification** — AXChart supports audio graphs on macOS: VoiceOver users hear data values as pitch variations. No extra code needed if you provide a proper `AXChartDescriptor`.

### Print & PDF Rendering

- **Known scrolling bug when capturing**: `ImageRenderer` fails to capture chart content when `.chartScrollableAxes(.horizontal)` is enabled in the view. Disable scrolling before capture or render at the desired visible domain programmatically.
  ```swift
  // Workaround: render without scrollable axes, set fixed visible domain
  let renderer = ImageRenderer(content: chartView.scaleEffect(x: 1, y: 1, anchor: .center))
  renderer.scale = 3.0
  ```
- **PDF path for printing** — Render to PDF via ImageRenderer then create `NSPrintOperation`:
  ```swift
  let renderer = ImageRenderer(content: chartView.frame(width: 1200, height: 600))
  renderer.render { size, context in
      let data = NSMutableData()
      var mediaBox = CGRect(origin: .zero, size: size)
      guard let consumer = CGDataConsumer(data: data as CFMutableData),
            let pdfContext = CGContext(consumer: consumer, &mediaBox, nil) else { return }
      context(pdfContext)
      // Save or print data
  }
  ```
- **Print entitlement** — Add `com.apple.security.print` entitlement for AppKit printing workflows.
- **Scroll position for print** — If you need to print the full dataset (not just visible), temporarily set `.chartXScale(domain:)` to your full data range, remove `.chartScrollableAxes`, render, then restore.
