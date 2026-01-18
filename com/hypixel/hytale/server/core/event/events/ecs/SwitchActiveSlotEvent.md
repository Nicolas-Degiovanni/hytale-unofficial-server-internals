---
description: Architectural reference for SwitchActiveSlotEvent
---

# SwitchActiveSlotEvent

**Package:** com.hypixel.hytale.server.core.event.events.ecs
**Type:** Transient Data Transfer Object (DTO)

## Definition
```java
// Signature
public class SwitchActiveSlotEvent extends CancellableEcsEvent {
```

## Architecture & Concepts
The SwitchActiveSlotEvent is a message object used within the server-side Entity Component System (ECS) to signal an entity's intent to change its active inventory slot. It is not a service or manager, but rather a piece of transient data that communicates state change requests between different game systems.

As a subclass of CancellableEcsEvent, its primary architectural feature is that the requested action can be vetoed by any system that processes it. This provides a powerful hook for game logic, such as status effects (e.g., being stunned) or zone-based restrictions, to prevent a player from changing their held item.

The event's design carefully distinguishes between client-initiated requests and server-initiated actions via the *serverRequest* flag. This is a critical control mechanism to prevent feedback loops where a server-side correction might otherwise be misinterpreted as a new, distinct client action.

The mutability of the *newSlot* field is a deliberate design choice, allowing intermediary systems to not only cancel the event but also to redirect it, changing the final destination slot before the action is committed.

### Lifecycle & Ownership
- **Creation:** An instance is created under two primary conditions:
    1. By a network packet handler when a client sends a request to change their active hotbar slot.
    2. By an internal game system or script that needs to programmatically force an entity to switch its held item.
- **Scope:** The object's lifetime is exceptionally short, confined to the duration of a single game tick's event processing phase. It is created, dispatched to all relevant listeners via the ECS event bus, and then immediately becomes eligible for garbage collection.
- **Destruction:** No system should maintain a reference to this event after its handler method completes. It is unmanaged and will be garbage collected once it falls out of the event bus's dispatch scope.

## Internal State & Concurrency
- **State:** The object is a container for both immutable and mutable state. The context of the event (*previousSlot*, *inventorySectionId*, *serverRequest*) is final and cannot be changed after instantiation. However, the outcome of the event (*newSlot*) is mutable, allowing for in-flight modification by event listeners.
- **Thread Safety:** This class is **not thread-safe**. All ECS events are processed sequentially on the main server game thread. Accessing or modifying an instance of this event from any other thread will result in race conditions, state corruption, and undefined behavior. The engine's single-threaded event processing model obviates the need for explicit locking.

## API Surface
The public contract is designed for inspection, modification, and cancellation of the slot switch request.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getNewSlot() | byte | O(1) | Retrieves the target slot for the switch. |
| setNewSlot(byte newSlot) | void | O(1) | Modifies the target slot. Allows a system to redirect the switch. |
| isClientRequest() | boolean | O(1) | Returns true if the event was initiated by a player's client. |
| isServerRequest() | boolean | O(1) | Returns true if the event was initiated by internal server logic. |
| cancel() | void | O(1) | Inherited. Flags the event to be ignored by subsequent systems. |

## Integration Patterns

### Standard Usage
The primary integration pattern is for an ECS System to subscribe to this event. The system inspects the event's properties, applies its own logic (e.g., checking for permissions or status effects), and then either ignores, cancels, or modifies it. The final system in the chain is responsible for committing the state change to the relevant inventory component.

```java
// Example: A system that processes the final slot switch
@EventHandler
public void onSwitchSlot(SwitchActiveSlotEvent event, Entity e, PlayerInventoryComponent inventory) {
    // Critical: Always check for cancellation first
    if (event.isCancelled()) {
        return;
    }

    // Apply the change requested by the event
    inventory.setActiveSlot(event.getNewSlot());
}
```

### Anti-Patterns (Do NOT do this)
- **Holding References:** Do not store an instance of SwitchActiveSlotEvent in a field or collection. Its lifecycle is bound to a single method invocation on the event bus. Storing it constitutes a memory leak and logic error.
- **Asynchronous Processing:** Do not dispatch this event to another thread for processing. All modifications and checks must occur synchronously on the main game thread.
- **Ignoring Cancellation:** Any system that enacts a permanent state change (e.g., modifying a component) based on this event *must* check the result of isCancelled before proceeding. Failure to do so breaks the core contract of the event system.

## Data Pipeline
The flow of data represented by this event typically follows a clear path from input to final state change within a single server tick.

> Flow:
> Client Input Packet -> Network Handler -> **SwitchActiveSlotEvent** (Created) -> ECS Event Bus -> [Validation Systems] -> [Modification Systems] -> [Finalization System] -> PlayerInventoryComponent (Updated)

