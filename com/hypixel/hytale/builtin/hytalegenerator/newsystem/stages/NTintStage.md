---
description: Architectural reference for NTintStage
---

# NTintStage

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.stages
**Type:** Component / Stage

## Definition
```java
// Signature
public class NTintStage implements NStage {
```

## Architecture & Concepts
The NTintStage is a concrete implementation of the NStage interface, designed to operate as a single, focused step within the world generation pipeline. Its primary function is to translate abstract biome data into concrete visual information, specifically the color tint applied to foliage and water within a world column.

This class acts as a data transformation node in the generation graph. It consumes a buffer of BiomeType data and produces a corresponding buffer of integer color values. The core design leverages the Strategy Pattern: NTintStage does not contain any biome-specific tinting logic itself. Instead, it queries the BiomeType object for a dedicated TintProvider, delegating the responsibility of color calculation. This decouples the pipeline stage from the game's content, allowing designers to define new biomes and their associated tinting rules without modifying the generator's core machinery.

NTintStage is designed for columnar processing, iterating over a 2D X/Z plane and assuming the data is consistent vertically for the purpose of tinting.

### Lifecycle & Ownership
- **Creation:** NTintStage is instantiated and configured by a higher-level system responsible for assembling the complete world generation stage pipeline. It is not created on-demand during generation but rather as part of a pre-configured template.
- **Scope:** An instance of NTintStage persists for the lifetime of the world generator configuration. As it is stateless, a single configured instance is used for all world generation tasks that utilize its parent pipeline.
- **Destruction:** The object is eligible for garbage collection when the world generator configuration is unloaded or replaced. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The NTintStage class is **immutable**. All of its configuration fields, such as buffer type descriptors and its stage name, are marked as final and set during construction. All mutable data, including the input and output pixel buffers, is passed into the run method via the NStage.Context object and is never stored as a field.

- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable nature, a single NTintStage instance can be safely shared and executed by multiple worker threads simultaneously. Each thread's execution of the run method operates on a distinct NStage.Context, preventing data races. The design anticipates parallel execution, as evidenced by the workerId being passed down to the TintProvider.

## API Surface
The public contract is defined by the NStage interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run(Context context) | void | O(N*M) | Executes the tinting logic. N and M are the X and Z dimensions of the output bounds. Throws NullPointerException if context or its internal buffers are null. |
| getInputTypesAndBounds_bufferGrid() | Map | O(1) | Declares the required input buffer types and the spatial region of interest. |
| getOutputTypes() | List | O(1) | Declares the buffer types this stage will produce. |
| getName() | String | O(1) | Returns the unique, configured name for this stage instance. |

## Integration Patterns

### Standard Usage
NTintStage is not intended to be invoked directly. It is configured and added to a generation pipeline, which is then executed by a stage runner. The developer's role is to instantiate it with the correct buffer type descriptors.

```java
// In a world generator setup class

// 1. Define the data buffer types that connect stages
NParametrizedBufferType biomeBufferId = new NParametrizedBufferType(
    "BIOMES", NCountedPixelBuffer.class, BiomeType.class
);
NParametrizedBufferType tintBufferId = new NParametrizedBufferType(
    "TINTS", NSimplePixelBuffer.class, Integer.class
);

// 2. Instantiate and configure the stage
NStage biomeTinter = new NTintStage("BiomeTinter", biomeBufferId, tintBufferId);

// 3. Add the stage to a pipeline builder
pipelineBuilder.addStage(biomeTinter);

// The pipeline runner will later invoke stage.run(context) internally.
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Do not call the run method directly. This bypasses the dependency management, buffer allocation, and parallel execution logic provided by the NStage runner framework. The NStage.Context must be supplied by the framework.
- **Incorrect Buffer Types:** Providing NParametrizedBufferType instances that do not match the internal assertions (e.g., using a buffer of Vector3i instead of BiomeType) will result in an AssertionError at construction time. This is a critical configuration error.
- **Assuming 3D Processing:** Do not rely on this stage to process data along the Y-axis. Its internal loop iterates only over X and Z, reading biome data from Y=0 and writing tint data to the corresponding X/Z column.

## Data Pipeline
NTintStage is a transformation step that consumes biome identifiers and produces color values.

> Flow:
> NStageRunner provides `NStage.Context` -> **NTintStage** reads `BiomeType` from input `NCountedPixelBuffer` -> Delegates to `BiomeType.getTintProvider()` -> **NTintStage** writes `Integer` tint value to output `NSimplePixelBuffer` -> Data is consumed by subsequent stages (e.g., rendering prep).

