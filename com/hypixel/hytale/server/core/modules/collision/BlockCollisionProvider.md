---
description: Architectural reference for BlockCollisionProvider
---

# BlockCollisionProvider

**Package:** com.hypixel.hytale.server.core.modules.collision
**Type:** Transient

## Definition
```java
// Signature
public class BlockCollisionProvider implements BoxBlockIterator.BoxIterationConsumer {
```

## Architecture & Concepts

The BlockCollisionProvider is a high-performance, stateful service object responsible for calculating the interaction between a moving axis-aligned bounding box (typically an entity's collider) and the static block world. It forms the core of the server's entity-world physics simulation.

This class is not a simple intersection checker. It implements a swept-volume collision detection algorithm, often referred to as "casting". It determines not only *if* a collision occurs, but precisely *where* along the path of motion it happens and with what surface.

Its key architectural features include:

-   **Dual-Strategy Evaluation:** The provider internally selects one of two strategies based on the magnitude of the motion vector. For very short movements, it uses a simple overlap check (`castShortDistance`). For longer movements, it performs a more computationally expensive iterative sweep test (`castIterative`). This is a critical optimization to reduce physics overhead for stationary or slow-moving entities.
-   **Decoupled Collision Response:** The provider detects collisions but does not resolve them. It uses a consumer pattern via the `IBlockCollisionConsumer` interface. During a `cast` operation, it streams collision events (solid impacts, damage, trigger activations) to the provided consumer, which is then responsible for implementing the game logic (e.g., stopping movement, applying damage, firing a script).
-   **Stateful, Per-Operation Design:** An instance of this class is designed to handle a single collision query at a time. It maintains significant internal state during the `cast` method's execution, such as the earliest collision time and trackers for blocks already processed. This state is meticulously reset upon completion, making the object reusable for subsequent, independent queries.
-   **Interaction Differentiation:** It is capable of distinguishing between multiple types of block interactions:
    -   **Solid Collisions:** Physical impacts that impede movement.
    -   **Damage Sources:** Blocks that harm entities upon contact (e.g., lava, spikes).
    -   **Triggers:** Volumes that activate game logic without impeding movement.
    -   **Fluids:** Special volumes with unique physics properties.

It orchestrates several specialized helper classes, including `MovingBoxBoxCollisionEvaluator` for swept math, `BlockDataProvider` for efficient world data access, and various `Tracker` classes to prevent redundant calculations on multi-part blocks or during complex sweeps.

### Lifecycle & Ownership
-   **Creation:** This class is not a singleton. Instances are intended to be created and managed by a higher-level system, such as the `CollisionModule`. It is highly probable that these objects are pooled to reduce garbage collection overhead in the main game loop.
-   **Scope:** The internal state of a BlockCollisionProvider is valid **only for the duration of a single call to the `cast` method**. The `cast` method is the transactional boundary for a collision query.
-   **Destruction:** The `cast` method cleans up its own internal state before returning. The object itself is either returned to an object pool for reuse or becomes eligible for garbage collection.

## Internal State & Concurrency
-   **State:** This class is highly **mutable**. It contains numerous fields that track the progress and results of the current `cast` operation, including the motion vector, collision consumer, and multiple trackers for damage, triggers, and solid contacts. This state is not persistent between `cast` calls.
-   **Thread Safety:** **This class is not thread-safe.** Its stateful, single-operation design makes it fundamentally unsafe for concurrent access. An instance must not be shared across threads. Any multi-threaded physics engine must ensure that each worker thread uses its own dedicated instance of BlockCollisionProvider or that access is strictly synchronized.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| cast(world, collider, pos, v, consumer, triggers, stop) | void | O(n) | **Primary entry point.** Executes a collision sweep for a collider moving along a vector. Results are streamed to the consumer. Complexity is proportional to the number of blocks along the sweep path. |
| setRequestedCollisionMaterials(materials) | void | O(1) | Configures the collision mask. This acts as a filter to determine which types of block materials the collider should interact with. |
| setReportOverlaps(report) | void | O(1) | Configures whether to report simple overlaps when no swept collision occurs. Useful for checking initial intersections. |

## Integration Patterns

### Standard Usage

The BlockCollisionProvider is used by systems that manage entity movement. The caller provides the entity's physical state and a callback handler (the consumer) to process the results. The provider performs the complex calculation, and the caller's consumer implements the game-specific reaction.

```java
// Obtain a provider from the managing module (DO NOT use 'new')
BlockCollisionProvider provider = CollisionModule.get().getProvider();

// Configure the query
provider.setRequestedCollisionMaterials(SOLID_MATERIALS);
provider.setReportOverlaps(true);

// Execute the cast with an implementation of IBlockCollisionConsumer
MyCollisionConsumer resultsHandler = new MyCollisionConsumer();
IBlockTracker activeTriggers = entity.getActiveTriggers();
provider.cast(world, entity.getCollider(), entity.getPosition(), entity.getVelocity(), resultsHandler, activeTriggers, 1.0);

// The resultsHandler now contains the collision results, which can be used
// to adjust the entity's final position and state.
Vector3d finalVelocity = resultsHandler.getFinalVelocity();
entity.setPosition(entity.getPosition().add(finalVelocity));
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not call `new BlockCollisionProvider()`. This bypasses any potential pooling mechanism managed by the `CollisionModule`, leading to poor performance and excessive garbage collection. Always acquire instances from the appropriate service.
-   **State Persistence:** Do not assume any state persists between calls to `cast`. The object's internal state is reset after every operation. Configuration methods like `setRequestedCollisionMaterials` must be called before each `cast`.
-   **Concurrent Modification:** Do not modify the input parameters (e.g., the position vector) from another thread while a `cast` operation is in progress. The provider is not designed for this and behavior will be undefined.

## Data Pipeline

The flow of data for a single `cast` operation is a multi-stage process orchestrated entirely within the provider.

> Flow:
> `cast` call with entity state -> **Strategy Selection** (Short vs. Iterative) -> **Block Iteration** along motion vector -> For each block: **Block Data Fetch** (`BlockDataProvider`) -> **Low-Level Math** (`MovingBoxBoxCollisionEvaluator`) -> **Event Firing** (Invoke `IBlockCollisionConsumer` callbacks) -> **State Update** (Update earliest collision time) -> Final Cleanup & `onCollisionFinished` call

