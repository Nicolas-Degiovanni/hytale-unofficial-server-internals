---
description: Architectural reference for MovingBoxBoxCollisionEvaluator
---

# MovingBoxBoxCollisionEvaluator

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient

## Definition
```java
// Signature
public class MovingBoxBoxCollisionEvaluator extends BlockContactData implements IBlockCollisionEvaluator {
```

## Architecture & Concepts
The MovingBoxBoxCollisionEvaluator is a low-level, high-performance calculator at the heart of the server-side physics engine. Its sole responsibility is to perform a continuous collision detection (CCD) query between a moving Axis-Aligned Bounding Box (AABB) and a static AABB.

This class implements a variant of the Separating Axis Theorem (SAT) optimized for AABBs. The core principle is that a 3D collision check can be decomposed into three independent 1D collision checks, one for each axis (X, Y, Z). This is handled internally by the **Collision1D** nested class.

By providing an entity's current position and its velocity vector for the current tick, this evaluator can determine *if* and *when* a collision will occur during that movement. It calculates the precise time of impact (a value between 0.0 and 1.0) and the surface normal of the collision, which are critical inputs for the physics resolution stage.

This component is not a general-purpose physics body but a specialized, stateful tool used by higher-level systems like the **CollisionSystem** to resolve entity movement against the static world geometry.

## Lifecycle & Ownership
-   **Creation:** Instances of MovingBoxBoxCollisionEvaluator are intended to be short-lived. They are typically managed by an object pool within the primary **CollisionSystem** to mitigate garbage collection overhead in the main game loop. Direct instantiation is heavily discouraged in performance-critical code.
-   **Scope:** The lifetime of an evaluator is scoped to a single, discrete collision query. It is configured, used to perform a calculation, and then either discarded or returned to its pool. Its internal state is only valid for the duration of that single query.
-   **Destruction:** If managed by a pool, the object is not destroyed but reset and returned for reuse. If instantiated directly, it becomes eligible for garbage collection as soon as it falls out of the immediate computational scope.

## Internal State & Concurrency
-   **State:** This class is highly **mutable**. Methods like **setCollider** and **setMove** directly modify its internal state to prepare it for a calculation. The primary execution method, **isBoundingBoxColliding**, further alters fields like **collisionStart**, **collisionNormal**, and **onGround**. This stateful design is a performance optimization to avoid passing numerous parameters.

-   **Thread Safety:** **WARNING:** This class is unequivocally **NOT thread-safe**. Its mutable, sequential nature makes it susceptible to race conditions if shared across threads. Any parallelized physics simulation must ensure that each worker thread acquires its own distinct instance of the evaluator from a pool. Do not share instances between threads under any circumstances.

## API Surface
The public API is designed as a fluent interface for configuration, followed by a single execution method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setCollider(Box collider) | MovingBoxBoxCollisionEvaluator | O(1) | Sets the AABB of the moving entity. |
| setMove(Vector3d pos, Vector3d v) | MovingBoxBoxCollisionEvaluator | O(1) | Sets the initial position and velocity vector for the query. |
| isBoundingBoxColliding(Box, double, double, double) | boolean | O(1) | Executes the collision test against a static AABB. Returns true if a collision occurs within the velocity vector. |
| setCollisionData(BlockCollisionData, ...) | void | O(1) | Populates a data transfer object with the results of the calculation. |
| getCollisionStart() | double | O(1) | Returns the time of impact (0.0 to 1.0). Only valid after a successful collision check. |
| setCheckForOnGround(boolean) | void | O(1) | Toggles a specialized check for "on ground" status, an important optimization. |
| setComputeOverlaps(boolean) | void | O(1) | Toggles whether to compute resolution data for initially overlapping boxes. |

## Integration Patterns

### Standard Usage
The evaluator is designed to be configured once per moving entity and then used to test against multiple static blocks in its path.

```java
// Assume 'collisionSystem' manages an object pool for evaluators
MovingBoxBoxCollisionEvaluator evaluator = collisionSystem.acquireEvaluator();
BlockCollisionData resultData = new BlockCollisionData();

// 1. Configure the moving entity's properties for this tick
evaluator.setCollider(entity.getHitbox());
evaluator.setMove(entity.getPosition(), entity.getVelocity());

// 2. Iterate over potential static colliders (e.g., nearby world blocks)
for (Box staticBlockBox : potentialColliders) {
    if (evaluator.isBoundingBoxColliding(staticBlockBox, block.x, block.y, block.z)) {
        // 3. A collision was found, populate the result object
        evaluator.setCollisionData(resultData, block.getCollisionConfig(), 0);
        
        // 4. Pass the result to the physics resolution system
        physicsResolver.handleCollision(resultData);
        break; // Or continue if multiple collisions are needed
    }
}

// 5. Return the evaluator to the pool for reuse
collisionSystem.releaseEvaluator(evaluator);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation in Loops:** Never call `new MovingBoxBoxCollisionEvaluator()` inside a physics tick or any other performance-sensitive loop. This will create extreme pressure on the garbage collector and degrade server performance. Always use a pooling mechanism.
-   **State Re-use:** Do not call **isBoundingBoxColliding** for a second, different static box without first re-configuring the evaluator with **setMove** if the moving entity's state has changed. The object's internal state is invalid after the first collision resolution.
-   **Concurrent Modification:** Do not access an instance from multiple threads. This will result in unpredictable and incorrect collision results.

## Data Pipeline
The evaluator is a processing node in the entity physics pipeline. It transforms positional and velocity data into actionable collision data.

> Flow:
> Entity State (Position, Velocity, Hitbox) -> **MovingBoxBoxCollisionEvaluator** -> Collision Result (Time of Impact, Normal, isTouching) -> Physics Resolution System -> Corrected Entity Position & Velocity

