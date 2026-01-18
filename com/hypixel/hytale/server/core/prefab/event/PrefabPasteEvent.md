---
description: Architectural reference for PrefabPasteEvent
---

# PrefabPasteEvent

**Package:** com.hypixel.hytale.server.core.prefab.event
**Type:** Transient Event Object

## Definition
```java
// Signature
public class PrefabPasteEvent extends CancellableEcsEvent {
```

## Architecture & Concepts
The PrefabPasteEvent is a message object used within the server's event-driven architecture. It represents a discrete, high-level action: the pasting of a prefab structure into the game world. This class does not perform the paste operation itself; rather, it serves as a notification and control mechanism that decouples the initiator of the paste from the systems that execute and validate it.

By extending CancellableEcsEvent, this event integrates directly with the Entity Component System (ECS). Systems can subscribe to this event to implement custom logic, such as:
*   **Validation:** A land-claiming system could listen for this event and cancel it if the paste operation would violate a player's claim.
*   **Resource Management:** A faction system could listen for the event and deduct the cost of the prefab from a faction's resources.
*   **Post-Processing:** A lighting system could listen for the completion event (when pasteStart is false) to trigger a relighting of the affected chunks.

The boolean flag, pasteStart, is critical. It transforms a single action into a two-phase transaction, providing hooks for both pre-operation validation and post-operation cleanup.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by a high-level system that needs to place a prefab into the world. This could be a player command handler, a quest script, or a world generation routine.
- **Scope:** Extremely short-lived. The object's lifetime is confined to a single tick and the duration of its dispatch through the server's event bus.
- **Destruction:** The event object becomes eligible for garbage collection as soon as the event bus has finished notifying all listeners. There are no long-term references to it.

## Internal State & Concurrency
- **State:** Immutable. The core data fields, prefabId and pasteStart, are final and set during construction. This guarantees that the event's context cannot change during its dispatch. The only mutable state is the cancellation flag inherited from CancellableEcsEvent, which is managed by the event bus.
- **Thread Safety:** The object itself is inherently thread-safe due to its immutability. It can be safely passed across threads.
    **WARNING:** While the event object is thread-safe, the systems that *listen* for it are almost certainly not. Listeners modifying world state in response to this event must implement their own synchronization or be designed to run exclusively on the main server thread.

## API Surface
The public contract is minimal, focused on data retrieval. The most important inherited method is *cancel()*.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPrefabId() | int | O(1) | Returns the unique identifier of the prefab being pasted. |
| isPasteStart() | boolean | O(1) | Returns true if the event is fired *before* the paste, and false if fired *after*. |
| cancel() | void | O(1) | Inherited from CancellableEcsEvent. Sets an internal flag to prevent the paste operation. Only effective if called on an event where isPasteStart() is true. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to create a system that subscribes to the event and performs logic based on its data.

```java
// Example of a validation system listening for the event
public class ClaimValidationSystem extends EcsSystem {
    
    @Subscribe
    public void onPrefabPaste(PrefabPasteEvent event) {
        // Only act on the pre-paste event
        if (!event.isPasteStart()) {
            return;
        }

        int prefabId = event.getPrefabId();
        // Fictional API to check if the paste is allowed
        if (!world.getClaimManager().canPasteAtLocation(prefabId, event.getLocation())) {
            // Veto the operation
            event.cancel();
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Holding References:** Do not cache or store an instance of PrefabPasteEvent in a field. Its data is only valid for the duration of the event dispatch.
- **Ignoring pasteStart:** Failing to check the isPasteStart flag is a common source of bugs. A system might incorrectly apply logic twice (before and after the paste) or attempt to cancel an event that has already completed.
- **Asynchronous Cancellation:** Do not attempt to cancel the event from another thread without proper synchronization. Event dispatch is typically synchronous, and cancellation is expected to occur within the listener's call stack.

## Data Pipeline
The event acts as a data carrier in a well-defined server-side process.

> Flow:
> Command or Script Execution -> **PrefabPasteEvent** (pasteStart=true) -> Server Event Bus -> Validation Systems (may cancel) -> Prefab Pasting System -> World State Modification -> **PrefabPasteEvent** (pasteStart=false) -> Server Event Bus -> Post-Processing Systems

