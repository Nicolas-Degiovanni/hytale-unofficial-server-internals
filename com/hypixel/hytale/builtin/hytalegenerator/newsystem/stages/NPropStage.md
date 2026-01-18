---
description: Architectural reference for NPropStage
---

# NPropStage

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.stages
**Type:** Transient

## Definition
```java
// Signature
public class NPropStage implements NStage {
```

## Architecture & Concepts
The NPropStage is a critical component of the procedural world generation pipeline, responsible for populating the world with discrete objects known as *Props*. These can range from simple environmental decorations like trees and rocks to more complex structures that may include entities.

This class operates as a stateful, single-execution "stage" within the larger `NStage` framework. Its primary function is to read foundational world data—specifically biome layout, biome boundary information, and the existing voxel terrain—and strategically place props based on a complex set of rules.

Architecturally, NPropStage acts as an enrichment layer. It does not generate the base terrain but rather decorates it. It is configured for a specific *runtime index*, allowing the generation system to orchestrate prop placement in a sequence. For example, a first pass (runtime 0) might place large foundational props like ruins, while a subsequent pass (runtime 1) places foliage on top of the terrain and ruins.

The core logic revolves around these key inputs:
*   **Biome Data:** Determines which types of props are allowed in a given column.
*   **Biome Distance Data:** Provides the distance to the nearest biome edge, allowing for effects like thinning forests near a plains biome.
*   **Position Provider:** A configurable strategy (e.g., Poisson disk sampling) that suggests candidate locations for prop placement, preventing uniform or grid-like patterns.
*   **Prop Distribution:** A weighted collection of props that can be selected based on position, worker ID, and other factors.

The stage is non-destructive to its inputs. It reads from input buffers and writes the final, modified world data to distinct output buffers, ensuring a clean data flow between stages.

## Lifecycle & Ownership
-   **Creation:** NPropStage is instantiated by a higher-level world generation orchestrator, often a pipeline builder or a `NStage` factory. The constructor is complex, requiring a complete configuration of input/output buffer types, a material cache, a list of all possible biomes, and a runtime index. This indicates it is a system-level component, not intended for direct user creation.
-   **Scope:** The object's lifetime is scoped to a single world generation task for a specific, bounded region of the world. It is created, its `run` method is invoked exactly once, and then it is discarded.
-   **Destruction:** The instance becomes eligible for garbage collection as soon as the `run` method completes and the reference from the pipeline executor is dropped. It holds no persistent resources that require manual cleanup.

## Internal State & Concurrency
-   **State:** The NPropStage is stateful, but its state is effectively immutable after construction. All configuration fields, such as buffer types and calculated input bounds, are declared `final` and set within the constructor. The `run` method does not modify this internal state; it operates exclusively on the data provided in the `NStage.Context`.
-   **Thread Safety:** The class instance is inherently thread-safe due to its immutable configuration. However, the `run` method is designed for execution by a single worker thread on a specific chunk of data. Concurrency is managed at the pipeline level, where multiple workers may execute separate NPropStage instances on different world regions simultaneously. The `context.workerId` parameter is passed down to prop selection and placement logic, enabling deterministic, yet varied, generation across threads.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run(Context context) | void | O(N * M) | Executes the prop placement logic for the configured region. Complexity depends on the number of candidate positions (N) and the cost of scanning and placing each prop (M). |
| getInputTypesAndBounds_bufferGrid() | Map | O(1) | Returns a map of required input buffer types and the spatial bounds from which this stage needs to read. |
| getOutputTypes() | List | O(1) | Returns a list of the buffer types this stage will write to. |
| getName() | String | O(1) | Returns the developer-assigned name for this stage instance. |

## Integration Patterns

### Standard Usage
The NPropStage is not used in isolation. It is assembled as part of a generation pipeline and executed by a stage runner. The runner is responsible for providing the `NStage.Context`, which contains access to the necessary data buffers.

```java
// Simplified conceptual example of a pipeline runner
NStage propStage = new NPropStage(
    "forest-trees",
    biomeBufferType,
    biomeDistBufferType,
    /* ... other configuration ... */
);

NStage.Context context = createGenerationContextForChunk(chunkPos);
propStage.run(context); // Executes the prop placement
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Avoid constructing NPropStage manually unless you are building a custom generation pipeline. The constructor's complexity implies it should be managed by a factory or builder that understands the relationships between different stages and buffer types.
-   **Instance Re-use:** An NPropStage instance is configured for a specific task and is not designed to be re-used. Calling the `run` method more than once will result in undefined behavior. A new instance should be created for each execution.
-   **Ignoring Dependencies:** This stage depends on prior stages (like `NBiomeStage` and `NBiomeDistanceStage`) to have run and populated the input buffers. Executing it out of order will lead to missing data and failed generation.

## Data Pipeline
The flow of data through the NPropStage is a multi-step process that transforms a base landscape into a decorated one.

> Flow:
> NBufferBundle (Input) -> **NPropStage** -> NBufferBundle (Output)

1.  **Initialization:** The stage begins by creating views into the required input buffers (Biome, Biome Distance, Material, and optionally Entity) and output buffers (Material and Entity). It defensively copies the input material and entity data to the output buffers, ensuring that any areas not touched by this stage remain intact.
2.  **Biome Analysis:** It scans the entire input region to build a set of all `BiomeType`s present. This optimizes the next step by avoiding iteration over props for biomes that do not exist in the current chunk.
3.  **Prop Field Aggregation:** It iterates through the present biomes and collects all `PropField`s that match the stage's configured `runtimeIndex`.
4.  **Position Generation:** For each collected `PropField`, it invokes a `PositionProvider`. This provider generates a series of potential 3D coordinates where a prop could be placed within the stage's bounds.
5.  **Candidate Validation & Placement:** For each generated position, the following sequence occurs:
    a. The biome at the candidate position is checked to ensure it matches the prop's intended biome.
    b. The distance to the nearest biome edge is retrieved.
    c. A specific `Prop` is selected from the `PropField`'s distribution, using the position and biome distance as inputs.
    d. The `Prop`'s `scan` method is called, which reads the `materialInputSpace` to determine if the location is suitable (e.g., checking for solid ground, avoiding water).
    e. If the scan is successful, the `Prop`'s `place` method is called, which writes voxel and entity data into the `materialOutputSpace` and `entityOutputSpace`.
6.  **Finalization:** Once all `PropField`s have been processed, the `run` method completes. The output buffers in the `NBufferBundle` now contain the original terrain plus all the props placed by this stage.

