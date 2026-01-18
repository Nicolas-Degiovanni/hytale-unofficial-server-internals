---
description: Architectural reference for NBiomeDistanceStage
---

# NBiomeDistanceStage

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.stages
**Type:** Transient

## Definition
```java
// Signature
public class NBiomeDistanceStage implements NStage {
```

## Architecture & Concepts

The NBiomeDistanceStage is a data transformation component within the Hytale world generation pipeline. It operates as a distinct step, or *Stage*, responsible for calculating proximity information between world locations and various biomes. Its primary function is to enrich the generation data by answering the question: "For any given (x, z) column, what is the precise distance to the nearest edge of every nearby biome?"

This stage consumes a grid of biome data and produces a new grid containing distance information. It acts as a foundational analysis step, providing critical data for subsequent stages that perform biome blending, feature placement, or environmental modifications that depend on proximity to biome borders.

The core algorithm is a neighborhood search. For each vertical column in the target generation area, the stage scans a surrounding region defined by the `maxDistance_voxelGrid` parameter. It identifies all unique biomes within this search radius and calculates the Euclidean distance from the target column to the nearest voxel of each of those biomes.

A key architectural concept is the use of two coordinate systems managed by the GridUtils class:
1.  **Voxel Grid:** The highest-resolution grid representing individual world blocks.
2.  **Buffer Grid:** A coarse grid where each cell corresponds to an NPixelBuffer, a chunk of world data.

The stage intelligently queries data on the Buffer Grid for efficiency but performs fine-grained distance calculations on the Voxel Grid for accuracy.

## Lifecycle & Ownership

-   **Creation:** An NBiomeDistanceStage is instantiated directly via its constructor, typically by a higher-level orchestrator or factory responsible for assembling the full world generation pipeline. Configuration, including the input and output buffer types and the maximum search distance, is provided at creation time and is immutable thereafter.
-   **Scope:** The object's lifetime is scoped to a single, complete execution of a world generation task. It is created, its `run` method is invoked once by the pipeline runner, and it is then discarded. It holds no state that persists between separate generation runs.
-   **Destruction:** The object becomes eligible for garbage collection as soon as the pipeline runner releases its reference, which typically occurs after the `run` method has completed.

## Internal State & Concurrency

-   **State:** The NBiomeDistanceStage instance is effectively immutable after construction. All its configuration fields are final. The `run` method is stateless concerning the class itself; all operational state is created and managed on the stack or within the provided `NStage.Context`. The primary data it modifies exists within the `NBufferBundle` supplied by the context.
-   **Thread Safety:** This class is not designed for concurrent execution of its `run` method on a single instance. It is expected to be executed by a single thread as part of a larger, potentially multi-threaded, generation system. The `NStage.Context` is assumed to provide thread-local or otherwise isolated access to the data buffers. The internal helper class, BiomeDistanceCounter, is a mutable object but is created, used, and destroyed entirely within the scope of a single iteration of the main processing loop, making it thread-safe by confinement.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run(Context context) | void | O(n * r²) | Executes the distance calculation. Populates the output buffer defined in the context. Throws exceptions if input buffers are missing or of the wrong type. |
| getInputTypesAndBounds_bufferGrid() | Map | O(1) | Returns the data dependencies of this stage, specifying which buffer types it needs and the spatial bounds required. |
| getOutputTypes() | List | O(1) | Returns a list of buffer types that this stage produces. |
| getName() | String | O(1) | Returns the unique, developer-assigned name for this stage instance. |

## Integration Patterns

### Standard Usage

The NBiomeDistanceStage is designed to be part of a sequence of operations managed by a pipeline runner. A developer defines the stage with its dependencies and adds it to the pipeline. The runner is responsible for invoking the `run` method with the correct context.

```java
// In a pipeline builder or generator configuration:

// 1. Define the data types for input and output
NParametrizedBufferType biomeInput = new NParametrizedBufferType(...);
NParametrizedBufferType distanceOutput = new NParametrizedBufferType(...);

// 2. Instantiate and configure the stage
double maxSearchDistance = 256.0;
NStage distanceStage = new NBiomeDistanceStage(
    "CalculateBiomeProximity",
    biomeInput,
    distanceOutput,
    maxSearchDistance
);

// 3. Add the stage to a generation pipeline (conceptual)
worldGenPipeline.addStage(distanceStage);

// The pipeline runner will later invoke distanceStage.run(context);
```

### Anti-Patterns (Do NOT do this)

-   **Excessive Search Distance:** Configuring `maxDistance_voxelGrid` with an extremely large value will cause severe performance degradation. The algorithm's complexity scales with the square of this radius, leading to an exponential increase in computation time.
-   **Reusing Instances:** Do not hold a reference to an NBiomeDistanceStage and call `run` on it for different generation tasks. While technically possible, it violates the intended transient lifecycle and offers no performance benefit.
-   **Incorrect Buffer Configuration:** Providing `biomeInputBufferType` or `biomeDistanceOutputBufferType` that do not match the actual buffer types in the `NBufferBundle` will result in a `ClassCastException` at runtime.

## Data Pipeline

The stage transforms biome map data into biome distance data. The flow is contained entirely within the `run` method's execution.

> Flow:
> NBufferBundle (with Biome Data) -> **NBiomeDistanceStage** -> Iterates each (x, z) column -> Scans neighborhood of biome buffers -> Calculates distance to nearest voxel of each biome -> Writes BiomeDistanceEntries to Output Buffer -> NBufferBundle (with Biome Distance Data)

