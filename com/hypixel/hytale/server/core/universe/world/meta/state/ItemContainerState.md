---
description: Architectural reference for ItemContainerState
---

# ItemContainerState

**Package:** com.hypixel.hytale.server.core.universe.world.meta.state
**Type:** State Object

## Definition
```java
// Signature
public class ItemContainerState extends BlockState implements ItemContainerBlockState, DestroyableBlockState, MarkerBlockState {
```

## Architecture & Concepts

The ItemContainerState is a server-side data component that represents the state and behavior of any block capable of storing items, such as chests, barrels, or machinery. It acts as the critical bridge between a physical block's presence in a WorldChunk and the abstract data model of an inventory, represented by an ItemContainer.

This class is not a standalone service; it is intrinsically bound to a specific block at a specific coordinate in the world. Its primary responsibilities include:
1.  **Data Persistence:** Managing the serialization and deserialization of its inventory and properties via a declarative Codec. This ensures that container contents are correctly saved and loaded with the world.
2.  **Lifecycle Management:** Implementing the DestroyableBlockState interface to define cleanup logic, ensuring that when the associated block is broken, its contents are dropped into the world as item entities.
3.  **Player Interaction State:** Tracking which players currently have the container's user interface open. This is managed via a concurrent map of player UUIDs to their corresponding ContainerBlockWindow instances.
4.  **Configuration:** The state is initialized based on properties defined in the corresponding BlockType asset, such as inventory capacity.

This component is fundamental to the server's item economy and world interactivity. It encapsulates all logic related to a block's inventory, preventing this logic from polluting higher-level systems like the World or WorldChunk.

### Lifecycle & Ownership

The lifecycle of an ItemContainerState is strictly managed by the world engine and is tied directly to the block it represents.

-   **Creation:** An instance is created under two conditions:
    1.  When a player places a new container block in the world.
    2.  When a WorldChunk is loaded from storage, and the saved data indicates a block at a given position has an ItemContainerState. The instance is deserialized from the chunk data using its static CODEC.
    Immediately following instantiation, the `initialize` method is called by the engine to configure the internal ItemContainer based on the block's type definition.

-   **Scope:** The object exists for the exact duration that its corresponding physical block exists in the world. It is held in memory as part of the WorldChunk's metadata.

-   **Destruction:** The `onDestroy` method is invoked by the WorldChunk system immediately before the block is removed from the world. This is a critical cleanup step that guarantees game-state consistency by dropping all contained items and closing any open user interfaces.

## Internal State & Concurrency

-   **State:** The ItemContainerState is highly **mutable**. Its primary state, the SimpleItemContainer, is frequently modified as players add or remove items. It maintains two categories of state:
    -   **Persistent State:** Fields defined in the `CODEC` (e.g., `itemContainer`, `custom`, `droplist`) are written to disk when the parent chunk is saved. Any modification to these fields *must* be followed by a call to `markNeedsSave`.
    -   **Runtime State:** The `windows` map is a transient, in-memory cache of active user interfaces. It is not saved and is rebuilt as players interact with the container.

-   **Thread Safety:** This class is **not inherently thread-safe** and requires careful management.
    -   Modifications to the `itemContainer` or other persistent fields must be performed on the main world thread or be properly synchronized. The standard pattern is to wrap world-modifying logic in a `world.execute(...)` call.
    -   The `windows` map is a ConcurrentHashMap, making it safe to add or remove player windows from multiple network threads simultaneously.
    -   The `onDestroy` method correctly schedules the creation of item entities on the main world thread, preventing race conditions with the EntityStore.

    **Warning:** Directly accessing and modifying the ItemContainer from an asynchronous task without scheduling it on the main world thread will lead to data corruption and server instability.

## API Surface

The public API provides access points for interaction logic and state management.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initialize(BlockType) | boolean | O(N) | **[Lifecycle]** Configures the internal container based on the block's asset definition. Drops N excess items if capacity is reduced. |
| onDestroy() | void | O(M) | **[Lifecycle]** Drops all M contained items into the world and performs cleanup. Critical for preventing item loss. |
| getItemContainer() | ItemContainer | O(1) | Returns the backing inventory container. This is the primary entry point for item manipulation. |
| getWindows() | Map | O(1) | Provides access to the map of currently open UI windows, keyed by player UUID. |
| canOpen(ref, accessor) | boolean | O(1) | Hook to determine if a specific entity is permitted to open this container. |
| onOpen(ref, world, store) | void | O(1) | Hook executed when an entity successfully opens the container. |

## Integration Patterns

### Standard Usage

Interaction logic, such as a player right-clicking a block, should retrieve the BlockState from the world, verify its type, and then operate on its ItemContainer.

```java
// In a player interaction handler...
Vector3i blockPosition = ...;
World world = ...;

// Retrieve the state from the world, do not create it
BlockState state = world.getBlockState(blockPosition);

if (state instanceof ItemContainerState containerState) {
    // Now interact with the container's inventory
    ItemContainer inventory = containerState.getItemContainer();
    inventory.addItem(new ItemStack(...));

    // The onItemChange listener within ItemContainerState will automatically
    // call markNeedsSave() to persist the change.
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new ItemContainerState()`. The world engine is solely responsible for its creation and lifecycle. Manually created instances will not be registered with the world, will not persist, and will cause memory leaks.

-   **Unsynchronized Modification:** Do not modify the ItemContainer from a separate thread without using the world's execution scheduler. This is a common source of concurrency bugs.
    ```java
    // BAD: Modifying state from a network or worker thread
    ItemContainer inventory = containerState.getItemContainer();
    inventory.clear(); // This will cause a ConcurrentModificationException or data corruption

    // GOOD: Scheduling the modification on the main world thread
    world.execute(() -> {
        ItemContainer inventory = containerState.getItemContainer();
        inventory.clear();
    });
    ```

-   **State Caching:** Avoid holding a reference to an ItemContainerState for longer than the current operation. The block could be destroyed at any time, invalidating the state object. Always re-fetch it from the world by its position when needed.

## Data Pipeline

The flow of data through this component is critical for both persistence and gameplay events like block destruction.

**Data Persistence Pipeline (Saving)**
> Flow:
> Player Action -> `ItemContainer.addItem()` -> `ItemContainerState.onItemChange()` -> `markNeedsSave()` -> WorldChunk Serialization -> **ItemContainerState.CODEC** -> Disk Storage

**Item Drop Pipeline (Block Destruction)**
> Flow:
> Block Break Event -> `World.setBlock()` -> `WorldChunk` -> **`ItemContainerState.onDestroy()`** -> `ItemComponent.generateItemDrops()` -> `World.execute()` -> `EntityStore.addEntities()` -> New Item Entities in World

