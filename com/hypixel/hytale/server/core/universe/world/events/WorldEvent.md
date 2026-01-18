---
description: Architectural reference for WorldEvent
---

# WorldEvent

**Package:** com.hypixel.hytale.server.core.universe.world.events
**Type:** Base Model / DTO

## Definition
```java
// Signature
public abstract class WorldEvent implements IEvent<String> {
```

## Architecture & Concepts
The WorldEvent class is an abstract base class that serves as the foundational data structure for all server-side events scoped to a specific World instance. It is a core component of the server's event-driven architecture, ensuring that any logic reacting to a world-specific occurrence has guaranteed, immediate access to the context of that world.

By enforcing that every subclass must provide a World instance upon construction, this design pattern eliminates ambiguity and prevents entire classes of bugs related to state management. It acts as a strict contract between event producers (e.g., physics engine, chunk loaders) and event consumers (e.g., AI systems, scripting engine), standardizing how world-contextual information is propagated through the system. Its implementation of the IEvent interface signals its role as a message payload intended for dispatch via the central Event Bus.

### Lifecycle & Ownership
- **Creation:** WorldEvent is an abstract class and is never instantiated directly. Concrete subclasses (e.g., ChunkLoadedEvent, EntitySpawnedInWorldEvent) are instantiated by various server systems when a relevant game state change occurs. For example, the ChunkManager would create a ChunkLoadedEvent when a new chunk is ready for gameplay.
- **Scope:** Instances are ephemeral and highly transient. An event object exists only for the duration of its dispatch cycle on the Event Bus. It is a fire-and-forget data packet.
- **Destruction:** The object becomes eligible for garbage collection as soon as the Event Bus has finished notifying all subscribers. No long-term references are maintained to event instances.

## Internal State & Concurrency
- **State:** The state of the WorldEvent class itself is immutable. The internal **world** reference is final and cannot be changed after construction. However, the World object it holds is a highly mutable, complex object representing the game state.
- **Thread Safety:** This class is inherently thread-safe due to its immutable fields. Callers can safely access its methods from any thread.
    - **WARNING:** While the event object is safe, the World object returned by getWorld is **not** thread-safe. All modifications to the World must be synchronized or queued to execute on the main server thread for that specific world to prevent race conditions, data corruption, and server instability.

## API Surface
The public API is minimal, focusing exclusively on providing the world context.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWorld() | World | O(1) | Retrieves the non-null World instance in which this event occurred. |

## Integration Patterns

### Standard Usage
Developers should not interact with this class directly but rather with its concrete subclasses. The primary interaction pattern is to listen for specific world events on the Event Bus and use the inherited getWorld method to get the operational context.

```java
// Example of a system subscribing to a concrete subclass of WorldEvent
@Subscribe
public void onChunkLoad(ChunkLoadedEvent event) {
    World eventWorld = event.getWorld();
    Chunk newChunk = event.getChunk();

    // Perform logic within the context of the specific world
    log.info("Chunk [" + newChunk.getPosition() + "] loaded in world: " + eventWorld.getName());
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring World Context:** Do not listen for a WorldEvent and then operate on a cached or global world reference. The event provides the precise context required for the operation.
- **Asynchronous World Modification:** Do not retrieve the World object from an event on a worker thread and modify its state directly. This will lead to severe concurrency issues. All state changes must be scheduled for the world's primary update tick.
- **Null World Injection:** Subclasses must never pass a null World to the super constructor. The system's integrity depends on this reference being valid.

## Data Pipeline
WorldEvent is a data payload. Its flow represents the propagation of a state change notification through the server.

> Flow:
> Server System (e.g., Physics Engine) -> `new ConcreteWorldEvent(world, ...)` -> Event Bus Dispatch -> Subscriber Method -> **event.getWorld()** -> World-Specific Logic Execution

