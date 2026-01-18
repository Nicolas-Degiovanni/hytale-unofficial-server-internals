---
description: Architectural reference for DropItemEvent
---

# DropItemEvent

**Package:** com.hypixel.hytale.server.core.event.events.ecs
**Type:** Transient

## Definition
```java
// Signature
public class DropItemEvent extends CancellableEcsEvent {
    // Nested static final classes provide specific event subtypes
    public static final class Drop extends DropItemEvent { /* ... */ }
    public static final class PlayerRequest extends DropItemEvent { /* ... */ }
}
```

## Architecture & Concepts

The DropItemEvent class is not a single event but a **protocol** representing the distinct stages of a player-initiated item drop sequence. It serves as a data contract that decouples player input from inventory logic and world simulation within the server's Entity Component System (ECS).

This class family follows a two-stage command pattern:

1.  **PlayerRequest:** An *intent* event. This is fired when the server receives a network packet indicating a player wishes to drop an item from a specific inventory slot. It contains only the raw identifiers needed to locate the item.
2.  **Drop:** An *action* event. This is fired by an authoritative system (e.g., InventorySystem) *after* it has validated the PlayerRequest, resolved the slot identifiers to a concrete ItemStack, and confirmed the action is permissible. This event contains the actual ItemStack to be dropped and physical parameters like throw speed.

By extending CancellableEcsEvent, any listener in the chain can veto the operation. This allows for flexible implementation of game rules, such as preventing item drops in protected zones or during combat, without modifying the core inventory or network code.

### Lifecycle & Ownership

-   **Creation:**
    -   A **PlayerRequest** is instantiated by a network packet handler upon receiving a corresponding client request.
    -   A **Drop** is instantiated by a core game system, typically an inventory manager, in response to processing a valid PlayerRequest.
-   **Scope:** Extremely short-lived. An instance of DropItemEvent exists only for the duration of the event processing phase within a single server tick. It is created, broadcast to all relevant ECS listeners, and immediately becomes eligible for garbage collection.
-   **Destruction:** Managed automatically by the Java Garbage Collector. No manual cleanup is required. Holding a reference to an event object past the current tick is a design error and will cause memory leaks.

## Internal State & Concurrency

-   **State:** The event objects are mutable by design. Specifically, the **Drop** subclass contains setters that allow intermediary systems to modify the ItemStack or its physical properties (e.g., throwSpeed) before it is ultimately consumed by the world simulation system. The **PlayerRequest** is effectively immutable as its fields are final.
-   **Thread Safety:** **Not thread-safe.** Event objects are stateful and are intended for synchronous processing on the main server thread only. Accessing or modifying an event from a worker thread will result in race conditions, data corruption, and non-deterministic behavior. All event handling logic must be confined to the server's primary game loop.

## API Surface

This API is divided between the two nested event types.

### PlayerRequest
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInventorySectionId() | int | O(1) | Returns the unique identifier for the inventory container. |
| getSlotId() | short | O(1) | Returns the index of the slot within the specified inventory section. |

### Drop
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getItemStack() | ItemStack | O(1) | Returns the non-null ItemStack instance being dropped. |
| setItemStack(ItemStack) | void | O(1) | Overwrites the ItemStack. Allows listeners to swap or modify the item. |
| getThrowSpeed() | float | O(1) | Returns the initial velocity for the item entity. |
| setThrowSpeed(float) | void | O(1) | Modifies the initial velocity. |

## Integration Patterns

### Standard Usage

Systems subscribe to these events via the ECS event bus. A typical implementation involves two separate systems: one for validation and one for execution.

```java
// System 1: Validates the player's request
@Subscribe
public void onPlayerRequestDrop(DropItemEvent.PlayerRequest event, Entity player) {
    // Logic to check if the player is in a no-drop zone
    if (isInProtectedZone(player.getPosition())) {
        event.setCancelled(true); // Veto the action
        return;
    }

    // If valid, the InventorySystem will see the un-cancelled event
    // and fire a DropItemEvent.Drop in response.
}

// System 2: Spawns the item in the world after validation
@Subscribe
public void onDropItem(DropItemEvent.Drop event, Entity player) {
    // This event only fires if the PlayerRequest was not cancelled.
    world.spawnItemEntity(player.getPosition(), event.getItemStack(), event.getThrowSpeed());
}
```

### Anti-Patterns (Do NOT do this)

-   **Bypassing Protocol:** Do not fire a Drop event directly without a preceding PlayerRequest. This bypasses critical validation, permission checks, and potential modifications from other systems, leading to inconsistent game state.
-   **Asynchronous Handling:** Do not process these events on a separate thread. The state of the player and their inventory can change between ticks, and asynchronous handling will cause severe race conditions.
-   **Reference Hoarding:** Do not store a reference to an event object in a component or system field. They are ephemeral and should be considered invalid after the event handler method returns.

## Data Pipeline

The flow of data for an item drop action is a clear, multi-stage pipeline mediated by the event system.

> Flow:
> Client Network Packet -> Server Network Handler -> **DropItemEvent.PlayerRequest** -> ECS Event Bus -> Validation Systems -> Inventory System -> **DropItemEvent.Drop** -> ECS Event Bus -> World Simulation System -> New Item Entity in World

