---
description: Architectural reference for BlockContactData
---

# BlockContactData

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class BlockContactData {
```

## Architecture & Concepts
BlockContactData is a fundamental data structure within the server-side physics and collision engine. It is not a service or manager, but rather a specialized data container designed to encapsulate the complete results of a collision query between a moving entity and a static world block.

This class acts as the primary information carrier between the low-level collision detection algorithms and the higher-level physics resolution logic. When a physics simulation checks for potential impacts, it populates an instance of BlockContactData with precise geometric and contextual information, such as the point of impact, the surface normal, and the time of collision. This data is then consumed by systems that must react to the collision, for example, by altering an entity's velocity, applying damage, or triggering game events.

Its design prioritizes performance and data locality for use in tight loops within the main server tick.

## Lifecycle & Ownership
- **Creation:** Instances of BlockContactData are intended to be created and destroyed rapidly. They are typically instantiated by a high-level physics module at the beginning of a collision test. The presence of a `clear` method strongly suggests the use of an object pooling pattern to reduce garbage collection overhead, where objects are recycled instead of being re-allocated each physics frame.

- **Scope:** The lifetime of a BlockContactData object is extremely short, typically confined to a single collision resolution step within one server tick. It is considered ephemeral state.

- **Destruction:** The object becomes eligible for garbage collection immediately after the physics response has been calculated. If managed by an object pool, it is returned to the pool for reuse. **WARNING:** Holding a reference to this object beyond the scope of the current physics calculation is a critical error, as its data will be stale or overwritten.

## Internal State & Concurrency
- **State:** The internal state is highly **Mutable**. The class is designed to be an empty shell that is progressively filled in by collision detection routines. The `final` keyword on the Vector3d fields only protects the reference, not the mutable state of the vector objects themselves.

- **Thread Safety:** This class is **not thread-safe**. It contains no synchronization primitives and is designed for exclusive use by a single thread, such as the main server game loop or a dedicated physics thread. Concurrent modification from multiple threads will lead to data corruption and unpredictable physics behavior.

## API Surface
The public API is focused on data assignment and retrieval. Standard getters are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| clear() | void | O(1) | Resets the object to a default state. Intended for object pool recycling. |
| assign(BlockContactData other) | void | O(1) | Performs a deep copy of all fields from another instance into this one. |
| setDamageAndSubmerged(...) | void | O(1) | Sets specific gameplay-related outcomes of the collision. |
| getCollisionNormal() | Vector3d | O(1) | Retrieves the immutable reference to the collision surface normal vector. |
| getCollisionPoint() | Vector3d | O(1) | Retrieves the immutable reference to the world-space point of impact. |
| getCollisionStart() | double | O(1) | Returns the fraction of the movement tick at which the collision begins. |
| isOnGround() | boolean | O(1) | A flag indicating if the collision represents a contact with a surface below the entity. |

## Integration Patterns

### Standard Usage
The canonical use case involves a physics system acquiring an instance, passing it to a world query function, and then consuming the results to adjust entity state.

```java
// Acquire an instance, preferably from an object pool
BlockContactData contactResult = new BlockContactData(); // Or pool.acquire();

// The physics engine populates the object during a world query
boolean hit = world.traceRay(entity.getPosition(), entity.getVelocity(), contactResult);

if (hit) {
    // The collision resolution system consumes the data
    Vector3d normal = contactResult.getCollisionNormal();
    entity.setVelocity(entity.getVelocity().reflect(normal));
    
    if (contactResult.getDamage() > 0) {
        entity.applyDamage(contactResult.getDamage());
    }
}

// Return to pool if applicable
// pool.release(contactResult);
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not store a BlockContactData object as a member field of a long-lived entity or component. Its data is only valid for the immediate frame in which it was generated.
- **Instantiation in Loops:** Avoid calling `new BlockContactData()` inside a tight physics loop. This pattern generates significant GC pressure. Use an object pool provided by the engine's core systems.
- **Asynchronous Access:** Never pass this object to another thread or an asynchronous callback. The data will be invalid by the time the other thread executes.

## Data Pipeline
BlockContactData serves as a critical data packet in the server's physics pipeline.

> Flow:
> Physics Engine Tick -> Collision Query (e.g., AABB Sweep) -> **BlockContactData (Populated)** -> Collision Resolution Logic -> Entity State Update (Position, Velocity, Health)

