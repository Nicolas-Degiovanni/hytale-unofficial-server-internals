---
description: Architectural reference for ChangeGameModeEvent
---

# ChangeGameModeEvent

**Package:** com.hypixel.hytale.server.core.event.events.ecs
**Type:** Transient

## Definition
```java
// Signature
public class ChangeGameModeEvent extends CancellableEcsEvent {
```

## Architecture & Concepts
The ChangeGameModeEvent is a server-side Data Transfer Object that represents a request to change a player entity's game mode. It is a fundamental component of the server's Entity Component System (ECS) event bus, acting as a message that decouples the initiator of the change from the systems that process and validate it.

By extending CancellableEcsEvent, this class participates in an interceptable workflow. Systems can subscribe to this event to implement custom logic, such as permission checks, logging, or gameplay rule enforcement. A listener can prevent the game mode change entirely by cancelling the event, or even modify the target game mode before it is applied. This design provides a powerful and centralized point of control for a critical game mechanic.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by high-level game logic when a game mode change is requested. Common sources include player-executed commands, script invocations, or automated game-world triggers. It is never created directly by the network layer.
- **Scope:** Extremely short-lived. An instance of this event exists only for the duration of its dispatch through the ECS event bus, which occurs synchronously within a single server tick.
- **Destruction:** The object has no explicit destruction logic. Once all registered listeners have processed the event, it falls out of scope and is reclaimed by the Java garbage collector. **WARNING:** Holding a reference to this event object beyond the scope of the event handler is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** The internal state is **Mutable**. The target GameMode can be altered in-flight by any event listener via the setGameMode method. This is a deliberate design choice to allow systems to modify the outcome of a request without needing to cancel it and fire a new one. For example, a permissions system could downgrade a requested change to Creative mode into Spectator mode.
- **Thread Safety:** This class is **Not Thread-Safe**. It is designed exclusively for use within the main server game loop thread. All creation, dispatch, and modification must occur on this thread to prevent race conditions and ensure deterministic behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getGameMode() | GameMode | O(1) | Retrieves the target game mode that will be applied if the event is not cancelled. |
| setGameMode(GameMode) | void | O(1) | Overwrites the target game mode. This allows a listener to modify the final outcome of the event. |

*Note: This event inherits cancellation methods like setCancelled(boolean) from its parent, CancellableEcsEvent, which are critical for its function.*

## Integration Patterns

### Standard Usage
The canonical use case is to create an ECS System that listens for this event to enforce game rules. The listener inspects the event's payload and cancels it if the request is invalid.

```java
// Example of a hypothetical PermissionSystem listening for the event
public class PermissionSystem extends EcsSystem {

    @Subscribe
    public void onChangeGameMode(ChangeGameModeEvent event, EntityId entityId) {
        Player player = world.getPlayer(entityId);
        GameMode requestedMode = event.getGameMode();

        // Prevent players without 'admin' permission from entering Creative mode
        if (requestedMode == GameMode.CREATIVE && !player.hasPermission("hytale.admin")) {
            player.sendMessage("You do not have permission to enter Creative mode.");
            event.setCancelled(true); // Veto the change
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Asynchronous Handling:** Do not dispatch this event to another thread for processing. The state of the world and the player entity can change between ticks, making asynchronous validation unreliable and dangerous. All logic must be executed synchronously within the event handler callback.
- **Holding References:** Do not store a reference to the event object in a field or collection that outlives the handler method. This prevents garbage collection and can lead to unpredictable behavior if the stale event is later modified.
- **Ignoring Cancellation:** When dispatching this event, the caller **must** check if the event was cancelled by a listener before applying the game mode change. Failure to do so bypasses all validation logic.

## Data Pipeline
The event acts as a message payload that flows from a high-level request through the core event bus to the systems that ultimately apply the state change to a player entity.

> Flow:
> Player Command -> Command Dispatcher -> **ChangeGameModeEvent** (Instantiation) -> ECS Event Bus -> Registered Listeners (e.g., PermissionSystem, LoggingSystem) -> Command Dispatcher (Checks for cancellation) -> PlayerManager (Applies final GameMode to Entity)

