---
description: Architectural reference for PlayerRefEvent
---

# PlayerRefEvent

**Package:** com.hypixel.hytale.server.core.event.events.player
**Type:** Base Class

## Definition
```java
// Signature
public abstract class PlayerRefEvent<KeyType> implements IEvent<KeyType> {
```

## Architecture & Concepts
The PlayerRefEvent is an abstract base class that serves as the foundation for all server-side events directly associated with a specific player. It is a critical component of the server's event-driven architecture, establishing a standardized contract for any game event that originates from or targets a player entity.

The primary design principle embodied by this class is the use of a **PlayerRef** instead of a full Player entity object. A PlayerRef is a lightweight, stable handle or identifier for a player. This design intentionally decouples the event payload from the player's complete, and potentially volatile, state.

This approach provides several key benefits:
1.  **Performance:** Passing a lightweight reference through the event bus is significantly more efficient than passing a heavy Player object, which may contain extensive state data like inventory, stats, and position.
2.  **State Consistency:** It prevents events from carrying stale data. Listeners receiving the event are expected to use the PlayerRef to look up the *current* state of the Player from a central authority (e.g., a World or PlayerManager), ensuring all logic operates on the most up-to-date information.
3.  **Decoupling:** Systems that only need to know *which* player an event pertains to do not need a dependency on the full Player class definition.

This class cannot be instantiated directly. Developers must extend it to create concrete, meaningful events such as PlayerJoinEvent or PlayerDamageEvent.

### Lifecycle & Ownership
-   **Creation:** Never created directly. Concrete subclasses are instantiated by various server systems when a player-specific event occurs. For example, the connection handling system would create a PlayerConnectedEvent.
-   **Scope:** Ephemeral and short-lived. An instance of a PlayerRefEvent subclass exists only for the duration of its dispatch through the event bus. It is a transient data transfer object.
-   **Destruction:** The object is eligible for garbage collection immediately after the event bus has finished notifying all registered listeners. No manual cleanup is required.

## Internal State & Concurrency
-   **State:** Immutable. The internal PlayerRef field is declared as final and is initialized only once in the constructor. This is a crucial design choice for event objects, as it guarantees that the event's context cannot be mutated while it propagates through the system.
-   **Thread Safety:** Inherently thread-safe due to its immutability. An instance can be safely read by multiple listeners across different threads without synchronization. However, the responsibility for dispatching events on the correct thread (e.g., the main server tick thread) lies with the event bus implementation, not the event object itself.

## API Surface
The public contract is minimal, focusing exclusively on providing the player context.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPlayerRef() | PlayerRef | O(1) | Returns the non-null, stable reference to the player associated with this event. |

## Integration Patterns

### Standard Usage
The primary pattern is to extend this class to create a new, specific event type. Systems then post instances of the concrete class to the event bus.

**1. Defining a Concrete Event:**
```java
// Create a specific event that inherits the player context
public class PlayerSentChatEvent extends PlayerRefEvent<PlayerRef> {
    private final String message;

    public PlayerSentChatEvent(PlayerRef playerRef, String message) {
        super(playerRef);
        this.message = message;
    }

    public String getMessage() {
        return this.message;
    }
}
```

**2. Listening for the Event:**
```java
// A listener would then consume the event
@Subscribe
public void onPlayerChat(PlayerSentChatEvent event) {
    PlayerRef ref = event.getPlayerRef();
    // Use the ref to look up the player and perform actions
    Player player = world.getPlayerByRef(ref);
    if (player != null) {
        log.info(player.getName() + " said: " + event.getMessage());
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** It is a compile-time error to attempt `new PlayerRefEvent()`. This class exists only to be extended.
-   **Storing the Event Object:** Listeners should not store references to event objects. They are transient and should be processed and discarded within the scope of the listener method. Holding onto them can prevent garbage collection and lead to memory leaks.
-   **Caching the PlayerRef:** While safer than caching a full Player object, listeners should avoid holding onto the PlayerRef for extended periods. The player may disconnect, rendering the reference invalid for subsequent lookups. Always resolve the PlayerRef to a Player object at the time of processing.

## Data Pipeline
The flow of data for any event extending PlayerRefEvent follows a clear, unidirectional path through the server's event bus.

> Flow:
> Game Logic (e.g., Chat System) -> Instantiates `PlayerSentChatEvent` -> `EventBus.post(event)` -> **PlayerRefEvent** (as payload) -> EventBus dispatches to Listeners -> Listener calls `event.getPlayerRef()` -> Logic Execution

