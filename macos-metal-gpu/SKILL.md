---
name: macos-metal-gpu
description: "Metal GPU programming for macOS: MTKView integration via NSViewRepresentable, render and compute pipelines, MTLHeap resource management, GPU selection and family feature detection, synchronization with fences and events, CAMetalLayer direct rendering, and MSL shader conventions. Use when building GPU-accelerated rendering, compute workloads, or custom Metal views in a macOS app."
sources: https://medium.com/@giikwebdeveloper/metal-4-for-swiftui-33209c1134e9, https://medium.com/@giikwebdeveloper/metal-view-for-swiftui-93f5f78ec36a, https://apple-docs.everest.mt/docs/metal/using-metal-to-draw-a-view's-contents/, https://developer.apple.com/forums/thread/119112, https://developer.apple.com/documentation/metal/mtlheap, https://developer.apple.com/documentation/quartzcore/cametallayer, https://developer.apple.com/videos/play/tech-talks/111373/, https://developer.apple.com/documentation/metal/selecting-device-objects-for-graphics-rendering/
researched: 2026-06-06
platform: macOS 26+
---

# macOS Metal & GPU

Metal GPU programming for macOS 26+ with Swift 6.3. Covers MTKView integration in SwiftUI, render and compute pipeline construction, GPU selection and family capabilities, heap-based resource management, synchronization primitives, CAMetalLayer direct rendering, and MSL shader conventions. For SwiftUI state management patterns, see `swiftui-patterns`. For NSViewRepresentable basics, see `swiftui-appkit-interop`. For AppKit layer-backing, see `appkit-foundations`.

See [references/metal-macos-patterns.md](references/metal-macos-patterns.md) for complete MSL shader examples, GPU family feature table, compute pipeline recipes, and synchronization patterns.

## Contents

