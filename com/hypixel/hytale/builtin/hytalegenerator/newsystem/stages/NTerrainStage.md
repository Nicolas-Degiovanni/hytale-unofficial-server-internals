---
description: Architectural reference for NTerrainStage
---

# NTerrainStage

**Package:** com.hypixel.hytale.builtin.hytalegenerator.newsystem.stages
**Type:** Transient Component

## Definition
```java
// Signature
public class NTerrainStage implements NStage {
```

## Architecture & Concepts

The NTerrainStage is a foundational component within the procedural world generation pipeline, specifically the NStagedChunkGenerator system. Its primary architectural role is to translate abstract, two-dimensional biome layout data into a tangible, three-dimensional voxel terrain. It acts as the bridge between conceptual map features (biomes) and the physical blocks (materials) that constitute the game world.

This stage executes a sophisticated two-phase process for each vertical chunk column it is assigned:

1.  **Density Field Generation:** The stage first computes a continuous scalar field of density values for the target space. A positive density value at a given coordinate indicates the presence of solid matter. This is not a simple binary calculation; the stage reads data from preceding pipeline stages, primarily the biome map and biome distance fields. It uses this information to interpolate density contributions from multiple nearby biomes, creating smooth, naturalistic transitions at biome borders. The `maxInterpolationRadius_voxelGrid` parameter is critical for controlling the smoothness and performance of this blend.

2.  **Materialization:** Once the density field is established, the stage iterates through it. For each coordinate, it consults the dominant biome's MaterialProvider. Using the density value and other contextual data—such as depth from the surface (floor) or distance from a cave ceiling—the MaterialProvider selects the appropriate Material (e.g., stone, dirt, grass). This result is then written into the final output voxel buffer.

This component is heavily optimized for parallel execution by worker threads, employing a thread-local storage pattern for its intermediate density calculations.

### Lifecycle & Ownership

-   **Creation:** An NTerrainStage is not created dynamically per-chunk. It is instantiated once during the configuration and assembly of a parent NStagedChunkGenerator. Its parameters, such as buffer types and interpolation radius, are defined at this time and remain constant for the generator's lifetime.
-   **Scope:** The object persists for the entire lifecycle of the world generator it belongs to. It is stateless between `run` invocations, allowing it to be reused for processing thousands of distinct chunks.
-   **Destruction:** The instance is eligible for garbage collection only when the NStagedChunkGenerator is destroyed, for example, during a server shutdown or a world unload event.

## Internal State & Concurrency

-   **State:** The NTerrainStage instance itself is effectively immutable after construction. Its fields store configuration data provided by the generator builder. However, it manages highly mutable, temporary state during execution. The `densityContainers` field, a WorkerIndexer.Data object, holds pre-allocated `FloatContainer3d` buffers—one for each potential worker thread. This intermediate buffer is cleared and reused for each chunk processed by a given thread.

-   **Thread Safety:** **Conditionally Thread-Safe**. The class is explicitly designed for concurrent execution by the NStagedChunkGenerator framework. The use of `WorkerIndexer.Data` to provide a thread-local density buffer is the key mechanism that prevents race conditions. Each worker thread operates on its own private data container. The safety guarantee relies on the assumption that the pipeline runner provides each invocation of the `run` method with a context for a distinct, non-overlapping region of the world. Direct, uncontrolled calls to `run` from multiple threads without this partitioning will lead to undefined behavior.

## API Surface

The public API is defined by the NStage interface, which dictates its interaction with the pipeline runner.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run(Context context) | void | O(N) | Executes the full terrain generation process for the chunk defined in the context. N is the number of voxels in the chunk. This is the primary entry point. |
| getInputTypesAndBounds_bufferGrid() | Map | O(1) | Declares the stage's data dependencies (Biome and BiomeDistance buffers) to the pipeline runner, allowing it to provision the necessary data. |
| getOutputTypes() | List | O(1) | Declares the stage's data product (the Material voxel buffer). |
| getName() | String | O(1) | Returns the unique name of this stage instance. |

## Integration Patterns

### Standard Usage

This class is not intended for direct invocation. It is designed to be registered as a component within a larger generation pipeline, which manages its lifecycle and execution.

```java
// Conceptual example of registering the stage
NStagedChunkGenerator.Builder builder = new NStagedChunkGenerator.Builder();

// ... other stages (e.g., NBiomeStage, NBiomeDistanceStage) are added first

builder.addStage(new NTerrainStage(
    "CoreTerrain",
    BIOME_BUFFER_TYPE,
    BIOME_DISTANCE_BUFFER_TYPE,
    MATERIAL_OUTPUT_BUFFER_TYPE,
    16, // maxInterpolationRadius_voxelGrid
    materialCache,
    workerIndexer
));

NStagedChunkGenerator generator = builder.build();
// The generator now owns and executes the NTerrainStage internally.
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation and Invocation:** Do not create an instance with `new` and call `run` manually. The `NStage.Context` object is complex and must be correctly constructed by the pipeline runner to ensure proper buffer access and thread safety.
-   **State Leakage:** Do not attempt to access or modify the internal `densityContainers` from outside the `run` method. These are thread-local and managed exclusively by the stage's internal logic.
-   **Incorrect Stage Ordering:** This stage critically depends on the outputs of biome and biome distance stages. Placing it before them in the pipeline will result in runtime errors or incorrect world generation.

## Data Pipeline

The flow of data through this stage involves transformation from 2D column data to a temporary 3D density field, and finally to a 3D voxel buffer.

> Flow:
> NCountedPixelBuffer<BiomeType> -> **NTerrainStage (Density Calculation)** -> Thread-Local FloatContainer3d (Density Field) -> **NTerrainStage (Materialization)** -> NVoxelBuffer<Material>

