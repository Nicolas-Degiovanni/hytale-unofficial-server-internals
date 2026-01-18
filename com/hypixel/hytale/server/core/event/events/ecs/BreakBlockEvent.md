---
description: Architectural reference for BreakBlockEvent
---

# BreakBlockEvent

**Package:** com.hypixel.hytale.server.core.event.events.ecs
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class BreakBlockEvent extends CancellableEcsEvent {
```

## Architecture & Concepts
The BreakBlockEvent is a high-level, semantic event that represents the *intent* of an entity to break a block within the game world. It is a fundamental component of the server's Entity Component System (ECS) event bus, acting as a message that decouples the cause of a block break (e.g., player input, TNT explosion) from its consequences (e.g., world modification, item drops, sound effects).

By extending CancellableEcsEvent, this class provides a critical interception point for other game systems. Systems responsible for permissions, protected zones, custom tool logic, or quest objectives can listen for this event and prevent the action from completing by cancelling it. This pattern promotes a highly modular and extensible game logic architecture, where the core world simulation does not need explicit knowledge of myriad special-case rules.

This event serves as a data carrier, encapsulating all necessary context about the attempted action: the item used, the target block's position, and the block's type.

### Lifecycle & Ownership
- **Creation:** An instance of BreakBlockEvent is created dynamically by a high-level game logic system when a block-breaking action is initiated. This is typically done by the server-side player interaction handler in response to a client network packet.
- **Scope:** The event's lifecycle is extremely short, confined to the duration of a single game tick's event processing phase. It is created, dispatched synchronously to all registered listeners, and then immediately becomes eligible for garbage collection.
- **Destruction:** The object is managed by the Java Garbage Collector. There is no explicit cleanup. Holding a reference to this event beyond the scope of the originating game tick is a severe anti-pattern and will lead to undefined behavior.

## Internal State & Concurrency
- **State:** The event's state is **Mutable**. While most fields are set at construction, the target block's position can be altered post-creation via the setTargetBlock method. This allows intercepting systems to redirect the break action to a different location, a powerful but potentially hazardous capability.
- **Thread Safety:** This class is **NOT THREAD-SAFE**. All creation, modification, and consumption of this event must occur on the main server thread. The server's event bus guarantees this synchronous, single-threaded dispatch. Any attempt to access or modify an event instance from an asynchronous task or worker thread will result in race conditions and world state corruption.

## API Surface
The public contract includes its own methods and key methods inherited from CancellableEcsEvent.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getItemInHand() | ItemStack | O(1) | Returns the item used in the break attempt. May be null. |
| getTargetBlock() | Vector3i | O(1) | Returns the coordinate of the block targeted for destruction. |
| getBlockType() | BlockType | O(1) | Returns the type definition of the targeted block. |
| setTargetBlock(pos) | void | O(1) | Modifies the target block's position. Use with extreme caution. |
| isCancelled() | boolean | O(1) | (Inherited) Checks if the event has been cancelled by a listener. |
| setCancelled(cancel) | void | O(1) | (Inherited) Cancels the event, preventing the block break. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to create a system that listens for the event and reacts to it. You should never create and dispatch this event unless your system is the authoritative source of a game action (e.g., a custom machine that breaks blocks).

```java
// Example of a "ProtectedZoneSystem" listening for the event
@Handler
public void onBlockBreak(BreakBlockEvent event) {
    Player player = event.getSourceEntity(Player.class);
    Vector3i blockPos = event.getTargetBlock();

    if (isPositionInProtectedZone(blockPos) && !player.hasPermission("zone.bypass")) {
        // Prevent the block from being broken
        event.setCancelled(true);
        player.sendMessage("You cannot break blocks here.");
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Asynchronous Processing:** Do not store a reference to the event to be processed later or on another thread. The event object is invalid outside the synchronous call stack of the event dispatch.
- **Unconditional State Modification:** Do not call setTargetBlock without a clear and contained game logic reason. Unpredictably redirecting break events can break other systems' assumptions and cause difficult-to-trace bugs.
- **Ignoring Cancellation:** If your system is the one that ultimately acts on the event (i.e., modifies the world), you MUST check `isCancelled()` before proceeding. Failure to do so violates the core contract of the event system.

## Data Pipeline
The flow of data and control for a typical player-initiated block break demonstrates where this event fits into the server architecture.

> Flow:
> Player Input -> Network Packet -> Server World Interaction System -> **BreakBlockEvent Instantiation** -> ECS Event Bus -> Listeners (Permissions, Logging, etc.) -> World System (Checks for cancellation) -> World State Change -> Network State Sync to Clients

