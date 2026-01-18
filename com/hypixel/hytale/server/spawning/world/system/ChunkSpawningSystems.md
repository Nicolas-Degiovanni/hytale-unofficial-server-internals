---
description: Architectural reference for ChunkSpawningSystems
---

# ChunkSpawningSystems

**Package:** com.hypixel.hytale.server.spawning.world.system
**Type:** Utility

## Definition
```java
// Signature
public class ChunkSpawningSystems {
```

## Architecture & Concepts

The ChunkSpawningSystems class is a static logic container responsible for the bookkeeping of the server's NPC spawning system at the world chunk level. It does not operate as a standalone system but provides the core implementation for several tightly-coupled Entity-Component-Systems (ECS) defined as nested classes.

Its primary architectural role is to act as a bridge between low-level world data, represented by the WorldChunk component, and the high-level, world-global spawning aggregator, the WorldSpawnData resource. The core design pattern is **incremental aggregation**. Instead of the main spawning algorithm repeatedly scanning every block of every loaded chunk, this system preprocesses chunks *once* as their state changes.

When a chunk becomes active (i.e., enters a "ticking" state), ChunkSpawningSystems scans its block data to calculate critical spawning metrics, such as the number of blocks belonging to each environment type. These metrics are stored in a ChunkSpawnData component and aggregated into the central WorldSpawnData resource. When the chunk becomes inactive, its contributions are atomically subtracted.

This design ensures that the primary spawning logic can make decisions based on pre-calculated, world-level data, which is significantly more performant than brute-force world analysis. The actual ECS integration is handled by the nested classes `ChunkRefAdded` and `TickingState`, which subscribe to ECS events and delegate all complex processing to the static methods of this container class.

### Lifecycle & Ownership
- **Creation:** The ChunkSpawningSystems class is a static utility and is never instantiated. Its nested system classes, `ChunkRefAdded` and `TickingState`, are instantiated by the SpawningPlugin during server bootstrap and registered with the world's ECS scheduler.
- **Scope:** The static logic is available for the entire server lifetime. The registered system instances persist for the lifetime of the world they are associated with.
- **Destruction:** The nested system instances are de-registered and garbage collected when the server world is shut down.

## Internal State & Concurrency
- **State:** The ChunkSpawningSystems class is fundamentally **stateless**. It holds no mutable fields and all its methods operate exclusively on parameters passed to them. All relevant state is stored and managed within the ECS, primarily in the WorldSpawnData resource and the ChunkSpawnData component.

- **Thread Safety:** This class is **not thread-safe** and is designed to be executed by a single thread within the server's main ECS update loop. The methods perform direct reads from the ECS Store and write operations via a CommandBuffer. Any external, concurrent invocation will bypass the engine's synchronization mechanisms, leading to race conditions, state corruption, and undefined behavior.

## API Surface

The primary public contract is fulfilled by the nested system classes. The protected static methods constitute the core logic but are considered internal implementation details.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| processStartedChunk(...) | boolean | O(C) | **Critical.** Prepares a newly activated chunk for spawning. Scans all 1024 block columns, calculates environment segment counts, attaches a ChunkSpawnData component, and aggregates results into the global WorldSpawnData. |
| processStoppedChunk(...) | boolean | O(E) | Decommissions an inactive chunk. Removes its contributions from the global WorldSpawnData to prevent NPCs from spawning relative to its data. Removes the ChunkSpawnData component. |
| updateChunkCount(...) | void | O(1) | A simple helper to adjust the total active chunk count in WorldSpawnData. |
| ChunkRefAdded | class | N/A | An ECS system that triggers `processStartedChunk` when a new, ticking chunk entity appears. |
| TickingState | class | N/A | An ECS system that monitors the NonTicking component on chunks. It triggers `processStartedChunk` when a chunk becomes active and `processStoppedChunk` when it becomes inactive. |

*Complexity Note: O(C) refers to the number of columns in a chunk (constant 1024). O(E) refers to the number of distinct environments within that chunk.*

## Integration Patterns

### Standard Usage

Direct interaction with this class is incorrect. The engine integrates this logic by registering its nested systems. The systems then react automatically to changes in chunk state managed by the world streaming and simulation engine.

```java
// During plugin initialization
// This is the ONLY correct way to integrate this logic.
// The engine will then invoke the systems automatically.

// Get the ECS scheduler for the world's ChunkStore
SystemScheduler<ChunkStore> scheduler = world.getChunkStore().getScheduler();

// Instantiate and register the systems
scheduler.add(new ChunkSpawningSystems.ChunkRefAdded(...));
scheduler.add(new ChunkSpawningSystems.TickingState(...));
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation:** Never call `processStartedChunk` or `processStoppedChunk` from other game systems. Doing so will desynchronize the spawning aggregates from the actual chunk state, as it bypasses the canonical ECS event triggers.
- **Manual Instantiation:** Do not use `new ChunkSpawningSystems()`. It is a static utility class and provides no instance-level functionality.
- **State Manipulation:** Do not manually add or modify a `ChunkSpawnData` component on a chunk entity. This component is owned and managed exclusively by these systems. Manual changes will be overwritten or cause calculation errors.

## Data Pipeline

The logic in ChunkSpawningSystems is event-driven, triggered by a change in a chunk's simulation state. The following flow describes the pipeline when a chunk becomes active and ready for simulation.

> Flow:
> World Streamer loads chunk data -> Engine creates a chunk entity -> Engine removes the **NonTicking** component -> **TickingState.onComponentRemoved** is invoked -> `processStartedChunk` is called -> Scans **WorldChunk** block data -> Populates a new **ChunkSpawnData** component -> Queues component addition in **CommandBuffer** -> Updates global **WorldSpawnData** resource -> Spawning System reads **WorldSpawnData** for decision making

---

