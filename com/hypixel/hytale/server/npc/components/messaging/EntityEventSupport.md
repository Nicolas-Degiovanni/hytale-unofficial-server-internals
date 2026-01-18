---
description: Architectural reference for EntityEventSupport
---

# EntityEventSupport

**Package:** com.hypixel.hytale.server.npc.components.messaging
**Type:** Abstract Component

## Definition
```java
// Signature
public abstract class EntityEventSupport extends EventSupport<EntityEventType, EntityEventNotification> {
```

## Architecture & Concepts
EntityEventSupport is an abstract base class that forms a critical part of the server-side NPC artificial intelligence framework. It functions as a per-entity, short-term sensory memory and event filtering system. Within the server's Entity-Component-System (ECS) architecture, this component is responsible for receiving raw world event notifications and determining their relevance to the host entity.

Its primary architectural role is to decouple high-level AI decision-making (e.g., Behavior Trees, State Machines) from the low-level noise of the global event stream. It achieves this by applying a series of contextual filters to incoming events, most notably:

*   **Spatial Relevance:** Events occurring outside a configurable maximum range are immediately discarded.
*   **Social Context (Flocking):** The component has first-class awareness of flocking behavior, allowing it to prioritize or filter events based on whether the initiator belongs to the same flock as the host entity.
*   **Saliency:** It maintains a record of the most relevant event of a given type, updating it only if a new event is closer or has a more important social context (e.g., a non-flock event being superseded by a flock event).

Subclasses are expected to be implemented for specific categories of NPC behavior, defining the concrete logic for handling different event types.

### Lifecycle & Ownership
- **Creation:** Concrete implementations of EntityEventSupport are instantiated by the ECS framework when an entity with the corresponding component is created. It is never created manually.
- **Scope:** The lifecycle of an EntityEventSupport instance is strictly tied to the lifecycle of its parent entity. It persists as long as the entity exists in the world.
- **Destruction:** The component is marked for garbage collection when its parent entity is removed from the world. No manual cleanup is required.

## Internal State & Concurrency
- **State:** This component is stateful. It inherits an array named messageSlots from its parent, EventSupport, which acts as a mutable cache for recent, filtered events. The `postMessage` method directly modifies the state of these slots.

- **Thread Safety:** **WARNING:** This class is not thread-safe. It contains no internal locking mechanisms and assumes all method calls originate from a single, synchronized game loop thread (the main server thread). Concurrent access from multiple threads will lead to race conditions, state corruption, and unpredictable AI behavior. All interactions with this component must be confined to the thread that owns the entity's world.

## API Surface
The public API provides methods for event ingestion and state querying, intended for use by other entity components or AI systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| postMessage(type, notification, parent, store) | void | O(1) | Ingests and filters a world event. This is the primary entry point for data. It may update internal state if the event is deemed relevant. |
| hasFlockMatchingMessage(index, position, range, flockOnly) | boolean | O(1) | Queries the internal state to check if a relevant message has been cached. Used by AI logic to make decisions. |

## Integration Patterns

### Standard Usage
This component is designed to be retrieved from an entity's component store and updated by a higher-level system that subscribes to world events. AI behaviors then query its state to inform their decisions.

```java
// In an AI Behavior or System that processes notifications for an entity:

// 1. An EntityEventNotification is received from a global bus or spatial query.
EntityEventNotification notification = ...;

// 2. Retrieve the component from the target entity.
// WARNING: The concrete class, not the abstract one, must be requested.
MyEventSupportComponent eventSupport = entityStore.getComponent(parentRef, MyEventSupportComponent.class);

// 3. Post the message for filtering and potential storage.
if (eventSupport != null) {
    eventSupport.postMessage(notification.getType(), notification, parentRef, entityStore);
}

// Elsewhere, in an AI decision-making tick:
// 4. Query the component to see if a "HeardNoise" event from a flock member is nearby.
boolean heardFlockNoise = eventSupport.hasFlockMatchingMessage(HEARD_NOISE_INDEX, selfPosition, 32.0, true);
if (heardFlockNoise) {
    // Change behavior to "Investigate"
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new MyEventSupportComponent()`. The ECS framework is solely responsible for the lifecycle of components. Manually created instances will not be registered with the entity or receive updates.
- **Multi-threaded Access:** Do not call `postMessage` or `hasFlockMatchingMessage` from asynchronous tasks, network threads, or any thread other than the main server tick thread for that entity's world. This will break state consistency.
- **State Tampering:** Do not attempt to access the inherited `messageSlots` array directly, even if visibility allows. Rely exclusively on the public API to ensure filtering logic is respected.

## Data Pipeline
The component acts as a stateful filter in the data flow from raw world events to AI-driven actions.

> Flow:
> World Event (e.g., Sound, Damage) -> `EntityEventNotification` Broadcast -> Entity Component Manager -> **`EntityEventSupport.postMessage`** -> Filtering (Range, Flock) -> Internal State Update (`messageSlots`) -> AI Behavior Query (`hasFlockMatchingMessage`) -> NPC Action

