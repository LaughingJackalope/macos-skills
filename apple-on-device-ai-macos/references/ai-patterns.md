# AI & ML Patterns — macOS

Deep-dive code patterns for on-device AI: Foundation Models with structured output and tool calling, CoreML compute configuration, MLTensor data pipelines, on-demand resource management, and XPC-isolated inference.

## Foundation Models — @Generable Structs & Tool Calling

### Structured Output with @Generable

`@Generable` derives the JSON schema from the type's property names and types. The model generates structured output matching this schema.

```swift
import FoundationModels

@Generable
struct Recipe {
    var name: String
    var ingredients: [String]
    @Guide(description: "Total preparation time in minutes")
    var prepTimeMinutes: Int
    @Guide(description: "Difficulty: easy, medium, or hard")
    var difficulty: String
}

let session = LanguageModelSession(model: SystemLanguageModel.default)
let response = try session.respond(to: "Suggest a recipe using chicken and rice.", generating: Recipe.self)

// Access typed content
let recipe = try response.content.decode(Recipe.self)
print(recipe.name)           // "Lemon Herb Chicken Rice Bowl"
print(recipe.prepTimeMinutes) // 35
```

`@Guide` properties shape generation. Without descriptions, the model infers intent from the property name — `difficulty` could produce anything. With `@Guide`, you constrain values explicitly.

### Multi-Turn Tool Calling

The model may chain multiple tool calls before producing a final answer. The session manages this loop internally.

```swift
struct SearchTool: Tool {
    let name = "search"
    let description = "Search for information. Returns a list of relevant snippets."

    @Generable
    struct Arguments {
        @Guide(description: "The search query")
        var query: String
    }

    func call(arguments: Arguments) async -> String {
        // Perform actual search
        let results = await searchEngine.search(arguments.query)
        return results.map(\.snippet).joined(separator: "\n")
    }
}

struct SummarizeTool: Tool {
    let name = "summarize"
    let description = "Summarize a text passage into a concise paragraph."

    @Generable
    struct Arguments {
        var text: String
    }

    func call(arguments: Arguments) async -> String {
        // Summarization logic
        return arguments.text.prefix(100) + "..."
    }
}

let session = LanguageModelSession(tools: [SearchTool(), SummarizeTool()])
let response = try session.respond(to: "Find recent developments in quantum computing and summarize them.")
// Model may call search("quantum computing recent"), then call summarize(...) on the results
```

**Key rules:**
- Tool names must be valid identifiers (no spaces, alphanumeric + underscore)
- The `description` is how the model decides *when* to use a tool — write it like you're explaining to another developer
- Return `String` from `call()` — complex return types should be serialized (e.g., JSON string) or flattened to text
- Tool execution is **synchronous from the model's perspective** — `call()` suspends generation until it returns

### Session Persistence

Save and restore conversation state via `Transcript`:

```swift
// Save transcript
let entries = session.transcript.entries
let encoded = try JSONEncoder().encode(entries.map { $0.serialized() })

// Restore later
let decoded = try JSONDecoder().decode([SerializedEntry].self, from: encoded)
let transcript = Transcript(entries: decoded.map { $0.turn })
let newSession = LanguageModelSession(transcript: transcript)
```

## CoreML — MLComputeUnits Configuration

Profile your specific model on target hardware before choosing compute units. The right choice depends on operator support and model architecture.

```swift
import CoreML

func loadModel(named name: String, strategy: ComputeStrategy) throws -> MLModel {
    let config = MLModelConfiguration()

    switch strategy {
    case .maximumCompatibility:
        config.computeUnits = .cpuOnly
    case .neuralEngineOptimized:
        config.computeUnits = .cpuAndNeuralEngine
    case .balanced:
        config.computeUnits = .all
    case .gpuCompute:
        config.computeUnits = .cpuAndGPU
        config.allowLowPrecisionAccumulationOnGPU = true
    }

    let url = Bundle.main.url(forResource: name, withExtension: "mlmodelc")!
    return try MLModel(contentsOf: url, configuration: config)
}

enum ComputeStrategy {
    case maximumCompatibility    // Any Mac, any OS version
    case neuralEngineOptimized   // Best perf/watt on supported hardware
    case balanced                // System decides per-layer
    case gpuCompute              // Use Metal GPU, optionally FP16
}
```

