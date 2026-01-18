---
description: Architectural reference for CollisionConfig
---

# CollisionConfig

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Reusable State Object

## Definition
```java
// Signature
public class CollisionConfig {
```

## Architecture & Concepts

The CollisionConfig class is a high-performance, mutable state container designed for use within the server's core physics and collision detection systems. It is not a service or manager, but rather a specialized parameter object that encapsulates both the configuration for a collision query and the live results from that query.

Its primary purpose is to minimize object allocation and garbage collection overhead during performance-critical operations, such as entity movement simulation or raycasting. A single CollisionConfig instance is intended to be reused across hundreds or thousands of individual block checks within a single logical operation.

The core of its functionality lies in the `canCollide` method. This method accepts world coordinates and performs all necessary lookups into the world's data structures (chunks, sections, block data, fluid states) to populate its own fields. It then evaluates these fields against its configured collision masks and predicates to return a simple boolean result. This design centralizes complex world data lookups and collision logic into a single, stateful object, which can be passed efficiently through the collision pipeline.

**WARNING:** This class is fundamentally stateful. The validity of its data fields, such as `blockType` or the result of `getBoundingBox`, is directly tied to the most recent call to `canCollide`. Accessing these fields without first calling `canCollide` will yield stale or uninitialized data.

## Lifecycle & Ownership

-   **Creation:** CollisionConfig instances are not meant to be instantiated directly using the `new` keyword. They are managed by a higher-level system, almost certainly an object pool, which is owned by the primary collision engine. A developer acquires a pre-allocated or new instance from this engine to begin a query.
-   **Scope:** The object's lifetime is scoped to a single, complete collision operation (e.g., one entity's physics tick). During this scope, it is mutated repeatedly as it is used to check a sequence of blocks in the world.
-   **Destruction:** The object is not typically destroyed by the garbage collector. Upon completion of the collision operation, the owner must explicitly call the `clear` method and release the instance back to its managing pool. Failure to do so constitutes a resource leak.

## Internal State & Concurrency

-   **State:** This object is highly mutable. Nearly all of its fields are public or modified as a side effect of method calls, particularly `canCollide`. It aggressively caches references to world data structures like `WorldChunk` and `BlockSection` to accelerate subsequent checks that fall within the same region, a common pattern in entity movement.
-   **Thread Safety:** **This class is not thread-safe and must not be shared across threads.** It is designed exclusively for single-threaded access within a contained physics tick or world query. Concurrent calls to `canCollide` or other state-modifying methods will result in race conditions, data corruption, and undefined behavior. Any system using this class must provide its own synchronization guarantees or, preferably, ensure a thread-per-instance or thread-per-pool model.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setWorld(World world) | void | O(1) | Binds the configuration to a specific world instance. Invalidates internal chunk caches. Must be called before any queries. |
| canCollide(int x, int y, int z) | boolean | O(N) | The primary operational method. Probes the world at the given coordinates, updates all internal state fields, and returns true if a collision is detected based on the current configuration. Complexity depends on chunk cache state. |
| setDefaultCollisionBehaviour() | void | O(1) | Configures the object for standard entity-vs-world collision, typically colliding with solid blocks and checking for damage sources. |
| getBoundingBox() | Box | O(1) | Returns the collision bounding box for the block at the coordinates of the *last* `canCollide` call. |
| clear() | void | O(1) | Resets all internal state, preparing the object for reuse or return to an object pool. |

## Integration Patterns

### Standard Usage

The intended use involves acquiring an instance, configuring it, using it in a loop to check multiple world positions, and finally releasing it.

```java
// Hypothetical CollisionEngine
CollisionConfig config = engine.acquireConfig();
config.setWorld(myWorld);
config.setDefaultCollisionBehaviour();

// Check a 3x3 area around a point
for (int x = -1; x <= 1; x++) {
    for (int z = -1; z <= 1; z++) {
        if (config.canCollide(entity.x + x, entity.y, entity.z + z)) {
            // A collision was found, react to it.
            // The config object now holds all relevant data about the colliding block.
            Box collisionBox = config.getBoundingBox();
            // ... perform physics response
        }
    }
}

// CRITICAL: Release the config back to the pool
engine.releaseConfig(config);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not call `new CollisionConfig()`. This bypasses the object pooling system, leading to significant garbage collection pressure and poor performance in physics-heavy scenarios.
-   **State Mismanagement:** Accessing state like `getBoundingBox` or `blockType` before a successful call to `canCollide` for the desired coordinates. The object's state is only guaranteed to be valid immediately following that call.
-   **Forgetting to Clear:** Failing to call `clear` (or the owning system's release method, which should call `clear`) before returning an instance to a pool. This will cause state from one collision query to leak into a completely separate and unrelated query, causing difficult-to-diagnose bugs.
-   **Concurrent Modification:** Sharing a single CollisionConfig instance across multiple threads. This will lead to catastrophic state corruption.

## Data Pipeline

CollisionConfig acts as a stateful probe rather than a step in a linear data pipeline. Its operational flow involves a query-response loop with the world data storage.

> Flow:
> 1.  **Collision Engine** acquires a `CollisionConfig` instance from an object pool.
> 2.  **Collision Engine** calls `setWorld` and other setup methods to define the query rules.
> 3.  For each block to be tested, the **Collision Engine** calls `canCollide(x, y, z)`.
> 4.  Inside `canCollide`, the **CollisionConfig** queries the `World` -> `ChunkStore` -> `WorldChunk` -> `BlockSection` and `FluidSection` to retrieve raw block, fluid, and rotation data.
> 5.  **CollisionConfig** updates its internal state (`blockType`, `boundingBoxes`, `blockMaterialMask`, etc.) based on the retrieved data.
> 6.  **CollisionConfig** evaluates its collision rules (e.g., `blockMaterialCollisionMask`) against its new internal state and returns a boolean result to the **Collision Engine**.
> 7.  The **Collision Engine** may then read the detailed state (e.g., `getBoundingBox`) from the **CollisionConfig** to perform a physics response.
> 8.  After the entire operation is complete, the **Collision Engine** releases the instance back to the pool, which triggers the `clear` method.

