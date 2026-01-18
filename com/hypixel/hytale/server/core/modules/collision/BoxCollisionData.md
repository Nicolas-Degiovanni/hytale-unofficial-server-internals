---
description: Architectural reference for BoxCollisionData
---

# BoxCollisionData

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient Data Object

## Definition
```java
// Signature
public class BoxCollisionData extends BasicCollisionData {
```

## Architecture & Concepts
BoxCollisionData is a specialized data transfer object (DTO) designed to encapsulate the results of a collision detection query between two box-aligned volumes. It is a fundamental data structure within the server-side physics and collision engine.

This class extends the more generic BasicCollisionData, augmenting it with precise impact details: the distance to the point of collision and the surface normal at that point. It serves as a high-performance, short-lived container for passing collision results from low-level intersection algorithms to higher-level collision resolution systems. Its design prioritizes speed and low memory allocation overhead over encapsulation, exposing its state via public fields for direct access within the tight loop of a physics tick.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by collision detection routines during the narrow-phase collision check. These objects are typically created on the stack or allocated from a pre-warmed object pool to avoid garbage collection pressure during physics simulation.
- **Scope:** Extremely short-lived. The lifetime of a BoxCollisionData instance is confined to a single collision query and its subsequent resolution. It is not intended to persist beyond a single frame or physics tick.
- **Destruction:** The object becomes eligible for garbage collection, or is returned to an object pool, immediately after the consuming system has processed the collision event. Holding references to these objects is a memory leak risk.

## Internal State & Concurrency
- **State:** Highly mutable. The primary purpose of this class is to be an empty container that is populated by a calculation. Both the collisionEnd and the internal state of the collisionNormal vector are modified after instantiation.
- **Thread Safety:** **This class is not thread-safe.** It is designed for use within a single, synchronous physics update loop. Its public, mutable fields and lack of any synchronization mechanisms make it completely unsafe for concurrent access. It must not be shared across threads without explicit, external locking, which would negate its performance-oriented design.

## API Surface
The public contract consists of one mutator method and direct field access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setEnd(double, Vector3d) | void | O(1) | Populates the object with collision results. The provided collisionNormal data is copied into the internal vector, preventing external mutation of the source vector from affecting this object's state. |
| collisionEnd | double | O(1) | Public field representing the fractional distance along a movement vector where the collision occurs. |
| collisionNormal | Vector3d | O(1) | Public field representing the surface normal of the face that was struck. The reference is final, but the vector's internal state is mutable. |

## Integration Patterns

### Standard Usage
A collision system creates an instance, passes it by reference to an intersection test, and then inspects its fields to resolve the collision if one occurred.

```java
// Conceptual example of a collision check
BoxCollisionData collisionResult = new BoxCollisionData(); // Or from a pool
boolean didCollide = AABBIntersector.test(entity.getBounds(), world.getBounds(), collisionResult);

if (didCollide) {
    // Use the populated data to adjust entity velocity or position
    Vector3d normal = collisionResult.collisionNormal;
    double impactTime = collisionResult.collisionEnd;
    entity.getPhysicsState().resolveImpact(normal, impactTime);
}
```

### Anti-Patterns (Do NOT do this)
- **State Caching:** Do not retain an instance of BoxCollisionData across multiple physics ticks. The data it contains is only valid for the specific moment in time it was calculated.
- **Concurrent Sharing:** Never pass an instance of this object to another thread for processing without a deep copy. Doing so will lead to race conditions and non-deterministic physics behavior.
- **Misinterpreting the Normal:** The collisionNormal is a final reference to a mutable object. While you cannot re-assign the field, you can modify the vector's internal components. Avoid modifying the result object; treat it as read-only after it has been populated by the collision system.

## Data Pipeline
BoxCollisionData acts as the data payload within the physics engine's collision pipeline. It does not process data itself but is rather the data being processed.

> Flow:
> Physics Tick Start -> Broad-Phase Check -> Narrow-Phase Intersection Test -> **BoxCollisionData Population** -> Collision Resolution Logic -> Entity State Update

