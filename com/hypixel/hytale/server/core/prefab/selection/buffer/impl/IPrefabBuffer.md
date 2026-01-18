---
description: Architectural reference for IPrefabBuffer
---

# IPrefabBuffer

**Package:** com.hypixel.hytale.server.core.prefab.selection.buffer.impl
**Type:** Interface / Contract

## Definition
```java
// Signature
public interface IPrefabBuffer {
```

## Architecture & Concepts
The IPrefabBuffer interface defines a contract for an in-memory, read-only representation of a game world structure, known as a Prefab. It serves as a high-performance data source for world generation and structure placement systems.

Its primary architectural purpose is to decouple the complex logic of storing prefab data (which may involve sparse arrays, octrees, or other optimized formats) from the systems that need to consume this data. Instead of exposing internal data structures, it provides a highly controlled, iterator-based access pattern via the forEach and forEachRaw methods.

This component acts as a blueprint. A system, such as a world generator, acquires an IPrefabBuffer for a desired structure (e.g., a dungeon room or a tree). It then uses the buffer's iteration methods to "stamp" the blocks, entities, and fluids defined in the blueprint into the live world data stores, such as the ChunkStore and EntityStore. The interface includes methods for querying the prefab's spatial boundaries and anchor point, which are essential for correct placement and collision detection.

## Lifecycle & Ownership
The lifecycle of an IPrefabBuffer implementation is designed to be transient and explicitly managed to minimize memory overhead and garbage collection pressure.

-   **Creation:** An IPrefabBuffer is not instantiated directly. Concrete implementations are created and returned by higher-level services, such as a PrefabManager or a world generation orchestrator, when a prefab is loaded from a persistent source.
-   **Scope:** The object's lifetime is strictly limited to a single, discrete operation, typically the placement of one prefab instance. It should be considered a short-lived, single-use object.
-   **Destruction:** The consumer of the IPrefabBuffer is **required** to call the release method once the operation is complete. Failure to do so will result in a severe memory leak, as the underlying implementation may hold significant native or pooled resources. The object is invalid after release is called.

## Internal State & Concurrency
-   **State:** As an interface, IPrefabBuffer is stateless. However, any concrete implementation is inherently stateful, containing the complete block, entity, and fluid data for a prefab. The contract implies that this state is immutable after the buffer has been created and handed to a consumer. There are no methods for modifying the buffer's contents.
-   **Thread Safety:** **This interface is not thread-safe.** Implementations are not expected to provide internal synchronization. An IPrefabBuffer instance must be confined to the thread that acquired it. Sharing an instance across multiple threads without external locking will lead to undefined behavior and data corruption. The typical pattern is for a world generation worker thread to acquire, use, and release its own private instance.

## API Surface
The public contract focuses on boundary queries and controlled data iteration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAnchorX() | int | O(1) | Returns the X coordinate of the prefab's anchor point. |
| getMinX(rotation) | int | O(1) | Returns the minimum X boundary of the prefab for a given rotation. |
| getMaxX(rotation) | int | O(1) | Returns the maximum X boundary of the prefab for a given rotation. |
| forEach(...) | void | O(N) | Iterates over all elements in the buffer, invoking consumer callbacks for blocks, entities, and child prefabs. This is the primary method for reading data. |
| forEachRaw(...) | void / boolean | O(N) | A lower-level, higher-performance iteration method that provides raw block data. Predicate-based overloads allow for early termination of the loop. |
| release() | void | O(1) | **CRITICAL.** Releases all resources held by the buffer. Must be called after use to prevent memory leaks. |
| getBlockId(x, y, z) | int | O(1) | Provides direct, random access to a block ID at a specific local coordinate. May be slower than bulk iteration. |

## Integration Patterns

### Standard Usage
The correct pattern involves acquiring the buffer, using it within a try-finally block to guarantee cleanup, and performing the world modification logic inside the provided consumer lambdas.

```java
// A world generator system acquires a buffer for a specific prefab
IPrefabBuffer buffer = prefabManager.getBuffer("hytale:dungeon_room_1");

try {
    // Use a PrefabBufferCall object to pass state into the consumers
    PlacementContext context = new PlacementContext(world, position);

    // The forEach method iterates and calls our lambda for each block
    buffer.forEach(
        IPrefabBuffer.iterateAllColumns(),
        (x, y, z, blockId, chunkStore, ...) -> {
            // Logic to place the block into the actual world
            context.getWorld().setBlock(context.getOriginX() + x, ...);
        },
        (x, z, entities, ...) -> {
            // Logic to spawn entities
        },
        null, // No child consumer
        context
    );
} finally {
    // CRITICAL: Always release the buffer to prevent memory leaks
    buffer.release();
}
```

### Anti-Patterns (Do NOT do this)
-   **Forgetting Release:** The most critical error is failing to call release(). This will cause a memory leak that can crash the server over time. Always use a try-finally block.
-   **Long-Term Storage:** Do not store references to an IPrefabBuffer in a long-lived object like a component or service. They are transient and may be invalidated or recycled by the parent system after release.
-   **Multi-threaded Access:** Never pass an IPrefabBuffer instance to another thread or access it from multiple threads concurrently. All operations on a given instance must be performed by the same thread.

## Data Pipeline
The IPrefabBuffer is a key transformation step, converting static asset data into commands that modify the live game world.

> Flow:
> Prefab Asset (Disk) -> Prefab Loading Service -> **IPrefabBuffer (In-Memory Blueprint)** -> World Generation System (via forEach) -> Live World State (ChunkStore, EntityStore)

