---
description: Architectural reference for InteractivelyPickupItemEvent
---

# InteractivelyPickupItemEvent

**Package:** com.hypixel.hytale.server.core.event.events.ecs
**Type:** Transient Data Object

## Definition
```java
// Signature
public class InteractivelyPickupItemEvent extends CancellableEcsEvent {
```

## Architecture & Concepts
The InteractivelyPickupItemEvent is a server-side, message-based data structure that represents a player's attempt to pick up an item entity from the game world. It is not a long-lived service but a transient payload that exists only for the duration of a single game tick's event processing phase.

This event is a critical component of the Entity Component System (ECS) event bus. Its primary architectural purpose is to decouple the *detection* of a pickup attempt (e.g., from a PlayerInteractionSystem) from the *logic* that validates and executes the pickup (e.g., an InventorySystem or a QuestSystem).

By extending CancellableEcsEvent, it participates in a chain-of-responsibility pattern. Multiple game systems can listen for this event, inspect its state, and choose to veto the action by cancelling the event. This allows for complex, emergent game logic without creating tight coupling between systems.

### Lifecycle & Ownership
-   **Creation:** Instantiated by a high-level game system that processes player interactions. For example, a system monitoring player proximity and input commands will create and dispatch this event when a pickup action is triggered.
-   **Scope:** Ephemeral. The object's lifetime is strictly confined to the event bus dispatch cycle within a single server tick. No system should ever retain a reference to this event object after the dispatch is complete.
-   **Destruction:** The object is eligible for garbage collection immediately after all registered listeners have processed it. Its memory footprint is reclaimed on the next GC cycle.

## Internal State & Concurrency
-   **State:** Mutable. The event encapsulates an ItemStack which can be read and, critically, modified by listeners via the setItemStack method. This allows systems to transform the item being picked up before it is added to an inventory.
-   **Thread Safety:** **This class is not thread-safe.** All ECS events are designed to be created, dispatched, and handled within the single main server thread. Accessing or modifying an event instance from an asynchronous task or a different thread will lead to state corruption and severe server instability.

## API Surface
The primary contract of this class includes its payload accessors and the cancellation mechanism inherited from its parent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getItemStack() | ItemStack | O(1) | Retrieves the item stack associated with the pickup attempt. Guaranteed non-null. |
| setItemStack(ItemStack) | void | O(1) | Modifies the item stack payload. Allows listeners to swap or alter the item being picked up. |
| setCancelled(boolean) | void | O(1) | Inherited. Vetoes the pickup action. Subsequent listeners can check this status. |

## Integration Patterns

### Standard Usage
The intended pattern is for systems to listen for this event, apply game logic, and potentially cancel it. A final system then acts upon un-cancelled events to complete the pickup.

```java
// In an InventorySystem listener
public void onPickupAttempt(InteractivelyPickupItemEvent event) {
    PlayerInventory inventory = getPlayerInventory(event.getEntityId());
    if (inventory.isFull()) {
        // Veto the event. The item will not be picked up.
        event.setCancelled(true);
        return;
    }
}

// In a finalization system that runs after other listeners
public void onFinalizePickup(InteractivelyPickupItemEvent event) {
    if (event.isCancelled()) {
        return;
    }
    // Logic to add event.getItemStack() to inventory and destroy world entity
}
```

### Anti-Patterns (Do NOT do this)
-   **Reference Caching:** Do not store a reference to this event in a component or system field. Its state is only valid for the instant it is being processed.
-   **Asynchronous Processing:** Do not hand this event off to another thread or a future task. All processing must be synchronous within the event callback.
-   **Ignoring Cancellation:** Systems that perform the final action of picking up an item **must** check the isCancelled flag. Failure to do so will break game logic from other systems.

## Data Pipeline
The event acts as a data carrier that flows through various server systems, each having an opportunity to inspect or halt the process.

> Flow:
> Player Input -> Interaction System -> **InteractivelyPickupItemEvent (Dispatch)** -> Event Bus -> [PermissionSystem, QuestSystem, InventorySystem] (Listeners) -> Final Pickup System (Acts on un-cancelled event)

