# Metal Patterns for macOS

Complete Metal patterns for macOS development: MSL shader examples, GPU family feature table, compute pipeline recipes, and synchronization patterns. Targets macOS 26+ with Swift 6.3.

## Contents

- [MSL Shader Examples](#msl-shader-examples)
- [GPU Family Feature Table](#gpu-family-feature-table)
- [Compute Pipeline Recipes](#compute-pipeline-recipes)
- [Synchronization Patterns](#synchronization-patterns)
- [Heap and Resource Aliasing](#heap-and-resource-aliasing)
- [Off-Screen Rendering](#off-screen-rendering)
- [Performance Profiling](#performance-profiling)

## MSL Shader Examples

### Basic Vertex + Fragment Shader

```metal
#include <metal_stdlib>
using namespace metal;

// Vertex input from the vertex descriptor
struct VertexIn {
    float3 position [[attribute(0)]];
    float3 normal   [[attribute(1)]];
    float2 texCoord [[attribute(2)]];
};

// Vertex output / fragment input
struct VertexOut {
    float4 position [[position]];
    float3 worldNormal;
    float2 texCoord;
    float3 worldPos;
};

// Uniform data — constant address space for per-frame data
struct FrameUniforms {
    float4x4 viewProjection;
    float4x4 model;
    float3   cameraPos;
    float    time;
};

vertex VertexOut vertex_lit(
    VertexIn in [[stage_in]],
    constant FrameUniforms& uniforms [[buffer(0)]]
) {
    VertexOut out;
    float4 worldPos = uniforms.model * float4(in.position, 1.0);
    out.position = uniforms.viewProjection * worldPos;
    out.worldNormal = (uniforms.model * float4(in.normal, 0.0)).xyz;
    out.texCoord = in.texCoord;
    out.worldPos = worldPos.xyz;
    return out;
}

fragment float4 fragment_lit(
    VertexOut in [[stage_in]],
    constant float3& lightDir [[buffer(1)]],
    constant float4& baseColor [[buffer(2)]],
    texture2d<float> albedoTexture [[texture(0)]],
    sampler linearSampler [[sampler(0)]]
) {
    float3 albedo = albedoTexture.sample(linearSampler, in.texCoord).rgb * baseColor.rgb;
    float3 normal = normalize(in.worldNormal);
    float NdotL = max(dot(normal, normalize(lightDir)), 0.0);
    float3 ambient = albedo * 0.1;
    float3 diffuse = albedo * NdotL;
    return float4(ambient + diffuse, 1.0);
}
```

### Compute Shader: Image Blur

```metal
#include <metal_stdlib>
using namespace metal;

kernel void gaussian_blur(
    texture2d<float, access::read>  inputTexture  [[texture(0)]],
    texture2d<float, access::write> outputTexture [[texture(1)]],
    constant float& radius [[buffer(0)]],
    uint2 gid [[thread_position_in_grid]]
) {
    uint width = inputTexture.get_width();
    uint height = inputTexture.get_height();

    if (gid.x >= width || gid.y >= height) return;

    float4 sum = float4(0.0);
    float weightSum = 0.0;
    int r = int(radius);

    for (int dy = -r; dy <= r; dy++) {
        for (int dx = -r; dx <= r; dx++) {
            int2 samplePos = int2(gid) + int2(dx, dy);
            samplePos = clamp(samplePos, int2(0), int2(width - 1, height - 1));

            float weight = exp(-float(dx*dx + dy*dy) / (2.0 * radius * radius));
            sum += inputTexture.read(uint2(samplePos)) * weight;
            weightSum += weight;
        }
    }

    outputTexture.write(sum / weightSum, gid);
}
```

### Compute Shader: Parallel Reduction (Sum)

```metal
#include <metal_stdlib>
using namespace metal;

kernel void reduction_sum(
    device const float* input  [[buffer(0)]],
    device float* output       [[buffer(1)]],
    threadgroup float* shared  [[threadgroup(0)]],
    uint gid  [[thread_position_in_grid]],
    uint lid  [[thread_position_in_threadgroup]],
    uint simd [[threads_per_simdgroup]]
) {
    // Load into threadgroup memory
    shared[lid] = (gid < inputLength) ? input[gid] : 0.0;
    threadgroup_barrier(mem_flags::mem_threadgroup);

    // Tree reduction within threadgroup
    for (uint stride = simd; stride > 0; stride >>= 1) {
        if (lid < stride) {
            shared[lid] += shared[lid + stride];
        }
        threadgroup_barrier(mem_flags::mem_threadgroup);
    }

    // Write result
    if (lid == 0) {
        output[threadgroup_id] = shared[0];
    }
}
```

### Tile Shader (Apple 8+)

Tile shaders run on the tile processor between vertex and fragment stages. Useful for culling, LOD selection, and per-tile data preparation.

```metal
#include <metal_stdlib>
using namespace metal;

// Tile shader receives the tile's bounding data
kernel void tile_culling(
    uint2 tid [[thread_position_in_tile]],
    uint2 tileSize [[threads_per_tile]],
    constant float4* frustumPlanes [[buffer(0)]],
    device uint* visibleIndices [[buffer(1)]],
    device atomic_uint& visibleCount [[buffer(2)]]
) {
    // Per-tile frustum culling logic
    // Write visible object indices to the output buffer
}
```

### Mesh Shader (Apple 9+)

Mesh shaders replace the traditional vertex/tessellation pipeline with a compute-like programming model.

```metal
#include <metal_stdlib>
using namespace metal;

struct MeshVertex {
    float3 position [[attribute(0)]];
    float3 normal;
};

struct Meshlet {
    uint vertexOffset;
    uint primitiveOffset;
    uint vertexCount;
    uint primitiveCount;
};

kernel void mesh_shader(
    uint gid [[thread_position_in_grid]],
    constant Meshlet* meshlets [[buffer(0)]],
    constant float4x4& mvp [[buffer(1)]],
    device MeshVertex* vertices [[buffer(2)]],
    device uint* primitives [[buffer(3)]]
) {
    Meshlet m = meshlets[gid];

    // Emit vertices
    for (uint i = 0; i < m.vertexCount; i++) {
        // Transform and write vertex
    }

    // Emit primitives (triangle indices)
    for (uint i = 0; i < m.primitiveCount; i++) {
        // Write triangle indices
    }
}
```

## GPU Family Feature Table

| Feature | `.apple7` | `.apple8` | `.apple9` | `.mac2` |
|---------|:---------:|:---------:|:---------:|:-------:|
| Function pointers | Yes | Yes (all stages) | Yes | Yes |
| Ray tracing (software) | Yes | Yes | Yes | No |
| Ray tracing (hardware) | No | No | Yes | No |
| Mesh shaders | No | No | Yes | No |
| Tile shaders | No | Yes | Yes (advanced) | No |
| Sparse textures | Yes | Yes | Yes | Yes |
| Sampler feedback | Yes | Yes | Yes | No |
| Argument buffers Tier 2 | No | Yes | Yes | Yes |
| Primitive motion blur | No | No | Yes | No |
| Barycentric coordinates | Yes | Yes | Yes | Yes |
| SIMD-group operations | Yes | Yes | Yes | Yes |
| Quad-group operations | Yes | Yes | Yes | Yes |
| SIMD-group matrix | No | Yes | Yes | Yes |
| Stack size (threads) | 1024 | 1024 | 1024 | 512 |
| Threadgroup memory | 32 KB | 32 KB | 32 KB | 32 KB |
| Max threads per threadgroup | 1024 | 1024 | 1024 | 512 |

**Checking at runtime:**

```swift
extension MTLDevice {
    var supportsMeshShaders: Bool {
        supportsFamily(.apple9)
    }

    var supportsHardwareRayTracing: Bool {
        supportsFamily(.apple9)
    }

    var supportsTileShaders: Bool {
        supportsFamily(.apple8)
    }

    var supportsFunctionPointers: Bool {
        supportsFamily(.apple7)
    }
}
```

## Compute Pipeline Recipes

### 1D Data Processing

For processing a linear array (audio, particle simulation, data transformation):

```swift
func dispatch1D(
    encoder: MTLComputeCommandEncoder,
    state: MTLComputePipelineState,
    elementCount: Int
) {
    let threadsPerGroup = MTLSize(width: 256, height: 1, depth: 1)
    let threadgroups = MTLSize(
        width: (elementCount + 255) / 256,
        height: 1,
        depth: 1
    )
    encoder.dispatchThreadgroups(threadgroups, threadsPerThreadgroup: threadsPerGroup)
}
```

### 2D Image Processing

For image filters, convolution, and per-pixel operations:

```swift
func dispatch2D(
    encoder: MTLComputeCommandEncoder,
    state: MTLComputePipelineState,
    width: Int,
    height: Int
) {
    let threadsPerGroup = MTLSize(width: 16, height: 16, depth: 1)
    let threadgroups = MTLSize(
        width: (width + 15) / 16,
        height: (height + 15) / 16,
        depth: 1
    )
    encoder.dispatchThreadgroups(threadgroups, threadsPerThreadgroup: threadsPerGroup)
}
```

### 3D Volume Processing

For volumetric data, 3D textures, and spatial simulations:

```swift
func dispatch3D(
    encoder: MTLComputeCommandEncoder,
    state: MTLComputePipelineState,
    width: Int,
    height: Int,
    depth: Int
) {
    let threadsPerGroup = MTLSize(width: 8, height: 8, depth: 4)  // 256 threads
    let threadgroups = MTLSize(
        width: (width + 7) / 8,
        height: (height + 7) / 8,
        depth: (depth + 3) / 4
    )
    encoder.dispatchThreadgroups(threadgroups, threadsPerThreadgroup: threadsPerGroup)
}
```

### Multi-Pass with Fence

When a compute pass produces data consumed by a render pass:

```swift
func encodeComputeThenRender(
    commandBuffer: MTLCommandBuffer,
    computeState: MTLComputePipelineState,
    renderState: MTLRenderPipelineState,
    renderPassDescriptor: MTLRenderPassDescriptor,
    device: MTLDevice
) {
    let fence = device.makeFence()!

    // Compute pass
    let computeEncoder = commandBuffer.makeComputeCommandEncoder()!
    computeEncoder.setComputePipelineState(computeState)
    // ... set buffers, textures, dispatch ...
    computeEncoder.updateFence(fence)
    computeEncoder.endEncoding()

    // Render pass — fragment stage waits for compute
    let renderEncoder = commandBuffer.makeRenderCommandEncoder(descriptor: renderPassDescriptor)!
    renderEncoder.setRenderPipelineState(renderState)
    renderEncoder.waitForFence(fence, before: .fragment)
    // ... draw commands ...
    renderEncoder.endEncoding()
}
```

## Synchronization Patterns

### Pattern 1: Single Queue, Sequential Encoders

The simplest pattern. Encoders on the same queue execute in submission order. No explicit synchronization needed for most cases.

```swift
let buffer = commandQueue.makeCommandBuffer()!

let computeEncoder = buffer.makeComputeCommandEncoder()!
// ... compute work ...
computeEncoder.endEncoding()

let renderEncoder = buffer.makeRenderCommandEncoder(descriptor: passDescriptor)!
// ... render work (reads compute output) ...
renderEncoder.endEncoding()

buffer.commit()
```

**When this works:** When the render encoder only reads the compute output in the fragment stage. The GPU's command processor handles the ordering.

**When this fails:** When the render encoder's vertex stage reads compute output, or when using `MTLHeap` resources with `.untracked` hazard mode. Use a fence in these cases.

### Pattern 2: Cross-Queue with Event

Use separate queues for compute and render to overlap work across frames.

```swift
let computeQueue = device.makeCommandQueue()!
let renderQueue = device.makeCommandQueue()!
let event = device.makeEvent()!

// Frame N: compute on queue 1
let computeBuffer = computeQueue.makeCommandBuffer()!
let computeEncoder = computeBuffer.makeComputeCommandEncoder()!
// ... compute work ...
computeEncoder.setSignaledValue(1)
computeEncoder.endEncoding()
computeBuffer.commit()

// Frame N: render on queue 2, waits for compute
let renderBuffer = renderQueue.makeCommandBuffer()!
let renderEncoder = renderBuffer.makeRenderCommandEncoder(descriptor: passDescriptor)!
renderEncoder.wait(for: event, value: 1)
// ... render work ...
renderEncoder.endEncoding()
renderBuffer.commit()
```

### Pattern 3: Triple Buffering with Fences

For CPU-GPU pipelining where the CPU writes to a buffer while the GPU reads from it:

```swift
let fence = device.makeFence()!
var frameIndex = 0
let bufferCount = 3

func frame() {
    let buffer = commandQueue.makeCommandBuffer()!

    let computeEncoder = buffer.makeComputeCommandEncoder()!
    computeEncoder.setComputePipelineState(computeState)
    computeEncoder.waitForFence(fence, before: .compute)  // Wait for GPU to finish reading
    // ... write to buffer[frameIndex] ...
    computeEncoder.updateFence(fence)  // Signal CPU is done writing
    computeEncoder.endEncoding()

    let renderEncoder = buffer.makeRenderCommandEncoder(descriptor: passDescriptor)!
    renderEncoder.waitForFence(fence, before: .vertex)  // Wait for CPU write
    // ... read from buffer[frameIndex] ...
    renderEncoder.updateFence(fence)  // Signal GPU is done reading
    renderEncoder.endEncoding()

    buffer.commit()
    frameIndex = (frameIndex + 1) % bufferCount
}
```

### Pattern 4: Shared Event for XPC

Synchronize Metal work between the main app and an XPC service:

```swift
// In the main app
let sharedEvent = device.makeSharedEvent()!
let handle = sharedEvent.sharedHandle

// Pass `handle` to the XPC service via XPC connection
xpcConnection.sendMessage(["eventHandle": handle])

// Signal from main app
computeEncoder.setSignaledValue(frameNumber)
computeEncoder.endEncoding()

// In the XPC service
let remoteEvent = device.makeSharedEvent(handle: receivedHandle)!

// Wait for signal from main app
renderEncoder.wait(for: remoteEvent, value: expectedFrame)
```

## Heap and Resource Aliasing

### Creating a Heap with Aliasing

```swift
let heapDesc = MTLHeapDescriptor()
heapDesc.size = 128 * 1024 * 1024  // 128 MB
heapDesc.storageMode = .private
heapDesc.hazardTrackingMode = .untracked  // Required for aliasing
let heap = device.makeHeap(descriptor: heapDesc)!

// Texture A: 2048x2048 RGBA16Float = 32 MB
let texADesc = MTLTextureDescriptor.texture2D(
    pixelFormat: .rgba16Float, width: 2048, height: 2048, mipmapped: false
)
texADesc.storageMode = .private
texADesc.hazardTrackingMode = .untracked
let textureA = heap.makeTexture(descriptor: texADesc)!

// Texture B: same size, same format — can alias with A
let texBDesc = MTLTextureDescriptor.texture2D(
    pixelFormat: .rgba16Float, width: 2048, height: 2048, mipmapped: false
)
texBDesc.storageMode = .private
texBDesc.hazardTrackingMode = .untracked
let textureB = heap.makeTexture(descriptor: texBDesc)!

// Use textureA in frame N, textureB in frame N+1
// They share the same heap memory — only one exists at a time
```

### Querying Size and Alignment

```swift
let alignInfo = device.heapTextureSizeAndAlign(descriptor: texADesc)
// alignInfo.size = actual bytes needed (including padding)
// alignInfo.align = required byte offset alignment

let alignInfoBuffer = device.heapBufferSizeAndAlign(length: 1024, options: [])
// Same for buffers
```

## Off-Screen Rendering

Render to a texture without displaying it. Useful for screenshot, video encoding, or generating textures at load time.

```swift
func renderToTexture(
    device: MTLDevice,
    commandQueue: MTLCommandQueue,
    size: CGSize,
    renderState: MTLRenderPipelineState,
    passDescriptor: MTLRenderPassDescriptor
) -> MTLTexture {
    let textureDesc = MTLTextureDescriptor.texture2D(
        pixelFormat: .rgba16Float,
        width: Int(size.width),
        height: Int(size.height),
        mipmapped: false
    )
    textureDesc.usage = [.renderTarget, .shaderRead]
    textureDesc.storageMode = .private
    let texture = device.makeTexture(descriptor: textureDesc)!

    let buffer = commandQueue.makeCommandBuffer()!
    let descriptor = MTLRenderPassDescriptor()
    descriptor.colorAttachments[0].texture = texture
    descriptor.colorAttachments[0].loadAction = .clear
    descriptor.colorAttachments[0].clearColor = MTLClearColor(red: 0, green: 0, blue: 0, alpha: 1)
    descriptor.colorAttachments[0].storeAction = .store

    let encoder = buffer.makeRenderCommandEncoder(descriptor: descriptor)!
    encoder.setRenderPipelineState(renderState)
    // ... draw commands ...
    encoder.endEncoding()

    buffer.commit()
    buffer.waitUntilCompleted()

    return texture
}
```

**Reading back to CPU:**

```swift
func readTextureToPixels(_ texture: MTLDevice, texture: MTLTexture) -> [UInt8] {
    let bytesPerRow = texture.width * 4  // RGBA8
    let region = MTLRegionMake2D(0, 0, texture.width, texture.height)
    var pixels = [UInt8](repeating: 0, count: bytesPerRow * texture.height)
    texture.getBytes(&pixels, bytesPerRow: bytesPerRow, from: region, mipmapLevel: 0)
    return pixels
}
```

**Note:** Reading back from a `.private` texture requires a blit encoder to copy to a `.shared` buffer first. The example above works for `.shared` storage mode.

## Performance Profiling

### GPU Capture with Metal System Trace

```swift
// Enable in Xcode: Product > Scheme > Edit Scheme > Run > Options > Metal API Validation
// Or programmatically:
let captureManager = MTLCaptureManager.shared()
let captureDescriptor = MTLCaptureDescriptor()
captureDescriptor.captureObject = commandQueue
captureDescriptor.destination = .developerTools

try captureManager.startCapture(with: captureDescriptor)
// ... render a frame ...
captureManager.stopCapture()
```

### Timing GPU Work

```swift
let buffer = commandQueue.makeCommandBuffer()!

// Add scheduled handler for timing
var startTime: UInt64 = 0
var endTime: UInt64 = 0

buffer.addScheduledHandler { buffer in
    startTime = mach_absolute_time()
}

buffer.addCompletedHandler { buffer in
    endTime = mach_absolute_time()
    let elapsed = endTime - startTime
    // Convert mach time to nanoseconds using mach_timebase_info
}

// ... encode work ...
buffer.commit()
```

### Best Practices

- **Minimize encoder switches.** Each encoder switch has overhead. Batch work into as few encoders as possible.
- **Reuse pipeline states.** Cache `MTLRenderPipelineState` and `MTLComputePipelineState` objects. Creation is expensive.
- **Use indirect command buffers** for repeated draw calls with varying parameters. Encode once, execute many times.
- **Prefer `loadAction = .dontCare`** for color attachments that are fully overwritten. This avoids a memory load.
- **Prefer `storeAction = .dontCare`** for intermediate render targets that are never read back. This avoids a memory store.
- **Use `MTLCommandBuffer.waitUntilCompleted()`** only for debugging. In production, use handlers to avoid CPU-GPU serialization.
- **Profile with Instruments.** Use the Metal System Trace template to identify encoder bottlenecks, texture bandwidth issues, and occupancy problems.
