---
description: Architectural reference for BlockDataProvider
---

# BlockDataProvider

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient

## Definition
```java
// Signature
public class BlockDataProvider extends BlockData {
```

## Architecture & Concepts
The BlockDataProvider serves as a high-performance, stateful cursor for querying block and fluid information at specific world coordinates. It is a critical component in the collision and physics systems, designed to abstract the underlying complexity of the server's chunked world storage.

Its primary architectural function is to act as a bridge between raw world data structures—such as World, WorldChunk, BlockSection, and FluidSection—and any system that requires a simplified, unified view of a block's properties.

To achieve high performance, the BlockDataProvider implements an aggressive caching strategy based on spatial locality. It maintains references to the last accessed WorldChunk and BlockSection. Subsequent reads for coordinates within the same chunk or section bypass expensive lookups, making it highly efficient for localized operations like entity physics updates or iterative raycasts. This design assumes that queries will frequently be clustered together in space.

## Lifecycle & Ownership
The BlockDataProvider is designed as a reusable, short-lived object, likely managed by an object pool to reduce garbage collection overhead. Its lifecycle is strictly defined and must be adhered to by the calling system.

-   **Creation:** Instances should not be created directly via the constructor. They are typically acquired from a higher-level system or pool. The object is considered inert until `initialize` is called.
-   **Scope:** The scope of an initialized BlockDataProvider is intended to be brief, often confined to a single method or a single game tick operation. It is initialized with a reference to a World and is used to perform a series of `read` operations.
-   **Destruction:** The `cleanup` method must be invoked when the object is no longer needed. This is a critical step that nullifies internal references to the World, WorldChunk, and other heavy objects, preventing memory leaks and preparing the instance for reuse or garbage collection. Failure to call `cleanup` will result in a hard-to-trace resource leak.

## Internal State & Concurrency
-   **State:** The BlockDataProvider is highly **mutable**. Its core purpose is to mutate its internal state (fields inherited from BlockData, such as blockId, fluidId, and blockType) with each call to the `read` method. It also caches references to world data structures like `chunk` and `chunkSection` to optimize performance.

-   **Thread Safety:** This class is **not thread-safe** and must be confined to a single thread. Its internal state, particularly the cached chunk and section references, would be corrupted by concurrent calls to `read` from different threads. Any system using BlockDataProvider must ensure that each instance is accessed by only one thread at a time.

## API Surface
The public API is minimal, reflecting its specialized role as a data accessor. All data is read into public fields inherited from the BlockData parent class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initialize(World world) | void | O(1) | Prepares the provider for use. Binds it to a specific World instance and resets internal state. |
| read(int x, int y, int z) | void | O(1) amortized | Reads all block and fluid data at the given world coordinates, populating the object's fields. Complexity is low if the chunk is cached, higher otherwise. |
| cleanup() | void | O(1) | Releases all references to world data structures. Must be called to prevent memory leaks. |

## Integration Patterns

### Standard Usage
The correct usage pattern involves acquiring an instance, initializing it, performing a series of reads, and ensuring cleanup is always called, typically within a try-finally block.

```java
// The provider is typically acquired from a pool or factory, not created with new.
BlockDataProvider provider = getProviderFromPool();

try {
    // 1. Initialize with the current world context.
    provider.initialize(world);

    // 2. Perform a series of read operations.
    for (Point p : pointsToSample) {
        provider.read(p.x, p.y, p.z);
        // Access data via public fields, e.g., provider.blockId, provider.blockType
        processBlockData(provider);
    }
} finally {
    // 3. CRITICAL: Always clean up to release resources.
    provider.cleanup();
    returnProviderToPool(provider);
}
```

### Anti-Patterns (Do NOT do this)
-   **Forgetting Cleanup:** The most severe anti-pattern is failing to call `cleanup`. This will cause the BlockDataProvider instance to retain a strong reference to the World and its associated chunks, leading to major memory leaks.
-   **Sharing Instances Across Threads:** Using a single instance of BlockDataProvider across multiple threads will lead to race conditions and unpredictable state corruption due to its internal caching. Each thread must have its own instance.
-   **Relying on Stale State:** The internal state of the provider is only valid for the coordinates of the most recent `read` call. Caching the provider itself instead of the data it provides is incorrect.

## Data Pipeline
The BlockDataProvider acts as a transformation stage in the data pipeline, converting raw, chunked storage data into an accessible object model for consumption by game logic.

> Flow:
> World Coordinate (x,y,z) -> **BlockDataProvider.read()** -> [Chunk/Section Lookup & Caching] -> [Raw ID & Metadata Extraction] -> [Asset Map Lookup] -> Populated `BlockData` fields -> Collision Engine / Physics System

