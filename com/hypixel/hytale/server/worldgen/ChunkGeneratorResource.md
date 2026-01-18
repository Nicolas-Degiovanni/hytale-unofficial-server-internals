---
description: Architectural reference for ChunkGeneratorResource
---

# ChunkGeneratorResource

**Package:** com.hypixel.hytale.server.worldgen
**Type:** Transient Resource Object

## Definition
```java
// Signature
public class ChunkGeneratorResource {
```

## Architecture & Concepts

The ChunkGeneratorResource is a stateful context object that serves as a "scratchpad" for the server-side world generation pipeline. Its primary architectural purpose is to aggregate all necessary buffers, caches, and mutable state required for the procedural generation of a single game chunk. This pattern is critical for performance, as it avoids repeated object allocation and garbage collection overhead in the computationally expensive world generation loop.

This class is not a standalone service. It is a transient, reusable container that is tightly coupled with a parent ChunkGenerator. It encapsulates the volatile state of a single generation task, including random number generators, coordinate caches, result buffers for noise calculations, and a dedicated cache for world prefabs. By bundling these resources, the engine can pass a single, coherent context object through the various stages of generation, from terrain heightmap calculation to prefab population.

**WARNING:** This object is fundamentally a collection of mutable buffers. It is designed for a single, specific purpose and should not be treated as a general-purpose data structure.

### Lifecycle & Ownership

-   **Creation:** Instances are created by a higher-level system, typically a worker pool managed by the ChunkGenerator. The public constructor exists for instantiation by the pool, not for direct use by feature developers. After construction, the `init` method must be called to link the resource to its parent ChunkGenerator, which provides access to global services like the master prefab cache.
-   **Scope:** The lifetime of a ChunkGeneratorResource is scoped to a single, discrete world generation task. A worker thread acquires an instance from a pool, uses it to generate a chunk, and then releases it back to the pool. It is a short-lived object from the perspective of any single task.
-   **Destruction:** The `release` method is the explicit cleanup hook. It is critical that this method is called when a generation task is complete. Its primary function is to shut down the internal `TimeoutCache` for prefabs, which in turn releases expensive `IPrefabBuffer` resources. Failure to call `release` will result in a severe memory leak.

## Internal State & Concurrency

-   **State:** This class is highly mutable by design. Nearly every field is a reusable buffer, cache, or stateful utility object (e.g., `BlockPriorityChunk`, `ResultBuffer3d`, `random`). The state is reset or overwritten at the beginning of each generation task. The internal `prefabs` cache is a notable piece of state, providing a short-term, 30-second cache for frequently accessed prefabs within a localized generation area.

-   **Thread Safety:** **This class is not thread-safe.** It is designed to be confined to a single worker thread for the duration of a generation task. Its internal state, especially the random number generators and data buffers, is not protected by locks or other synchronization primitives. Sharing an instance across multiple threads will lead to race conditions, data corruption, and non-deterministic world generation. All access must be externally synchronized, which is typically handled by the worker pool that manages these objects.

## API Surface

The public API is minimal, reflecting its role as an internal component. The primary interaction is through its public fields, which are accessed directly by different stages of the world generation algorithm.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(ChunkGenerator) | void | O(1) | Binds this resource to a parent ChunkGenerator. Must be called once after construction. |
| release() | void | O(N) | Releases all held resources, primarily shutting down the internal prefab cache. N is the number of cached prefabs. |
| getRandom() | Random | O(1) | Provides access to the primary Random instance for deterministic generation. |

## Integration Patterns

### Standard Usage

The intended use is within a try-with-resources or a similar pooling pattern where acquisition and release are guaranteed. The object is acquired, used to perform a series of generation steps, and then released.

```java
// Pseudo-code demonstrating the intended lifecycle
ChunkGeneratorResource resource = generator.getResourcePool().acquire();
try {
    // The resource is now ready for a specific chunk coordinate
    configureResourceForTask(resource, chunkX, chunkZ);

    // Various generation stages access the public buffers and fields
    HeightmapPass.execute(resource.resultBuffer2d, resource.random);
    CavePass.execute(resource.resultBuffer3d, resource.cacheCaveCoordinateKey);
    PopulationPass.execute(resource.priorityChunk, resource.prefabPopulator);

} finally {
    // CRITICAL: The resource must be released to prevent leaks
    generator.getResourcePool().release(resource);
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new ChunkGeneratorResource()` in feature code. The object is useless until `init` is called, and its lifecycle must be managed by the world generation engine's resource pool.
-   **Forgetting Release:** Failure to call `release()` (or return the object to its pool) is a critical resource leak. The `IPrefabBuffer` objects held by the internal `TimeoutCache` will not be deallocated, leading to rapid memory exhaustion.
-   **Cross-Thread Sharing:** Never pass an instance of this class between threads. The mutable, unsynchronized state will be corrupted immediately.
-   **Long-Term Storage:** Do not store a reference to a ChunkGeneratorResource in a long-lived object. These are transient resources that are recycled. A stored reference may point to an object that has been reconfigured for a completely different task, leading to unpredictable behavior.

## Data Pipeline

ChunkGeneratorResource does not define a data pipeline itself, but it serves as the central state container *within* the larger chunk generation pipeline.

> Flow:
> Chunk Generation Task -> Worker Thread acquires **ChunkGeneratorResource** -> Generation stages read/write to its internal buffers (e.g., `resultBuffer2d`, `priorityChunk`) -> Prefab data is loaded into its `prefabs` cache -> Final block data is assembled in `priorityChunk` -> **ChunkGeneratorResource** is released back to pool.

