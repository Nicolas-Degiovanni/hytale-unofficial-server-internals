---
description: Architectural reference for TickableBlockState
---

# TickableBlockState

**Package:** com.hypixel.hytale.server.core.universe.world.chunk.state
**Type:** Interface

## Definition
```java
// Signature
public interface TickableBlockState {
```

## Architecture & Concepts
The TickableBlockState interface defines the contract for any block entity or state that requires per-tick updates from the server's main simulation loop. It is a core component of the world simulation engine, enabling dynamic and interactive block behaviors such as crop growth, furnace smelting, or custom machinery operations.

This interface acts as a specialization within an Entity Component System (ECS) architecture. By implementing TickableBlockState, a block's state component signals to the chunk processing system that it must be included in the active simulation set. This decouples the generic chunk ticking mechanism from the specific logic of thousands of potential block types, adhering to the Strategy Pattern. The `tick` method is the primary entry point, receiving the necessary world context to safely query data and enqueue state changes.

**WARNING:** Implementations of this interface are performance-critical. Logic within the `tick` method is executed for every instance of the block in every loaded chunk, every single game tick. Inefficient code will have a significant, direct impact on server performance.

## Lifecycle & Ownership
- **Creation:** An object implementing this interface is instantiated when a corresponding block is placed into the world. This process is typically managed by a `BlockFactory` or the `WorldChunk` itself, which then registers the new instance with the chunk's internal list of tickable states.
- **Scope:** The instance persists as long as the block exists in the world and its parent `WorldChunk` is loaded in memory. Its lifecycle is directly bound to the lifecycle of the block entity it represents.
- **Destruction:** The instance is marked for removal when the associated block is destroyed by a player or world event. The `invalidate` method is called to signal this state change, allowing the chunk ticking system to safely deregister and garbage collect the object at the end of the current tick cycle. An instance also becomes eligible for destruction when its parent chunk is unloaded from memory.

## Internal State & Concurrency
- **State:** This interface is stateless by definition. However, concrete implementations are expected to be highly stateful, managing properties like burn time, growth progress, or internal inventories. The state is inherently mutable, as the purpose of the `tick` method is to evolve this state over time.
- **Thread Safety:** This interface is **not** thread-safe, and implementations should not be assumed to be. The server's chunk processing engine may execute `tick` methods for different chunks in parallel on multiple threads. To prevent race conditions and ensure world consistency, all world-modifying operations (e.g., changing a block type, spawning an entity) **must** be deferred through the provided `CommandBuffer`. Direct modification of the `WorldChunk` or other shared state from within the `tick` method is a severe violation that will lead to world corruption and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| tick(float, int, ArchetypeChunk, Store, CommandBuffer) | void | O(N) | Executes one tick of simulation logic for the block. Complexity depends on implementation. |
| getPosition() | Vector3i | O(1) | Returns the world-space position of the block. |
| getBlockPosition() | Vector3i | O(1) | Returns the local, within-chunk position of the block. |
| getChunk() | WorldChunk | O(1) | Returns a reference to the parent chunk. May be null if the state is detached. |
| invalidate() | void | O(1) | Marks this state as invalid, signaling for its removal from the simulation loop. |

## Integration Patterns

### Standard Usage
Implementations of TickableBlockState are not meant to be invoked directly. The server's chunk ticking system queries for these components and executes their `tick` method as part of the main game loop. A typical implementation will read world state via the `Store` and schedule changes via the `CommandBuffer`.

```java
// Example implementation for a hypothetical growing plant
public class PlantBlockState implements TickableBlockState {
    private float growth;
    // ... other fields and constructor

    @Override
    public void tick(float deltaTime, int tickNumber, ArchetypeChunk<ChunkStore> chunk, Store<ChunkStore> store, CommandBuffer<ChunkStore> commands) {
        if (this.getChunk() == null) {
            return; // Do not tick if detached
        }

        // Read light level from the world via the Store
        int lightLevel = store.getSunlight(this.getPosition());

        if (lightLevel > 8) {
            this.growth += 0.1f * deltaTime;
        }

        if (this.growth >= 1.0f) {
            // Use the CommandBuffer to change the block to its next stage
            commands.setBlock(this.getPosition(), Block.MATURE_PLANT);
        }
    }

    // ... other interface methods
}
```

### Anti-Patterns (Do NOT do this)
- **Direct World Modification:** Never modify the `WorldChunk` or other world state directly from inside the `tick` method. Always use the provided `CommandBuffer` to ensure thread safety and transactional consistency.
- **Long-Term Caching:** Do not cache and hold references to other `TickableBlockState` instances or entities. They may be invalidated at any time. Re-query for necessary data from the `Store` each tick.
- **Ignoring Invalidation:** Failure to check if `getChunk()` is null or to handle the `invalidate` call can lead to memory leaks and attempts to operate on stale, detached block states, causing crashes.
- **Manual Invocation:** Never call the `tick` method manually. It must only be driven by the server's chunk simulation engine to guarantee correct context and execution order.

## Data Pipeline
The TickableBlockState is not part of a data pipeline but rather a participant in the server's simulation loop.

> Flow:
> Server Tick Begins -> Chunk Simulation System -> **Iterates active TickableBlockState instances** -> `tick()` method is invoked -> Logic executes, enqueuing changes in `CommandBuffer` -> Server Tick Ends -> `CommandBuffer` is flushed, atomically applying all changes to the world state.