**Profiling snippet:**

```swift
import CoreML

func benchmark(model: MLModel, input: MLFeatureProvider, iterations: Int = 50) -> Double {
    // Warmup
    _ = try? model.prediction(from: input)

    let start = CFAbsoluteTimeGetCurrent()
    for _ in 0..<iterations {
        _ = try? model.prediction(from: input)
    }
    let elapsed = CFAbsoluteTimeGetCurrent() - start
    return elapsed / Double(iterations)  // seconds per inference
}
```

Run this with each `MLComputeUnits` value on your minimum supported device. The Neural Engine path only wins if the majority of operators are supported — otherwise, fallback transfers to CPU/GPU create overhead that negates the benefit.

## MLTensor Data Pipeline

Build preprocessing and postprocessing graphs with MLTensor:

```swift
import CoreML

func preprocessAndPredict(pixelBuffer: CVPixelBuffer, model: MLModel) throws -> String {
    // 1. Wrap input as MLTensor
    let inputTensor = MLTensor(pixelBuffer: pixelBuffer)  // shape [1, H, W, C]

    // 2. Preprocessing (lazy)
    let resized = inputTensor.resized(to: [224, 224])
    let normalized = (resized / 255.0 - MLTensor([0.485, 0.456, 0.406]))
        / MLTensor([0.229, 0.224, 0.225])
    let channelsFirst = normalized.transposed(withPermutation: [0, 3, 1, 2])  // NCHW

    // 3. Create graph with device placement
    let graph = MLTensorGraph()
    let prepared = graph.node(channelsFirst, on: .neuralEngine)
    let modelOutput = try graph.node(
        model.prediction(from: prepared),
        on: .neuralEngine
    )
    let argmax = modelOutput.argmax(alongAxis: 1)  // lazy

    // 4. Single evaluation at the boundary
    let classIndex = try await argmax.evaluated()

    // 5. Map to label
    let labels = ["cat", "dog", "bird", "fish"]
    return labels[Int(classIndex[0].item()!)]
}
```

**Important:** Call `evaluated()` only at the final step. Each evaluation forces synchronization between compute units and materializes tensors in memory. Chaining lazy operations lets the system fuse kernels and schedule efficiently.

## On-Demand Resources for Large Models

For models too large for the bundle, use `NSBundleResourceRequest`:

```swift
final class ModelManager {
    private var resourceRequest: NSBundleResourceRequest?
    private var downloadProgress: Progress?

    func loadModel(tag: String, completion: @escaping (Result<URL, Error>) -> Void) {
        let request = NSBundleResourceRequest(tags: [tag])
        self.resourceRequest = request

        // Progress observation
        self.downloadProgress = request.progress
        // Observe with KVO or SwiftUI @ObservedObject

        request.beginAccessingResources { [weak self] error in
            if let error {
                completion(.failure(error))
                return
            }
            // Resource available
            guard let url = Bundle.main.url(forResource: "Classifier", withExtension: "mlmodelc") else {
                completion(.failure(ModelError.resourceNotFound))
                return
            }
            completion(.success(url))
        }
    }

    func releaseModel() {
        resourceRequest?.endAccessingResources()
        resourceRequest = nil
    }
}

enum ModelError: Error {
    case resourceNotFound
    case downloadFailed
}
```

**Xcode setup:** In the asset catalog / build settings, assign tags to `.mlmodelc` folders. Tags are strings like `"coreml-classifier-v2"`. The download size shown in App Store Connect uses compressed sizes.

