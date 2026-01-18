---
description: Architectural reference for BlockData
---

# BlockData

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient Data Container

## Definition
```java
// Signature
public class BlockData {
```

## Architecture & Concepts
The BlockData class is a high-performance, mutable data container that represents the fully resolved state of a single block for physics and collision calculations. It is not a persistent entity but rather a transient snapshot of a block's properties at a specific world coordinate.

This class acts as a crucial bridge between raw world data (like block and fluid IDs) and the systems that need to interact with them. It aggregates primitive identifiers, resolved asset references (BlockType, Fluid), and derived state (collisionMaterials, fillHeight) into a single, optimized structure.

Its design, featuring explicit `assign` and `clear` methods, strongly indicates its use within an object pooling system. This pattern is critical for minimizing garbage collection overhead in performance-sensitive loops, such as the server's main physics tick. The engine acquires a BlockData instance, populates it for a specific query, performs calculations, and then releases it back to the pool.

## Lifecycle & Ownership
- **Creation:** Instances are not intended for direct instantiation by developers. They are created and managed by an internal object pool, likely initialized once when the collision module starts.
- **Scope:** The *data* within a BlockData instance is extremely short-lived, valid only for the duration of a single, atomic operation (e.g., one entity's collision check against one block). The *object itself*, however, persists for the entire server session within its management pool.
- **Destruction:** The object is never truly destroyed until server shutdown. Instead, its `clear` method is invoked upon release to the pool, resetting its state and making it available for immediate reuse.

## Internal State & Concurrency
- **State:** Highly **Mutable**. The object is explicitly designed to be modified repeatedly. It also features lazy-initialization for the BlockBoundingBoxes field, which acts as a micro-optimization to defer asset loading until absolutely necessary.
- **Thread Safety:** This class is **not thread-safe**. It is designed for single-threaded access within a tight loop. Passing a BlockData instance between threads without external locking or creating a deep copy will result in race conditions and severe data corruption. It is expected that a thread acquires an instance from a pool, uses it exclusively, and then releases it.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| assign(BlockData other) | void | O(1) | Performs a shallow copy of all fields from another instance. Used for state duplication. |
| clear() | void | O(1) | Resets all fields to default values. Critical for object pool integration. |
| isFiller() | boolean | O(1) | Returns true if this block is a non-origin part of a multi-block structure. |
| originX(int x) | int | O(1) | Calculates the world X coordinate of the origin block for a multi-block structure. |
| getBlockBoundingBoxes() | BlockBoundingBoxes | O(1) Amortized | Retrieves the collision hitbox. Lazily loads and caches the asset on first call. |
| getBlockDamage() | int | O(1) | Calculates the total potential damage from the block and any fluid it contains. |

## Integration Patterns

### Standard Usage
BlockData should never be instantiated directly. It is acquired from a pool, populated by a world-aware service, used for immediate calculations, and then promptly released.

```java
// Hypothetical usage within a physics engine
BlockData pooledData = context.getService(BlockDataPool.class).acquire();

try {
    // A world service populates the transient object with data for a specific coordinate
    worldQueryService.resolveBlockAt(x, y, z, pooledData);

    if (pooledData.isTrigger()) {
        eventBus.post(new EntityEnteredTriggerEvent(entity, pooledData));
    }

    // ... further physics calculations
} finally {
    // CRITICAL: The object must be returned to the pool to prevent leaks.
    context.getService(BlockDataPool.class).release(pooledData);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockData()`. This bypasses the object pool, creating GC pressure and degrading server performance.
- **Long-Term Storage:** Do not store a reference to a BlockData instance in a component or entity. The object you are holding is transient and will be cleared and reused by another system, leading to unpredictable behavior and bugs that are difficult to trace.
- **Asynchronous Access:** Do not use a BlockData instance across different threads or in an asynchronous callback without deep-copying its contents into a stable structure.

## Data Pipeline
BlockData is the output of the world data resolution stage and the input for the collision and physics simulation stage.

> Flow:
> Raw Chunk Data (int blockId) -> World Query Service -> **BlockData** (populated with resolved BlockType, Fluid, etc.) -> Collision Engine -> Physics Resolution

