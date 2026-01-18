---
description: Architectural reference for WorldChunk
---

# WorldChunk

**Package:** com.hypixel.hytale.server.core.universe.world.chunk
**Type:** Transient Component

## Definition
```java
// Signature
public class WorldChunk implements BlockAccessor, Component<ChunkStore> {
```

## Architecture & Concepts

The WorldChunk class is the primary server-side representation of a 16x320x16 vertical column of blocks in the game world. It serves as a high-level container and controller, aggregating specialized data-oriented components that manage different aspects of the chunk's state. It is the central point of interaction for any system that needs to read or modify world data.

Fundamentally, a WorldChunk does not store block or entity data directly. Instead, it holds references to more granular components:
- **BlockChunk:** Manages the raw block ID, rotation, and heightmap data.
- **BlockComponentChunk:** Manages block entities and their associated state, such as the contents of a chest.
- **EntityChunk:** Manages entities residing within the chunk's boundaries.

This composite structure allows the engine to selectively load and process only the necessary data for a given task. WorldChunk implements the BlockAccessor interface, establishing it as the authoritative gateway for all block manipulation operations within its volume.

As a component within the server's Entity-Component-System (ECS) framework, a WorldChunk is part of a larger ChunkStore entity. This integration allows chunk state and behavior to be dynamically modified by attaching or detaching other components, such as those related to ticking, physics, or saving.

## Lifecycle & Ownership

The lifecycle of a WorldChunk is tightly managed by the World and its associated chunk loading systems. It is a transient object, created on-demand and destroyed when no longer needed.

- **Creation:** A WorldChunk is instantiated by the World's chunk management system when a region of the map needs to be loaded into memory. This is typically triggered by a player moving into a new area or by a game system requesting access to an unloaded chunk. The instance is populated with data loaded from disk via a ChunkStore or generated for the first time.

- **Scope:** An instance of WorldChunk exists only as long as it is considered "active". Activity is determined by proximity to players, server configuration (e.g., spawn chunks), or an explicit `keepLoaded` flag. The `keepAlive` and `activeTimer` fields are part of a countdown mechanism; if these timers expire, the chunk becomes a candidate for unloading.

- **Destruction:** When a chunk is unloaded, its associated WorldChunk instance is marked for garbage collection. Before destruction, its state is typically serialized and saved to disk via the ChunkStore if it has been modified (indicated by the `needsSaving` flag).

## Internal State & Concurrency

**WARNING:** The WorldChunk object and its sub-components are not thread-safe. All modifications must be performed on the main World ticking thread.

- **State:** The state of a WorldChunk is highly mutable. It acts as a live, in-memory representation of a piece of the world. It caches references to its constituent data chunks (`BlockChunk`, `EntityChunk`) and manages a set of lifecycle flags (`ChunkFlag`). The `needsSaving` flag tracks if its state is dirty and needs to be persisted.

- **Thread Safety:** Concurrency is managed with extreme care.
    - **General Access:** Most methods, particularly those that modify state like `setBlock` or `setState`, contain assertions (`world.isInThread()`) to prevent illegal off-thread access. Attempts to modify a WorldChunk from another thread will either throw an exception or delegate the operation to the main world thread via a `CompletableFuture`.
    - **Flag Management:** The critical exception is the management of `ChunkFlag` values. These flags are protected by a `StampedLock`. This high-performance lock allows for optimistic, non-blocking reads of chunk status by various subsystems (e.g., networking, saving), falling back to a full read-lock only when contention occurs. Writes to flags always acquire a full write-lock to ensure atomic updates.

## API Surface

The public API provides a controlled interface for interacting with the chunk's data and lifecycle.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setBlock(x, y, z, id, ...) | boolean | O(N) | The primary method for modifying a block. Complexity is high and variable, as it can trigger a cascade of updates including lighting, physics, block entity creation, and neighbor block updates. |
| getState(x, y, z) | BlockState | O(1) | Retrieves the block entity state (e.g., chest inventory) at a specific coordinate. Requires execution on the main world thread. |
| setState(x, y, z, state, ...) | void | O(1) | Sets or removes a block entity state at a specific coordinate. Requires execution on the main world thread. |
| setFlag(flag, value) | void | O(1) | Atomically sets a lifecycle flag. This is a thread-safe operation that triggers side effects, such as moving a chunk into or out of the ticking system. |
| is(flag) | boolean | O(1) | Atomically checks a lifecycle flag. This is a thread-safe operation using an optimistic read lock. |
| markNeedsSaving() | void | O(1) | Marks the chunk as modified, ensuring its data will be persisted during the next save cycle. |

## Integration Patterns

### Standard Usage

Interaction with a WorldChunk should always be mediated by the World object. A developer should retrieve a chunk from the world, perform operations, and assume the engine will handle its lifecycle.

```java
// Correctly retrieve a chunk and modify a block
World world = ...;
WorldChunk chunk = world.getChunk(chunkX, chunkZ);

if (chunk != null) {
    // This operation is implicitly executed on the world thread
    chunk.breakBlock(worldX, worldY, worldZ);
}
```

### Anti-Patterns (Do NOT do this)

- **Direct Instantiation:** Never create a WorldChunk using `new WorldChunk()`. The object is deeply integrated into the world's management and ECS systems. Bypassing the official creation path (via `World` or `ChunkStore`) will result in a non-functional and unstable object.

- **Off-Thread Modification:** Do not call methods like `setBlock` or `setState` from an asynchronous task or a different thread without using the engine's provided mechanisms for thread synchronization. This will lead to race conditions, data corruption, and server crashes.

```java
// INCORRECT: This will cause severe concurrency issues.
WorldChunk chunk = ...;
new Thread(() -> {
    // FATAL: Modifying chunk state from an unmanaged thread.
    chunk.setBlock(x, y, z, Block.STONE.getId(), ...);
}).start();
```

- **Flag Mismanagement:** Avoid directly manipulating lifecycle flags unless you are part of the core chunk management system. Forcing a flag like `TICKING` can desynchronize the chunk from the world's ticking loop, causing unpredictable behavior.

## Data Pipeline

A block modification operation triggers a complex data pipeline that flows through the WorldChunk and radiates out to other engine systems.

> Flow:
> Player Input or Game System -> `World.setBlock()` -> **WorldChunk.setBlock()**
>
> **Inside WorldChunk.setBlock():**
> 1.  Update raw block ID in **BlockChunk**.
> 2.  Create or destroy associated **BlockState** in **BlockComponentChunk**.
> 3.  Invalidate lighting information, queuing a recalculation in the **ChunkLighting** system.
> 4.  Update **BlockPhysics** components if the block has support properties.
> 5.  Propagate changes to adjacent blocks for multi-block structures.
> 6.  Send block update and particle effect packets via the **WorldNotificationHandler**.
> 7.  Set the `needsSaving` flag to true.
>
> **Later, in a separate process:**
> Chunk Saving System -> Detects `needsSaving` flag -> Serializes **WorldChunk** state -> **ChunkStore** -> Disk I/O

