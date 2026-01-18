---
description: Architectural reference for EventTypeRegistration
---

# EventTypeRegistration

**Package:** com.hypixel.hytale.server.npc.blackboard.view.event
**Type:** Stateful Component

## Definition
```java
// Signature
public class EventTypeRegistration<EventType extends Enum<EventType>, NotificationType extends EventNotification> {
```

## Architecture & Concepts

The EventTypeRegistration class is a high-performance, specialized dispatcher responsible for managing all entity subscriptions for a *single, specific type of event*. It is a core component of the server-side NPC AI sensory system, acting as the bridge between an event occurrence and the AI logic of interested NPCs.

This class is not a generic event bus. Instead, a distinct instance of EventTypeRegistration is created for each unique event type (e.g., HEARD_SOUND, SAW_ENEMY). This design choice optimizes for performance by partitioning the event space.

The central architectural pattern is the use of integer-based "sets" for entity grouping. An NPC can be associated with multiple sets, which might represent factions, physical locations, or behavioral states. When an event is relayed, this class efficiently iterates only through the sets relevant to the event's sender, avoiding a costly iteration over every NPC in the world.

Filtering is achieved via a functional interface, `BiIntPredicate setTester`, which is provided during construction. This predicate decouples the event sender from the receivers, allowing for complex, data-driven rules to determine if a group of NPCs should be notified. For example, a rule could specify that NPCs in set 10 (Faction: Guardians) should only process events from senders in type group 2 (Players), but not type group 3 (Monsters).

## Lifecycle & Ownership

-   **Creation:** EventTypeRegistration instances are not intended for direct instantiation by feature developers. They are created and managed by a higher-level system, typically a central `BlackboardEventService` or equivalent manager. This manager will create one instance per `EventType` enum value that the system needs to handle.
-   **Scope:** An instance persists for the lifetime of the managing service, which is typically tied to the server or world session. It accumulates entity registrations as they are initialized and remains active to process events.
-   **Destruction:** The object is eligible for garbage collection when the parent event management service is shut down. The `cleanup` method is **not** a destructor; it is a periodic maintenance routine that must be called to prune stale entity references and prevent memory leaks.

## Internal State & Concurrency

-   **State:** This class is highly stateful and mutable. Its primary state is stored in two fields:
    -   `eventSets`: A BitSet that tracks which integer sets currently have at least one listening entity. This provides a fast way to iterate only over active sets.
    -   `entitiesBySet`: A map from a set ID to a list of entity references. This is the primary data cache for the dispatcher.

-   **Thread Safety:** **This class is not thread-safe.** All internal data structures (Int2ObjectOpenHashMap, ObjectArrayList, BitSet) are unsynchronized. All method calls that mutate or read state, such as `initialiseEntity`, `relayEvent`, and `cleanup`, must be executed from a single, controlled thread, typically the main server tick thread.

    **WARNING:** Concurrent access will lead to `ConcurrentModificationException`, data corruption, or other unpredictable behavior. External synchronization is required if multi-threaded access is unavoidable.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initialiseEntity(ref, changeSets) | void | O(k) | Registers an entity with one or more sets. *k* is the number of sets provided. |
| relayEvent(...) | void | O(S * E) | The primary dispatch function. Iterates through relevant sets and notifies all registered entities. *S* is the number of active sets, and *E* is the average number of entities per set. |
| cleanup() | void | O(N) | Performs internal maintenance, removing invalid entity references. *N* is the total number of registered entity references across all sets. Must be called periodically. |
| getSetCount() | int | O(1) | Returns the number of sets that have active listeners. |
| forEach(setConsumer, npcConsumer) | void | O(N) | Iterates over every registered set and every valid entity within those sets. |

## Integration Patterns

### Standard Usage

This class is designed to be managed by a singleton service. The service holds a map of `EventType` to `EventTypeRegistration`. The standard flow involves the manager delegating calls based on the event.

```java
// In a hypothetical BlackboardEventService

// When an NPC is spawned or its sets change
EventTypeRegistration registration = getRegistrationForType(npc.getListenType());
registration.initialiseEntity(npc.getRef(), npc.getEventSets());

// When an event occurs in the world
NotificationType notification = createNotification(source);
EventTypeRegistration registration = getRegistrationForType(notification.getType());
registration.relayEvent(senderTypeId, notification, ...);

// During a periodic maintenance tick
for (EventTypeRegistration reg : allRegistrations) {
    reg.cleanup();
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Do not use `new EventTypeRegistration()`. The lifecycle must be controlled by a central service to ensure events are routed correctly and state is managed globally.
-   **State Leakage:** Do not hold a reference to an EventTypeRegistration instance outside of a managing service. This can prevent proper garbage collection and lead to confusion about the authority of state.
-   **Omitting Cleanup:** Failure to periodically call the `cleanup` method will result in a memory leak. The internal lists will grow indefinitely with references to destroyed entities, and performance will degrade as `relayEvent` attempts to process invalid targets.
-   **Concurrent Modification:** Do not call `initialiseEntity` or `cleanup` from another thread while `relayEvent` could be executing. All interactions must be serialized onto the main game thread.

## Data Pipeline

The flow of data for a single event through this component is linear and synchronous.

> Flow:
> World Event (e.g., Sound Emitter) -> Central Event Service -> **EventTypeRegistration.relayEvent** -> Filtering via `setTester` -> Entity List Iteration -> `IEventCallback.notify` -> NPC Blackboard Update

