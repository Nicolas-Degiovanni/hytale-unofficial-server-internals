---
description: Architectural reference for FloodLightCalculation
---

# FloodLightCalculation

**Package:** com.hypixel.hytale.server.core.universe.world.lighting
**Type:** Transient

## Definition
```java
// Signature
public class FloodLightCalculation implements LightCalculation {
```

## Architecture & Concepts
The FloodLightCalculation class is the primary implementation of the server's lighting engine, responsible for simulating the propagation of both skylight and block-emitted light through the world. It operates as a core component of the ChunkLightingManager, executing a sophisticated two-phase flood-fill algorithm on discrete 32x32x32 block volumes known as chunk sections.

This system is designed for incremental, asynchronous updates. Rather than recalculating the entire world's lighting on every change, it precisely invalidates and requeues only the affected chunk sections and their immediate neighbors.

The calculation process is divided into two distinct, sequential phases for each chunk section:

1.  **Local Light Calculation:** This phase computes the initial light values *within* a section. It determines which blocks are exposed to the sky and assigns them the maximum skylight value. It also identifies blocks that emit their own light (e.g., torches, lava) and seeds the light map with their values. This phase is self-contained and does not consider light from adjacent sections.

2.  **Global Light Propagation:** Once a section's local light is stable, this phase calculates how light "bleeds" into it from its 26 neighbors (sides, edges, and corners). It reads the final local light values from adjacent sections, attenuates them based on distance and block opacity, and propagates them into the current section. This ensures seamless lighting across chunk boundaries.

The algorithm heavily relies on a LocalCachedChunkAccessor to efficiently and safely access data from the target chunk section and its neighbors, preventing expensive lookups during the intensive flood-fill process.

### Lifecycle & Ownership
- **Creation:** An instance of FloodLightCalculation is created and owned exclusively by the ChunkLightingManager upon its initialization. It is a mandatory, tightly-coupled component of the server's lighting subsystem.
- **Scope:** The object's lifetime is bound directly to its parent ChunkLightingManager. It persists as long as the lighting system is active for a given World instance.
- **Destruction:** The object is eligible for garbage collection when the World, and consequently its ChunkLightingManager, is unloaded. It does not manage any native resources and has no explicit destruction method.

## Internal State & Concurrency
- **State:** FloodLightCalculation is stateful. It maintains a direct reference to its parent ChunkLightingManager for context. It also contains several AverageCollector instances for performance metric tracking and a `fromSections` 2D array. This array serves as a short-lived cache during the global light phase to hold references to neighboring chunk sections, optimizing data access.

- **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded execution within the lighting system's worker pool. Methods like `calculateLight` are expected to be called by a managed thread. The implementation contains world thread assertions (`debugAssertInTickingThread`) and blocking calls (`join()`) that assume a specific threading model controlled by the ChunkLightingManager.

**WARNING:** Concurrent calls to `calculateLight` on the same instance will result in race conditions on the internal `fromSections` cache and corrupt performance metrics. All interaction must be serialized through the ChunkLightingManager's work queue.

## API Surface
The public API is designed for internal use by the ChunkLightingManager. Direct invocation by other systems is a design violation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init(WorldChunk chunk) | void | O(1) | Initializes and queues a newly loaded chunk and its neighbors for lighting calculation. |
| calculateLight(Vector3i pos) | CalculationResult | O(N) | Executes the full two-phase lighting calculation for a single chunk section. N is the number of blocks in the section. |
| invalidateLightAtBlock(...) | boolean | O(1) | Invalidates lighting for a vertical column of sections when a single block changes. This is the primary entry point for world updates. |
| invalidateLightInChunkSections(...) | boolean | O(C) | Invalidates a contiguous vertical range of chunk sections. C is the number of sections in the range. |

## Integration Patterns

### Standard Usage
Developers should never interact with this class directly. The system is driven entirely by world modifications. The ChunkLightingManager orchestrates the entire process.

```java
// This is a conceptual example of how the ChunkLightingManager uses this class.
// DO NOT CALL THIS DIRECTLY.

// 1. A block is changed in the world.
world.setBlock(x, y, z, newBlock);

// 2. The world change triggers an invalidation call.
// This is the ONLY supported way to interact with the lighting system.
lightCalculation.invalidateLightAtBlock(chunk, x, y, z, ...);

// 3. The ChunkLightingManager's worker thread later dequeues the task.
Vector3i sectionToLight = lightingQueue.poll();
if (sectionToLight != null) {
    lightCalculation.calculateLight(sectionToLight);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new FloodLightCalculation()`. The instance is managed by the ChunkLightingManager and requires its context to function correctly.
- **Concurrent Execution:** Do not call `calculateLight` from multiple threads on the same instance. This will lead to data corruption.
- **Manual Invalidation:** Avoid calling `invalidateLightAtBlock` or related methods without a corresponding, preceding block update in the world. Doing so creates unnecessary work and can lead to desynchronization between the block state and the lighting state.

## Data Pipeline
The flow of data for a single block update demonstrates how FloodLightCalculation fits into the broader server architecture.

> Flow:
> World::setBlock -> **FloodLightCalculation::invalidateLightAtBlock** -> BlockSection::invalidateLocalLight -> ChunkLightingManager Work Queue -> **FloodLightCalculation::calculateLight** -> LocalCachedChunkAccessor (Data Fetch) -> ChunkLightDataBuilder (Computation) -> BlockSection::setLocalLight / setGlobalLight (Data Commit) -> BlockChunk::invalidateChunkSection (Client Update)

