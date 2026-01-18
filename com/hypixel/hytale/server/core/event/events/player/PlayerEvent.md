---
description: Architectural reference for PlayerEvent
---

# PlayerEvent

**Package:** com.hypixel.hytale.server.core.event.events.player
**Type:** Transient Base Class

## Definition
```java
// Signature
public abstract class PlayerEvent<KeyType> implements IEvent<KeyType> {
```

## Architecture & Concepts
The PlayerEvent class is an abstract base class that serves as the foundational template for all server-side events directly associated with a specific Player entity. It is a critical component of the server's event-driven architecture, ensuring a standardized data contract for any system that needs to publish or subscribe to player-related occurrences.

Its primary architectural function is to bundle a reference to the Player entity (`Player`) and its corresponding data store handle (`Ref<EntityStore>`) into a single, immutable payload. This design decouples event producers from consumers. A system, such as the chat handler, can fire a `PlayerChatEvent` without needing to know which other systems, like logging or command processing, will consume it. The consumers, in turn, receive all necessary context to act upon the player without performing expensive lookups.

This class is not a service or manager; it is a pure Data Transfer Object (DTO) that flows through the central Event Bus.

## Lifecycle & Ownership
- **Creation:** Concrete subclasses of PlayerEvent are instantiated by various server-side systems in direct response to a player action or a change in their state. For example, the connection manager would instantiate a `PlayerJoinEvent` when a player successfully connects.
- **Scope:** Extremely short-lived and ephemeral. An instance of a PlayerEvent exists only for the duration of its dispatch cycle within the event bus. It is created, passed to all relevant listeners, and then becomes eligible for garbage collection.
- **Destruction:** Managed automatically by the Java Garbage Collector. Once the event bus has finished dispatching the event to all subscribers, no strong references to the event object should remain.

**WARNING:** Systems acting as event listeners must not retain references to PlayerEvent objects after their handling method completes. Caching or storing these transient objects can lead to memory leaks and unpredictable behavior with stale data.

## Internal State & Concurrency
- **State:** Immutable. The internal fields `playerRef` and `player` are marked as final and are assigned only once during construction. The object's state cannot be changed after it is created.
- **Thread Safety:** The PlayerEvent object itself is inherently thread-safe due to its immutability. However, the objects it provides access to, specifically the `Player` entity, are **not** guaranteed to be thread-safe. All modifications to the `Player` object or its components must be synchronized with the main server tick thread.

**WARNING:** Event handlers executing on asynchronous or worker threads must not directly mutate the state of the `Player` object returned by `getPlayer()`. State modifications must be scheduled to run on the main game thread to prevent race conditions and data corruption.

## API Surface
The public contract is minimal, providing read-only access to the core player context.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getPlayerRef() | Ref<EntityStore> | O(1) | Returns a stable reference to the player's underlying entity data store. |
| getPlayer() | Player | O(1) | Returns the direct reference to the Player entity instance associated with this event. |

## Integration Patterns

### Standard Usage
The intended pattern is to extend this abstract class to create specific, concrete event types. Developers do not interact with PlayerEvent directly but rather with its subclasses through the event bus subscription model.

```java
// 1. Define a concrete event
public class PlayerSentChatEvent extends PlayerEvent<Void> {
    private final String message;

    public PlayerSentChatEvent(Ref<EntityStore> ref, Player p, String msg) {
        super(ref, p);
        this.message = msg;
    }

    public String getMessage() {
        return this.message;
    }
}

// 2. Listen for the concrete event
@Subscribe
public void onPlayerChat(PlayerSentChatEvent event) {
    Player player = event.getPlayer();
    String message = event.getMessage();
    // Process the chat message for the specific player
    System.out.println(player.getName() + " said: " + message);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** You cannot instantiate PlayerEvent directly as it is an abstract class. Attempting `new PlayerEvent(...)` will result in a compilation error.
- **Stateful Listeners:** Do not design an event listener that stores the PlayerEvent object in a field for later use. The event and the player state it captured may be stale by the next server tick.
- **Cross-Thread Mutation:** Never call setters or modify fields on the `Player` object from an asynchronous event handler. This is the most common source of concurrency bugs in the event system.

## Data Pipeline
PlayerEvent is a data payload that moves through the server's core systems. Its flow is unidirectional and follows a clear path from producer to consumer.

> Flow:
> Player Action (e.g., movement, chat) -> Game System (Producer) -> Instantiates **Concrete PlayerEvent** -> Event Bus Dispatch -> Game Systems (Consumers) -> State Change / Network Response

