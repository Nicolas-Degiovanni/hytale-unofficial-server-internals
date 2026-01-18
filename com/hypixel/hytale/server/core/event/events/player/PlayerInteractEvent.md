---
description: Architectural reference for PlayerInteractEvent
---

# PlayerInteractEvent

**Package:** com.hypixel.hytale.server.core.event.events.player
**Type:** Transient

## Definition
```java
// Signature
@Deprecated
public class PlayerInteractEvent extends PlayerEvent<String> implements ICancellable {
```

## Architecture & Concepts
The PlayerInteractEvent is a data-transfer object that represents a discrete player action within the server's event-driven architecture. It serves as a crucial decoupling mechanism, translating low-level network packets into a high-level, semantic event that game logic systems can subscribe and react to.

This class encapsulates all context surrounding a player's attempt to interact with the game world, such as attacking an entity, placing a block, or using an item. By implementing the ICancellable interface, it allows subscribers in the event pipeline to veto the action, preventing the default game behavior from executing.

**Warning:** This class is marked as **Deprecated**. It should not be used for new development. Its presence indicates a legacy event structure that has likely been superseded by a more granular or performant system. Systems should migrate to the recommended alternative.

## Lifecycle & Ownership
- **Creation:** An instance is created by a server-side network packet handler immediately after decoding a client's interaction packet. The handler populates the event with data directly from the network stream.
- **Scope:** The object's lifetime is extremely short. It exists only for the duration of its dispatch through the server's central EventBus.
- **Destruction:** Once all registered listeners have processed the event, it falls out of scope and becomes eligible for garbage collection. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The event's state is primarily immutable after construction, with one critical exception: the *cancelled* boolean flag. All fields like targetEntity and itemInHand are final, but the objects they reference may be mutable. The core purpose of this object's mutable state is to allow event listeners to communicate a cancellation request down the processing pipeline.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be created, dispatched, and handled within a single, synchronous game loop tick (e.g., the main server thread or a specific world thread). Accessing or modifying an instance from multiple threads will result in race conditions on the *cancelled* flag, leading to unpredictable behavior.

## API Surface
The public contract is composed of simple data accessors and the cancellation mechanism.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setCancelled(boolean) | void | O(1) | Sets the cancellation state. If set to true, subsequent listeners and default game logic should not process the action. |
| isCancelled() | boolean | O(1) | Returns the current cancellation state of the event. |
| getActionType() | InteractionType | O(1) | Returns the specific type of interaction, e.g., ATTACK, USE_ITEM. |
| getTargetEntity() | Entity | O(1) | Returns the entity being targeted, if any. May be null. |
| getTargetBlock() | Vector3i | O(1) | Returns the block position being targeted, if any. May be null. |
| getItemInHand() | ItemStack | O(1) | Returns a snapshot of the item stack the player used to perform the action. |

## Integration Patterns

### Standard Usage
The standard pattern is to create a listener method that subscribes to this event type. The listener inspects the event's properties to determine if it should act, and potentially cancels it to override default behavior.

```java
// A listener within a game logic system or plugin
@Subscribe
public void onPlayerInteract(PlayerInteractEvent event) {
    // Prevent players from breaking a specific block type
    if (event.getActionType() == InteractionType.BLOCK_BREAK) {
        Block target = world.getBlockAt(event.getTargetBlock());
        if (target.isUnbreakable()) {
            event.setCancelled(true);
            event.getPlayer().sendMessage("You cannot break this block.");
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Do not use `new PlayerInteractEvent()` to simulate player actions. This bypasses the network layer, validation, and anti-cheat systems, and will not produce the expected behavior. Player actions must originate from a client connection.
- **Ignoring Cancellation:** Listeners, especially those with a lower priority, must check `event.isCancelled()` before executing logic. Proceeding with an action on a cancelled event can cause exploits, item duplication, or server instability.
- **Asynchronous Modification:** Do not hold a reference to the event and modify it on another thread or in a later game tick. The event object is invalid after the initial dispatch cycle is complete.

## Data Pipeline
The event is a single step in the flow of data from the client's input to a change in the server's world state.

> Flow:
> Client Input Packet -> Server Network Handler -> **PlayerInteractEvent** (Instantiation) -> Event Bus (Dispatch) -> System Listeners (e.g., Land Protection, Anti-Cheat) -> Default Game Logic (e.g., Damage Calculation, Block Removal)

