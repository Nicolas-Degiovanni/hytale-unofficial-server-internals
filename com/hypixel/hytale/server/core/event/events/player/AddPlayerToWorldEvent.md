---
description: Architectural reference for AddPlayerToWorldEvent
---

# AddPlayerToWorldEvent

**Package:** com.hypixel.hytale.server.core.event.events.player
**Type:** Transient Data Transfer Object

## Definition
```java
// Signature
public class AddPlayerToWorldEvent implements IEvent<String> {
```

## Architecture & Concepts
The AddPlayerToWorldEvent is a data-carrying class that represents a discrete, transactional request to introduce a player's entity into a specific game world. It functions as a command object within the server's event-driven architecture, decoupling the network-level player connection logic from the game simulation and world management systems.

When a player successfully completes the connection and authentication sequence, a higher-level manager (such as a PlayerConnectionHandler) instantiates this event. It is then dispatched onto the central server Event Bus. Systems concerned with world state, such as the WorldManager or EntityLifecycleManager, subscribe to this event type. Upon receiving it, these listeners perform the necessary operations to add the player's EntityStore representation to the target World's simulation tick, making the player visible and interactive.

The implementation of IEvent suggests that event handlers can potentially return a result, although this specific class is primarily used for one-way notification.

## Lifecycle & Ownership
- **Creation:** Instantiated by a network or session management component immediately after a player is authenticated and ready to enter the game simulation.
- **Scope:** Extremely short-lived. The object's lifetime is confined to a single event dispatch cycle. It exists only for the time it takes for the Event Bus to pass it to all relevant subscribers.
- **Destruction:** The event object holds no persistent resources and is not managed by any container. Once all listeners have processed it, it falls out of scope and is reclaimed by the Java garbage collector.

## Internal State & Concurrency
- **State:** The object's state is **Mutable**. While the core references to the player's Holder and the target World are final, the boolean flag *broadcastJoinMessage* can be modified post-construction. This allows intermediary event listeners (e.g., a plugin system) to intercept the event and alter its subsequent behavior, such as suppressing the global join message under certain conditions.

- **Thread Safety:** This class is **Not Thread-Safe**. It is designed to be created and processed within a single, synchronized context, typically the main server game loop or "tick" thread. The server's Event Bus is responsible for guaranteeing that listeners are invoked sequentially, preventing concurrent modification issues. Accessing or modifying this object from multiple threads will result in unpredictable behavior.

## API Surface
The public API is designed for data retrieval and minor state modification by event listeners.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| AddPlayerToWorldEvent(Holder, World) | constructor | O(1) | Creates a new event to add the specified player entity to the target world. |
| getHolder() | Holder | O(1) | Returns the handle to the player's core entity data. |
| getWorld() | World | O(1) | Returns the target world instance for the player to be added to. |
| setBroadcastJoinMessage(boolean) | void | O(1) | Modifies the event to control whether a server-wide join message is broadcast. |

## Integration Patterns

### Standard Usage
The event should be created and immediately dispatched to the server's event bus. Listeners will then act upon the data contained within the event.

```java
// In a PlayerConnectionManager or similar system...
Holder<EntityStore> playerEntityHolder = ...;
World targetWorld = worldProvider.getDefaultWorld();

// Create and dispatch the event. Do not hold a reference to it.
AddPlayerToWorldEvent event = new AddPlayerToWorldEvent(playerEntityHolder, targetWorld);
server.getEventBus().fire(event);
```

### Anti-Patterns (Do NOT do this)
- **Event Re-use:** Do not cache and re-fire an instance of this event. Each request to add a player to a world must be represented by a new event object.
- **Asynchronous Modification:** Do not modify the event's state from another thread after it has been fired. For example, calling setBroadcastJoinMessage from a network thread while the main game thread is processing the event is a severe race condition.
- **Direct System Invocation:** Do not bypass the event bus. The purpose of this object is to decouple systems. Manually passing an event instance to a manager circumvents this design.
    ```java
    // BAD: Bypasses the event bus and creates tight coupling
    AddPlayerToWorldEvent event = new AddPlayerToWorldEvent(holder, world);
    worldManager.manuallyHandlePlayerAdd(event); 
    ```

## Data Pipeline
The AddPlayerToWorldEvent is a critical link in the data flow that transitions a player from a simple network connection to a fully simulated entity within the game.

> Flow:
> Network Connection Established -> Authentication Success -> **AddPlayerToWorldEvent (Created & Fired)** -> Server Event Bus -> WorldManager Listener -> Player Entity Injected into World Simulation -> Client Receives World Data

