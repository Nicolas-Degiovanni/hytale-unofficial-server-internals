---
description: Architectural reference for UseBlockEvent
---

# UseBlockEvent

**Package:** com.hypixel.hytale.server.core.event.events.ecs
**Type:** Transient Data Model

## Definition
```java
// Signature
public abstract class UseBlockEvent extends EcsEvent {

   // Concrete event fired before the action is processed
   public static final class Pre extends UseBlockEvent implements ICancellableEcsEvent {
      // ...
   }

   // Concrete event fired after the action is processed
   public static final class Post extends UseBlockEvent {
      // ...
   }
}
```

## Architecture & Concepts
UseBlockEvent is a foundational event within the server's Entity Component System (ECS) that signals an entity's interaction with a block in the game world. It serves as a data-rich message payload, encapsulating the context of the interaction: who acted, what they did, where it happened, and what block was affected.

This class embodies a critical **Pre/Post** event pattern, which provides robust hooks for modifying and observing game behavior.

-   **UseBlockEvent.Pre:** This variant is dispatched *before* the server executes the logic for the block interaction. Its primary purpose is to allow other systems to intercept and potentially cancel the action. This is the main extension point for implementing custom game rules, permissions, or special tool behaviors. If this event is cancelled, the corresponding world change and the Post event will never occur.

-   **UseBlockEvent.Post:** This variant is dispatched *after* the block interaction has been successfully validated, processed, and the world state has been modified. It serves as a notification mechanism for systems that need to react to completed actions, such as logging, statistics tracking, or triggering achievements.

These events are exclusively managed and fired by the server's core input processing loop and are fundamental to the server's interactive gameplay logic.

## Lifecycle & Ownership
-   **Creation:** An instance of UseBlockEvent is created by the server's network packet handler when a player input related to block interaction is received. A **Pre** event is instantiated first and dispatched to the event bus. If it is not cancelled by any listener, the server performs the action and then instantiates a **Post** event for subsequent dispatch.
-   **Scope:** The lifecycle of a UseBlockEvent instance is extremely brief, confined to the single game tick in which it is created and processed. It is a transient object by design.
-   **Destruction:** The object becomes eligible for garbage collection as soon as the event bus has finished dispatching it to all registered listeners. No long-term references should ever be held to these event objects.

## Internal State & Concurrency
-   **State:**
    -   The base UseBlockEvent and its **Post** subclass are **immutable**. All their fields are final and are set at construction time.
    -   The **Pre** subclass is **mutable** in one specific way: its cancellation status. The `cancelled` boolean flag can be modified by event listeners.

-   **Thread Safety:**
    -   UseBlockEvent instances are **not thread-safe**. They are designed to be created, dispatched, and handled exclusively on the main server game thread.
    -   **WARNING:** Accessing or modifying an event from an asynchronous task or a different thread is a critical error. All event handling logic, including cancellation, must be performed synchronously within the listener's callback to prevent race conditions and ensure a consistent game state.

## API Surface
The public contract is focused on data access and, for the Pre event, cancellation control.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getInteractionType() | InteractionType | O(1) | Returns the type of interaction, e.g., primary or secondary click. |
| getContext() | InteractionContext | O(1) | Provides context about the interacting entity. |
| getTargetBlock() | Vector3i | O(1) | Returns the world coordinates of the block being interacted with. |
| getBlockType() | BlockType | O(1) | Returns the type definition of the target block. |
| isCancelled() | boolean | O(1) | (Pre only) Checks if the event has been cancelled. |
| setCancelled(boolean) | void | O(1) | (Pre only) Sets the cancellation state of the event. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to create a system that listens for one of the concrete subclasses on the ECS event bus.

A system for managing protected zones would listen for the **Pre** event to prevent modifications.
```java
// Example of a listener cancelling an event
@Subscribe
public void onBlockUse(UseBlockEvent.Pre event) {
    Player player = event.getContext().getPlayer();
    Vector3i location = event.getTargetBlock();

    if (ProtectionManager.isRegionProtected(location, player)) {
        event.setCancelled(true);
    }
}
```

A system for tracking statistics would listen for the **Post** event to log successful actions.
```java
// Example of a listener reacting to a completed event
@Subscribe
public void afterBlockUse(UseBlockEvent.Post event) {
    if (event.getInteractionType() == InteractionType.PRIMARY_ATTACK) {
        Player player = event.getContext().getPlayer();
        StatsTracker.incrementBlocksBroken(player.getUUID(), event.getBlockType());
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Manual Instantiation:** Do not create instances of UseBlockEvent using `new`. These events are authoritative and must only be generated by the server's core input loop. Manually firing this event will not correctly simulate player interaction and will bypass network validation logic.
-   **Holding References:** Do not store a reference to an event object in a field or collection that persists beyond the scope of the event handler method. The event's data represents a point-in-time snapshot and will be stale in subsequent ticks. This is a common source of memory leaks.
-   **Asynchronous Cancellation:** Never call `setCancelled(true)` from a separate thread or a delayed task. The event bus processes listeners synchronously, and the cancellation state is checked immediately after the dispatch is complete. Asynchronous modification will have no effect and constitutes a race condition.

## Data Pipeline
The flow of data and control for a block interaction is strictly ordered and mediated by the event system.

> Flow:
> Player Input Packet -> Server Network Handler -> **UseBlockEvent.Pre** -> Event Bus Dispatch -> System Listeners (e.g., Permissions) -> **[Action Cancelled?]**
>
> **If Not Cancelled:**
> World State Mutation (e.g., Block is broken) -> **UseBlockEvent.Post** -> Event Bus Dispatch -> System Listeners (e.g., Logging, Achievements)

