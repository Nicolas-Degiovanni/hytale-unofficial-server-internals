---
description: Architectural reference for ChunkUnloadingSystem
---

# ChunkUnloadingSystem

**Package:** com.hypixel.hytale.server.core.universe.world.storage.component
**Type:** System Component

## Definition
```java
// Signature
public class ChunkUnloadingSystem extends TickingSystem<ChunkStore> implements RunWhenPausedSystem<ChunkStore> {
```

## Architecture & Concepts
The ChunkUnloadingSystem is a critical server-side memory management service operating within the Entity Component System (ECS) framework. Its sole responsibility is to identify and unload world chunks that are no longer required to be in memory, thereby preventing excessive RAM consumption and ensuring server stability.

This system functions as a garbage collector for chunks. It operates by periodically scanning all loaded WorldChunk entities and evaluating their relevance based on proximity to players. The core logic is driven by player-attached ChunkTracker components, which define a "hot" and "cold" radius around each player.

-   **Hot Chunks:** Chunks directly within a player's view distance. These are actively ticked and their unload timers are reset.
-   **Cold Chunks:** Chunks adjacent to the hot zone. These are not actively ticked but are kept in memory for quick access. Their unload timers are allowed to decrement.
-   **Inactive Chunks:** Chunks outside any player's hot or cold zones. These are primary candidates for unloading.

The system also features a **Desperate Unload** mechanism. If server RAM usage surpasses a critical threshold (DESPERATE_UNLOAD_RAM_USAGE_THRESHOLD), the system becomes more aggressive, accelerating the rate at which unload timers decrement to reclaim memory more quickly.

Finally, the system provides a hook, ChunkUnloadEvent, allowing other game logic to intercept and cancel an unload operation. This is essential for scenarios where a chunk must be kept loaded for programmatic reasons, even without a nearby player.

### Lifecycle & Ownership
-   **Creation:** Instantiated and registered by the server's ECS scheduler during the initialization of a World. It is a foundational system required for any world that permits chunk unloading.
-   **Scope:** The lifecycle of the ChunkUnloadingSystem is bound to its parent World. It persists for the entire server session of that world.
-   **Destruction:** De-registered and marked for garbage collection when the World is shut down.

## Internal State & Concurrency
-   **State:** The system itself is largely stateless, containing only a simple counter for logging reminders. The primary state it operates on is external, distributed across all WorldChunk components (specifically their keep-alive and active timers) and a shared resource, ChunkUnloadingSystem.Data. This Data resource holds a timer to throttle the system's execution frequency and a temporary list of all active ChunkTrackers, which is rebuilt each cycle.

-   **Thread Safety:** This system is designed for high-performance, parallel execution. The core evaluation logic, tryUnload, is executed concurrently across multiple worker threads via the forEachEntityParallel method.

    **WARNING:** Thread safety is achieved through the **CommandBuffer** pattern. Direct state mutations (e.g., removing an entity, changing a component flag) are forbidden within the parallel execution block. Instead, these operations are recorded as commands in a thread-local CommandBuffer. The ECS framework guarantees that all queued commands are synchronized and executed safely on the main thread after the parallel phase completes. This architecture prevents race conditions and ensures world state consistency.

## API Surface
The primary interaction with this system is through its registration with the ECS scheduler. Direct invocation is not intended.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(dt, systemIndex, store) | void | O(N + M) | The main entry point called by the ECS scheduler. N is the number of player entities with ChunkTrackers; M is the number of loaded WorldChunk entities. |

## Integration Patterns

### Standard Usage
Developers do not, and should not, interact with this system directly. It is a core background process. The standard way to influence its behavior is indirect:

1.  **Preventing Unloading:** To programmatically keep a chunk loaded, an event listener for ChunkUnloadEvent can be registered. Calling `event.setCancelled(true)` will prevent the unload for that cycle.

    ```java
    // In another system or event handler
    @Subscribe
    public void onChunkUnload(ChunkUnloadEvent event) {
        WorldChunk chunk = event.getChunk();
        if (isChunkNeededForQuest(chunk.getX(), chunk.getZ())) {
            event.setCancelled(true);
            event.setResetKeepAlive(true); // Resets the timer
        }
    }
    ```

2.  **Forcing Unloading:** There is no direct API to force an unload. The correct pattern is to ensure no players are near the chunk and that no logic is cancelling the ChunkUnloadEvent.

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never create an instance using `new ChunkUnloadingSystem()`. The ECS framework manages its lifecycle.
-   **Manual Ticking:** Do not call the tick method manually. This will bypass the scheduler and the CommandBuffer synchronization, leading to severe concurrency issues and world corruption.
-   **State Tampering:** Do not modify a WorldChunk's internal keep-alive timers directly. This interferes with the system's state machine and will cause unpredictable chunk loading and unloading behavior.

## Data Pipeline
The system's data flow is a cyclical process of collection, parallel evaluation, and synchronized mutation.

> Flow:
> System Tick -> Collect all player ChunkTrackers -> **ChunkUnloadingSystem** (begins parallel evaluation) -> For each WorldChunk, check visibility against trackers -> Decrement timers or queue unload command in CommandBuffer -> ECS Scheduler synchronizes and executes all commands -> WorldChunk entity is removed from memory.

