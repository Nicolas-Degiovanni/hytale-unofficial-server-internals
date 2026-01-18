---
description: Architectural reference for BoxBlockIntersectionEvaluator
---

# BoxBlockIntersectionEvaluator

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient

## Definition
```java
// Signature
public class BoxBlockIntersectionEvaluator extends BlockContactData implements IBlockCollisionEvaluator {
```

## Architecture & Concepts
The BoxBlockIntersectionEvaluator is a specialized, stateful calculator within the server-side physics engine. Its primary function is to perform and evaluate fine-grained intersection tests between two Axis-Aligned Bounding Boxes (AABBs), typically representing an entity's hitbox and a static world block.

This class embodies the **Strategy Pattern**. By implementing the IBlockCollisionEvaluator interface, it provides a concrete algorithm for box-on-box collision checks. The broader physics system can select this evaluator dynamically when it determines both collision participants are box-shaped, decoupling the high-level collision loop from the low-level mathematical details.

It inherits from BlockContactData, which serves as its data store for collision results. The evaluator's methods compute contact information (like collision normal and penetration depth) and populate the fields of its parent class, which are then consumed by physics resolution systems.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor, `new BoxBlockIntersectionEvaluator()`. It is a lightweight, plain Java object, not a managed service. It is typically created on the stack or as a short-lived field within a higher-level physics process for the duration of a single entity's collision update.
- **Scope:** The object's lifetime is intentionally brief, usually confined to a single method or a single game tick's physics pass. It is designed to be configured, used for a series of related intersection tests, and then immediately discarded.
- **Destruction:** The object is reclaimed by the standard Java garbage collector once it falls out of scope. No manual resource management is necessary.

## Internal State & Concurrency
- **State:** This class is highly mutable and stateful. Its primary purpose is to hold the configuration for a collision test (the box, its position) and the results of the last intersection calculation (resultCode, onGround, touchCeil, collisionNormal). Each call to an `intersect` method modifies this internal state.

- **Thread Safety:** **This class is not thread-safe.** Its mutable, sequential nature makes it fundamentally unsafe for concurrent access. It must only be used within a single thread. Sharing an instance across threads without external locking will result in severe race conditions and non-deterministic physics behavior.

## API Surface
The public API is designed as a fluent interface for configuration, followed by methods to execute tests and query results.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setBox(Box box) | BoxBlockIntersectionEvaluator | O(1) | Configures the primary AABB for the intersection test. |
| setPosition(Vector3d pos) | BoxBlockIntersectionEvaluator | O(1) | Sets the world position of the primary AABB. |
| intersectBoxComputeTouch(Box, double, double, double) | int | O(1) | Executes the core intersection test and computes detailed contact data, including the collision normal. Updates all internal state. |
| intersectBoxComputeOnGround(Box, double, double, double) | int | O(1) | Executes a specialized intersection test optimized for determining ground contact. Updates internal state. |
| isTouching() | boolean | O(1) | Returns true if the last intersection test resulted in a touching contact. **Warning:** Relies on state from a prior `intersect` call. |
| touchesCeil() | boolean | O(1) | Returns true if the last intersection test resulted in contact with a ceiling. **Warning:** Relies on state from a prior `intersect` call. |
| setCollisionData(...) | void | O(1) | Populates a BlockCollisionData object with the results of the last intersection test. This is the primary mechanism for exporting results. |

## Integration Patterns

### Standard Usage
The evaluator follows a strict configure-execute-read pattern. An instance is created, configured with the entity's hitbox, and then used to test against one or more block hitboxes within a loop.

```java
// 1. Create a new, clean evaluator for this physics update
BoxBlockIntersectionEvaluator evaluator = new BoxBlockIntersectionEvaluator();

// 2. Configure it with the entity's current state
evaluator.setBox(entity.getHitbox(), entity.getPosition());

// 3. Loop through potential colliders (e.g., nearby blocks)
for (Block block : nearbyBlocks) {
    Box blockHitbox = block.getHitbox();
    Vector3d blockPos = block.getPosition();

    // 4. Execute the intersection test
    if (evaluator.isBoxIntersecting(blockHitbox, blockPos.x, blockPos.y, blockPos.z)) {
        // 5. A collision was found. Package the results for the physics solver.
        BlockCollisionData result = new BlockCollisionData();
        evaluator.setCollisionData(result, block.getCollisionConfig(), 0);
        
        // ... process the collision result ...
        break; // Exit loop if only the first collision matters
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Instance Caching:** Do not cache and reuse a single evaluator instance across unrelated physics updates (e.g., for different entities or different game ticks). Its internal state will become invalid and cause unpredictable collision results. Always create a new instance for a new, distinct collision resolution task.
- **Stale State Reading:** Do not call result-querying methods like `isTouching` or `touchesCeil` before executing an `intersect` method. The returned value will be based on default or stale data from a previous, unrelated operation.
- **Concurrent Access:** Never share an instance of this class between multiple threads. The physics engine must ensure that each thread performing collision detection has its own private instance of the evaluator.

## Data Pipeline
The BoxBlockIntersectionEvaluator acts as a computational stage in the broader server physics pipeline. It transforms geometric data into actionable collision information.

> Flow:
> Entity State (Position, Hitbox) -> **BoxBlockIntersectionEvaluator** (Configuration) -> World State (Block Hitboxes) -> **BoxBlockIntersectionEvaluator** (Intersection Test) -> Internal State (resultCode, onGround, normal) -> BlockCollisionData (Formatted Result) -> Physics Resolution Engine

