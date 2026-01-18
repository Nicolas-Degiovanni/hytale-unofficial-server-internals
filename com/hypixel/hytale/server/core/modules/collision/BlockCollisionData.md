---
description: Architectural reference for BlockCollisionData
---

# BlockCollisionData

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient Data Structure

## Definition
```java
// Signature
public class BlockCollisionData extends BoxCollisionData {
```

## Architecture & Concepts
BlockCollisionData is a specialized data container that encapsulates the results of a collision query between a moving entity and a static world block. It is not a service or manager, but rather a fundamental data structure used extensively by the server's physics and collision detection engine.

This class extends the more generic BoxCollisionData, inheriting geometric collision details like the point of impact and collision normal. It enriches this base data with block-specific context, such as the block's world coordinates (x, y, z), its type, material, and fluid properties.

In essence, BlockCollisionData acts as a detailed "collision report". When the physics system detects an interaction, it populates an instance of this class to provide all necessary information for game logic to react appropriately, whether by stopping movement, applying damage, or triggering a fluid interaction.

## Lifecycle & Ownership
- **Creation:** Instances are created and populated by the core collision detection system during physics ticks. Due to the high frequency of collision checks, these objects are almost certainly managed by an object pool to avoid constant memory allocation and garbage collection overhead. The existence of a public `clear` method strongly supports this pooling pattern.
- **Scope:** The scope of a BlockCollisionData instance is extremely narrow, typically lasting for only a fraction of a single server tick. It is created, populated, consumed by game logic, and then released back to its pool.
- **Destruction:** An instance is effectively "destroyed" when it is released. For pooled objects, this means its `clear` method is called to nullify references, preparing it for immediate reuse in a subsequent collision check.

**WARNING:** Holding a reference to a BlockCollisionData object beyond the immediate scope of the collision event handler is a critical defect. The object will be recycled and its data will become invalid, leading to unpredictable and difficult-to-debug behavior.

## Internal State & Concurrency
- **State:** The state is entirely mutable. The class is designed as a simple data bag with public fields, intended to be written to once by the collision system and then read by game logic.
- **Thread Safety:** This class is **not thread-safe** and must not be shared across threads without explicit and robust synchronization. It is designed for synchronous, single-threaded access within the server's main game loop or a dedicated physics thread.

## API Surface
The public API is minimal, focusing on populating and resetting the object's state. Direct field access is expected for reading data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setBlockData(CollisionConfig) | void | O(1) | Populates the object from a CollisionConfig. This is the primary initialization entry point. |
| setDetailBoxIndex(int) | void | O(1) | Specifies which sub-box of a complex block model was involved in the collision. |
| setTouchingOverlapping(boolean, boolean) | void | O(1) | Sets final state flags determined by the collision resolution phase. |
| clear() | void | O(1) | Resets object state for reuse. Critical for object pooling patterns. |

## Integration Patterns

### Standard Usage
The canonical usage involves receiving this object as the result of a physics query. The consumer reads the relevant fields to implement game logic and then immediately discards its reference to the object.

```java
// A physics system provides the populated BlockCollisionData object
BlockCollisionData collision = physics.raycast(start, end);

if (collision != null) {
    // React to the collision based on the data
    if (collision.willDamage) {
        entity.applyDamage(10);
    }
    if (collision.fluid != null) {
        entity.setWet(true);
    }
    // The reference to 'collision' is now discarded
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new BlockCollisionData()`. The server's collision module manages a pool of these objects for performance. Manually creating instances bypasses this optimization and can lead to increased garbage collection pressure.
- **Long-Term Storage:** Do not store a BlockCollisionData instance in a component or field for later use. The object is transient and will be cleared and reused by the pooling system, corrupting your stored reference.
- **Asynchronous Processing:** Do not pass this object to another thread or an asynchronous task. Its data is only valid for the synchronous duration of the collision callback.

## Data Pipeline
BlockCollisionData is the terminal output of the low-level collision detection pipeline. It serves as the bridge between raw physics calculations and high-level game logic.

> Flow:
> Entity Movement Query -> World Octree Pruning -> AABB Intersection Test -> **BlockCollisionData** Population -> Game Logic Consumer (e.g., Movement Component)

