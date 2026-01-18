---
description: Architectural reference for CharacterCollisionData
---

# CharacterCollisionData

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient Data Object

## Definition
```java
// Signature
public class CharacterCollisionData extends BasicCollisionData {
```

## Architecture & Concepts
CharacterCollisionData is a specialized, server-side data structure that encapsulates the result of a physics collision query involving a character entity, such as a player or an NPC. It extends the more generic BasicCollisionData with character-specific information critical for gameplay logic.

This class is not a standalone service but rather a data-transfer object (DTO) used within the server's collision detection pipeline. Its primary purpose is to convey precise collision details from a low-level physics module to higher-level game systems like combat, AI, or interaction handlers.

The inclusion of a `Ref<EntityStore>` is a key architectural choice. It provides a safe, reference-counted handle to the collided entity, preventing dangling pointers or crashes if the entity is destroyed between the time of collision detection and the time the data is processed. The `isPlayer` boolean serves as a fast-path optimization, allowing systems to quickly filter or branch logic for player-specific interactions without needing to resolve the entity reference and inspect its components.

## Lifecycle & Ownership
The lifecycle of a CharacterCollisionData instance is ephemeral and tightly controlled by an object pooling system to minimize garbage collection overhead in the performance-critical physics loop.

-   **Creation:** Instances are not created directly via the `new` keyword. Instead, they are pre-allocated and managed by a central pool, likely owned by the server's primary CollisionModule. When a collision test occurs, an available instance is acquired from this pool.
-   **Scope:** The object's state is valid only for the immediate duration of the collision event processing. Its lifetime is typically confined to a single method scope or a single frame tick.
-   **Destruction:** The object is not destroyed. After the consuming system has finished processing the collision result, the instance is "released" back to the object pool, where its internal state is cleared or marked as invalid, ready for reuse in a subsequent collision test.

## Internal State & Concurrency
-   **State:** Highly mutable. The `assign` method is designed to completely overwrite the object's state for each new collision event. It is a reusable container, not an immutable value object.
-   **Thread Safety:** **This class is not thread-safe.** It is designed for synchronous, single-threaded access within the server's main game loop or a dedicated physics thread. Passing instances between threads without explicit synchronization will result in data corruption and undefined behavior. The object pooling pattern inherently relies on a single-threaded acquire-use-release cycle.

## API Surface
The public contract consists of the `assign` method for re-initialization and direct field access for data retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| assign(...) | void | O(1) | Re-initializes the object with new collision data. This is the primary mechanism for populating a pooled instance. |
| entityReference | Ref<EntityStore> | - | A public, reference-counted handle to the entity involved in the collision. |
| isPlayer | boolean | - | A public flag for efficiently identifying if the collided entity is a player. |

## Integration Patterns

### Standard Usage
CharacterCollisionData is never instantiated directly. It is provided by a collision query system, which manages the object's lifecycle. The consumer inspects its public fields to inform game logic.

```java
// Hypothetical usage within a game system
CollisionModule collisionModule = server.getModule(CollisionModule.class);
RaycastResult result = collisionModule.raycast(start, end);

if (result.hasHit() && result.getData() instanceof CharacterCollisionData) {
    CharacterCollisionData charData = (CharacterCollisionData) result.getData();
    
    // Process the collision based on the populated data
    if (charData.isPlayer) {
        applyPlayerSpecificEffect(charData.entityReference);
    }
}
// The 'charData' object is now considered invalid and will be reused by the pool
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new CharacterCollisionData()`. This bypasses the engine's object pool, leading to significant performance degradation due to increased GC pressure in the physics loop.
-   **State Caching:** Do not store a reference to a CharacterCollisionData object beyond the immediate scope of the event handler or tick. The object will be reclaimed by the pool and its data will be overwritten by a subsequent collision test, leading to severe and difficult-to-debug logical errors.
-   **Modification:** Do not modify the fields of a CharacterCollisionData object after it has been received from a system. It is intended to be a read-only result container for the consumer.

## Data Pipeline
The flow of data through this component is linear and confined to a single server tick.

> Flow:
> Physics Query (e.g., Raycast) -> CollisionModule acquires **CharacterCollisionData** from pool -> Physics engine populates data -> **CharacterCollisionData** is returned to caller -> Game System reads data -> **CharacterCollisionData** is released to pool