- [MTKView in SwiftUI](#mtkview-in-swiftui)
- [GPU Selection](#gpu-selection)
- [GPU Family Features](#gpu-family-features)
- [Render Pipeline](#render-pipeline)
- [Compute Pipeline](#compute-pipeline)
- [MTLHeap Resource Management](#mtlheap-resource-management)
- [Synchronization](#synchronization)
- [CAMetalLayer Direct Path](#cametallayer-direct-path)
- [MSL Conventions](#msl-conventions)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## MTKView in SwiftUI

Wrap `MTKView` as `NSViewRepresentable` to embed Metal rendering in a SwiftUI view hierarchy. `MTKView` manages the drawable, frame timing, and display link for you.

```swift
import SwiftUI
import MetalKit

struct MetalCanvasView: NSViewRepresentable {
    var renderer: MetalRenderer

    func makeNSView(context: Context) -> MTKView {
        let view = MTKView()
        view.device = renderer.device
        view.colorPixelFormat = .bgra8Unorm_srgb
        view.depthStencilPixelFormat = .depth32Float
        view.clearColor = MTLClearColor(red: 0, green: 0, blue: 0, alpha: 1)
        view.preferredFramesPerSecond = 60
        view.isPaused = false
        view.enableSetNeedsDisplay = false  // use display-link-driven draw loop
        view.delegate = renderer
        return view
    }

    func updateNSView(_ nsView: MTKView, context: Context) {
        // Push state changes (e.g., new drawable size, renderer swap).
    }
}
```

**Key properties:**

| Property | Purpose |
|----------|---------|
| `preferredFramesPerSecond` | Target frame rate. Set to 0 for uncapped (matches display refresh). |
| `isPaused` | When `true`, the display link stops. Use for static content or backgrounded apps. |
| `enableSetNeedsDisplay` | When `true`, you call `setNeedsDisplay()` manually instead of using the display link. Default `false` — the view drives its own draw loop. |
| `autoResizeDrawable` | When `true` (default), drawable size tracks the view's bounds. Set `false` for fixed-resolution rendering. |
| `drawableSize` | The actual drawable dimensions in pixels. Use this for viewport setup, not `bounds`. |

The Coordinator pattern is the recommended way to bridge `MTKViewDelegate` into SwiftUI:

```swift
struct Metal4View: NSViewRepresentable {
    func makeCoordinator() -> Renderer {
        Renderer(device: MTLCreateSystemDefaultDevice()!)
    }

    func makeNSView(context: Context) -> MTKView {
        let view = MTKView()
        view.device = context.coordinator.device
        view.delegate = context.coordinator
        return view
    }

    func updateNSView(_ nsView: MTKView, context: Context) { }
}
```

**Display-link vs manual:** The default mode (`enableSetNeedsDisplay = false`) uses a `CVDisplayLink` that fires at the display's refresh rate. This is the correct choice for real-time rendering. Switch to `enableSetNeedsDisplay = true` only for content that updates on demand (e.g., a static visualization that changes when the user interacts).

**Drawable size vs bounds:** `drawableSize` is in pixels (accounting for the view's `contentsScale`). `bounds` is in points. Always use `drawableSize` for render target sizing and viewport configuration.

**SwiftUI layout note:** SwiftUI controls the layout of the wrapped `NSView`. Do not manually set `frame`/`bounds` on the MTKView from your code. Implement `sizeThatFits(_:nsView:context:)` for custom sizing if needed.

## GPU Selection

`MTLCopyAllDevices()` returns all available GPUs. On Apple Silicon Macs, this is typically a single unified GPU. On Intel Macs with discrete GPUs, multiple devices may appear.

```swift
let allDevices = MTLCopyAllDevices()

// Prefer the low-power (integrated) GPU for lightweight work
let lowPower = allDevices.first { $0.isLowPower }

// Prefer the high-performance GPU for heavy rendering
let highPerf = allDevices.first { !$0.isLowPower && !$0.isRemovable }

// Detect eGPU
let eGPU = allDevices.first { $0.isRemovable }

// Headless GPU (no display output — compute only)
let headless = allDevices.first { $0.isHeadless }
```

**Device properties:**

| Property | Meaning |
|----------|---------|
| `isLowPower` | Integrated GPU. Lower performance, lower power. |
| `isRemovable` | eGPU connected via Thunderbolt. May disconnect at runtime. |
| `isHeadless` | No display output. Suitable for compute-only workloads. |
| `hasUnifiedMemory` | CPU and GPU share the same memory pool (Apple Silicon). |
| `recommendedMaxWorkingSetSize` | Upper bound for GPU memory allocation. Respect this to avoid paging. |
| `maxBufferLength` | Maximum size of a single buffer in bytes. |
| `supportsFamily(_:)` | Check GPU family for feature availability. |

**Apple Silicon (2025+):** On Apple Silicon Macs, `MTLCreateSystemDefaultDevice()` returns the single unified GPU. There is no discrete/integrated split — all GPU work runs on the same chip. `isLowPower`, `isRemovable`, and `isHeadless` are always `false`. `hasUnifiedMemory` is always `true`.

**Multi-GPU (Intel/Studio Macs):** When multiple GPUs are present, choose the optimal device per display. Use `CGDirectDisplayID` to identify which GPU drives a given monitor and render with that device to avoid cross-GPU drawable transfers. Observe `NSApplicationDidChangeScreenParametersNotification` to handle display reconfiguration at runtime, and use `MTLCopyAllDevicesWithObserver` for hot-plug eGPU support.

**eGPU handling:** When `isRemovable` is `true`, the device can disappear at any time. Observe `MTLDeviceWasAddedNotification` and `MTLDeviceWasRemovedNotification` to handle hot-plug. Always fall back to the integrated GPU if the eGPU disconnects.

**Unified memory (Apple Silicon):** `hasUnifiedMemory` is `true`. CPU and GPU access the same physical memory, so `StorageModeShared` is the default and most efficient mode. `StorageModeManaged` (explicit CPU/GPU sync) and `StorageModePrivate` (GPU-only) are still available but rarely needed for buffers.

## GPU Family Features

GPU families define feature availability. Check with `supportsFamily(_:)` before using advanced features.

```swift
let device = MTLCreateSystemDefaultDevice()!

if device.supportsFamily(.apple9) {
    // Mesh shaders, hardware ray tracing, advanced tile shaders
} else if device.supportsFamily(.apple8) {
    // Tile shaders, argument buffers, function pointers
} else if device.supportsFamily(.apple7) {
    // Function pointers, ray tracing (software), sparse textures
}
```

**Family hierarchy:** Each family is a superset of the previous. `.apple9` includes all `.apple8` features, which includes all `.apple7` features, and so on. Check from newest to oldest and fall back gracefully.

**Key family capabilities:**

| Family | Key Features |
|--------|-------------|
| `.apple7` | Function pointers, ray tracing (software), sparse textures, sampler feedback |
| `.apple8` | Tile shaders, argument buffers tier 2, function pointers in all stages |
| `.apple9` | Mesh shaders, hardware ray tracing, advanced tile shaders, primitive motion blur |
| `.mac2` | macOS-specific: argument buffers, tessellation, layered rendering |

**macOS-specific:** `.mac2` is the baseline for macOS Metal. All Apple Silicon Macs support at least `.apple7` + `.mac2`. When targeting both iOS and macOS, check both the Apple family and the macOS family.

## Render Pipeline

Build a render pipeline descriptor with vertex/fragment functions and color attachment configuration.

```swift
let descriptor = MTLRenderPipelineDescriptor()
descriptor.vertexFunction = library.makeFunction(name: "vertex_main")
descriptor.fragmentFunction = library.makeFunction(name: "fragment_main")
descriptor.colorAttachments[0].pixelFormat = .bgra8Unorm_srgb
descriptor.colorAttachments[0].isBlendingEnabled = true
descriptor.colorAttachments[0].rgbBlendOperation = .add
descriptor.colorAttachments[0].alphaBlendOperation = .add
descriptor.colorAttachments[0].sourceRGBBlendFactor = .sourceAlpha
descriptor.colorAttachments[0].sourceAlphaBlendFactor = .sourceAlpha
descriptor.colorAttachments[0].destinationRGBBlendFactor = .oneMinusSourceAlpha
descriptor.colorAttachments[0].destinationAlphaBlendFactor = .oneMinusSourceAlpha
descriptor.depthAttachmentPixelFormat = .depth32Float

let state = try device.makeRenderPipelineState(descriptor: descriptor)
```

**Color attachment blending:** The example above configures standard alpha blending. Disable blending (`isBlendingEnabled = false`) for opaque passes to avoid unnecessary GPU work. For additive blending (particles, glow), use `.one` for both source and destination factors.

**Depth/stencil:** Set `depthAttachmentPixelFormat` to `.depth32Float` for standard depth testing. Use `.depth32Float_stencil8` when you need stencil operations. The pixel format must match the `MTKView`'s `depthStencilPixelFormat`.

**Library creation:** Compile shaders at build time using the Metal compiler (`metal -c shader.msl -o shader.air`) and load with `makeLibrary(bundle:)`, or compile from source at runtime with `makeLibrary(source:options:)`. Runtime compilation is slower and should be cached.

## Compute Pipeline

Compute pipelines dispatch threadgroups across the GPU. Threadgroup sizing is critical for performance on Apple Silicon.

```swift
let function = library.makeFunction(name: "compute_kernel")!
let state = try device.makeComputePipelineState(function: function)

let encoder = commandBuffer.makeComputeCommandEncoder()!
encoder.setComputePipelineState(state)

// Threadgroup size: 1D for simple kernels, 2D/3D for image/volume work
let threadsPerThreadgroup = MTLSize(width: 256, height: 1, depth: 1)

// Threadgroups: cover the full dispatch size
let threadgroups = MTLSize(
    width: (width + threadsPerThreadgroup.width - 1) / threadsPerThreadgroup.width,
    height: (height + threadsPerThreadgroup.height - 1) / threadsPerThreadgroup.height,
    depth: 1
)

encoder.dispatchThreadgroups(threadgroups, threadsPerThreadgroup: threadsPerThreadgroup)
encoder.endEncoding()
```

**Threadgroup sizing on Apple Silicon:**

- The GPU executes threadgroups in SIMD groups of 32 threads (a "warp"). Threadgroup width should be a multiple of 32.
- For 1D kernels, use 256 or 512 threads per threadgroup. 256 is a safe default; 512 may improve occupancy for compute-heavy kernels.
- For 2D image processing, use `MTLSize(width: 16, height: 16, depth: 1)` — 256 threads total, matching the GPU's execution width.
- Query `state.maxTotalThreadsPerThreadgroup` to find the hardware limit. Never exceed it.
- `state.threadExecutionWidth` gives the SIMD width (32 on Apple Silicon). Align threadgroup dimensions to this value.

**Threadgroup memory:** Use `setThreadgroupMemoryLength(_:at:)` for data shared within a threadgroup. This is on-chip memory — much faster than buffer access. Typical use: reduction operations, shared tile data for image filters.

**Dispatch best practices (2025):** Use a 3D grid of threadgroups distributed uniformly across GPU cores. Each core has its own fast, limited threadgroup memory (typically 32 KB on Apple Silicon). Use `dispatchThreads(_:threadsPerThreadgroup:)` (Xcode 15+) for simpler sizing when Metal can calculate threadgroup counts automatically:

```swift
let gridSize = MTLSize(width: width, height: height, depth: 1)
encoder.dispatchThreads(gridSize, threadsPerThreadgroup: threadsPerThreadgroup)
```

## MTLHeap Resource Management

Heaps allocate GPU memory in bulk and let you create placed resources within the heap. This reduces allocation overhead and enables aliasing (reusing the same memory for different resources at different times).

```swift
let heapDescriptor = MTLHeapDescriptor()
heapDescriptor.size = 64 * 1024 * 1024  // 64 MB
heapDescriptor.storageMode = .private     // GPU-only (Apple Silicon: still unified)
heapDescriptor.hazardTrackingMode = .tracked

let heap = try device.makeHeap(descriptor: heapDescriptor)

// Create a placed buffer within the heap
let bufferDescriptor = MTLBufferDescriptor()
bufferDescriptor.length = 1024 * 1024
bufferDescriptor.storageMode = .private
bufferDescriptor.hazardTrackingMode = .untracked

let buffer = heap.makeBuffer(descriptor: bufferDescriptor)!

// Create a placed texture within the heap
let textureDescriptor = MTLTextureDescriptor.texture2D(
    pixelFormat: .rgba16Float,
    width: 1024,
    height: 1024,
    mipmapped: false
)
textureDescriptor.storageMode = .private
textureDescriptor.hazardTrackingMode = .untracked

let texture = heap.makeTexture(descriptor: textureDescriptor)!
```

**Hazard tracking modes:**

| Mode | Behavior |
|------|----------|
| `.tracked` | Metal tracks resource access across encoders. Safe default. |
| `.untracked` | You are responsible for synchronization. Use with fences/events for maximum performance. |
| `.default` | Same as `.tracked`. |

**`MTLSizeAndAlign`:** When placing resources in a heap, each resource has alignment and size requirements. Query with `heap.makeTexture(descriptor:)` — Metal handles alignment automatically. For manual placement (advanced), use `heap.makeTexture(descriptor:offset:)` and check alignment with `device.heapTextureSizeAndAlign(descriptor:)`.

**Aliasing:** Multiple resources can share heap memory if they are not used simultaneously. Use `makeTexture(descriptor:offset:)` to place a texture at a specific offset within the heap, overlapping with another texture that is no longer in use. This is the primary performance benefit of heaps — reduced memory footprint for transient resources.

**Heap sizing:** Size heaps large enough for their resources. Query available space with `heap.maxAvailableSize(alignment:)` before sub-allocating. Keep `heap.usedSize` and `heap.currentAllocatedSize` within your memory budget. On macOS, using a small number of well-managed heaps reduces memory pressure and minimizes memory compression delays.

**Fragmentation:** Creating and deleting many sub-allocated resources can fragment heap memory. Use multiple heaps or allocate similarly sized resources in contiguous blocks to mitigate fragmentation. Keep separate heaps for aliasable resources vs non-aliasable resources (e.g., compressible depth textures and MSAA textures cannot alias).

**Sync with fences:** A single `MTLFence` can track multiple sub-allocated resources with identical synchronization requirements. Use `updateFence`/`waitForFence` to enforce ordering between compute and render passes that access heap-backed resources.

## Synchronization

Metal provides three synchronization primitives with different scopes.

### MTLFence (within a single encoder)

Fences synchronize access to resources within a single command encoder. They are the lightest-weight option.

```swift
let fence = device.makeFence()!

let computeEncoder = commandBuffer.makeComputeCommandEncoder()!
computeEncoder.setComputePipelineState(computeState)
// ... dispatch compute ...
encoder.updateFence(fence)  // GPU signals fence after this point
computeEncoder.endEncoding()

let renderEncoder = commandBuffer.makeRenderCommandEncoder(descriptor: passDescriptor)!
renderEncoder.waitForFence(fence, before: .fragment)  // Fragment stage waits
// ... draw ...
renderEncoder.endEncoding()
```

**Fence stages:** `before: .vertex` waits at the vertex stage; `before: .fragment` waits at the fragment stage. Use `.fragment` when the render pass reads compute output as a texture — the vertex stage doesn't need to wait.

### MTLEvent (cross-queue)

Events synchronize across different command queues within the same device.

```swift
let event = device.makeEvent()!

// Queue 1: signal
let computeBuffer = queue1.makeCommandBuffer()!
let computeEncoder = computeBuffer.makeComputeCommandEncoder()!
// ... compute work ...
computeEncoder.setSignaledValue(1)
computeEncoder.endEncoding()

// Queue 2: wait
let renderBuffer = queue2.makeCommandBuffer()!
let renderEncoder = renderBuffer.makeRenderCommandEncoder(descriptor: passDescriptor)!
renderEncoder.wait(for: event, value: 1)
// ... render work ...
renderEncoder.endEncoding()
```

### MTLSharedEvent (cross-process)

Shared events extend `MTLEvent` to work across process boundaries. Use for synchronizing Metal work between an app and an XPC service, or between two apps.

```swift
let sharedEvent = device.makeSharedEvent()!

// In the other process, obtain the event via handle
let otherEvent = device.makeSharedEvent(handle: sharedEvent.sharedHandle)!

// Signaling and waiting work the same as MTLEvent
```

**Fence vs Event vs SharedEvent:**

| Primitive | Scope | Overhead | Use Case |
|-----------|-------|----------|----------|
| `MTLFence` | Within encoder | Lowest | Compute-to-render within one pass |
| `MTLEvent` | Cross-queue, same device | Medium | Async compute on separate queue |
| `MTLSharedEvent` | Cross-process | Highest | App-to-XPC or app-to-app sync |

## CAMetalLayer Direct Path

When `MTKView` is insufficient — off-screen rendering, custom AppKit layer hierarchies, or direct control over the drawable — use `CAMetalLayer` directly.

```swift
final class MetalHostingView: NSView {
    var metalLayer: CAMetalLayer!
    var device: MTLDevice!
    var commandQueue: MTLCommandQueue!

    override init(frame: NSRect) {
        super.init(frame: frame)
        setupMetal()
    }

    required init?(coder: NSCoder) {
        super.init(coder: coder)
        setupMetal()
    }

    private func setupMetal() {
        wantsLayer = true
        device = MTLCreateSystemDefaultDevice()!
        commandQueue = device.makeCommandQueue()!

        metalLayer = CAMetalLayer()
        metalLayer.device = device
        metalLayer.pixelFormat = .bgra8Unorm_srgb
        metalLayer.framebufferOnly = false  // allow texture readback
        metalLayer.contentsScale = window?.backingScaleFactor ?? 1.0
        metalLayer.maximumDrawableCount = 3  // triple buffering
        layer = metalLayer
    }

    override func viewDidChangeBackingProperties() {
        super.viewDidChangeBackingProperties()
        metalLayer.contentsScale = window?.backingScaleFactor ?? 1.0
    }

    func draw() {
        guard let drawable = metalLayer.nextDrawable() else { return }

        let buffer = commandQueue.makeCommandBuffer()!
        let descriptor = MTLRenderPassDescriptor()
        descriptor.colorAttachments[0].texture = drawable.texture
        descriptor.colorAttachments[0].loadAction = .clear
        descriptor.colorAttachments[0].clearColor = MTLClearColor(red: 0, green: 0, blue: 0, alpha: 1)
        descriptor.colorAttachments[0].storeAction = .store

        let encoder = buffer.makeRenderCommandEncoder(descriptor: descriptor)!
        // ... draw commands ...
        encoder.endEncoding()

        buffer.present(drawable)
        buffer.commit()
    }
}
```

**When to use CAMetalLayer directly:**

- Off-screen rendering to a texture (no display output). Render into a drawable obtained from `nextDrawable()` without presenting to screen, or render to a standalone `MTLTexture`.
- Custom layer composition (multiple Metal layers in one view)
- Integrating with Core Animation animations on the layer
- Rendering to a specific `CGContext` or `IOSurface`

**Off-screen rendering pattern:** For pure off-screen work, render to a drawable obtained from `nextDrawable()` and read back its texture, or better yet, render directly to a private `MTLTexture` and avoid the drawable overhead entirely.

**`maximumDrawableCount`:** Set to 2 (double buffering) or 3 (triple buffering). Triple buffering improves GPU utilization at the cost of one frame of latency. Use 2 for latency-sensitive applications (VR, real-time input).

**`framebufferOnly`:** Set to `true` (default) when the drawable texture is only used as a render target. Set to `false` when you need to read the texture back (e.g., for screenshot, video encoding). `false` may use more memory.

**Contents scale:** Update `contentsScale` in `viewDidChangeBackingProperties()` to handle display changes (moving between Retina and non-Retina displays).

**Drawable lifecycle:** Drawables are reused only if not onscreen and there are no strong references. If `nextDrawable()` has no available drawable, the system waits. Release references promptly — wrap rendering in an autoreleasepool to release drawables promptly after presenting and avoid deadlocks or nil drawables.

## MSL Conventions

Metal Shading Language (MSL) is based on C++14 with GPU-specific attributes and qualifiers.

### Buffer and Texture Bindings

```metal
#include <metal_stdlib>
using namespace metal;

struct VertexIn {
    float3 position [[attribute(0)]];
    float2 texCoord [[attribute(1)]];
};

struct VertexOut {
    float4 position [[position]];
    float2 texCoord;
};

vertex VertexOut vertex_main(
    VertexIn in [[stage_in]],
    constant float4x4& modelViewProjection [[buffer(0)]],
    constant float3& lightDir [[buffer(1)]]
) {
    VertexOut out;
    out.position = modelViewProjection * float4(in.position, 1.0);
    out.texCoord = in.texCoord;
    return out;
}

fragment float4 fragment_main(
    VertexOut in [[stage_in]],
    texture2d<float> colorTexture [[texture(0)]],
    sampler textureSampler [[sampler(0)]]
) {
    float4 color = colorTexture.sample(textureSampler, in.texCoord);
    return color;
}
```

**Binding indices:** `[[buffer(n)]]`, `[[texture(n)]]`, `[[sampler(n)]]` map to the indices set in the encoder:

```swift
renderEncoder.setVertexBuffer(buffer, offset: 0, index: 0)
renderEncoder.setFragmentTexture(texture, index: 0)
renderEncoder.setFragmentSamplerState(sampler, index: 0)
```

**`[[threadgroup(n)]]`:** For compute kernels, `threadgroup` address space declares memory shared within a threadgroup:

```metal
kernel void blur(
    texture2d<float, access::read> input [[texture(0)]],
    texture2d<float, access::write> output [[texture(1)]],
    threadgroup float4* sharedMem [[threadgroup(0)]],
    uint2 gid [[thread_position_in_grid]],
    uint2 lid [[thread_position_in_threadgroup]]
) {
    // Load into threadgroup memory, synchronize, then compute
    threadgroup_barrier(mem_flags::mem_threadgroup);
}
```

### Precision Qualifiers

| Qualifier | Precision | Use Case |
|-----------|-----------|----------|
| `half` | 16-bit float | Colors, normals, UVs — most fragment shader data |
| `float` | 32-bit float | Positions, matrices, compute |
| `float2`, `float3`, `float4` | Vector forms | Packed data |

On Apple Silicon, `half` operations are as fast as `float` in many cases. Prefer `half` for fragment shader inputs (colors, texture coordinates) to reduce register pressure and improve occupancy.

### Address Spaces

| Space | Meaning |
|-------|---------|
| `device` | Read-write, persistent. Default for buffers and textures. |
| `constant` | Read-only, cached. For uniform data that doesn't change per-thread. |
| `threadgroup` | On-chip shared memory within a threadgroup. |
| `thread` | Per-thread private memory (registers/stack). |

### MSL Performance Best Practices (2025)

From Apple's Tech Talks and the MSL specification:

- **Function groups and attributes:** Use function group attributes at call sites to enable indirect function calls and unlock more shader optimizations, especially for statically linked functions. Enables the Metal compiler to perform cross-function analysis.

- **Threadgroup memory vs. device/constant:** On Apple Family 9 (M3/A17 Pro), the benefit of threadgroup memory as a software cache is reduced compared to older GPUs. If data fits in cache, reading from device/constant buffers directly may be faster than copying to threadgroup memory first. Profile on your target hardware.

- **Occupancy tuning:** Optimize thread occupancy through efficient resource usage (registers, memory). Use Metal profiling tools (updated for M3/A17 Pro) to identify lower thread occupancy bottlenecks. Occupancy management is more important on newer GPU generations.

- **Language revisions:** MSL supports language revisions from 1.x through 3.x (Metal 3.0/3.1). Metal 3 introduces mesh shaders and hardware ray tracing. Use `-std=metal3.0` or later when targeting recent Apple Silicon.

- **Max threadgroup size attribute:** Tune total threads per threadgroup via `-fmax-total-threads-per-threadgroup` compile option or `[[max_total_threads_per_threadgroup]]` attribute on compute kernels. This is important for PSO compatibility and hardware-specific tuning.

- **SIMD-group synchronization:** Use `threadgroup_barrier(mem_flags::mem_threadgroup)` for proper synchronization after writing to threadgroup memory. For SIMD-group-level sync within a threadgroup, use SIMD-group functions for finer control.

## Constraints & Gotchas

- **Main thread requirement:** `MTKView` and `CAMetalLayer` must be created and accessed from the main thread. Command buffer encoding can happen on any thread.
- **Drawable lifetime:** A `CAMetalDrawable` is valid only until the next `present()` call. Do not retain it.
- **Pipeline state caching:** `makeRenderPipelineState(descriptor:)` is expensive. Cache the result and reuse it across frames.
- **Command buffer reuse:** Never reuse a command buffer after `commit()`. Create a new one each frame.
- **Encoder nesting:** You cannot nest render encoders. Compute encoders can be nested inside a render encoder on supported GPUs.
- **Threadgroup memory limit:** Maximum threadgroup memory is 32 KB on Apple Silicon. Query `device.maxThreadgroupMemoryLength` for the exact value.
- **Buffer alignment:** Buffers bound to `[[buffer(n)]]` must be aligned to 16 bytes (or the data type's natural alignment, whichever is larger).
- **Texture format compatibility:** Not all texture formats support all operations. Check `device.supportsFamily(_:)` and the Metal Feature Set Tables for format capabilities.

## Common Mistakes

- **Using `bounds` instead of `drawableSize`** for viewport setup — results in incorrect rendering on Retina displays.
- **Setting `enableSetNeedsDisplay = true`** for real-time rendering — the display link stops and frames are only drawn on explicit `setNeedsDisplay()` calls.
- **Ignoring `isPaused`** when the app is backgrounded — the display link continues firing, wasting GPU cycles. Set `isPaused = true` in `sceneDidEnterBackground`.
- **Not checking `supportsFamily(_:)`** before using mesh shaders or ray tracing — crashes on older hardware.
- **Exceeding `maxTotalThreadsPerThreadgroup`** — the dispatch fails silently or produces incorrect results.
- **Using `StorageModePrivate` on Apple Silicon** for buffers that the CPU needs to read — use `StorageModeShared` instead; unified memory makes private storage unnecessary for most cases.
- **Forgetting to call `buffer.present(drawable)`** — the drawable is never displayed, and the layer's drawable pool eventually exhausts.
- **Not updating `contentsScale`** when the view moves to a different display — rendering appears at the wrong resolution.
- **Not using autorelease pool** around CAMetalLayer rendering — drawables may not be released promptly, exhausting the drawable pool and causing `nextDrawable()` to block.

## Review Checklist

- [ ] MTKView wrapped with correct `colorPixelFormat` and `depthStencilPixelFormat`
- [ ] `drawableSize` used for viewport, not `bounds`
- [ ] `preferredFramesPerSecond` set appropriately (0 for uncapped, 60 for standard)
- [ ] GPU selection checks `isLowPower`, `isRemovable`, `isHeadless` as needed
- [ ] `supportsFamily(_:)` checked before using advanced features
- [ ] Render pipeline descriptor matches attachment pixel formats
- [ ] Compute threadgroup size is a multiple of 32 and within `maxTotalThreadsPerThreadgroup`
- [ ] Heap resources use appropriate hazard tracking mode
- [ ] Heaps sized appropriately; used for aliasable and non-aliasing resources separately
- [ ] Fences used for intra-encoder sync, events for cross-queue sync
- [ ] CAMetalLayer `contentsScale` updated on display change
- [ ] `maximumDrawableCount` set to 2 or 3; autorelease pool used for rendering
- [ ] MSL buffer/texture/sampler indices match encoder bindings
- [ ] Pipeline states cached, not recreated per frame
- [ ] Command buffers not reused after commit
- [ ] MSL language revision aligned to target OS version (metal3.0+ for recent features)

## References

- [references/metal-macos-patterns.md](references/metal-macos-patterns.md) — Complete MSL shader examples, GPU family feature table, compute pipeline recipes, synchronization patterns
- [Apple Developer: MTLHeap](https://developer.apple.com/documentation/metal/mtlheap) — Heap API reference
- [Apple Developer: CAMetalLayer](https://developer.apple.com/documentation/quartzcore/cametallayer) — Layer reference
- [Apple Developer: Selecting Device Objects for Graphics Rendering](https://developer.apple.com/documentation/metal/selecting-device-objects-for-graphics-rendering/) — Multi-GPU guidance
- [Apple Developer: Learn performance best practices for Metal shaders (Tech Talks 111373)](https://developer.apple.com/videos/play/tech-talks/111373/) — MSL performance best practices
- [Apple Developer: NSViewRepresentable](https://developer.apple.com/documentation/swiftui/nsviewrepresentable) — SwiftUI wrapping pattern
- [Metal Shading Language Specification](https://developer.apple.com/metal/Metal-Shading-Language-Specification.pdf) — MSL language reference
