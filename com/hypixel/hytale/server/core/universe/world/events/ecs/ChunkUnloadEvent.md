---
description: Architectural reference for ChunkUnloadEvent
---

# ChunkUnloadEvent

**Package:** com.hypixel.hytale.server.core.universe.world.events.ecs
**Type:** Transient

## Definition
```java
// Signature
public class ChunkUnloadEvent extends CancellableEcsEvent {
```

## Architecture & Concepts
The ChunkUnloadEvent is a message object used within the server's Entity Component System (ECS) to signal the *intent* to unload a WorldChunk from active memory. It is a fundamental component of the server's dynamic world streaming and memory management architecture.

This class does not perform any action itself; it is a data container that represents a proposed state change. Its primary architectural function is to decouple the system that identifies an unloadable chunk (the *producer*) from the various systems that may have a vested interest in that chunk's state (the *consumers*).

By extending CancellableEcsEvent, it provides a powerful interception point. Any system listening for this event can prevent the unload operation by cancelling the event. This allows for complex, rule-based chunk lifecycle management without creating tight dependencies between systems. For example, a system managing protected player zones can veto a chunk unload request originating from a generic proximity-based garbage collector.

The *resetKeepAlive* flag introduces a secondary behavior, suggesting a distinction between a "soft" unload (where the chunk might be quickly reloaded) and a "hard" unload where its resources are fully relinquished.

## Lifecycle & Ownership
- **Creation:** Instantiated by high-level world management systems, such as a WorldManager or a dedicated ChunkGarbageCollector. This typically occurs when the server determines a chunk is no longer within any player's view distance or simulation distance.
- **Scope:** Extremely short-lived. An instance of ChunkUnloadEvent exists only for the duration of the event dispatch phase within a single server tick.
- **Destruction:** The object becomes eligible for garbage collection immediately after all subscribed ECS systems have processed it. There is no manual destruction or pooling mechanism.

## Internal State & Concurrency
- **State:** Mutable. While the reference to the WorldChunk is final, the event's disposition—specifically its cancelled status and the *resetKeepAlive* flag—is designed to be mutated by event listeners during the dispatch cycle.
- **Thread Safety:** **This class is not thread-safe and must only be accessed from the world's main update thread.** The entire ECS event bus architecture relies on a single-threaded, sequential processing model to guarantee a deterministic outcome and prevent race conditions. Modifying this event from an asynchronous task will lead to undefined behavior and world corruption.

## API Surface
The public contract includes methods inherited from CancellableEcsEvent, which are critical to its function.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getChunk() | WorldChunk | O(1) | Returns the non-null WorldChunk that is the target of the unload operation. |
| setResetKeepAlive(boolean) | void | O(1) | Allows a listener to modify whether the chunk's keep-alive timer should be reset. |
| willResetKeepAlive() | boolean | O(1) | Checks the current state of the keep-alive flag. |
| setCancelled(boolean) | void | O(1) | *Inherited.* Vetoes the unload operation. This is the primary interaction mechanism. |
| isCancelled() | boolean | O(1) | *Inherited.* Checks if the unload operation has been vetoed by any listener. |

## Integration Patterns

### Standard Usage
The intended pattern is to create an ECS System that listens for this event to implement custom game logic related to chunk persistence. The system should inspect the event's payload and, if necessary, cancel it.

```java
// Inside an ECS System that manages protected zones
@Subscribe
public void onChunkUnload(ChunkUnloadEvent event) {
    WorldChunk chunk = event.getChunk();
    if (isChunkInProtectedZone(chunk.getChunkPos())) {
        // Veto the unload operation to keep the protected zone in memory
        event.setCancelled(true);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Firing:** Do not create and dispatch this event manually unless you are replacing a core part of the world management engine. Firing this event without the corresponding logic in the WorldManager to handle the post-event state will have no effect or could lead to state desynchronization.
- **Ignoring Cancellation:** Any system that performs a destructive action based on this event (e.g., writing chunk data to a final save file) **must** check `event.isCancelled()` before proceeding. Failure to do so violates the architectural contract and can lead to data loss.
- **Asynchronous Handling:** Do not hold a reference to the event and modify it on another thread. Event objects are recycled or garbage collected immediately after the tick's dispatch cycle.

## Data Pipeline
The event acts as a control signal in the chunk unloading data pipeline.

> Flow:
> WorldManager identifies candidate chunk -> **ChunkUnloadEvent created** -> Dispatched to ECS Event Bus -> Multiple systems listen and potentially cancel the event -> WorldManager checks `isCancelled()` -> If not cancelled, WorldChunk is serialized and removed from memory.

