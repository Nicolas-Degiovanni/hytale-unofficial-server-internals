---
description: Architectural reference for ChunkSaveEvent
---

# ChunkSaveEvent

**Package:** com.hypixel.hytale.server.core.universe.world.events.ecs
**Type:** Transient

## Definition
```java
// Signature
public class ChunkSaveEvent extends CancellableEcsEvent {
```

## Architecture & Concepts
The ChunkSaveEvent is a message-oriented data structure used within the server-side Entity Component System (ECS) event bus. It does not represent a completed action, but rather a **cancellable intent** to persist a single WorldChunk to disk.

This event acts as a critical decoupling mechanism. It separates the component that identifies a chunk as needing a save (the *producer*, e.g., a world persistence manager) from the components that execute or modify the save process (the *consumers* or *listeners*).

By extending CancellableEcsEvent, it participates in an interceptor pattern. Systems can listen for this event not only to perform the save operation but also to veto it under specific conditions, such as a server-wide maintenance mode or if the chunk is in an invalid state. This provides a powerful, centralized hook for controlling world persistence logic.

### Lifecycle & Ownership
-   **Creation:** Instantiated on-demand by high-level world management systems when a chunk is marked as dirty and requires persistence. This is typically triggered by block changes, entity movements, or periodic background processes.
-   **Scope:** Extremely short-lived. An instance of ChunkSaveEvent exists only for the duration of its dispatch through the ECS event bus. It is a fire-and-forget object.
-   **Destruction:** The event object becomes eligible for garbage collection immediately after all registered listeners have processed it. No system should retain a reference to this event post-dispatch.

## Internal State & Concurrency
-   **State:** Effectively immutable. The internal reference to the WorldChunk is a final field, assigned once at creation. The only mutable state is the cancellation flag inherited from its parent class, which is managed by the event bus and its listeners.
-   **Thread Safety:** The ChunkSaveEvent object itself is safe to read from multiple threads. However, the contained WorldChunk object is **not guaranteed to be thread-safe**. Listeners processing this event must adhere to the server's threading model, typically by executing logic on the main world thread or by employing explicit locking if interacting with the chunk from a worker thread.

## API Surface
The primary contract is defined by its parent, CancellableEcsEvent. The direct API is minimal, serving only to provide the necessary context.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getChunk() | WorldChunk | O(1) | Returns the non-null chunk instance that is the subject of the save request. |
| isCancelled() | boolean | O(1) | (Inherited) Checks if any listener has cancelled this event. Critical check before I/O. |
| setCancelled(boolean) | void | O(1) | (Inherited) Sets the cancellation state. Used by listeners to prevent the save. |

## Integration Patterns

### Standard Usage
The correct pattern involves two types of systems: a producer that creates and dispatches the event, and one or more consumers (listeners) that react to it. A saving system **must** respect the cancellation flag.

```java
// A listener responsible for writing the chunk to disk
@Subscribe
public void onChunkSave(ChunkSaveEvent event) {
    // CRITICAL: Always check for cancellation first.
    if (event.isCancelled()) {
        return;
    }

    WorldChunk chunkToSave = event.getChunk();
    // Proceed with asynchronous I/O operation to save the chunk...
    ioScheduler.submit(() -> saveToDisk(chunkToSave));
}
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring Cancellation:** The most severe anti-pattern is a listener that performs a save operation without first checking `event.isCancelled()`. This violates the event contract and can lead to data corruption or inconsistent world state.
-   **Retaining References:** Storing an instance of ChunkSaveEvent in a field or collection after the event dispatch is complete. This constitutes a memory leak and serves no valid purpose.
-   **Modifying the WorldChunk:** Listeners should treat the WorldChunk object as read-only. Modifying the chunk from within a save event listener can introduce severe race conditions with the main game loop.

## Data Pipeline
The ChunkSaveEvent is a control signal within the world persistence pipeline. Its flow is not one of data transformation, but of broadcast and reaction.

> Flow:
> World Ticking Logic -> Identifies Dirty Chunk -> **new ChunkSaveEvent(chunk)** -> ECS Event Bus Dispatch -> Listeners -> Cancellation Check -> Asynchronous I/O System -> Disk Storage

