---
description: Architectural reference for AllWorldsLoadedEvent
---

# AllWorldsLoadedEvent

**Package:** com.hypixel.hytale.server.core.universe.world.events
**Type:** Transient Data Object

## Definition
```java
// Signature
public class AllWorldsLoadedEvent implements IEvent<Void> {
```

## Architecture & Concepts
The AllWorldsLoadedEvent is a critical **signal event** within the server's startup and initialization sequence. It serves as a system-wide notification that the server's world-loading process has completed successfully and all configured worlds are now active and ready for interaction.

As an implementation of IEvent with a Void generic type, this event carries no data payload. Its sole purpose is to act as a marker for a specific state transition: the moment the server universe becomes fully operational. It is a cornerstone of the engine's event-driven architecture, decoupling the WorldManager from numerous other systems that depend on a fully initialized game state. Systems such as player connection handlers, AI schedulers, and quest managers subscribe to this event to trigger their own startup logic.

## Lifecycle & Ownership
- **Creation:** Instantiated and fired exactly once by the central WorldManager or an equivalent server bootstrap coordinator. This occurs immediately after the final world in the load queue reports a successful initialization.
- **Scope:** Extremely short-lived. The object exists only for the duration of its dispatch through the event bus. Once all subscribers have processed the event, it becomes eligible for garbage collection.
- **Destruction:** Managed automatically by the Java Garbage Collector. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no fields and its identity is its only meaningful attribute.
- **Thread Safety:** Inherently thread-safe. An instance of AllWorldsLoadedEvent can be safely passed across any thread boundary. Concurrency concerns are managed by the EventBus implementation, not the event object itself.

## API Surface
The primary contract of this class is its type, which is used for event subscription. It has no meaningful methods to be called by a consumer.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| (Class Type) | AllWorldsLoadedEvent | N/A | The type itself is the API, used by the EventBus for routing to subscribers. |

## Integration Patterns

### Standard Usage
Systems should not create this event. Instead, they subscribe to it to perform initialization logic that depends on the game worlds being fully available.

```java
// Example: A service that begins accepting player connections only after worlds are loaded.
public class PlayerConnectionService {

    @Subscribe
    public void onAllWorldsLoaded(AllWorldsLoadedEvent event) {
        // The server is now in a state to accept incoming players.
        this.startAcceptingConnections();
        log.info("All worlds loaded. Player connections are now enabled.");
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Premature Firing:** Firing this event manually or from a component other than the primary WorldManager can lead to a catastrophic race condition, causing systems to interact with partially initialized or non-existent worlds.
- **Assuming World Population:** This event signals that world *structures* are loaded. It does **not** guarantee that all initial chunks are generated or that all persistent entities have been spawned. Subscribing systems must be prepared to handle a structurally sound but potentially empty world.

## Data Pipeline
This component does not process a data stream; it represents a control flow signal that unblocks other processes.

> Flow:
> WorldManager Load Loop -> Final World Initialized -> **AllWorldsLoadedEvent Instantiated** -> EventBus Dispatch -> Subscribers (e.g., NetworkManager, AIScheduler) Activated

