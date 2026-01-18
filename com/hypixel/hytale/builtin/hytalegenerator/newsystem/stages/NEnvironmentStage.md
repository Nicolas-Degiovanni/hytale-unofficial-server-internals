---
description: Architectural reference for NEnvironmentStage
---

# NEnvironmentStage

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.stages
**Type:** Transient

## Definition
```java
// Signature
public class NEnvironmentStage implements NStage {
```

## Architecture & Concepts
The NEnvironmentStage is a critical component within the Hytale world generation pipeline. It functions as a data transformation stage, responsible for converting high-level, two-dimensional biome data into a granular, three-dimensional environmental map.

Architecturally, this stage acts as a bridge between biome definition and voxel-level world properties. It consumes a 2D pixel buffer where each pixel represents a BiomeType for a vertical column of the world. It then iterates through every voxel within the target 3D bounds, queries the biome's associated EnvironmentProvider, and populates an output voxel buffer with the resulting environment value.

This process effectively "extrudes" the 2D biome map into a 3D volume, applying biome-specific environmental rules. The output data is likely consumed by subsequent stages for tasks such as atmospheric rendering, ambient sound selection, or biome-specific particle effects. The use of an EnvironmentProvider interface decouples this stage from the specific logic of any single biome, allowing for a flexible and extensible generation system.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level world generation orchestrator or pipeline builder. The constructor requires specific input and output buffer definitions, indicating it is configured as part of a declarative pipeline definition, not created dynamically during generation.
-   **Scope:** The object itself is stateless and holds only configuration. An instance persists for the lifetime of the world generator's pipeline configuration. The same instance is reused for processing numerous distinct world chunks.
-   **Destruction:** The object is eligible for garbage collection when the world generation pipeline is reconfigured or the server shuts down. No explicit cleanup is required.

## Internal State & Concurrency
-   **State:** This class is immutable and stateless. All member fields are final and are initialized in the constructor. It does not cache any data between invocations of the run method. All state is passed in via the NStage.Context object.
-   **Thread Safety:** The class instance is inherently thread-safe due to its immutability. The primary method, run, is designed to be executed by a single worker thread on a given chunk. The NStage.Context object, which contains the data buffers, is **not** thread-safe and must not be shared across multiple threads executing this stage's run method simultaneously. The presence of a workerId in the context is a strong indicator of a single-threaded execution model per task.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run(NStage.Context context) | void | O(width * depth * height) | Executes the stage. Reads from the biome buffer and writes to the environment buffer for the entire volume defined by the output bounds. |
| getInputTypesAndBounds_bufferGrid() | Map | O(1) | Declares the required input buffer types and their relative bounds. Used by the pipeline orchestrator for dependency resolution. |
| getOutputTypes() | List | O(1) | Declares the buffer types this stage produces. |
| getName() | String | O(1) | Returns the configured, human-readable name of this stage instance. |

## Integration Patterns

### Standard Usage
A developer does not typically invoke methods on this class directly. Instead, it is instantiated and registered as part of a larger NStage pipeline managed by a central orchestrator.

```java
// Pseudo-code for pipeline setup
NParametrizedBufferType biomeBuffer = new NParametrizedBufferType("biomes", ...);
NParametrizedBufferType envBuffer = new NParametrizedBufferType("environment", ...);

NStage environmentStage = new NEnvironmentStage(
    "GenerateEnvironment",
    biomeBuffer,
    envBuffer
);

// The stage is added to a list and passed to a pipeline runner
List<NStage> pipelineStages = List.of(..., environmentStage, ...);
WorldGenPipeline pipeline = new WorldGenPipeline(pipelineStages);

// The pipeline runner will invoke stage.run() with the correct context
pipeline.executeForChunk(chunkPos);
```

### Anti-Patterns (Do NOT do this)
-   **Manual Invocation:** Do not call the run method directly. Doing so bypasses the pipeline's buffer management, dependency tracking, and parallel execution logic, which will lead to unpredictable behavior and likely a system crash.
-   **Incorrect Buffer Configuration:** The constructor performs assertions to validate the buffer types. Providing buffer types that do not match the expected NCountedPixelBuffer of BiomeType for input or NVoxelBuffer of Integer for output will result in an AssertionError during pipeline initialization.

## Data Pipeline
The NEnvironmentStage is a pure data transformation component. Its flow is linear and predictable.

> Flow:
> NCountedPixelBuffer<BiomeType> -> **NEnvironmentStage** (reads BiomeType, queries EnvironmentProvider) -> NVoxelBuffer<Integer> -> Subsequent Generation Stages

