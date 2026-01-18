---
description: Architectural reference for ItemMergeSystem
---

# ItemMergeSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.item
**Type:** System

## Definition
```java
// Signature
public class ItemMergeSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The ItemMergeSystem is a server-side entity system responsible for optimizing the game world by reducing the number of active item entities. It operates within the Hytale Entity Component System (ECS) framework to automatically find and combine compatible, nearby item stacks that have been dropped in the world.

This system's primary function is performance enhancement. A large number of individual item entities can create significant overhead in physics, rendering, and network updates. By merging items, this system reduces the entity count, alleviating server load.

It identifies target entities by querying for those with an **ItemComponent** but specifically excluding any that have an **Interactable** component (meaning a player is currently targeting it) or a **PreventItemMerging** component (an explicit flag to disable this behavior).

The core of its logic relies on spatial queries. For each eligible item, the system queries a **SpatialResource**—a server-wide spatial partitioning structure like a grid or octree—to efficiently find other items within a fixed radius. This avoids costly brute-force checks against every other entity in the world.

## Lifecycle & Ownership
-   **Creation:** The ItemMergeSystem is instantiated once by the server's core module loader during the server bootstrap sequence. It is registered with the main ECS world scheduler.
-   **Scope:** This system is a long-lived, stateless service. It persists for the entire lifetime of the server session. Its own state is immutable after construction; it only reads from and writes to the shared ECS world state.
-   **Destruction:** The system is destroyed and its resources are released only when the server process is shut down.

## Internal State & Concurrency
-   **State:** The ItemMergeSystem is effectively stateless. Its instance fields are final references to component and resource types, which are ECS metadata. All runtime state it manipulates (e.g., item quantities, entity positions) is external, located in components within the **EntityStore**.

-   **Thread Safety:** This system is explicitly marked as **not thread-safe**. The `isParallel` method returns `false`, which instructs the ECS scheduler to execute its `tick` method serially. This is a critical design choice to prevent race conditions. If two nearby items were processed in parallel, they might both attempt to merge with a third item simultaneously, leading to item duplication, loss, or world state corruption. All merging logic must be executed from a single thread to ensure transactional integrity.

## API Surface
The public API is exclusively for consumption by the ECS scheduler. Developers do not interact with these methods directly.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getQuery() | Query | O(1) | Returns the pre-built query used by the scheduler to identify entities this system should process. |
| isParallel(...) | boolean | O(1) | Returns `false`, indicating the system's logic cannot be executed in parallel. **Warning:** Overriding this behavior will cause severe state corruption. |
| tick(...) | void | O(k) | Executes the merge logic for a single entity. Complexity is dependent on `k`, the number of entities within the 2.0 unit search radius. |

## Integration Patterns

### Standard Usage
Developers do not call this system directly. Its operation is implicit. To make an entity eligible for item merging, simply spawn a server-side entity with an **ItemComponent** and a **TransformComponent**. The system will automatically discover and process it on subsequent server ticks.

To prevent a specific item from merging, add the **PreventItemMerging** component to the entity.

```java
// Example: Spawning an entity that will be processed by this system
// This code would exist within another system or entity creation logic.

Ref<EntityStore> newItemEntity = commandBuffer.createEntity();
commandBuffer.putComponent(newItemEntity, TransformComponent.getComponentType(), new TransformComponent(position));
commandBuffer.putComponent(newItemEntity, ItemComponent.getComponentType(), new ItemComponent(someItemStack));

// The ItemMergeSystem will now automatically handle this entity.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ItemMergeSystem()`. The server's ECS framework is solely responsible for its lifecycle. Manually creating an instance will result in a non-functional object that is not registered with the game loop.
-   **Manual Invocation:** Do not call the `tick` method directly. This bypasses the ECS scheduler and its crucial **CommandBuffer** mechanism, which can lead to immediate and unpredictable world state corruption. All entity modifications must be queued through the CommandBuffer provided to the `tick` method.
-   **Forcing Parallelism:** Do not alter the system to return `true` from `isParallel`. The merging logic is fundamentally sequential and relies on exclusive access to entities within a local area. Enabling parallelism will introduce race conditions, guaranteed to cause item duplication and data loss.

## Data Pipeline
The system processes data for a single entity within one invocation of its `tick` method. The flow is deterministic and transactional through the use of a CommandBuffer.

> Flow:
> 1.  **ECS Scheduler** identifies an entity matching the system's query.
> 2.  **ItemMergeSystem.tick** is invoked with the target entity.
> 3.  **Pre-check:** The system validates that the item is not at max stack size and that its internal merge cooldown has elapsed.
> 4.  **Spatial Query:** The entity's position is read from its **TransformComponent**. The **SpatialResource** is queried to find all other entities within a 2.0 unit radius.
> 5.  **Filtering & Validation:** The system iterates through the query results, applying a chain of validation checks for each potential merge partner:
>     - Is the partner a valid, distinct entity?
>     - Does it have an **ItemComponent**?
>     - Is it stackable with the source item (`isStackableWith`)?
>     - Is it not currently being interacted with (lacks an **Interactable** component)?
> 6.  **Command Generation:** If a valid merge partner is found, the system calculates the resulting item stack quantities. It then queues commands into the **CommandBuffer**:
>     - If the partner is fully absorbed, a `removeEntity` command is queued.
>     - `setItemStack` is used to update the **ItemComponent** of one or both entities.
>     - Other components like **DespawnComponent** and **DynamicLight** may be updated to reflect the new stack size and value.
> 7.  **Execution:** The **CommandBuffer** collects all queued changes. At the end of the tick, the scheduler applies all commands atomically, ensuring a consistent world state for the next tick.

