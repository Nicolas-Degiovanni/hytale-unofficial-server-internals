---
description: Architectural reference for EntityContactData
---

# EntityContactData

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class EntityContactData {
```

## Architecture & Concepts
EntityContactData is a high-performance, mutable data container designed to encapsulate the results of a single collision detection query, such as a raycast or shape test. It is a fundamental component of the server-side physics and interaction engine.

Its primary architectural purpose is to eliminate memory allocation overhead during the physics simulation loop. In high-frequency operations like collision detection, creating new objects for each result would exert significant pressure on the garbage collector, leading to performance degradation. This class is designed to be pooled and reused, with its internal state being overwritten for each new query.

It serves as a standardized data transfer object between the low-level collision detection routines and higher-level game logic systems (e.g., combat, scripting, AI) that need to react to physical contact.

### Lifecycle & Ownership
-   **Creation:** Instances of EntityContactData are not intended for direct instantiation by game logic developers. They are typically managed by an object pool owned by a central service, such as a CollisionModule or PhysicsWorld. A system performing a query requests a pre-allocated instance from this pool.
-   **Scope:** The object's valid lifetime is extremely brief and ephemeral. It is scoped to the immediate execution of the function that populates it. Its data should be considered invalid after the current call stack unwinds.
-   **Destruction:** The object is not destroyed. After its data has been consumed, the `clear` method is invoked, and the instance is returned to its owner pool to be recycled for a future query. Failure to return the object to its pool constitutes a resource leak.

## Internal State & Concurrency
-   **State:** The object is highly mutable. The `assign` method completely overwrites the internal state. Note that while the `collisionPoint` field is final, the Vector3d object it references is mutable and is modified in place for performance reasons.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded access within the server's main game loop. Passing an instance across threads without external locking mechanisms will result in race conditions and undefined behavior. It must not be held or modified by asynchronous tasks.

## API Surface
The public API is minimal, focusing on state mutation and data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| assign(position, start, end, entity, detailName) | void | O(1) | Overwrites the object's state with new collision results. This is the primary method for populating the data structure. |
| clear() | void | O(1) | Resets the internal entity reference to null. This is a critical step in the object's lifecycle to prepare it for reuse and prevent memory leaks. |
| getCollisionPoint() | Vector3d | O(1) | Returns a reference to the internal, mutable Vector3d representing the point of impact. |
| getEntityReference() | Ref<EntityStore> | O(1) | Returns a reference to the entity that was struck in the collision query. May be null. |

## Integration Patterns

### Standard Usage
The canonical use case involves a system requesting a collision check, receiving a populated EntityContactData object, processing the results immediately, and then releasing the object.

```java
// Hypothetical CollisionSystem usage
CollisionSystem collisionSys = context.getService(CollisionSystem.class);

// The CollisionSystem manages the lifecycle of EntityContactData internally
EntityContactData contact = collisionSys.raycast(origin, direction, maxDistance);

if (contact != null && contact.getEntityReference() != null) {
    // CRITICAL: Consume data immediately. Do not store the 'contact' object itself.
    Vector3d impactPoint = contact.getCollisionPoint();
    Ref<EntityStore> hitEntity = contact.getEntityReference();
    
    // Trigger game logic based on the copied data
    CombatModule.applyDamage(hitEntity, impactPoint);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new EntityContactData()`. This defeats the object pooling mechanism and will lead to unnecessary garbage collection. Always acquire instances from the appropriate system.
-   **Long-Term Storage:** Storing a reference to an EntityContactData object is a severe error. The object will be recycled and its data will be overwritten by a subsequent, unrelated collision query, leading to unpredictable and hard-to-debug bugs. If collision data must be persisted, copy its values into a separate, stable data structure.
-   **Modifying Returned Vector:** The Vector3d returned by `getCollisionPoint` is the internal, live vector. Modifying it will corrupt the state of the EntityContactData object and may have unintended side effects if the object is still in use by the collision system.

## Data Pipeline
EntityContactData acts as a transient carrier in the data flow from a physics query to game logic.

> Flow:
> Physics Engine Query -> **EntityContactData (Populated)** -> Game System (e.g., Combat, AI) -> Data is read/copied -> **EntityContactData (Cleared & Pooled)**


