---
description: Architectural reference for EventSupport
---

# EventSupport

**Package:** com.hypixel.hytale.server.npc.components.messaging
**Type:** Component / Abstract Base Class

## Definition
```java
// Signature
public abstract class EventSupport<EventType extends Enum<EventType>, NotificationType extends EventNotification> extends MessageSupport {
```

## Architecture & Concepts

EventSupport is an abstract base class that provides a generic, stateful mechanism for Non-Player Characters (NPCs) to receive and process spatially-aware events. It functions as a filtered mailbox or event buffer, specifically designed to manage notifications that have a location in the game world.

This component is a foundational element of the NPC sensory system. It decouples the source of an event (e.g., a player making a sound, an explosion) from the NPC's reaction logic (e.g., a behavior tree node). Its primary responsibility is to receive a high volume of world notifications, filter them based on type and proximity to the parent NPC, and store the most relevant ones in pre-allocated "message slots".

The core design revolves around a two-tiered lookup system. An incoming notification is first categorized by its `EventType`. Then, within that type, it is mapped to a specific integer-based slot using a secondary map. This allows for highly efficient O(1) lookups to determine which slot should handle a given event, which is critical for performance on a server with many NPCs and events.

### Lifecycle & Ownership

-   **Creation:** Concrete subclasses of EventSupport are not instantiated directly via a public constructor. They are created as part of an NPC's component assembly, typically when an NPC entity is spawned or its AI is initialized. The critical initialization step is the explicit call to the `initialise` method, which allocates and configures the internal message slots.
-   **Scope:** The lifecycle of an EventSupport instance is tightly bound to its parent NPC entity. It persists for the entire duration that the NPC is active in the world.
-   **Destruction:** The component is eligible for garbage collection when the parent NPC entity is despawned or destroyed. There are no explicit teardown methods.

## Internal State & Concurrency

-   **State:** EventSupport is highly stateful and mutable. Its primary state is stored in the `messageSlots` array, which contains `EventMessage` objects. These objects are continuously updated by `postMessage` (activating slots) and `pollMessage` (deactivating slots). The component effectively caches the most recent and relevant event for each configured slot. The `cloneTo` method performs a deep copy of the `messageSlots` array but a shallow copy of the `messageIndices` map, a critical detail for systems that duplicate NPC state.

-   **Thread Safety:** This class is **not thread-safe**. All internal collections and state variables are accessed without any synchronization mechanisms. It is designed to be operated on exclusively by the main server thread that executes the NPC's update tick. Concurrent calls to `postMessage` and `pollMessage` from different threads will lead to race conditions and unpredictable behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| initialise(setIndices, messageRanges, count) | void | O(N) | **Required.** Configures and allocates the internal message slots. Must be called before any other method. |
| postMessage(type, notification, parent, store) | void | O(1) | Primary entry point for incoming events. Filters the event by distance and activates the appropriate slot. |
| hasMatchingMessage(messageIndex, parentPosition, range) | boolean | O(1) | Non-destructively checks if a specific message slot is active and within a given range. |
| pollMessage(messageIndex) | Ref<EntityStore> | O(1) | Consumes the message from a slot, deactivating it and returning the event initiator. |
| getMessageSlot(type, notification) | EventMessage | O(1) | Retrieves the corresponding message slot for a given event type and notification. Returns null if no mapping exists. |
| cloneTo(other) | void | O(N) | Performs a state copy to another instance. Critically, the index map is shared, not copied. |

## Integration Patterns

### Standard Usage

EventSupport is designed to be extended by a concrete class. The typical pattern involves an external system (e.g., a world event bus) pushing notifications to the NPC, and the NPC's internal AI logic polling this component for relevant events to act upon.

```java
// In an NPC's AI update tick
// Assumes 'eventComponent' is a concrete subclass of EventSupport

// Check if a "HeardNoise" event (mapped to slot 0) is nearby
if (eventComponent.hasMatchingMessage(0, myPosition, 10.0)) {
    // Consume the message to get the source of the noise
    Ref<EntityStore> noiseSource = eventComponent.pollMessage(0);
    if (noiseSource != null) {
        // Turn to face the source of the noise
        turnTowards(noiseSource);
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Usage Before Initialization:** Calling any method such as `postMessage` or `pollMessage` before `initialise` has been successfully invoked will result in a NullPointerException.
-   **Cross-Thread Access:** Do not access an instance of this component from any thread other than the one responsible for the parent NPC's AI updates. This will corrupt its internal state.
-   **State Sharing:** Using the `cloneTo` method implies that the `messageIndices` map is shared between the original and the clone. Modifying this map in one instance will affect the other, which is almost never the desired behavior. This method should be used with extreme caution.

## Data Pipeline

The flow of data through this component is designed for filtering and temporal storage. It transforms a transient world event into a persistent, queryable state for an NPC.

> Flow:
> External `EventNotification` -> `postMessage` -> Spatial & Type Filtering -> **EventSupport (activates internal `EventMessage` slot)** -> `pollMessage` by NPC AI -> `Ref<EntityStore>` (Event Initiator) -> AI Behavior Execution

