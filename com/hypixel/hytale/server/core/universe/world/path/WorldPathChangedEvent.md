---
description: Architectural reference for WorldPathChangedEvent
---

# WorldPathChangedEvent

**Package:** com.hypixel.hytale.server.core.universe.world.path
**Type:** Transient

## Definition
```java
// Signature
public class WorldPathChangedEvent implements IEvent<Void> {
```

## Architecture & Concepts
The WorldPathChangedEvent is a fundamental messaging object within the server's event-driven architecture. It serves as an immutable notification that the active game world has been changed. This class is not a service or manager; it is a simple, transient data carrier whose sole purpose is to decouple the system that initiates a world change from the various systems that need to react to it.

By implementing the IEvent interface, it signals its role as a standard message to be published on the server's central event bus. Systems such as the WorldManager, AssetLoader, or AI Director subscribe to this event type to trigger their respective state updates, resource loading, or cleanup procedures without needing direct coupling to the source of the change. The generic type parameter of Void indicates that handlers of this event are not expected to return a value; it is a fire-and-forget notification.

## Lifecycle & Ownership
- **Creation:** Instantiated by a high-level server authority, typically the WorldManager or a related lifecycle service, at the exact moment a world transition is committed. It is never created by subscribers or low-level systems.
- **Scope:** Extremely short-lived. An instance of this event exists only for the duration of its dispatch through the event bus. It is created, passed to all relevant subscribers in a single operation, and then becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Virtual Machine's garbage collector. There are no native resources or explicit cleanup requirements. Holding a long-term reference to an event object is a memory leak anti-pattern.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal WorldPath field is private and assigned only once during construction. There are no setters or methods to modify its state post-creation. This design is critical for ensuring that all event subscribers receive the exact same, untampered information.
- **Thread Safety:** The object itself is inherently thread-safe due to its immutability. It can be safely passed between threads without locks. However, subscribers (listeners) are responsible for ensuring their own thread safety. Events are typically dispatched on a specific, well-defined thread (e.g., the main server tick thread), and any handler that performs long-running operations or interacts with state on other threads must manage its own concurrency.

## API Surface
The public contract is minimal, focusing exclusively on data retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| WorldPathChangedEvent(WorldPath) | constructor | O(1) | Creates a new event instance. Throws NullPointerException if the provided WorldPath is null. |
| getWorldPath() | WorldPath | O(1) | Returns the new world path that has become active. This is the primary data payload. |

## Integration Patterns

### Standard Usage
The event is consumed by a subscriber class that has registered itself with the server's event bus. The handler method extracts the payload and acts upon it.

```java
// In a service that needs to react to world changes
@Subscribe
public void onWorldPathChanged(WorldPathChangedEvent event) {
    WorldPath newPath = event.getWorldPath();
    log.info("World has changed. Unloading old assets and preparing for: " + newPath);
    // Trigger internal state change, resource loading, etc.
    this.unloadAllChunks();
    this.beginLoadingWorld(newPath);
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation by Consumers:** A listener or subscriber should **never** create an instance of this event. Its role is to react, not to command. Falsely firing this event can lead to severe server state desynchronization.
- **Storing Event References:** Do not store the WorldPathChangedEvent object in a field. Extract the WorldPath data you need from it within the handler method and discard the event object itself. Storing the event can prevent garbage collection and lead to memory leaks.
- **Payload Modification:** While the event's internal field is final, if the contained WorldPath object were mutable, modifying it within a listener would be a critical error. This would create unpredictable side effects for other listeners in the dispatch chain. Treat all event payloads as read-only.

## Data Pipeline
The flow of this event through the system is linear and unidirectional, serving as a broadcast notification.

> Flow:
> WorldManager Command -> **WorldPathChangedEvent Instantiation** -> Event Bus Publication -> Subscriber Notification (e.g., ChunkManager, AssetLoader, PlayerSessionManager) -> State Update

