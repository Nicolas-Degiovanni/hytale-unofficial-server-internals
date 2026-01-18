---
description: Architectural reference for PlayerConnectEvent
---

# PlayerConnectEvent

**Package:** com.hypixel.hytale.server.core.event.events.player
**Type:** Transient

## Definition
```java
// Signature
public class PlayerConnectEvent implements IEvent<Void> {
```

## Architecture & Concepts
The PlayerConnectEvent is a foundational event in the server's player lifecycle management system. It signifies the critical transition point where a player has successfully authenticated with the server but has not yet been spawned into a world.

Architecturally, this class serves two primary purposes:
1.  **Notification:** It broadcasts the fact that a new player is ready to join the game universe, allowing various systems (e.g., logging, analytics, social notifications) to react.
2.  **Mutable Context:** It acts as a mutable data carrier that allows event listeners to intercept and modify the player's destination. The ability to change the target World via the setWorld method is the class's most significant feature, enabling dynamic server balancing, lobby redirection, and instance management.

The implementation of IEvent with a Void generic type indicates that handlers are not expected to return a value. Instead, they influence the outcome of the connection process by modifying the state of the event object itself. The presence of a deprecated getPlayer method suggests a deliberate architectural shift away from direct entity manipulation within event handlers, favoring the more abstract and stable PlayerRef for identification.

## Lifecycle & Ownership
-   **Creation:** Instantiated by the server's core session management layer immediately after a client's network handshake and authentication sequence is successfully completed.
-   **Scope:** Ephemeral. The object's lifespan is strictly confined to the duration of its dispatch on the server's main event bus. It is created, passed to all relevant listeners, and then becomes eligible for garbage collection.
-   **Destruction:** The object is not managed or explicitly destroyed. It is garbage collected once all references are dropped after the event dispatch cycle concludes. **WARNING:** Systems must not retain references to this event object after their handling method completes.

## Internal State & Concurrency
-   **State:** This object is a mutable container. While the initial Holder and PlayerRef are final, the World field is explicitly designed to be modified by downstream consumers. This mutability is central to its role in the connection pipeline.
-   **Thread Safety:** **Not thread-safe.** This event is designed to be processed synchronously on a single thread, typically the main server tick thread. Any attempt to read or write its state from another thread will result in severe race conditions and undefined server behavior. All modifications must occur within the call stack of the event handler.

## API Surface
The public API is minimal, focusing on providing context and allowing modification of the player's destination.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getHolder() | Holder | O(1) | Retrieves the EntityStore container for the connecting player, holding their core components. |
| getPlayerRef() | PlayerRef | O(1) | Returns a stable, lightweight reference to the player. Use this for identification. |
| getWorld() | World | O(1) | Gets the currently proposed destination world. May be null initially. |
| setWorld(World) | void | O(1) | Overrides the player's destination world. This is the primary control mechanism for handlers. |

## Integration Patterns

### Standard Usage
The primary pattern is to implement a listener that intercepts this event to apply custom connection logic, such as routing players to a specific lobby or hub world.

```java
// In a service that listens to events
public void onPlayerConnect(PlayerConnectEvent event) {
    // Check if the player is new or should be sent to the hub
    if (isNewPlayer(event.getPlayerRef()) || event.getWorld() == null) {
        World hubWorld = worldManager.getHubWorld();
        
        // WARNING: This mutation directs the subsequent spawning logic
        event.setWorld(hubWorld);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Retaining References:** Do not store the PlayerConnectEvent object in a field or list for later processing. Its state is only valid during the event dispatch.
-   **Asynchronous Modification:** Do not dispatch a task to another thread to decide the world and then call setWorld. The event dispatch will have completed long before the asynchronous task finishes, rendering the modification useless and causing a silent failure.
    ```java
    // DO NOT DO THIS
    public void onPlayerConnect(PlayerConnectEvent event) {
        // This will NOT work as intended
        CompletableFuture.runAsync(() -> {
            World complexWorld = someSlowWorldCalculation();
            event.setWorld(complexWorld); // This call happens too late
        });
    }
    ```
-   **Ignoring Nullability:** The initial world provided in the event constructor can be null. Handlers must always perform null checks on the result of getWorld before using it.

## Data Pipeline
The PlayerConnectEvent is a key link between the network session layer and the game's world management and entity spawning systems.

> Flow:
> Client Handshake -> Server Session Manager -> **PlayerConnectEvent** (Creation & Dispatch) -> Event Bus -> Custom Listeners (World Redirection Logic) -> Player Spawning Service -> Entity Added to World