## XPC-Isolated MLModel Inference

Run inference in an XPC service so crashes and memory spikes don't take down the main app.

**XPC Service entry point:**

```swift
// MLInferenceService.swift (XPC Helper)
import Foundation
import CoreML

final class MLInferenceService: NSObject, MLInferenceProtocol {
    private var model: MLModel?

    func loadModel(data: Data, reply: @escaping (Error?) -> Void) {
        // Load from data or bundle
        guard let url = Bundle(for: type(of: self)).url(forResource: "Model", withExtension: "mlmodelc") else {
            reply(ModelError.modelNotFound)
            return
        }
        do {
            let config = MLModelConfiguration()
            config.computeUnits = .cpuAndNeuralEngine
            model = try MLModel(contentsOf: url, configuration: config)
            reply(nil)
        } catch {
            reply(error)
        }
    }

    func predict(inputData: Data, reply: @escaping (Data?, Error?) -> Void) {
        guard let model else {
            reply(nil, ModelError.modelNotLoaded)
            return
        }
        do {
            let input = try JSONDecoder().decode(InputFeatures.self, from: inputData)
            let provider = try MLDictionaryFeatureProvider(dictionary: input.featureDict)
            let output = try model.prediction(from: provider)
            let result = try OutputFeatures(from: output)
            let encoded = try JSONEncoder().encode(result)
            reply(encoded, nil)
        } catch {
            reply(nil, error)
        }
    }
}

// Main entry
let service = MLInferenceService()
let listener = NSXPCListener.service()
listener.delegate = service
listener.resume()
```

**App side — connection management:**

```swift
final class RemoteInferenceService {
    private var connection: NSXPCConnection?
    private var service: MLInferenceProtocol? {
        connection?.remoteObjectProxy as? MLInferenceProtocol
    }

    func connect() {
        let conn = NSXPCConnection(serviceName: "com.myapp.MLInferenceService")
        conn.remoteObjectInterface = NSXPCInterface(with: MLInferenceProtocol.self)
        conn.exportedInterface = NSXPCInterface(with: NSObjectProtocol.self)

        // Handle crashes / unwanted termination
        conn.invalidationHandler = { [weak self] in
            self?.connection = nil  // Will reconnect on next use
        }

        conn.interruptionHandler = { [weak self] in
            // Connection interrupted (e.g., system memory pressure)
            // Connection will typically auto-reconnect, but re-load model state
            self?.reloadModel()
        }

        conn.resume()
        self.connection = conn
    }

    func predict(input: InputFeatures) async throws -> OutputFeatures {
        guard let service else { throw ServiceError.notConnected }
        let inputData = try JSONEncoder().encode(input)

        return try await withCheckedThrowingContinuation { continuation in
            service.predict(inputData: inputData) { outputData, error in
                if let error {
                    continuation.resume(throwing: error)
                    return
                }
                guard let outputData else {
                    continuation.resume(throwing: ServiceError.nilResponse)
                    return
                }
                do {
                    let result = try JSONDecoder().decode(OutputFeatures.self, from: outputData)
                    continuation.resume(returning: result)
                } catch {
                    continuation.resume(throwing: error)
                }
            }
        }
    }

    private func reloadModel() {
        service?.loadModel(data: Data()) { error in
            if let error {
                print("Failed to reload model in XPC helper: \(error)")
            }
        }
    }

    deinit {
        connection?.invalidate()
    }
}

enum ServiceError: Error {
    case notConnected
    case nilResponse
}
```

**Entitlements** — The XPC helper needs its own entitlements file. Request only what's needed: `com.apple.security.app-sandbox` + network access if the model downloads from a server.

**Testing XPC services:** Run the helper as an XPC service in Xcode scheme, not directly. The test target connects via `NSXPCConnection(serviceName:)` same as production. Use `NSXPCConnection(serviceName:)` for named services; don't instantiate the service class manually in production code — it won't be in the right process context.
