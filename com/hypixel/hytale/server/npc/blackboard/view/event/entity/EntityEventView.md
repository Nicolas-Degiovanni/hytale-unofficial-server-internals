---
description: Architectural reference for EntityEventView
---

# EntityEventView

**Package:** com.hypixel.hytale.server.npc.blackboard.view.event.entity
**Type:** Scoped Service

## Definition
```java
// Signature
public class EntityEventView extends EventView<EntityEventView, EntityEventType, EntityEventNotification> {
```

## Architecture & Concepts

The EntityEventView is a critical component of the server-side NPC AI system. It acts as a translation and dispatch layer, converting low-level server events into high-level, semantically meaningful AI stimuli. Its primary responsibility is to observe entity-related actions within a specific World, filter them for relevance, and broadcast them to interested NPCEntity instances via the Blackboard system.

This class is fundamentally world-bound; a distinct instance of EntityEventView exists for each active World on the server. This design ensures that event processing is partitioned by world, preventing event leakage and simplifying logic in multi-world environments.

Internally, it subscribes to the core server event bus for specific events, such as PlayerInteractEvent. When a subscribed event occurs, EntityEventView extracts relevant context—such as the initiator, target, and location—and maps it to a specific EntityEventType (e.g., INTERACTION). It then delegates to its parent, EventView, which utilizes efficient spatial query structures to find and notify only the NPCs in the vicinity that have registered an interest in that event type.

The filtering mechanism is role-based, using the TagSetPlugin to determine if an NPC's configured role (defined in NPCGroup) should react to a given event. This allows designers to create NPCs that are, for example, only interactive with players, while others might only react to being attacked by other NPCs.

## Lifecycle & Ownership

-   **Creation:** An EntityEventView is not instantiated directly. It is created and managed by a parent Blackboard instance, which is itself tied to a World. The view is instantiated when the Blackboard service is first initialized for a given world.
-   **Scope:** The lifecycle of an EntityEventView is strictly coupled to the lifecycle of its parent World. It persists for the entire duration that the world is loaded and active on the server.
-   **Destruction:** The view is marked for garbage collection when its parent World is unloaded. Its subscriptions to the global event bus are managed by the EventRegistry, which handles listener cleanup.

## Internal State & Concurrency

-   **State:** This class is highly stateful and mutable. It maintains a map of EventTypeRegistration objects, which in turn hold spatial data structures containing references to all NPCs that are listening for specific events. It also maintains a single, reusable EntityEventNotification object to minimize memory allocations during the event processing loop.

-   **Thread Safety:** **WARNING:** This class is not thread-safe and is designed for single-threaded access. All interactions with an EntityEventView instance **must** occur on the main server thread corresponding to its parent World. Asynchronous or multi-threaded access will lead to state corruption, concurrent modification exceptions, and unpredictable AI behavior. The use of a reusable notification object is a clear indicator of this single-threaded design contract.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getUpdatedView(ref, accessor) | EntityEventView | O(1) | Returns the correct view instance for a given entity, crucial for cross-world interactions. |
| initialiseEntity(ref, npcComponent) | void | O(N) | Registers an NPCEntity with the view, subscribing it to the events defined in its blackboard configuration. N is the number of event types. |
| processAttackedEvent(victim, attacker, accessor, eventType) | void | O(Spatial Query) | Primary entry point for the combat system to report an attack. Triggers a spatial dispatch to notify nearby NPCs. |

## Integration Patterns

### Standard Usage

The most common integration is from another server system, like the combat handler, which needs to inform the AI of an event. The system must first retrieve the correct world's Blackboard, then get the EntityEventView from it.

```java
// Example: A combat system reporting that a victim was attacked.
World victimWorld = victim.getStore().getExternalData().getWorld();
Blackboard blackboard = victimWorld.getResource(Blackboard.getResourceType());
EntityEventView eventView = blackboard.getView(EntityEventView.class);

// This call will dispatch the event to relevant NPCs near the victim.
eventView.processAttackedEvent(victimRef, attackerRef, componentAccessor, EntityEventType.ATTACKED_BY_MELEE);
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new EntityEventView(world)`. This creates an unmanaged instance that is not registered with the Blackboard system, resulting in a "phantom" view that will not receive entity registrations or function correctly. Always retrieve it from the world's Blackboard.
-   **Cross-Thread Access:** Do not call `processAttackedEvent` or any other method from an asynchronous task or a different thread. All interactions must be synchronized with the world's main tick loop.
-   **State Leakage:** Do not hold a reference to the `reusableEventNotification` object obtained from an event. This object is mutable and will be overwritten by the next event processed by the view.

## Data Pipeline

The flow of data for entity events is a multi-stage process that starts with a raw server action and ends with a specific NPC behavior trigger.

> Flow:
> Core Server Event (e.g., PlayerInteractEvent) -> **EntityEventView** Event Handler (e.g., onPlayerInteraction) -> Generic `onEvent` Dispatch -> Parent `EventView` Spatial Query -> `NPCEntity::notifyEntityEvent` Callback -> NPC Behavior Tree Activation

