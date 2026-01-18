---
description: Architectural reference for ChunkEvent
---

# ChunkEvent

**Package:** com.hypixel.hytale.server.core.universe.world.events
**Type:** Transient Event Base

## Definition
```java
// Signature
public abstract class ChunkEvent implements IEvent<String> {
```

## Architecture & Concepts
The ChunkEvent class is an abstract base for all server-side events related to the state changes of a WorldChunk. It is a foundational component of the server's world management and event-driven architecture. It is not intended to be instantiated directly; rather, concrete subclasses such as ChunkLoadEvent or ChunkSaveEvent are used to signal specific lifecycle transitions.

This class acts as a standardized message payload. Its primary responsibility is to carry a reference to the specific WorldChunk that is the subject of an event. By conforming to the IEvent interface, instances of ChunkEvent subclasses can be dispatched through the central server event bus, allowing disparate systems to react to changes in the world state in a decoupled manner. For example, the networking system might listen for these events to send chunk data to clients, while a persistence system might listen to save chunk data to disk.

## Lifecycle & Ownership
-   **Creation:** Concrete subclasses of ChunkEvent are instantiated by core world management systems, such as the ChunkLoadingService or the WorldGenerator. Creation is a direct response to a significant change in a chunk's state, often triggered by player proximity or scheduled world ticks.
-   **Scope:** An instance of a ChunkEvent is extremely short-lived. Its scope is confined to the duration of its dispatch through the event bus. It is a transient, fire-and-forget data object.
-   **Destruction:** Once the event has been processed by all registered listeners, it falls out of scope and becomes eligible for garbage collection. Systems should **never** maintain long-term references to an event object.

## Internal State & Concurrency
-   **State:** The state of a ChunkEvent is immutable. The internal WorldChunk reference is declared as final and is assigned exclusively at construction time. This design guarantees that the event payload cannot be altered during its dispatch, which is critical for predictable event handling across multiple listeners.
-   **Thread Safety:** The ChunkEvent object itself is inherently thread-safe due to its immutability. It can be safely passed between threads without synchronization.
    **WARNING:** While the event object is thread-safe, the WorldChunk it references is **not**. The WorldChunk represents mutable game state. Any listener that accesses or modifies the WorldChunk must adhere to the world simulation's threading model, typically by queuing modifications to be executed on the main world thread or by using prescribed locking mechanisms. Failure to do so will lead to race conditions and world corruption.

## API Surface
The public contract is minimal, focusing solely on data access.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getChunk() | WorldChunk | O(1) | Returns the non-null WorldChunk that is the subject of this event. |

## Integration Patterns

### Standard Usage
The standard pattern is not to create ChunkEvent instances, but to listen for them. Systems implement event handlers that subscribe to specific, concrete subclasses of ChunkEvent to perform work.

```java
// Example of a system listening for a hypothetical ChunkLoadEvent
@EventHandler
public void onChunkLoaded(ChunkLoadEvent event) {
    WorldChunk newlyLoadedChunk = event.getChunk();
    
    // Queue work on the world thread to safely interact with the chunk
    world.getScheduler().run(() -> {
        populateChunkWithEntities(newlyLoadedChunk);
    });
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** You cannot use `new ChunkEvent()` as the class is abstract. Furthermore, developers should not instantiate concrete subclasses; this is the sole responsibility of the core world management systems. Manually firing events can desynchronize world state.
-   **Unsynchronized State Modification:** Do not directly modify the WorldChunk returned by `getChunk()` from an asynchronous event listener. This is the most common source of severe concurrency bugs. All modifications must be synchronized with the main world update loop.
-   **Storing Event References:** Do not cache or store event objects in services or components. They are transient and their state reflects a single point in time. Process them immediately and discard the reference.

## Data Pipeline
The flow of data for a chunk-related event follows a clear, decoupled path from cause to effect.

> Flow:
> World State Trigger (e.g., Player Movement) -> Chunk Management Service -> **Instantiation of a concrete ChunkEvent** -> Server Event Bus -> Registered Listeners (e.g., Network Broadcaster, AI Spawner, Persistence Manager) -> System-Specific Actions

