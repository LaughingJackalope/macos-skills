---
name: apple-on-device-ai-macos
description: "On-device AI on macOS — Foundation Models (LanguageModelSession, @Generable, tool calling, SystemLanguageModel), CoreML (MLModel, MLTensor, Neural Engine), MLX Swift, llama.cpp. MLComputeUnits (CPU+NE+GPU), model deployment in bundle vs on-demand resources, XPC isolation for model inference. Use when implementing ML/AI features on macOS."
sources: https://developer.apple.com/documentation/coreml, https://developer.apple.com/documentation/foundationmodels
researched: 2026-06-06
platform: macOS 26+
---

# Apple On-Device AI — macOS

On-device machine learning and AI on macOS 26+ covers Apple's Foundation Models framework for text generation, CoreML for running custom models, MLX for open-weight Swift-native inference, and llama.cpp for local LLM deployment. Covers compute unit selection, model packaging, streaming responses, structured output with `@Generable`, tool calling, and isolating inference in XPC helper processes.

See [references/ai-patterns.md](references/ai-patterns.md) for detailed code recipes covering Foundation Models with `@Generable` and tool calling, CoreML configuration, MLTensor pipelines, on-demand resources, and XPC-isolated inference.

## Contents

- [Foundation Models](#foundation-models)
- [Generation](#generation)
- [Tool Calling](#tool-callling)
- [CoreML Setup](#coreml-setup)
- [CoreML Inference](#coreml-inference)
- [MLTensor](#mltensor)
- [Model Deployment](#model-deployment)
- [XPC Isolation](#xpc-isolation)
- [MLX & llama.cpp](#mlx--llamacpp)
- [Constraints & Gotchas](#constraints--gotchas)
- [Common Mistakes](#common-mistakes)
- [Review Checklist](#review-checklist)
- [References](#references)

## Foundation Models

Apple's **Foundation Models** framework provides access to Apple's on-device language model through `SystemLanguageModel`. The model runs entirely on-device using the Neural Engine, GPU, and CPU.

```swift
import FoundationModels

let model = SystemLanguageModel.default

// Check availability before use
guard model.availability == .available else {
    // Handle unavailable — model may not be supported on this device,
    // or the user disabled the feature
    switch model.availability {
    case .unavailable(let reason):
        // reason: .modelNotSupported, .disabled
        break
    case .available:
        break
    @unknown default:
        break
    }
    return
}
```

`SystemLanguageModel.default` is a singleton representing the on-device model. Availability checks are essential — the model is not available on all Macs (requires Apple Silicon) and users can disable it in Settings. Always branch on `model.availability` before attempting inference.

The model object is stateless. Session and state are managed by `LanguageModelSession`, which you create per-conversation or per-task.

## Generation

Create a `LanguageModelSession` with the model and an optional `GenerationOptions`. Use `session.respond(to:)` for single-shot generation or `session.streamResponse(to:)` for streaming.

```swift
let session = LanguageModelSession(model: model)

// Single-shot
let response = try session.respond(to: "Summarize quantum entanglement in one sentence.")
print(response.content)

// With options
var options = GenerationOptions()
options.temperature = 0.3
options.maximumResponseTokens = 256
options.seed = 42  // reproducible output

let precise = try session.respond(to: "List three sorting algorithms.", options: options)
```

**Streaming** delivers content incrementally via an `AsyncSequence`:

```swift
for try await chunk in session.streamResponse(to: "Write a short poem about Rust.") {
    print(chunk.content, terminator: "")
}
```

Stream chunks include partial content as it's generated. Use streaming for any response >1s perceived latency — it dramatically improves UX for long-form output.

**Prompt style:** Use natural language instructions. The model supports conversational context via session history — pass prior turns in the `LanguageModelSession(transcript:)` initializer to resume a conversation.

```swift
let transcript = Transcript(entries: [
    .prompt(Turn(role: .user, content: "My name is Kai.")),
    .response(Turn(role: .assistant, content: "Nice to meet you, Kai!")),
])
let continuingSession = LanguageModelSession(model: model, transcript: transcript)
```

## Tool Calling

To enable the model to call your code, conform types to the `Tool` protocol and pass them to the session. Tool inputs use `@Generable` for structured, schema-derived parameters.

```swift
import FoundationModels

/// A tool that looks up weather by city
struct WeatherLookup: Tool {
    let name = "get_weather"
    let description = "Get the current weather for a city."

    @Generable
    struct Arguments {
        var city: String
    }

    func call(arguments: Arguments) async -> String {
        // Perform the actual lookup
        return "72°F, sunny in \(arguments.city)"
    }
}

let session = LanguageModelSession(tools: [WeatherLookup()])
let response = try session.respond(to: "What's the weather in Tokyo?")
```

The model decides when to invoke tools based on the tool name and description. After a tool call, the session automatically feeds the result back to the model for a final response. This is **multi-turn**: the model sees the tool output and may call additional tools before synthesizing a `@Generable` guidance type for structured output:

```swift
@Generable
struct Itinerary {
    var city: String
    var activities: [String]
    @Guide(description: "1-5 rating")
    var rating: Int
}
```

Apply `@Generable` to structs describing the shape you want. Use `@Guide` properties for field-level instructions the model should follow. The session can return typed output via `respond(to: Itinerary.self)`.

## CoreML Setup

**CoreML** is the execution engine for custom ML models. You create an `MLModel` from a `.mlmodel` or `.mlpackage` file, optionally with configuration.

```swift
import CoreML

// Simple load
let model = try MLModel(contentsOf: modelURL)

// With configuration
let config = MLModelConfiguration()
config.computeUnits = .cpuAndNeuralEngine  // target Neural Engine
config.allowLowPrecisionAccumulationOnGPU = true
config.parameters = [.numThreads: 4]

let model = try MLModel(contentsOf: modelURL, configuration: config)
```

**`MLComputeUnits` options:**

| Value | Behavior |
|---|---|
| `.cpuOnly` | CPU-only execution — most compatible, slowest for heavy models |
| `.cpuAndGPU` | Use GPU for parallelizable layers, CPU for control-flow ops |
| `.cpuAndNeuralEngine` | Prefer Neural Engine for compatible layers — best perf/watt |
| `.all` | Let the system choose per-layer — recommended default |

For image classification / transformer inference, `.cpuAndNeuralEngine` typically gives the best throughput. Use `.all` when unsure. The Neural Engine is a dedicated ML accelerator — it's not the GPU and has different operator support. Always measure real workloads; theoretical preference doesn't always win.

## CoreML Inference

CoreML inference uses `MLPredictionProvider`'s `prediction(from:)` with an `MLFeatureProvider` input.

```swift
// Single prediction
let output = try model.prediction(from: inputFeatureProvider)

// Batch prediction for throughput
let batchProvider = MLArrayBatchProvider(array: [input1, input2, input3])
let batchOutput = try model.predictions(from: batchProvider)
```

**Async prediction** (iOS 17+/macOS 14+):

```swift
let output = try await model.prediction(from: input, options: MLModel.PredictionOptions())
```

**`MLFeatureProvider`** maps `MLFeatureValue` instances by key. For typed wrappers, use CoreML's auto-generated `Input` / `Output` classes from `.mlmodel` files, or create custom providers.

```swift
final class ImageFeatureProvider: MLFeatureProvider {
    let image: CVPixelBuffer

    var featureNames: Set<String> { ["image"] }

    func featureValue(for featureName: String) -> MLFeatureValue? {
        guard featureName == "image" else { return nil }
        return MLFeatureValue(pixelBuffer: image)
    }
}
```

## MLTensor

`MLTensor` is CoreML's tensor abstraction for data pipelines. Tensors are **lazy** — operations build a graph that materializes on evaluation.

```swift
import CoreML

let a = MLTensor(shape: [3, 3], values: Array(0..<9).map { Float($0) })
let b = a transposed()
let c = a + b  // lazy: no computation yet

// Materialize with explicit device placement
let graph = MLTensorGraph()
let result = graph.node(c, on: .neuralEngine)  // target Neural Engine
let evaluated = try await result.evaluated()
```

**Device placement:**

- `.cpu` — always available, highest precision
- `.gpu` — Metal GPU, good for parallelizable ops
- `.neuralEngine` — dedicated ML accelerator, best for inference workloads
- `.automatic` — system decides

Use `.neuralEngine` for inference sub-graphs, `.cpu` for preprocessing that the NE doesn't support. Lazy evaluation means chains of operations fuse together — avoid breaking chains with intermediate `evaluated()` calls unless debugging.

## Model Deployment

Models can ship **inside the app bundle** (guaranteed available) or via **on-demand resources** (downloaded after install, purgable).

**Bundle placement:** Place `.mlmodelc` compiled models in the app bundle. Xcode compiles `.mlmodel` → `.mlmodelc` at build time. Access via `Bundle.main.url(forResource:withExtension:)`.

**On-demand resources (ODR):** Tag `.mlmodelc` files in Xcode with an asset tag. Use `NSBundleResourceRequest` to download on demand.

```swift
let resourceRequest = NSBundleResourceRequest(tags: ["coreml-classifier"])

resourceRequest.beginAccessingResources { error in
    guard error == nil else { return }
    // Model downloaded and available — load from bundle
    let modelURL = Bundle.main.url(forResource: "Classifier", withExtension: "mlmodelc")!
}

// When done (or the system needs space):
resourceRequest.endAccessingResources()
```

ODR models can be purged when storage is tight. Always check availability and re-request if needed. For large models (>50MB), ODR is strongly recommended — it reduces initial download size and avoids App Store size limits.

**Model compilation:** If you ship an uncompiled `.mlmodel`, compile it on first launch:

```swift
let compiledURL = try MLModel.compileModel(at: uncompiledURL)
let model = try MLModel(contentsOf: compiledURL)
try FileManager.default.moveItem(at: compiledURL, to: permanentURL)
```

Use BackgroundTasks to schedule compilation off the main thread on app launch.

## XPC Isolation

Run model inference in a separate XPC helper process to isolate crashes, reduce memory footprint of the main app, and enforce sandbox boundaries.

```
Main App ←→ NSXPCConnection ←→ MLInferenceService (XPC Helper)
```

**XPC Service definition (Info.plist):**

```xml
<key>NSXPCServices</key>
<dict>
    <key>com.myapp.MLInferenceService</key>
    <dict>
        <key>NSXPCServiceEndpoint</key>
        <string>com.myapp.MLInferenceService</string>
    </dict>
</dict>
```

**XPC protocol:**

```swift
@objc protocol MLInferenceProtocol {
    func predict(input: Data, reply: @escaping (Data?, Error?) -> Void)
}
```

**Main app side:**

```swift
let connection = NSXPCConnection(serviceName: "com.myapp.MLInferenceService")
connection.remoteObjectInterface = NSXPCInterface(with: MLInferenceProtocol.self)
connection.resume()

let service = connection.remoteObjectProxyWithErrorHandler { error in
    // Handle connection invalidation / crash
} as? MLInferenceProtocol

service?.predict(input: inputData) { outputData, error in
    // Process result
}
```

**Helper side:** The XPC helper loads the ML model. If it crashes, only the helper dies — the app stays alive. Implement the resolver if the model isn't encoded in the bundle; pass it as a parameter.

**Why XPC isolate ML:**
- ML model crashes (corrupt model, Neural Engine errors) don't take down the main app
- Neural Engine context is process-scoped; heavy inference doesn't compete with UI compositing
- Sandboxing: the helper can run with a tighter entitlement profile
- Memory pressure: the helper's memory is fully reclaimable on completion

## MLX & llama.cpp

For open-weight models not supported by CoreML/Foundation Models, use **MLX Swift** (Apple's Swift-native ML framework) or **llama.cpp**.

**MLX Swift (`mlx-swift`):**

```swift
import MLX

let a = MLXArray([1.0, 2.0, 3.0])
let b = MLXArray([4.0, 5.0, 6.0])
let c = a + b  // runs on GPU by default
```

MLX uses a unified memory model (CPU/GPU share address space) and lazy evaluation. It supports LoRA adapters, quantized models, and transformer architectures. Install via Swift Package Manager from `https://github.com/ml-explore/mlx-swift`.

**llama.cpp (Swift bindings):**

Use `llama.swift` or similar community bindings to run GGUF-quantized models entirely in Metal GPU. Best for larger models (7B-70B range) where CoreML's model format is impractical. llama.cpp approaches CoreML Neural Engine performance on GPU-bound workloads.

## Constraints & Gotchas

- **Neural Engine availability:** Not all Macs have the same NE capabilities. M1 through M4 each support different operator subsets. Test on your minimum supported hardware.
- **Thermal throttling:** Sustained Neural Engine load causes thermal throttling. The system may throttle NE frequency after ~30s of continuous inference. Design workloads in bursts when possible.
- **Model size limits:** On-device Foundation Models model is sized by Apple — you don't control it. CoreML models in the bundle are limited by App Store download thresholds (200MB ODR warning). For models >1GB, use OD download.
- **Seed reproducibility:** `FoundationModels` seeding is best-effort on Neural Engine — GPU-based generation varies slightly between runs due to floating-point ordering. For truly deterministic output, use CoreML with CPU execution.
- **Lazy MLTensor graphs** must be `evaluated()` before reading values. If you pass a lazy tensor to a function expecting concrete values, you'll get graph objects, not data.
- **ODR re-download:** The system purges on-demand resources under storage pressure. Your code must handle re-download gracefully — never treat ODR models as permanently cached.

## Common Mistakes

1. **Not checking `SystemLanguageModel.availability`** before use — crashes on unsupported devices or when disabled in Settings
2. **Running all CoreML on CPU** (`.cpuOnly`) when the Neural Engine would give 3-5x better throughput — profile before choosing
3. **Blocking the main thread** with model inference — always use `async` prediction or dispatch to a background queue
4. **Deploying large models in the bundle** instead of ODR — increases initial download size and may trigger App Store warnings
5. **XPC helper that doesn't handle connection invalidation** — always implement `interruptionHandler` and `invalidationHandler` on `NSXPCConnection`
6. **Assuming Neural Engine supports all operators** — unsupported ops silently fall back to GPU or CPU silently; CoreML splits the model across compute units, which can create unexpected bottlenecks at transfer points

## Review Checklist

- [ ] `SystemLanguageModel.availability` checked before any session creation
- [ ] CoreML models tagged for on-demand resources if >50MB
- [ ] `MLComputeUnits` chosen intentionally (not just `.all` everywhere — profile your model)
- [ ] Inference dispatched off the main thread; UI remains responsive during generation
- [ ] `@Guidance` annotations cover all fields on `@Generable` types
- [ ] Tool descriptions are specific enough for the model to know when to invoke them
- [ ] XPC helpers have connection invalidation handling
- [ ] First-launch model compilation handled in background, not blocking app start
- [ ] Tested on minimum supported hardware (older Apple Silicon = fewer NE ops)
- [ ] ODR purging path tested — app recovers gracefully when models are evicted
- [ ] Thermal throttling scenario tested — sustained inference doesn't hang or crash

## References

- [Foundation Models Documentation](https://developer.apple.com/documentation/foundationmodels)
- [CoreML Documentation](https://developer.apple.com/documentation/coreml)
- [MLTensor Reference](https://developer.apple.com/documentation/coreml/mltensor)
- [On-Demand Resources Guide](https://developer.apple.com/library/archive/documentation/FileManagement/Conceptual/Understanding_Resources_on_Apple_Devices/)
- [NSXPCConnection](https://developer.apple.com/documentation/foundation/nsxpcconnection)
- [MLX Swift](https://github.com/ml-explore/mlx-swift)
- [llama.cpp](https://github.com/ggerganov/llama.cpp)

## Research Addendum — 2026 Updates

### Foundation Models Enhancements

- **`SystemLanguageModel`** — Still the entry point. On macOS 26, ensure you check `model.availability` — users can disable the feature in System Settings, and not all Macs support it (requires Apple Silicon).
- **`LanguageModelSession` with tools** — Multi-turn tool calling is fully supported. The session feeds tool results back to the model automatically, and the model may chain multiple tool calls before synthesizing a final response.
- **`@Generable` with `@Guide`** — Use `@Guide(description:)` on struct properties to provide field-level instructions. Critical for getting structured output that matches your expected schema.
- **Streaming is the default UX pattern** — For any generation that could exceed ~1s, use `session.streamResponse(to:)`. Users see partial output immediately.
- **Seed reproducibility** — Best-effort on Neural Engine due to floating-point ordering. For deterministic output, use CoreML with CPU execution.

CoreML & Neural Engine (2026)

- **Apple Silicon NE optimization**: Core ML dynamically partitions models across CPU (BNNS/AMX), GPU (Metal), and Neural Engine (ANE). You don't control per-layer assignment — the runtime decides based on operator support.
- **NE operator coverage varies by chip**: M1 through M4 support different operator subsets. Test on your minimum supported device. Unsupported operators silently fall back to GPU or CPU.
- **`MLComputeUnits.cpuAndNeuralEngine`** — Preferred for inference workloads. Avoid `.all` in production without profiling; the system might choose a suboptimal path for your specific model.
- **`MLNeuralEngineComputeDevice.totalCoreCount`** — Query the number of Neural Engine cores available on the current device.
- **Thermal throttling** — Sustained NE load (~30s+) triggers thermal throttling. Design bursty workloads when possible and monitor performance on thermally constrained devices (MacBook Air vs. Mac Studio).
- **Typed execution**: ML programs run with precision determined by the runtime's dynamic partitioning. CPU-only ensures highest precision. NE may use lower precision for some operators.

### On-Demand Resources (ODR)

- **`NSBundleResourceRequest`** remains the API. Tag `.mlmodelc` files in Xcode with an asset tag. Check availability, begin access, load model.
- **Purging**: The system purges ODR under storage pressure. Your code must handle re-download gracefully — never treat ODR models as permanently cached.
- **Size thresholds**: For models >50MB, ODR is strongly recommended to reduce initial download and avoid App Store limits.
- **Background compilation**: Use `BackgroundTasks` framework to compile `.mlmodel` → `.mlmodelc` on first launch without blocking the main thread. Store the compiled `.mlmodelc` in the app's cache directory.
