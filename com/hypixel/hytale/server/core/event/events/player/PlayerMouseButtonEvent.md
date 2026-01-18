---
description: Architectural reference for PlayerMouseButtonEvent
---

# PlayerMouseButtonEvent

**Package:** com.hypixel.hytale.server.core.event.events.player
**Type:** Transient

## Definition
```java
// Signature
public class PlayerMouseButtonEvent extends PlayerEvent<Void> implements ICancellable {
```

## Architecture & Concepts
The PlayerMouseButtonEvent is a server-side, state-carrying event object that represents a player's mouse button interaction with the game world. It is a fundamental component of the server's input processing and gameplay interaction systems.

This class acts as a data transfer object (DTO) that decouples the raw network input from the game logic that responds to it. When a client sends a mouse click packet, the server's network layer translates it into this rich event object. It is then dispatched to the central server Event Bus, allowing various systems—such as block interaction handlers, entity attack logic, or custom script listeners—to react to the player's action.

A critical feature is its implementation of the ICancellable interface. This allows any listener in the event chain to prevent the default action associated with the click from occurring. For example, a land-protection system can listen for this event, check if the player has permission to interact with the target block, and cancel the event to prevent unauthorized modification. This makes the event system a powerful interception point for gameplay logic and server administration.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively by the server's network input handler when it receives and decodes a mouse button packet from a game client. The handler populates the event with a snapshot of the relevant game state at the moment of the click.
- **Scope:** Extremely short-lived. An instance of this event exists only for the duration of its dispatch through the event bus, which typically occurs within a single server tick.
- **Destruction:** The object is not managed explicitly. Once all registered listeners have processed the event, it falls out of scope and becomes eligible for garbage collection. There are no persistent references to it.

## Internal State & Concurrency
- **State:** The object is primarily a state container. Its core data fields (targetBlock, targetEntity, itemInHand, etc.) are effectively immutable, as they are set once in the constructor. The only mutable state is the boolean *cancelled* flag, which is designed to be modified by event listeners.
- **Thread Safety:** **This class is not thread-safe and must only be accessed from the main server thread.** The server's event system operates on a single-threaded model where events are processed sequentially during the game tick. Attempting to access or modify this event from an asynchronous task or a different thread will lead to severe concurrency issues, including race conditions and inconsistent state reads.

## API Surface
The public API consists almost entirely of accessors for the event's state, plus the methods for cancellation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setCancelled(boolean) | void | O(1) | Sets the cancellation state of the event. If set to true, subsequent listeners and the default game logic will not process the action. |
| isCancelled() | boolean | O(1) | Returns true if the event has been cancelled by a prior listener. |
| getPlayer() | Player | O(1) | Inherited. Returns the Player entity that initiated the event. |
| getItemInHand() | Item | O(1) | Returns the Item configuration the player was holding at the time of the click. |
| getTargetBlock() | Vector3i | O(1) | Returns the world coordinates of the block the player was targeting. |
| getTargetEntity() | Entity | O(1) | Returns the Entity the player was targeting, if any. |
| getMouseButton() | MouseButtonEvent | O(1) | Returns an enum value indicating which mouse button was pressed. |

## Integration Patterns

### Standard Usage
The standard interaction pattern is to create a method in a listener class and subscribe it to the event bus. The method receives the event, performs its logic, and optionally cancels it.

```java
// In a class registered with the server's EventBus
@Subscribe
public void onPlayerClick(PlayerMouseButtonEvent event) {
    Player player = event.getPlayer();
    Vector3i blockPos = event.getTargetBlock();

    // Example: Prevent interaction in a protected zone
    if (ZoneManager.isProtected(blockPos) && !player.hasPermission("zone.bypass")) {
        player.sendMessage("You cannot interact with this protected block.");
        event.setCancelled(true); // Prevents the block from being broken or used
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new PlayerMouseButtonEvent()`. These events are meaningful only when created and dispatched by the core server engine in response to actual client input. Manually created events will not be processed by the game and will have no effect.
- **Asynchronous Handling:** Do not pass the event object to another thread for processing. The game state it references (like the Player or target Entity) can be modified or removed on subsequent ticks, leading to unpredictable behavior or crashes. All logic must be executed synchronously within the event handler.
- **Storing References:** Do not store a reference to the event object in a field or list for later use. It is a transient DTO and should be considered invalid as soon as the handler method returns.

## Data Pipeline
The flow of data from client action to server-side game logic is linear and synchronous.

> Flow:
> Client Mouse Click -> Network Packet (Client to Server) -> Server Packet Handler -> **PlayerMouseButtonEvent Instantiation** -> Server Event Bus Dispatch -> Registered Event Listeners -> Default Game Action (if not cancelled)

