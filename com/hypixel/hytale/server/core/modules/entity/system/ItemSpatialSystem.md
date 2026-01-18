---
description: Architectural reference for ItemSpatialSystem
---

# ItemSpatialSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.system
**Type:** System Component

## Definition
```java
// Signature
public class ItemSpatialSystem extends SpatialSystem<EntityStore> {
```

## Architecture & Concepts
The ItemSpatialSystem is a specialized processor within the server-side Entity Component System (ECS) framework. It does not contain complex logic itself; rather, it serves as a configuration and data-access bridge for the generic, high-performance SpatialSystem it extends.

Its primary architectural role is to identify all "mergeable" item entities and integrate them into the world's spatial partitioning grid. This enables efficient proximity queries, which are fundamental for game mechanics like stacking dropped items.

The system operates on a simple principle:
1.  **Filtering:** It defines a precise query to select entities that possess an ItemComponent and a TransformComponent, but explicitly lack the PreventItemMerging component.
2.  **Data Extraction:** For each entity matching the filter, it provides a mechanism to extract its world position from its TransformComponent.

The core spatial indexing logic (e.g., updating the grid or octree) is handled by the parent SpatialSystem. This class merely provides the necessary inputs: *what* to track and *where* it is.

### Lifecycle & Ownership
-   **Creation:** Instantiated by the server's central ECS scheduler during world initialization. It is registered as one of many systems to be executed in a deterministic order each game tick.
-   **Scope:** The instance persists for the entire lifetime of a game world. It is not re-created on a per-tick or per-session basis.
-   **Destruction:** The object is marked for garbage collection when the world is unloaded or the server shuts down. The ECS framework is the sole owner and manager of its lifecycle.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. Its behavior is dictated by its static QUERY constant and the implementation of its methods. The true state it operates upon—the component data in the EntityStore and the SpatialResource grid—is external and passed in during the tick cycle.

-   **Thread Safety:** **Not thread-safe.** This system is designed to be executed exclusively by the main server game loop thread. Any concurrent access to the component Store or invocation of its methods from other threads will lead to state corruption, race conditions, and server instability. The parent ECS scheduler is responsible for ensuring serialized execution.

## API Surface
The public API is minimal, intended for consumption by the ECS scheduler, not by general game logic developers.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the static component query used by the scheduler to identify relevant entities for this system. |
| tick(dt, systemIndex, store) | void | O(N) | Executes the parent SpatialSystem logic for one game tick. N is the number of entities matching the query. |
| getPosition(archetypeChunk, index) | Vector3d | O(1) | Retrieves the world position of a specific entity from its TransformComponent. Called internally by the parent system. |

## Integration Patterns

### Standard Usage
Direct interaction with this class is not a standard development practice. Instead, developers influence its behavior by manipulating entity components. To make an entity processable by this system, ensure it has the required components.

```java
// EXAMPLE: Making a newly dropped item processable by the spatial system
// This code would exist in a different part of the engine, like item drop logic.

Entity newItem = world.createEntity();
newItem.addComponent(new ItemComponent(...));
newItem.addComponent(new TransformComponent(position));

// The ItemSpatialSystem will now automatically pick up and process this entity on the next tick.
// To prevent merging, add the PreventItemMerging component.
// newItem.addComponent(new PreventItemMerging());
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ItemSpatialSystem()`. The ECS framework is responsible for its creation and lifecycle. Manually created instances will not be registered with the game loop and will be non-functional.
-   **Manual Invocation:** Do not call the `tick` or `getPosition` methods directly. This bypasses the scheduler's state management and will break data consistency guarantees, likely causing crashes.
-   **Stateful Logic:** Do not add mutable instance fields to this class. Systems in the ECS should be stateless to ensure predictable, idempotent behavior.

## Data Pipeline
The ItemSpatialSystem acts as a critical node in the server's entity processing pipeline. It feeds positional data into the spatial partitioning structure, which is then consumed by other game systems.

> Flow:
> ECS Component Store -> **ItemSpatialSystem (Filters entities & provides position)** -> SpatialResource (Grid/Octree) -> Downstream Systems (e.g., ItemMergingSystem)

---

