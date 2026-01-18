---
description: Architectural reference for RestingSupport
---

# RestingSupport

**Package:** com.hypixel.hytale.server.core.modules.physics
**Type:** Transient

## Definition
```java
// Signature
public class RestingSupport {
```

## Architecture & Concepts
The RestingSupport class is a performance-optimization component within the server-side physics engine. Its primary function is to reduce the computational cost of simulating entities that are stationary or "at rest".

Instead of performing expensive collision detection and physics calculations for a resting entity every game tick, the engine can use a RestingSupport instance to take a snapshot of the blocks immediately surrounding and supporting the entity. On subsequent ticks, the engine can perform a very cheap check using the hasChanged method to see if any of those blocks have been altered.

This system acts as a "dirty flag" for a resting entity's physical environment. An entity is only "woken up" for a full physics recalculation if its support structure changes, such as when a player breaks a block beneath it. This avoids redundant processing for the vast majority of stationary entities in the world. It is a stateful helper object, owned by a higher-level physics component associated with a game entity.

## Lifecycle & Ownership
- **Creation:** A RestingSupport object is instantiated directly by its consumer, typically a per-entity physics controller, when an entity first comes to rest. It is not managed by a central registry or factory.
- **Scope:** The lifecycle of a RestingSupport instance is tightly coupled to its owning entity's physics state. It persists as long as the entity exists and may be cleared and reused multiple times.
- **Destruction:** The object is eligible for garbage collection when its owning physics component is destroyed, for example, when an entity is removed from the world. The clear method can be invoked to release internal memory (the supportBlocks array) without destroying the object itself.

## Internal State & Concurrency
- **State:** This class is highly mutable. Its core state consists of a cached bounding box (supportMin/Max coordinates) and a snapshot of block IDs within that box (supportBlocks). The rest method completely overwrites this state, and the clear method nullifies the block cache.

- **Thread Safety:** **This class is not thread-safe.** All methods read and write to instance fields without any synchronization primitives. Concurrent calls to rest or hasChanged will lead to race conditions, inconsistent state, and undefined behavior.

    **WARNING:** All interactions with a RestingSupport instance must be strictly confined to the single thread responsible for the physics simulation, which is typically the main server thread. Do not access it from network threads or asynchronous tasks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hasChanged(World world) | boolean | O(N) | Compares the cached block snapshot against the current world state. N is the number of blocks in the support volume. |
| rest(World world, Box boundingBox, Vector3d position) | void | O(N) | Captures a new snapshot of the blocks within the specified bounding box. Allocates memory if no cache exists. |
| clear() | void | O(1) | Resets the internal state, releasing the block snapshot array. |

## Integration Patterns

### Standard Usage
The intended use is within an entity's state update loop to determine if a resting entity needs its physics re-evaluated.

```java
// In a hypothetical EntityPhysicsController update method
RestingSupport support = this.getRestingSupport();

if (entity.isAtRest()) {
    if (support.hasChanged(world)) {
        // The ground changed, wake the entity up
        entity.setAtRest(false);
        runFullPhysicsAndCollision();
    }
    // Otherwise, do nothing.
} else {
    runFullPhysicsAndCollision();
    if (entity.hasComeToRest()) {
        // Entity just stopped moving, take a snapshot
        support.rest(world, entity.getBoundingBox(), entity.getPosition());
    } else {
        // Entity is still moving, ensure support is cleared
        support.clear();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **State Mismatch:** Do not call hasChanged if the entity has moved from the position where rest was last called. The check will be performed against an incorrect location in the world, leading to missed physics updates or phantom triggers.
- **Shared Instances:** Never share a single RestingSupport instance across multiple entities. Each entity's physics state requires its own unique instance to avoid state corruption.
- **Premature Optimization:** Do not use this for highly dynamic or constantly moving entities. The overhead of calling rest frequently will negate any performance benefits. It is designed exclusively for stationary or near-stationary objects.

## Data Pipeline
RestingSupport acts as a stateful cache in a control flow rather than a traditional data pipeline. It reads from the world state to determine if a control signal (a boolean) should be sent back to the physics engine.

> Flow:
> Physics Engine identifies resting entity -> Calls **RestingSupport.rest()** -> World/WorldChunk block data is read and cached internally -> On next tick, Physics Engine calls **RestingSupport.hasChanged()** -> **RestingSupport** re-reads World/WorldChunk block data -> Compares new data to cache -> Returns boolean result to Physics Engine -> Physics Engine decides whether to run full simulation.

