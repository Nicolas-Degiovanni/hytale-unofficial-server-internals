---
description: Architectural reference for BasicCollisionData
---

# BasicCollisionData

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient Data Object

## Definition
```java
// Signature
public class BasicCollisionData {
```

## Architecture & Concepts
The BasicCollisionData class is a fundamental, high-frequency data structure within the server-side physics and collision module. It serves as a lightweight data carrier, encapsulating the result of a single collision intersection test. Its primary role is to convey the precise point and "time" of impact from a low-level collision detection routine to a higher-level physics resolution system.

The design prioritizes performance and low memory allocation overhead over strict encapsulation. Publicly accessible fields and mutable state allow for rapid, allocation-free updates, which is critical in tight physics loops where thousands of collision checks may occur per tick.

The inclusion of the static COLLISION_START_COMPARATOR is a strong indicator of its intended use case: to be part of a collection of potential collision points that are then sorted to determine the *first* point of contact along an object's trajectory.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand by collision detection algorithms, such as raycasting or shape-sweep tests. This is not a managed service and is typically instantiated directly via its constructor. In performance-critical systems, these objects may be managed by an object pool to reduce garbage collector pressure.
- **Scope:** Ephemeral. The lifetime of a BasicCollisionData object is extremely short, usually confined to the scope of a single method or physics tick. It holds temporary data that is consumed almost immediately.
- **Destruction:** The object is reclaimed by the Java Garbage Collector once it is no longer referenced. There are no manual cleanup or disposal methods.

## Internal State & Concurrency
- **State:** This object is highly **mutable**. The `setStart` method directly modifies its internal fields. While the `collisionPoint` field is a final reference, the underlying Vector3d object it points to is itself mutable. This design facilitates object reuse and avoids costly re-allocations.
- **Thread Safety:** This class is **not thread-safe** and provides no internal synchronization. Its public mutable fields make it inherently unsafe for concurrent access.

**WARNING:** All operations on a BasicCollisionData instance must be externally synchronized or, more appropriately, confined to a single thread, such as the main server physics thread. Sharing instances across threads will lead to race conditions and non-deterministic physics behavior.

## API Surface
The public contract is designed for high-speed data access and modification.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setStart(point, start) | void | O(1) | Populates the object with the result of a collision test. Assigns the impact location and the fractional time of collision. |
| COLLISION_START_COMPARATOR | Comparator | N/A | A static utility for sorting collections of BasicCollisionData. Essential for resolving multiple potential collisions and finding the earliest impact. |
| collisionPoint | Vector3d | O(1) | Public field providing direct access to the world-space point of collision. |
| collisionStart | double | O(1) | Public field representing the fraction along a movement vector where the collision occurred (typically in the range [0, 1]). |

## Integration Patterns

### Standard Usage
This object is typically used as an "out" parameter, where an empty instance is passed to a collision utility which then populates it upon detecting a hit.

```java
// A physics routine receives a pre-allocated data object to fill
private boolean findEarliestCollision(Entity entity, List<BasicCollisionData> results) {
    // ... logic to query the world for potential collisions ...

    for (Shape potentialCollider : nearbyShapes) {
        BasicCollisionData hitData = new BasicCollisionData(); // Or retrieved from a pool
        if (CollisionUtil.castShape(entity.getShape(), entity.getVelocity(), potentialCollider, hitData)) {
            results.add(hitData);
        }
    }

    if (!results.isEmpty()) {
        // Sort to find the very first collision in the movement path
        results.sort(BasicCollisionData.COLLISION_START_COMPARATOR);
        return true;
    }
    return false;
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Access:** Never pass a BasicCollisionData instance to another thread for reading or writing without explicit and robust synchronization. The class is not designed for this.
- **Long-Term Storage:** Do not retain instances of BasicCollisionData beyond the immediate physics step that generated them. The data represents a point-in-time event and will become stale. Storing these objects can also lead to memory leaks if not managed carefully.
- **Relying on Reference Stability:** While the `collisionPoint` field reference is final, the Vector3d object it points to is not. Do not assume the contents of `collisionPoint` will remain unchanged if the BasicCollisionData object is passed to other systems.

## Data Pipeline
The flow of information through this component is linear and localized within the physics engine.

> Flow:
> Physics Query (e.g., Raycast, Shape Sweep) -> **BasicCollisionData** (Result) -> Collision Resolution Logic -> Entity State Update (Position, Velocity)

