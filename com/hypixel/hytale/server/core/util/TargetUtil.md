---
description: Architectural reference for TargetUtil
---

# TargetUtil

**Package:** com.hypixel.hytale.server.core.util
**Type:** Utility

## Definition
```java
// Signature
public final class TargetUtil {
```

## Architecture & Concepts
TargetUtil is a static, server-side utility class that provides a suite of functions for spatial querying and raycasting within the game world. It serves as the primary interface for systems that need to determine line-of-sight, identify the block or entity a player is aiming at, or collect all entities within a specific geometric volume.

This class is a critical component of server logic, bridging high-level gameplay systems (like AI, player interaction, and combat) with low-level world data structures. It directly interacts with the world's chunk storage via `ChunkStore` and the entity spatial partitioning system via `SpatialResource`. Its design emphasizes performance by operating directly on world data and leveraging efficient iteration patterns like `BlockIterator`.

TargetUtil is fundamentally a stateless computational library; it does not manage or own any persistent data. Its methods are pure functions that take the current world state as input and produce a result without side effects.

## Lifecycle & Ownership
- **Creation:** As a final class with only static methods, TargetUtil is never instantiated. Its methods are loaded by the JVM ClassLoader and become available when the class is first referenced.
- **Scope:** The utility's methods are available for the entire lifetime of the server application.
- **Destruction:** The class is unloaded when the JVM shuts down. There is no instance-level cleanup.

## Internal State & Concurrency
- **State:** TargetUtil is entirely stateless. It holds no instance or static fields that represent world state. All necessary context, such as the `World` or `ComponentAccessor`, is passed as arguments to its methods. Internal helper classes like `TargetBuffer` are instantiated per-call and are confined to the method's stack frame, ensuring no state leaks between invocations.

- **Thread Safety:** The methods in TargetUtil are **not** inherently thread-safe. They are safe only if the underlying data sources (`World`, `ComponentAccessor`) are accessed in a thread-safe manner. The caller is responsible for ensuring that no other thread is modifying the world's chunk or entity data while a raycast or spatial query is in progress.

    - **Warning:** Calling these methods with a `ComponentAccessor` on a thread other than the main server tick thread will lead to severe concurrency issues, including crashes and data corruption.

    - The `getAllEntitiesIn...` methods utilize `SpatialResource.getThreadLocalReferenceList()`, a deliberate pattern to avoid heap allocations and collection contention in a multi-threaded environment by using a pre-allocated list that is unique to the calling thread.

## API Surface
The API provides two main categories of functionality: raycasting (getTarget...) and volumetric queries (getAllEntitiesIn...).

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTargetBlock(...) | Vector3i | O(N) | Performs a raycast against the world's block grid. N is proportional to maxDistance. Returns the integer coordinates of the first non-air block that satisfies the predicate. |
| getTargetLocation(...) | Vector3d | O(N) | Similar to getTargetBlock, but returns the precise floating-point intersection point on the face of the block. N is proportional to maxDistance. |
| getTargetEntity(...) | Ref | O(M) | Finds the first entity intersected by an entity's line-of-sight. M is the number of entities within the initial sphere check. Involves a broad-phase sphere check followed by a narrow-phase ray-AABB intersection test for each candidate. |
| getAllEntitiesInSphere(...) | List | O(M) | Collects all entities within a spherical volume. M is the number of entities returned. Relies on the efficiency of the underlying SpatialResource data structure. |
| getAllEntitiesInCylinder(...) | List | O(M) | Collects all entities within a cylindrical volume. M is the number of entities returned. |
| getAllEntitiesInBox(...) | List | O(M) | Collects all entities within an axis-aligned bounding box. M is the number of entities returned. |

## Integration Patterns

### Standard Usage
TargetUtil should be invoked from server-side systems that require world-aware targeting information. The most common pattern involves retrieving the necessary `World` and `ComponentAccessor` from the current execution context and passing them to the desired static method.

```java
// Example: A server-side AI system determining if it can see a block
public void aiBehaviorTick(Ref<EntityStore> self, ComponentAccessor<EntityStore> accessor) {
    World world = accessor.getExternalData().getWorld();
    
    // Define a predicate for what constitutes a valid target block
    BiIntPredicate targetPredicate = (blockId, fluidId) -> {
        return blockId == BlockTypes.STONE;
    };

    // Get the entity's look transform
    Transform look = TargetUtil.getLook(self, accessor);
    Vector3d pos = look.getPosition();
    Vector3d dir = look.getDirection();

    // Perform the raycast
    Vector3i targetBlock = TargetUtil.getTargetBlock(
        world,
        targetPredicate,
        pos.x, pos.y, pos.z,
        dir.x, dir.y, dir.z,
        32.0 // Max distance
    );

    if (targetBlock != null) {
        // Logic to handle seeing the target block
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Thread Safety:** Do not call TargetUtil methods from asynchronous tasks or worker threads without explicit, engine-provided synchronization mechanisms for the `World` state. This is the most common source of instability.
- **Excessive Polling:** Avoid calling raycasting methods every single tick for every entity. This is computationally expensive. For AI, prefer running these checks on a staggered schedule or in response to specific events.
- **Unbounded Distance:** Passing an extremely large `maxDistance` to raycasting functions can cause significant performance degradation by forcing the iterator to check a vast number of blocks. Always use the smallest reasonable distance for the gameplay mechanic.
- **Result List Mismanagement:** The lists returned by `getAllEntitiesIn...` methods are thread-local and reused. Do not store a reference to this list beyond the immediate scope of the calling function, as its contents will be overwritten by the next spatial query on the same thread. Copy the contents if persistence is required.

## Data Pipeline
The data flow for a raycasting operation like `getTargetBlock` demonstrates how TargetUtil orchestrates various engine components.

> Flow:
> 1. **Input:** An origin `Vector3d`, a direction `Vector3d`, a max distance, and a `World` reference.
> 2. **Iteration:** `TargetUtil` invokes `BlockIterator.iterate`, which calculates each block coordinate along the ray's path.
> 3. **Chunk Loading:** For each coordinate, the internal `TargetBuffer` determines the corresponding chunk coordinates (`chunkX`, `chunkZ`). It queries the `World`'s `ChunkStore` to get a reference to the `BlockChunk` for that location.
> 4. **Data Sampling:** If the chunk is loaded, the `BlockSection` is retrieved and the block ID at the precise `(x, y, z)` coordinate is sampled.
> 5. **Predicate Test:** The sampled block ID is passed to the user-provided `blockIdPredicate`.
> 6. **Termination:** If the predicate returns true, the iteration stops, and `TargetUtil` returns the current block's coordinate vector. If the ray exceeds `maxDistance` without a match, it returns null.

