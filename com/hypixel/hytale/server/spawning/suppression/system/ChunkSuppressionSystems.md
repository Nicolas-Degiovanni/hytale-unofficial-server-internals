---
description: Architectural reference for ChunkSuppressionSystems.ChunkAdded
---

# ChunkSuppressionSystems.ChunkAdded

**Package:** com.hypixel.hytale.server.spawning.suppression.system
**Type:** Engine System

## Definition
```java
// Signature
public static class ChunkAdded extends RefSystem<ChunkStore> {
```

## Architecture & Concepts
The ChunkAdded system is a reactive component within the server's Entity Component System (ECS) framework. Its sole responsibility is to ensure the persistence of spawn suppression state for game world chunks as they are loaded into memory.

When a chunk is streamed into the active world (represented by the addition of a BlockChunk component to a chunk entity), this system is automatically triggered. It queries the world-level SpawnSuppressionController to check if the newly loaded chunk has a pre-existing suppression record. If a record exists, this system re-attaches the corresponding ChunkSuppressionEntry component to the chunk entity.

This mechanism is critical for state integrity. It prevents a suppressed area from becoming active again simply because it was unloaded and subsequently reloaded due to player movement. It acts as the bridge between the persistent, long-term suppression map and the live, in-memory component state of a chunk entity.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's internal SystemManager during the initialization of the SpawningPlugin. This system is not intended for manual creation.
- **Scope:** Singleton per World instance. Its lifecycle is strictly bound to the game world it services.
- **Destruction:** Decommissioned and garbage collected when the SpawningPlugin is unloaded or the parent World is shut down.

## Internal State & Concurrency
- **State:** This system is entirely stateless. It holds immutable references to component and resource type definitions, which are injected via its constructor during engine bootstrap. It does not cache or maintain any session or per-chunk data itself.
- **Thread Safety:** The Hytale ECS framework guarantees that the onEntityAdded method is invoked exclusively from the main world-update thread. It is not thread-safe and must not be accessed from other threads. All component modifications are marshaled through the provided CommandBuffer, ensuring transactional safety within the scope of a single game tick.

## API Surface
The public API is for engine-internal use only. Direct invocation will destabilize the ECS state.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| onEntityAdded(ref, reason, store, cmd) | void | O(1) | Engine callback. Re-applies a persistent suppression state to a newly loaded chunk. |
| onEntityRemove(ref, reason, store, cmd) | void | O(1) | No-op. This system does not react to chunk unloading. |
| getQuery() | Query | O(1) | Configuration method. Declares interest in entities possessing a BlockChunk component. |

## Integration Patterns

### Standard Usage
This system operates implicitly and is not designed for direct interaction. Its functionality is triggered automatically by the core engine's chunk management lifecycle. Developers influence this system indirectly by manipulating the global SpawnSuppressionController.

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new ChunkAdded()`. The system requires engine-managed dependencies injected at startup and will fail if created manually.
- **Manual Invocation:** Do not call onEntityAdded directly. Bypassing the ECS framework's event loop will lead to severe data corruption, race conditions, and unpredictable server behavior.

## Data Pipeline
This system's data flow is reactive, initiated by the world engine.

> Flow:
> World Engine streams in a chunk -> ECS Framework adds BlockChunk component to entity -> **ChunkAdded.onEntityAdded** is triggered -> Reads from global SpawnSuppressionController map -> Writes ChunkSuppressionEntry component to CommandBuffer -> ECS applies CommandBuffer at the end of the tick.

---
description: Architectural reference for ChunkSuppressionSystems.Ticking
---

# ChunkSuppressionSystems.Ticking

**Package:** com.hypixel.hytale.server.spawning.suppression.system
**Type:** Engine System

## Definition
```java
// Signature
public static class Ticking extends TickingSystem<ChunkStore> {
```

## Architecture & Concepts
The Ticking system functions as a deferred command processor for managing chunk-based spawn suppression. It runs once per server game tick to apply pending suppression state changes in a centralized and synchronized manner.

Other game logic systems do not modify ChunkSuppressionEntry components directly. Doing so could introduce mid-tick state inconsistencies or concurrency hazards. Instead, they enqueue requests to add or remove suppression into a global ChunkSuppressionQueue resource.

This Ticking system is the sole, authoritative consumer of that queue. During its execution phase in the game loop, it drains both the addition and removal queues, applying the component changes directly to the ChunkStore. This pattern guarantees that all suppression state modifications are resolved atomically within a single, well-defined phase of the tick, simplifying logic for all other systems and eliminating entire classes of state-related bugs.

## Lifecycle & Ownership
- **Creation:** Instantiated by the server's internal SystemManager during the initialization of the SpawningPlugin.
- **Scope:** Singleton per World instance. It persists for the entire lifetime of the game world.
- **Destruction:** Decommissioned when the SpawningPlugin is unloaded or the parent World is shut down.

## Internal State & Concurrency
- **State:** This system is stateless. It holds immutable references to component and resource types. The state it operates on is external, contained entirely within the ChunkSuppressionQueue resource.
- **Thread Safety:** The tick method is executed by the ECS framework exclusively on the main world-update thread. While the system itself is not entered concurrently, the ChunkSuppressionQueue it reads from may be written to by other threads. The queue implementation must therefore be thread-safe. This system acts as the synchronized commit point for all queued operations.

## API Surface
The public API is for engine-internal use only.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, index, store) | void | O(N+M) | Engine callback. Processes all pending suppression changes. N is additions, M is removals. |

## Integration Patterns

### Standard Usage
This system is not invoked directly. Other systems interact with it by submitting requests to the world's ChunkSuppressionQueue resource.

```java
// In another system, requesting to suppress a chunk
ChunkSuppressionQueue queue = world.getChunkStore().getStore().getResource(ChunkSuppressionQueue.class);
ChunkSuppressionEntry newEntry = createSuppressionEntry(); // Business logic
Ref<ChunkStore> chunkRef = getChunkReference(); // Business logic

// Enqueue the request. The Ticking system will process this on the next tick.
queue.getToAdd().add(new AbstractMap.SimpleImmutableEntry<>(chunkRef, newEntry));
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new Ticking()`. It cannot function outside the engine's managed lifecycle.
- **Manual Invocation:** Calling the tick method manually will process suppression changes out of sync with the game loop, causing severe temporal bugs and state desynchronization.
- **Bypassing the Queue:** Systems must not add or remove ChunkSuppressionEntry components directly. This violates the deferred execution pattern, re-introducing the risk of race conditions and making system behavior unpredictable. Always use the ChunkSuppressionQueue.

## Data Pipeline
This system's data flow is proactive, driven by the game loop and an external data queue.

> Flow:
> External Game Logic -> Enqueues add/remove request in ChunkSuppressionQueue -> Server Game Loop advances to system processing phase -> **Ticking.tick** is triggered -> Drains requests from ChunkSuppressionQueue -> Applies component changes directly to the live ChunkStore.

