---
description: Architectural reference for ChunkTracker
---

# ChunkTracker

**Package:** com.hypixel.hytale.server.core.modules.entity.player
**Type:** Stateful Component

## Definition
```java
// Signature
public class ChunkTracker implements Component<EntityStore> {
```

## Architecture & Concepts

The ChunkTracker is a server-side, stateful component responsible for managing the "Area of Interest" (AoI) for a single player entity. It is a core element of the server's world streaming and visibility system, acting as the bridge between a player's position in the world and the network packets required to render that world on their client.

Fundamentally, this component maintains the set of world chunks that should be loaded for its associated player. It continuously compares the player's current position and view distance against the set of currently loaded chunks. When discrepancies are found, it orchestrates the loading of new chunks and the unloading of distant ones.

Key architectural concepts include:

*   **Spiral Iteration:** To discover new chunks to load as a player moves, it employs a CircleSpiralIterator. This ensures that chunks closer to the player are prioritized, creating a natural "loading-in" effect from the center outwards.
*   **Throttling and Pacing:** Chunk loading is a network and I/O intensive operation. The ChunkTracker implements a throttling mechanism using an accumulator and configurable `maxChunksPerSecond` and `maxChunksPerTick` values. This prevents the server from being overwhelmed by a fast-moving player and ensures a smooth, paced delivery of world data.
*   **Asynchronous Loading:** The component initiates chunk loading asynchronously via `ChunkStore.getChunkReferenceAsync`. This prevents the main server tick from blocking on disk I/O or world generation, which is critical for server performance.
*   **Hot vs. Cold Chunks:** The system differentiates between a "hot" radius and the standard view radius. Chunks within the hot radius are marked with the TICKING flag, enabling entity simulation and other active game logic. Chunks outside this radius but within the view distance are considered "cold" and are primarily for rendering purposes.
*   **State Segregation:** The component carefully manages three distinct sets of chunk indices: `loading` (chunks requested but not yet confirmed), `loaded` (chunks confirmed sent to the client), and `reload` (chunks that need to be resent due to updates).

## Lifecycle & Ownership

The lifecycle of a ChunkTracker is rigidly bound to its parent player entity within the Entity-Component-System (ECS) framework.

*   **Creation:** A ChunkTracker instance is created and attached to a player entity when that entity is first initialized in the world, typically upon the player joining the server. It is not instantiated directly but is managed by the EntityModule and EntityStore.
*   **Scope:** The component persists for the entire duration of the player's session. Its internal state (the set of loaded chunks) evolves continuously as the player moves through the world.
*   **Destruction:** The component is destroyed when its parent player entity is removed from the EntityStore, for example, when the player disconnects. The `unloadAll` method is a critical part of this teardown process, ensuring that explicit `UnloadChunk` packets are sent to the client to prevent memory leaks on the client-side.

## Internal State & Concurrency

The ChunkTracker is a highly mutable and concurrent component. Correctly managing its state is critical to server stability.

*   **State:** The core state is composed of three `HLongSet` collections: `loading`, `loaded`, and `reload`. It also maintains scalar values for the player's last known chunk coordinates, current view radius, and throttling accumulator. This state is volatile and changes on almost every tick.
*   **Thread Safety:** **HIGHLY CRITICAL.** The component is not thread-safe by default. All access to the internal chunk sets (`loading`, `loaded`, `reload`) is protected by a `StampedLock`. The `tick` method, which performs the primary logic, acquires a full write lock. Asynchronous loading callbacks also acquire locks to safely transition a chunk's state from `loading` to `loaded`. Any external system attempting to read the state, such as for debug commands, must use the provided read-lock-protected accessors like `getLoadedChunksCount`. Direct field access from other threads will lead to data corruption and server instability.

## API Surface

The public API is designed to be driven by an ECS System and to provide safe accessors for external queries.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(ref, dt, buffer) | void | O(R²) | The primary update method. Called every server tick. Complexity is relative to the view radius (R) as it may iterate chunks in a spiral. |
| unloadAll(playerRef) | void | O(N) | Instructs the client to unload all tracked chunks. N is the number of loaded chunks. Critical for player logout. |
| isLoaded(long index) | boolean | O(1) | Checks if a specific chunk is in the `loaded` set. Acquires a read lock. |
| removeForReload(long index) | void | O(1) | Moves a chunk from the `loaded` set to the `reload` set, queueing it to be sent again. Acquires a write lock. |
| setReadyForChunks(bool) | void | O(1) | A control flag to enable or disable the tracker's operation. Used to prevent chunk loading before the player is fully spawned. |
| getChunkVisibility(long) | ChunkVisibility | O(1) | Determines if a chunk is HOT, COLD, or NONE relative to the player's current position. |

## Integration Patterns

### Standard Usage

The ChunkTracker is not intended for direct manipulation. It is managed exclusively by a server-side ECS System (e.g., a PlayerSystem) which calls the `tick` method once per frame for each online player.

```java
// Inside an ECS System's update loop
for (Ref<EntityStore> playerEntity : players) {
    ChunkTracker tracker = commandBuffer.getComponent(playerEntity, ChunkTracker.getComponentType());
    if (tracker != null) {
        tracker.tick(playerEntity, deltaTime, commandBuffer);
    }
}
```

### Anti-Patterns (Do NOT do this)

*   **Direct Instantiation:** Never use `new ChunkTracker()`. The component must be created and managed by the `EntityStore` to be correctly associated with an entity.
*   **Unsynchronized Access:** Do not access the `loading` or `loaded` fields directly from another thread. This will bypass the `StampedLock` and cause severe concurrency issues. Use methods like `isLoaded` or `getLoadedChunksCount`.
*   **External Ticking:** Do not call the `tick` method from anywhere other than the designated main-thread ECS update system. The component's logic is stateful and not designed for concurrent or out-of-order execution.

## Data Pipeline

The ChunkTracker orchestrates the flow of data from the server's world representation to the player's client.

> Flow:
> Player Movement (TransformComponent update) -> ECS System calls **ChunkTracker.tick()** -> **ChunkTracker** identifies required chunks -> Asynchronous request to `ChunkStore` -> Chunk data loaded/generated from disk/cache -> Chunk data serialized into Packets -> Packets sent via `PlayerRef.getPacketHandler()` -> Client Network Layer -> Client World Renderer

