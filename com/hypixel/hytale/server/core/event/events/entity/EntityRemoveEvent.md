---
description: Architectural reference for EntityRemoveEvent
---

# EntityRemoveEvent

**Package:** com.hypixel.hytale.server.core.event.events.entity
**Type:** Transient Data Object

## Definition
```java
// Signature
public class EntityRemoveEvent extends EntityEvent<Entity, String> {
```

## Architecture & Concepts
The EntityRemoveEvent is a server-side, immutable data object that signals a critical state transition in the game world: the permanent removal of an entity. It is a core component of the server's event-driven architecture, designed to decouple the entity management system from other game systems that must react to an entity's destruction.

This class acts as a message payload within the server's central Event Bus. When the core game logic determines an entity must be removed—due to death, despawning, or chunk unloading—it instantiates and dispatches this event. Subscribed systems, such as loot droppers, quest trackers, or network synchronizers, can then execute their logic without being tightly coupled to the entity lifecycle controller. This follows a classic Observer pattern, where the EntityRemoveEvent is the notification sent to all observers.

**WARNING:** This event signifies that an entity is *about to be* removed. By the time listeners receive this event, the entity is already in a terminal state and should be considered read-only.

### Lifecycle & Ownership
-   **Creation:** Instantiated exclusively by the server's core entity management system (e.g., an EntityManager or World instance) when an entity is scheduled for removal from the game state. Application-level code or plugins should never create this event directly.
-   **Scope:** Extremely short-lived. An instance of this event exists only for the duration of its dispatch through the event bus. It is a fire-and-forget signal.
-   **Destruction:** The object becomes eligible for garbage collection immediately after the event dispatch cycle is complete and all listener invocations have returned. There are no persistent references to event instances.

## Internal State & Concurrency
-   **State:** The class is effectively immutable. Its primary state is the reference to the `Entity` being removed, which is set once in the constructor and cannot be changed. It holds no other mutable state.
-   **Thread Safety:** As an immutable object, EntityRemoveEvent is inherently thread-safe for read access. However, the event bus that dispatches it typically operates on the main server thread. Listeners processing this event must assume they are executing on the primary game loop tick and must not perform blocking operations. Any modifications to shared game state must be synchronized or queued if they interact with other threads.

## API Surface
The public contract is minimal, focusing on providing access to the entity being removed. Most interaction is via the inherited `getEntity` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getEntity() | Entity | O(1) | (Inherited) Returns the entity instance that is being removed. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to create a listener method within a game system and annotate it to subscribe to the event. The EventManager will automatically invoke the method when the event is dispatched.

```java
// Example within a hypothetical QuestSystem
import com.hypixel.hytale.server.core.event.EventSubscribe;

public class QuestSystem {

    @EventSubscribe
    public void onEntityRemoved(EntityRemoveEvent event) {
        Entity removedEntity = event.getEntity();
        
        // Check if this entity was a target for any active quests
        // and update quest state accordingly.
        updateQuestObjectives(removedEntity.getId());
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new EntityRemoveEvent()`. Doing so outside the core entity manager will broadcast a false signal, leading to game state corruption, desynchronization, and potential null pointer exceptions if other systems act on an entity that is not actually being removed.
-   **State Modification:** Do not attempt to modify the state of the entity returned by `getEntity()`. The entity is in its final moments; changing its position, health, or inventory can cause race conditions with the final cleanup logic and is undefined behavior.
-   **Event Cancellation:** This event is a notification, not a request. It cannot be cancelled. It is a statement of fact that an entity's removal is already in progress.

## Data Pipeline
The EntityRemoveEvent is a terminal point in an entity's data lifecycle. Its flow through the server systems is linear and synchronous within a single game tick.

> Flow:
> Game Logic (e.g., HealthSystem, DespawnSystem) -> EntityManager.removeEntity(entity) -> **new EntityRemoveEvent(entity)** -> EventManager.dispatch() -> System Listeners (QuestSystem, LootSystem, NetworkSystem) -> Final Entity Deregistration

