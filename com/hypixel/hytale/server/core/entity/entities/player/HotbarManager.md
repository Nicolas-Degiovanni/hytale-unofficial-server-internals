---
description: Architectural reference for HotbarManager
---

# HotbarManager

**Package:** com.hypixel.hytale.server.core.entity.entities.player
**Type:** Component Data Object

## Definition
```java
// Signature
public class HotbarManager {
```

## Architecture & Concepts
The HotbarManager is a server-side, stateful component responsible for managing the saved hotbars feature available to players in Creative Mode. It is not a standalone service but rather a data-holding object that is attached to a specific player entity within the Entity-Component-System (ECS) architecture.

Its primary role is to encapsulate the state and logic for persisting and restoring up to 10 distinct hotbar layouts. This allows a player to save a toolkit of items and recall it on demand without manually rebuilding it.

A critical architectural feature is the static **CODEC** field. This self-contained serialization contract allows the Hytale persistence engine to automatically save and load the state of this component as part of the player entity's data. When a player is written to the world database, the CODEC serializes the `savedHotbars` array and `currentHotbar` index. When the player is loaded, the CODEC reconstructs the HotbarManager to its exact previous state.

This component acts as a data model that is manipulated by server-side systems, typically in response to player input packets.

## Lifecycle & Ownership
- **Creation:** A HotbarManager instance is not created manually via its constructor. It is instantiated by the server's persistence layer when a player entity is loaded from storage, using the logic defined in its static CODEC. A new, default instance is created when a player entity is first generated.
- **Scope:** The lifecycle of a HotbarManager is strictly bound to the lifecycle of its parent player entity. It persists as long as the player exists in the game world.
- **Destruction:** The object is eligible for garbage collection when the player entity is unloaded from the server, for example, when the player logs out or the server shuts down. There is no manual destruction method.

## Internal State & Concurrency
- **State:** The HotbarManager is highly mutable. Its core state consists of an array of ItemContainer objects, `savedHotbars`, and an integer, `currentHotbar`, tracking the active selection. It also contains a transient boolean flag, `currentlyLoadingHotbar`, to prevent re-entrant or overlapping modifications during a single operation.

- **Thread Safety:** **This class is not thread-safe.** All methods mutate internal state without any synchronization mechanisms. It is designed to be accessed exclusively from the main server thread that processes entity updates (the "game loop"). Unsynchronized access from other threads will lead to race conditions, data corruption, and server instability. The `currentlyLoadingHotbar` flag is a state-management tool, not a concurrency primitive.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| saveHotbar(playerRef, hotbarIndex, accessor) | void | O(N) | Clones the player's current hotbar and saves it to the specified index. N is the number of slots. Fails if the player is not in Creative Mode. |
| loadHotbar(playerRef, hotbarIndex, accessor) | void | O(N) | Clears the player's current hotbar and replaces its contents with the saved hotbar from the specified index. N is the number of slots. Fails if the player is not in Creative Mode. |
| getCurrentHotbarIndex() | int | O(1) | Returns the index of the last hotbar that was saved or loaded. |
| getIsCurrentlyLoadingHotbar() | boolean | O(1) | Returns a transient flag indicating if a load operation is in progress. |

## Integration Patterns

### Standard Usage
The HotbarManager is designed to be retrieved as a component from a player entity and then operated upon by a server-side system. Direct interaction is always mediated by the ECS framework.

```java
// Example within a hypothetical PlayerCommandSystem
void handleHotbarSaveCommand(Ref<EntityStore> playerEntityRef, short targetSlot) {
    ComponentAccessor<EntityStore> accessor = world.getComponentAccessor();
    
    // Retrieve the HotbarManager component from the player entity
    HotbarManager hotbarManager = accessor.getComponent(playerEntityRef, HotbarManager.class);

    if (hotbarManager != null) {
        hotbarManager.saveHotbar(playerEntityRef, targetSlot, accessor);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new HotbarManager()`. An instance created this way will be detached from any player entity, will not be persisted, and will be useless. The component must be managed by the ECS.
- **Multi-threaded Access:** Do not access a HotbarManager instance from any thread other than the main server thread. This will corrupt player data.
- **State Tampering:** Do not attempt to get and modify the internal `savedHotbars` array directly. All interactions must go through the `saveHotbar` and `loadHotbar` methods to ensure game logic is correctly applied.

## Data Pipeline
The data flow for a typical `loadHotbar` operation demonstrates the component's role within the larger server architecture.

> Flow:
> Client Input (Key Press) -> Network Packet -> Server Packet Handler -> Command/Event on Server -> Game System (e.g., PlayerInteractionSystem) -> Retrieves **HotbarManager** from Player Entity -> **HotbarManager.loadHotbar()** -> Mutates Player Inventory Component -> Player receives updated inventory state.

