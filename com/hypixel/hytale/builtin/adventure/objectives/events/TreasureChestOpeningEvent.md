---
description: Architectural reference for TreasureChestOpeningEvent
---

# TreasureChestOpeningEvent

**Package:** com.hypixel.hytale.builtin.adventure.objectives.events
**Type:** Transient

## Definition
```java
// Signature
public class TreasureChestOpeningEvent implements IEvent<String> {
```

## Architecture & Concepts
The TreasureChestOpeningEvent is an immutable Data Transfer Object (DTO) that functions as a message on the server-side event bus. It represents a specific, high-level gameplay moment: a player opening a treasure chest that is tracked by an adventure objective.

This class is a cornerstone of the engine's decoupled architecture. It allows the system responsible for world interaction (detecting the chest opening) to signal other, unrelated systems (like the objective or quest tracker) without requiring direct dependencies.

The design heavily favors indirection and serialization-safe identifiers over direct object references. By using UUID for entities and a Ref for the player, the event remains lightweight and avoids pinning large objects like the EntityStore in memory. The inclusion of a Store object provides the necessary context for listeners to resolve these references back into live game objects.

## Lifecycle & Ownership
- **Creation:** Instantiated by server-side game logic when a player's interaction with a chest entity is validated. This is typically handled by a system managing block or entity interactions, which then publishes the event to the central event bus.
- **Scope:** The object's lifetime is exceptionally brief. It exists only for the duration of its dispatch through the event bus. It is a fire-and-forget message.
- **Destruction:** Once all registered event listeners have processed the event, it falls out of scope within the event bus dispatcher and becomes eligible for garbage collection. No manual cleanup is required.

## Internal State & Concurrency
- **State:** **Immutable**. All member fields are final and are assigned exclusively during construction. This is a critical design choice for event objects, as it guarantees that no listener can mutate the event's data, which would lead to unpredictable side effects for other listeners processing the same event instance.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutability, an instance of TreasureChestOpeningEvent can be safely read by multiple listeners across different threads without any need for locks or synchronization primitives. This is essential for a high-performance, multi-threaded server architecture.

## API Surface
The public API consists solely of accessors for retrieving the event's context data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getObjectiveUUID() | UUID | O(1) | Returns the unique identifier for the associated adventure objective. |
| getChestUUID() | UUID | O(1) | Returns the unique identifier of the chest entity that was opened. |
| getPlayerRef() | Ref<EntityStore> | O(1) | Returns a lightweight, serializable reference to the player who opened the chest. |
| getStore() | Store<EntityStore> | O(1) | Returns the data store context required to resolve the playerRef. |

## Integration Patterns

### Standard Usage
This event is not intended to be created or managed directly by most systems. Instead, systems subscribe to it via the engine's event bus to react when a relevant chest is opened.

```java
// In an ObjectiveTrackingSystem or similar service
@Subscribe
public void onTreasureChestOpened(TreasureChestOpeningEvent event) {
    UUID objectiveId = event.getObjectiveUUID();
    
    // Check if this objective is active for the player
    if (isObjectiveActive(event.getPlayerRef(), objectiveId)) {
        // Resolve the player reference to get the actual entity
        EntityStore player = event.getStore().get(event.getPlayerRef());
        
        // Update objective progress
        updateProgress(player, objectiveId, event.getChestUUID());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Holding References:** Do not store a reference to this event object after the listener method completes. Its data may become stale, and it prevents the object from being garbage collected.
- **Manual Instantiation:** Listeners or general game systems should never create this event using `new TreasureChestOpeningEvent()`. Its creation is the sole responsibility of the core interaction logic that validates the physical action in the game world.
- **Resolving Unrelated Entities:** The provided Store should only be used to resolve the playerRef contained within the event. Using it to fetch other, unrelated entities from within the event listener creates hidden dependencies and makes the system harder to reason about.

## Data Pipeline
The event acts as a message that flows from the core world simulation to various gameplay logic systems.

> Flow:
> Player Interaction Packet -> Server World Simulation -> **TreasureChestOpeningEvent** (Creation & Dispatch) -> Event Bus -> ObjectiveTrackingSystem (Listener) -> Player State Update & Database Commit

