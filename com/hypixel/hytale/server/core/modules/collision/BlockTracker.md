---
description: Architectural reference for BlockTracker
---

# BlockTracker

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient State Object

## Definition
```java
// Signature
public class BlockTracker implements IBlockTracker {
```

## Architecture & Concepts
The BlockTracker is a low-level, performance-oriented data structure designed to maintain a small, dynamic collection of unique block coordinates. It resides within the server's collision module and serves as a specialized, high-speed alternative to a generic Set of Vector3i objects.

Its primary architectural goal is to minimize garbage collection pressure during intensive physics calculations. This is achieved through two key strategies:
1.  **Array Pooling:** It pre-allocates and reuses Vector3i objects within its internal array, avoiding the constant creation and destruction of vector objects during track and untrack operations.
2.  **Contiguous Memory:** It uses a simple array, which provides better memory locality compared to node-based structures like a HashSet, potentially improving cache performance for the small datasets it is designed to handle.

This class is not a general-purpose collection. It is optimized for scenarios where the number of tracked blocks is small and the collection is cleared and reused frequently, such as tracking the blocks an entity is colliding with during a single game tick.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly via its public constructor: `new BlockTracker()`. It is expected to be created by a higher-level system that requires temporary, localized block coordinate tracking, such as a CollisionResolver or an EntityPhysicsState manager.
-   **Scope:** The lifetime of a BlockTracker instance is explicitly tied to its owner. It is designed to be short-lived or reused across operations. The presence of the `reset` method strongly indicates an intended pattern of create-use-reset-reuse within a single-threaded loop, like the main server tick.
-   **Destruction:** The object is reclaimed by the Java garbage collector when all references to it are released. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
-   **State:** The BlockTracker is a highly mutable, stateful object. Its core state consists of an array of Vector3i objects and an integer `count` that tracks the number of active elements. The internal array grows dynamically to accommodate new blocks.

-   **Thread Safety:** **This class is not thread-safe.** All methods that modify its internal state (positions array and count) are unsynchronized. Concurrent access from multiple threads will lead to race conditions, inconsistent state, `ArrayIndexOutOfBoundsException`, and other undefined behavior. All interactions with a BlockTracker instance **must** be confined to a single thread.

## API Surface
The public API is designed for high-frequency add, remove, and check operations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| track(x, y, z) | boolean | O(N) | Adds the block if not present. Returns true if the block was already tracked. Complexity is driven by the internal `isTracked` linear search. |
| trackNew(x, y, z) | void | Amortized O(1) | Unconditionally adds a new block coordinate. This is O(N) during array reallocation but O(1) otherwise. |
| untrack(x, y, z) | void | O(N) | Removes a block by coordinate. Involves a linear search to find the index, followed by an array copy. |
| isTracked(x, y, z) | boolean | O(N) | Performs a linear search to check if a coordinate is currently tracked. |
| reset() | void | O(1) | Clears the tracker by setting its count to zero. The internal array and its Vector3i objects are preserved for reuse. |
| getCount() | int | O(1) | Returns the number of currently tracked blocks. |

**Warning:** The O(N) complexity of core operations like `track` and `isTracked` makes this class unsuitable for tracking a large number of blocks. Performance will degrade linearly as the count of tracked blocks increases.

## Integration Patterns

### Standard Usage
The intended use is to instantiate a BlockTracker for a specific, scoped task, perform operations, and then reset it for the next task. This is common within a game loop for per-entity or per-tick calculations.

```java
// A physics system uses a BlockTracker to resolve collisions for one entity
BlockTracker collisionBlocks = new BlockTracker();

// In the update loop for a specific entity:
collisionBlocks.reset(); // Prepare for the new tick

for (Block potentialCollision : getNearbyBlocks()) {
    if (entity.isCollidingWith(potentialCollision)) {
        collisionBlocks.trackNew(
            potentialCollision.getX(),
            potentialCollision.getY(),
            potentialCollision.getZ()
        );
    }
}

// ... use the populated collisionBlocks to apply physics responses
```

### Anti-Patterns (Do NOT do this)
-   **Sharing Instances:** Do not share a single BlockTracker instance across different threads or for unrelated contexts (e.g., for two different entities' physics updates) without explicit `reset` calls. This will cause state to leak between computations.
-   **Concurrent Modification:** Never read from a BlockTracker in one thread while another thread is writing to it. This will corrupt its internal state. All access must be externally synchronized or, preferably, confined to a single thread.
-   **Large-Scale Tracking:** Do not use this class to track thousands of block positions. The linear-scan search algorithm is inefficient for large N. For larger datasets, a `HashSet<Vector3i>` or a more specialized spatial partitioning structure like an Octree should be used.

## Data Pipeline
BlockTracker does not act as a pipeline component itself, but rather as a stateful "bucket" used during a step in a larger data processing flow, such as collision resolution.

> Flow:
> Entity Physics Tick -> Broad-Phase Check (Finds candidate blocks) -> **BlockTracker (Populated with candidates)** -> Narrow-Phase Resolution (Uses BlockTracker data) -> Apply Physics Forces

