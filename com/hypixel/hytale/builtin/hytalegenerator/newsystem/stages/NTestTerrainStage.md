---
description: Architectural reference for NTestTerrainStage
---

# NTestTerrainStage

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.stages
**Type:** Transient Component

## Definition
```java
// Signature
public class NTestTerrainStage implements NStage {
```

## Architecture & Concepts
NTestTerrainStage is a concrete implementation of the NStage interface, serving as a foundational *Generator Stage* within the procedural world generation pipeline. Its primary architectural role is to bootstrap the terrain creation process by populating a designated voxel buffer with an initial, simple landscape based on a deterministic noise algorithm.

This class embodies the concept of a "source node" in a directed acyclic graph (DAG) of generation tasks. By declaring zero input dependencies, it signals to the NStage execution engine that it can be run at the beginning of a generation pass without waiting for any preceding stages. Its output, a buffer filled with solid and empty materials, serves as the raw input for subsequent stages like cave carvers, ore placers, or biome-specific modifiers.

The core logic uses SimplexNoise to produce a continuous field of values, which are then thresholded against the vertical position to create a basic ground plane with rolling hills. This is a common and computationally efficient technique for establishing the macro-structure of a game world.

## Lifecycle & Ownership
-   **Creation:** An NTestTerrainStage is instantiated and configured by a higher-level authority, typically a world generator factory or a stage graph builder. It is not intended for direct instantiation by game logic. The constructor requires all dependencies, including the target buffer type and the materials to use, ensuring an instance is fully configured and immutable upon creation.
-   **Scope:** The lifetime of an instance is ephemeral, strictly bound to the execution of a single world generation task for a specific world chunk or region. It is not a long-lived service or a singleton.
-   **Destruction:** The object holds no persistent resources and is eligible for garbage collection as soon as the stage graph that contains it has completed execution and is dereferenced by the generation scheduler.

## Internal State & Concurrency
-   **State:** The internal state of an NTestTerrainStage instance is **immutable**. The output buffer type and the ground and empty materials are final fields set during construction. The `run` method is stateful only within its own execution scope; it does not modify the instance's internal fields.
-   **Thread Safety:** The instance itself is inherently thread-safe due to its immutable state. The `run` method, however, is designed for execution by a single thread within a given context. The world generation system guarantees thread safety by providing each stage with exclusive or properly synchronized access to its declared buffer regions.

    **Warning:** While multiple NTestTerrainStage instances can be executed concurrently on separate threads, they must operate on distinct, non-overlapping buffer regions to prevent data corruption. The NStage scheduler is responsible for enforcing this boundary.

## API Surface
The public contract is defined by the NStage interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run(Context context) | void | O(N) | Executes the terrain generation logic. N is the number of voxels in the target buffer region. This method is the computational core of the class. |
| getInputTypesAndBounds_bufferGrid() | Map | O(1) | Declares data dependencies. Critically, this returns an empty map, identifying the stage as a generator. |
| getOutputTypes() | List | O(1) | Declares the buffer type this stage is responsible for populating. |
| getName() | String | O(1) | Provides a human-readable identifier for logging and debugging. |

## Integration Patterns

### Standard Usage
NTestTerrainStage is not used in isolation. It is designed to be composed into a larger pipeline or graph of NStage objects. The generation system orchestrates its execution.

```java
// Conceptual example of pipeline assembly
NStageGraphBuilder builder = new NStageGraphBuilder();
NBufferType materialBufferType = NBufferTypes.SOLID_MATERIAL;

// The stage is configured and added to the graph.
// The graph runner will later invoke run() with a valid context.
builder.addStage(new NTestTerrainStage(
    materialBufferType,
    KnownMaterials.STONE,
    KnownMaterials.AIR
));

// ... add other stages that consume the material buffer
builder.addStage(new NCaveCarverStage(materialBufferType, ...));

NStageGraph graph = builder.build();
```

### Anti-Patterns (Do NOT do this)
-   **Direct Invocation:** Never call the `run` method directly. The `NStage.Context` object it requires is a complex dependency supplied exclusively by the generation scheduler. Bypassing the scheduler will break buffer access, dependency resolution, and thread management.
-   **State Reuse:** Do not attempt to modify an NTestTerrainStage instance after creation. It is configured to be immutable. If a different configuration is needed, create a new instance.

## Data Pipeline
As a generator stage, NTestTerrainStage originates a data flow. It does not consume data from other stages.

> Flow:
> SimplexNoise Algorithm -> **NTestTerrainStage** -> NVoxelBuffer (of SolidMaterial) -> Subsequent Stages (e.g., NCaveCarverStage, NOrePlacementStage)

