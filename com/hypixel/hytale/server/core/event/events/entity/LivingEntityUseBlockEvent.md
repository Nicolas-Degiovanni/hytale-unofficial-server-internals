---
description: Architectural reference for LivingEntityUseBlockEvent
---

# LivingEntityUseBlockEvent

**Package:** com.hypixel.hytale.server.core.event.events.entity
**Type:** Transient

## Definition
```java
// Signature
@Deprecated(forRemoval = true)
public class LivingEntityUseBlockEvent implements IEvent<String> {
```

## Architecture & Concepts
The LivingEntityUseBlockEvent is a data-only message object used within the server-side event bus system. It represents a discrete, high-level game action: a living entity has interacted with or "used" a specific type of block.

This class is a fundamental component of an event-driven architecture, designed to decouple the cause of an action (e.g., a player's input or an AI's decision) from its effects (e.g., opening a door, triggering a trap, or updating a quest objective). Systems that generate this event do not need to know which other systems will react to it.

**WARNING:** This event is explicitly marked as deprecated and scheduled for removal. Its existence implies a legacy implementation. Production code must migrate to its replacement, which is likely a more generic or extensible interaction event. Continued use of this class will result in broken functionality in future engine updates.

### Lifecycle & Ownership
- **Creation:** Instantiated by a high-level game logic system when an entity-block interaction is confirmed. For example, the server-side player controller would create this event after validating a "use" request from a client.
- **Scope:** Extremely short-lived. The object's scope is confined to the single game tick in which it is created and dispatched through the event bus.
- **Destruction:** The event object has no owner after being processed by all subscribed listeners. It becomes eligible for garbage collection at the end of the event dispatch cycle. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Immutable. The internal fields for the entity reference and block type are private and set only once during construction. This is a critical design feature for event objects, ensuring that state is consistent and predictable as it passes through multiple, potentially concurrent, listeners.
- **Thread Safety:** This class is thread-safe by immutability. An instance can be safely read by multiple listeners on different threads without requiring locks or other synchronization primitives. However, the systems that *consume* this event are responsible for their own thread safety when mutating shared game state in response to it.

## API Surface
The public contract is minimal, providing read-only access to the event's payload.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBlockType() | String | O(1) | Returns the identifier for the block that was used. |
| getRef() | Ref<EntityStore> | O(1) | Returns a managed reference to the entity that performed the action. |

## Integration Patterns

### Standard Usage
This event is not meant to be used directly. The following example is for historical and illustrative purposes only, demonstrating how a system would have posted it to an event bus.

```java
// A hypothetical system detecting a block interaction
void onPlayerInteracts(Player player, Block targetBlock) {
    // WARNING: Do not use this deprecated event.
    Ref<EntityStore> entityRef = player.getEntityStoreRef();
    String blockId = targetBlock.getTypeId();

    LivingEntityUseBlockEvent event = new LivingEntityUseBlockEvent(entityRef, blockId);
    
    // The event is fired and forgotten.
    serverContext.getEventBus().post(event);
}
```

### Anti-Patterns (Do NOT do this)
- **Active Usage:** Do not write new code that references, creates, or listens for this event. It is deprecated and will be removed. Find the modern equivalent, such as a generic EntityInteractEvent.
- **State Modification:** Do not attempt to use reflection or other means to modify the event's state after creation. Event objects are designed to be immutable records of a past occurrence.
- **Listener-Side Instantiation:** Systems that listen for events should never create and post new events of the same type. This can lead to infinite loops and stack overflows.

## Data Pipeline
The flow for this event follows a standard fire-and-forget pattern within the engine's event bus.

> Flow:
> Game Logic (Player Input, AI Behavior) -> **LivingEntityUseBlockEvent** (Instantiation) -> EventBus Dispatch -> Subscribed Systems (Quest Trackers, Block Logic Handlers)

