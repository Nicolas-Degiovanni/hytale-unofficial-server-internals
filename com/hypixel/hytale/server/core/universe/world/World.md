---
description: Architectural reference for the World class, the core simulation environment for Hytale.
---

# World

**Package:** com.hypixel.hytale.server.core.universe.world
**Type:** Core Service / Manager

## Definition
```java
// Signature
public class World extends TickingThread implements Executor, ExecutorMetricsRegistry.ExecutorMetric, ChunkAccessor<WorldChunk>, IWorldChunks, IMessageReceiver {
```

## Architecture & Concepts

The World class is the central orchestrator for a single, self-contained game instance. Each dimension, zone, or instanced dungeon is represented by its own World object, running on a dedicated thread. This design provides strong isolation between game environments, preventing performance issues in one world from affecting others.

Architecturally, World acts as a container and lifecycle manager for the primary gameplay systems:

*   **Entity-Component-System (ECS) Stores:** It owns and manages the `EntityStore` and `ChunkStore`. These stores are the ground truth for all entity and terrain data within the world's boundaries.
*   **Dedicated Game Loop:** By extending `TickingThread`, each World instance executes its own simulation loop. This loop is responsible for processing entity updates, chunk ticking, and executing scheduled tasks.
*   **Thread-Safe Command Queue:** It implements the `Executor` interface, exposing a task queue (`taskQueue`). This is the **only** sanctioned mechanism for other server threads to safely interact with and modify the world's state. All cross-thread communication is marshaled through this queue, ensuring sequential, thread-safe execution.
*   **Sub-System Orchestration:** It initializes and coordinates major background services like the `ChunkLightingManager` for lighting calculations and the `WorldMapManager` for minimap and compass data.
*   **Player Session Management:** The World is the authority for player presence. It handles the entire lifecycle of a player joining, participating in, and leaving the simulation, including network packet orchestration and initial chunk loading.

A World is defined by its `WorldConfig`, which specifies its generator, save behavior, gameplay rules, and other metadata. The `Universe` service is responsible for the creation and management of all active World instances.

### Lifecycle & Ownership
- **Creation:** A World is instantiated and initialized by the central `Universe` service, typically in response to a server startup configuration or a dynamic command. The constructor sets up initial state, but the asynchronous `init()` method must be called to load world generators and configuration files from disk. A World is not considered fully operational until the `CompletableFuture` from `init()` completes successfully.
- **Scope:** The object's lifetime is tied to the duration the world is loaded on the server. For a default world, this is the entire server session. For temporary worlds (e.g., dungeons), it exists only until it is explicitly unloaded.
- **Destruction:** Shutdown is initiated via `stopIndividualWorld()`. This begins a graceful shutdown sequence managed by the `onShutdown()` method. The sequence is critical and follows a strict order:
    1.  Drain all players to a fallback world or disconnect them.
    2.  Wait for any in-flight chunk loading operations to complete.
    3.  Signal sub-systems like `ChunkLightingManager` to stop.
    4.  Invoke `shutdown()` on `ChunkStore` and `EntityStore`, which flushes all pending changes to disk.
    5.  Save the final `WorldConfig`.
    6.  Set the `alive` flag to false and unregister from the `Universe`.

**WARNING:** Failure to complete this sequence, for example during a server crash, may result in data loss for any changes not yet persisted by the stores.

## Internal State & Concurrency
- **State:** The World object is highly stateful and mutable. It maintains direct references to its core data stores (`chunkStore`, `entityStore`), a collection of active players (`players`), a queue of pending tasks (`taskQueue`), and numerous configuration flags (`isPaused`, `isTicking`). Its state is a live representation of the entire simulation.

- **Thread Safety:** The class is fundamentally **not thread-safe**. All methods that mutate state are designed to be called exclusively from the World's own thread ("WorldThread - {name}").

    To enforce this, the World implements a single-threaded execution model. External threads **must not** call mutating methods directly. Instead, they must schedule work using the `execute(Runnable)` method. This places the operation in a queue that is consumed sequentially at the beginning of each tick cycle within the `tick()` method. Methods intended for cross-thread access, such as `getChunkAsync`, are explicitly asynchronous and return a `CompletableFuture` to avoid blocking the caller and to marshal the final result safely.

    **WARNING:** Directly accessing or modifying any internal collection or state object (e.g., `getChunkStore().getStore()`) from an external thread will lead to race conditions, data corruption, and server instability. All interactions must be funneled through the `execute` method or designated async APIs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| init() | CompletableFuture<World> | O(IO) | Asynchronously initializes the world, loading generators and configs. Must be called after construction. |
| addPlayer(playerRef, transform) | CompletableFuture<PlayerRef> | O(IO) | Manages the complex process of adding a player. Involves network negotiation, chunk loading, and entity registration. |
| getChunkAsync(index) | CompletableFuture<WorldChunk> | O(IO) | Asynchronously loads or retrieves a chunk. The returned future completes on the World thread. |
| execute(command) | void | O(1) | Queues a Runnable to be executed safely on the World's next tick. This is the primary entry point for cross-thread state mutation. |
| stopIndividualWorld() | void | O(IO) | Initiates the graceful shutdown and save process for the world. |
| setPaused(paused) | void | O(N) | Pauses or resumes the world simulation. Broadcasts the state change to all N players. |

## Integration Patterns

### Standard Usage
Interaction with a World instance should always be mediated through the `Universe` service. State modifications must be scheduled via the executor service pattern.

```java
// How a developer should normally use this
Universe universe = HytaleServer.get().getUniverse();
World targetWorld = universe.getWorld("default");

// Schedule a task to run safely on the world's thread
targetWorld.execute(() -> {
    // This code is now running inside the world's game loop
    // It is safe to interact with the EntityStore and other systems here
    targetWorld.setPaused(true);
    System.out.println("World has been paused.");
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new World()`. The `Universe` manages the lifecycle, registration, and threading of World objects. Manual creation will result in a non-functional, unmanaged world instance.
- **Cross-Thread State Mutation:** Never modify World state directly from another thread. This will bypass all safety mechanisms and corrupt the simulation.
    ```java
    // BAD: This will cause a race condition and likely crash the server.
    World world = universe.getWorld("default");
    world.getEntityStore().getStore().addEntity(...);
    ```
- **Blocking the World Thread:** Submitting a task to `execute` that performs blocking IO or enters a long-running loop will freeze the entire world simulation, making it unresponsive. All expensive operations must be performed asynchronously.

## Data Pipeline

The World acts as a central processing node for many data flows. A primary example is the player joining pipeline.

> Flow:
> Network Handshake -> `Universe.transferPlayer()` -> `World.addPlayer()` -> **World Thread Execution** -> `chunkStore.getChunkReferenceAsync()` -> **Disk/Cache IO** -> Chunk Loaded -> `onSetupPlayerJoining()` -> Send `JoinWorld` Packet -> `onFinishPlayerJoining()` -> Add Player `Ref` to `EntityStore` -> Player is now part of the `tick()` loop.

