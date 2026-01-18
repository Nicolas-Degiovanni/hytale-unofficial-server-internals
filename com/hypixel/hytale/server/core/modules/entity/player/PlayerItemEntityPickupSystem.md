---
description: Architectural reference for PlayerItemEntityPickupSystem
---

# PlayerItemEntityPickupSystem

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** System

## Definition
```java
// Signature
public class PlayerItemEntityPickupSystem extends EntityTickingSystem<EntityStore> {
```

## Architecture & Concepts
The PlayerItemEntityPickupSystem is a server-side system within the Entity Component System (ECS) framework responsible for managing the logic when a player picks up a dropped item entity. It acts as the bridge between a physical item entity in the world and a player's inventory.

This system operates on a specific set of entities defined by its internal query: item entities that are eligible for pickup. For each eligible item, it performs a spatial search to find nearby players. The core architectural feature is its dual-path processing logic:

1.  **Interaction-Driven Pickup:** If the item's asset definition specifies a custom pickup interaction, this system defers all logic to the InteractionModule. It constructs an InteractionContext, populates it with the item and player references, and executes the defined InteractionChain. This allows for highly customizable pickup behaviors (e.g., quest items, cursed items) without modifying engine code.

2.  **Direct Inventory Pickup:** If no custom interaction is defined, the system falls back to a default, hard-coded behavior. It directly accesses the player's inventory containers and attempts to add the item stack. This path handles partial pickups, where only a portion of a stack is taken, and updates the world entity accordingly.

The system relies heavily on a SpatialResource, which is an optimized spatial data structure (like a grid or k-d tree) for efficiently querying entities by location. This prevents the system from performing a brute-force check against every player on the server for every item, making it highly performant.

## Lifecycle & Ownership
-   **Creation:** Instantiated once by the server's primary ECS world builder during the initialization of server-side entity modules. It is not intended for manual instantiation.
-   **Scope:** Singleton within the context of an EntityStore. It persists for the entire lifetime of the game world.
-   **Destruction:** The system is destroyed and garbage collected only when the server shuts down and the associated EntityStore is deconstructed.

## Internal State & Concurrency
-   **State:** This class is stateless. Its fields are final references to component types and resources, configured during construction. All state it modifies (item position, player inventory, etc.) is stored externally in ECS components.
-   **Thread Safety:** This system is **not thread-safe**. The implementation of the isParallel method explicitly returns false, signaling to the ECS scheduler that its tick method must be executed serially. This is a critical design choice to prevent race conditions, such as two item entities attempting to modify the same player's inventory simultaneously from different threads.

## API Surface
The primary API is the contract with the ECS scheduler. Direct invocation is an anti-pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, chunk, store, commands) | void | Varies | The main entry point called by the ECS scheduler each tick. Complexity depends on player density near the item, as it performs a spatial query. |
| getDependencies() | Set | O(1) | Declares an execution dependency, ensuring it runs *after* the PlayerSpatialSystem has updated the player location index for the current frame. |
| getQuery() | Query | O(1) | Defines the set of entities this system will operate on. Used by the ECS scheduler to feed the system with relevant data chunks. |

## Integration Patterns

### Standard Usage
This system is not invoked directly. Its functionality is triggered by creating an entity in the world that matches its query. To make an item pickup-able, a developer would spawn an entity with, at a minimum, an ItemComponent and a TransformComponent.

```java
// A separate system or game logic module would create an item entity like this.
// The PlayerItemEntityPickupSystem will then automatically process it.

Holder<EntityStore> itemEntityHolder = world.createEntity();
itemEntityHolder.addComponent(new TransformComponent(position));
itemEntityHolder.addComponent(new ItemComponent(someItemStack));

commandBuffer.addEntity(itemEntityHolder, AddReason.SPAWN);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new PlayerItemEntityPickupSystem()`. The system is managed entirely by the ECS world.
-   **Manual Invocation:** Never call the `tick` method directly. Doing so bypasses the ECS scheduler, dependency management, and the CommandBuffer system, which will lead to concurrency issues, state corruption, and unpredictable behavior.
-   **Stateful Logic:** Do not modify this system to hold state related to a specific entity or player. All state must be stored in components.

## Data Pipeline
The flow of data and logic for a single item entity during one tick is as follows.

> Flow:
> ECS Scheduler -> **PlayerItemEntityPickupSystem.tick()**
> 1. Reads item's **TransformComponent** for its world position.
> 2. Queries **SpatialResource** with the item's position to find nearby player entities.
> 3. Checks **Item** asset data for a defined pickup **Interaction**.
> 4. **IF** Interaction exists:
>    - Creates **InteractionContext**.
>    - Dispatches to **InteractionManager** to execute an **InteractionChain**.
>    - Writes `removeEntity` command to **CommandBuffer**.
> 5. **ELSE** (no Interaction):
>    - Accesses player's **ItemContainer**.
>    - Performs **ItemStackTransaction**.
>    - **IF** fully picked up: Writes `removeEntity` command to **CommandBuffer**.
>    - **IF** partially picked up: Writes `setComponent` command to **CommandBuffer** to update the item's remaining stack.
> 6. Writes `addEntity` command to **CommandBuffer** to spawn a visual pickup effect.
>
> **Output:** A series of commands enqueued in the **CommandBuffer** to be executed at the end of the tick.

