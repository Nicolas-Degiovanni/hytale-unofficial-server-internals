---
description: Architectural reference for ChunkPreLoadProcessEvent
---

# ChunkPreLoadProcessEvent

**Package:** com.hypixel.hytale.server.core.universe.world.events
**Type:** Transient

## Definition
```java
// Signature
public class ChunkPreLoadProcessEvent extends ChunkEvent implements IProcessedEvent {
```

## Architecture & Concepts
The ChunkPreLoadProcessEvent is a specialized, short-lived message object used within the server's world chunk management pipeline. It signals a critical transition point: a WorldChunk has been either loaded from storage or newly generated, and is now entering the "pre-processing" phase before it becomes fully active in the world.

Its primary role is not just to carry data, but to act as a **performance monitoring instrument**. By implementing the IProcessedEvent interface, it provides a mechanism for the event dispatch system to measure the time elapsed between different processing stages (hooks). If any single stage of the pre-load process exceeds the world's configured tick duration, this event is responsible for logging a severe performance warning.

This event is a key component of an asynchronous, multi-stage data processing pipeline. It decouples the initial chunk retrieval (I/O or generation) from subsequent processing tasks like populator execution, structure placement, and neighbor data linking.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's internal chunk loading and generation orchestrator when a chunk is ready for its first stage of processing. This is a system-level event and is never created by gameplay logic.
- **Scope:** The event's lifetime is extremely brief, confined to the execution scope of a single chunk's pre-load pipeline. It exists only for the time it takes to be passed through all registered pre-load event listeners.
- **Destruction:** The object is eligible for garbage collection immediately after the event bus finishes dispatching it to all subscribers. There are no long-term references held to it.

## Internal State & Concurrency
- **State:** This object is **Mutable**. Its internal state, specifically the lastDispatchNanos and didLog fields, is modified each time the processEvent method is invoked by the event dispatcher. This state is essential for its function as a performance timer.
- **Thread Safety:** This class is **Not Thread-Safe**. It is designed to be created, dispatched, and processed within a single, well-defined thread, typically a world worker thread. Concurrent calls to processEvent on the same instance would corrupt its timing metrics and lead to incorrect logging. The engine's event bus guarantees sequential, single-threaded delivery to its listeners, thus ensuring safe usage.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isNewlyGenerated() | boolean | O(1) | Returns true if the associated chunk was just generated, false if loaded from storage. |
| getHolder() | Holder<ChunkStore> | O(1) | Provides access to the ChunkStore, allowing processors to interact with chunk persistence. |
| processEvent(hookName) | void | O(1) | **Internal Callback.** Updates internal timers and logs a performance warning if the time since the last call exceeds the world tick step. |
| didLog() | boolean | O(1) | Returns true if this event instance has already triggered a performance warning log. |

## Integration Patterns

### Standard Usage
This event is not created or managed directly. Instead, server systems subscribe to it via an event bus to perform chunk processing tasks. The processEvent method is invoked by the event bus itself, not by the subscriber.

```java
// A system listening for the event
@Subscribe
public void onChunkPreLoad(ChunkPreLoadProcessEvent event) {
    WorldChunk chunk = event.getChunk();
    if (event.isNewlyGenerated()) {
        // This is a new chunk, run terrain populators
        runPopulators(chunk);
    }
    // Perform other pre-load logic...
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Invocation of processEvent:** Never call `event.processEvent()` from a listener. This method is exclusively for the event dispatching system to manage performance timings between hooks. Manual calls will destroy the integrity of performance metrics.
- **Storing Event References:** Do not store a reference to this event in a field or long-lived collection. It holds references to core world components like WorldChunk and ChunkStore. Retaining the event can prevent the chunk from being unloaded, causing a severe memory leak.
- **Re-Dispatching:** An event instance is single-use. Do not attempt to post the same event object to the bus a second time, as its internal timing state will be invalid.

## Data Pipeline
The event acts as a stateful token that flows through the chunk pre-loading system, measuring the time spent in each processing stage.

> Flow:
> Chunk Load Request -> Chunk I/O or Generation -> **ChunkPreLoadProcessEvent Created** -> Event Bus Dispatch -> Listener 1 (e.g., Terrain Populator) -> `processEvent` called by bus -> Listener 2 (e.g., Structure Placer) -> `processEvent` called by bus -> ... -> Finalization -> Event Discarded

