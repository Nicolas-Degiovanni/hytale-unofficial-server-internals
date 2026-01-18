---
description: Architectural reference for ChunkBlockTickSystem
---

# ChunkBlockTickSystem

**Package:** com.hypixel.hytale.builtin.blocktick.system
**Type:** Utility / Namespace

## Definition
```java
// Signature
public class ChunkBlockTickSystem {
```

## Architecture & Concepts
The ChunkBlockTickSystem is a container for the core server-side logic that processes scheduled block updates within world chunks. It is not a single system but a namespace for two distinct, ordered phases of block ticking, implemented as nested static classes: **PreTick** and **Ticking**. This two-phase approach ensures deterministic execution and prevents race conditions between systems that prepare data and systems that act on it.

This system is a fundamental part of the server's Entity Component System (ECS) framework. In this context, each chunk is treated as an entity, and its data (e.g., BlockChunk, WorldChunk) are its components. The ChunkBlockTickSystem operates on every entity that possesses these chunk components.

-   **Phase 1: PreTick**
    The PreTick system runs first. Its sole responsibility is to prepare each BlockChunk for the main ticking phase by calling its internal preTick method. This step may involve updating internal timers, compacting data structures, or identifying which blocks need a tick in the current game loop iteration. It is designed to be highly parallelizable across all loaded chunks.

-   **Phase 2: Ticking**
    The Ticking system executes *after* the PreTick system is complete for all chunks. This dependency is explicitly declared to the ECS scheduler. This phase iterates through the blocks identified for updates during the PreTick phase. For each block, it retrieves the corresponding TickProcedure from the BlockTickPlugin and executes it. This is where the actual game logic, such as crop growth or block decay, is performed.

The system acts as a bridge between the low-level chunk data and high-level, asset-driven game logic defined in TickProcedure assets.

### Lifecycle & Ownership
-   **Creation:** The inner systems, PreTick and Ticking, are not instantiated manually. They are discovered and instantiated by the ECS scheduler during world initialization. The outer ChunkBlockTickSystem class is never instantiated.
-   **Scope:** The system instances persist for the entire lifetime of a game world. They are invoked by the ECS scheduler on every server tick as part of the main game loop.
-   **Destruction:** The systems are destroyed and garbage collected when the world is unloaded or the server shuts down. Their lifecycle is entirely managed by the engine.

## Internal State & Concurrency
-   **State:** The ChunkBlockTickSystem and its inner system classes are **stateless**. They do not store any data between ticks. All state is read from and written to components (BlockChunk, WorldChunk) or global resources (WorldTimeResource). This stateless design is critical for enabling parallelism and ensuring predictable behavior.

-   **Thread Safety:** The system is designed for high-concurrency environments.
    -   The PreTick phase explicitly enables parallel execution via the isParallel method, allowing the engine to process many chunks simultaneously across multiple threads.
    -   The Ticking phase, while not explicitly parallel in the provided code, operates on a per-chunk basis. The underlying ECS framework guarantees that operations on a single chunk's components are thread-safe. All mutations are funneled through the CommandBuffer, which serializes changes and prevents data corruption.

    **WARNING:** Custom TickProcedure implementations must be thread-aware. While the system guarantees safe access to the provided chunk and world arguments, any access to external, shared state from within a TickProcedure must be properly synchronized.

## API Surface
The ChunkBlockTickSystem itself exposes no public API. Interaction with its functionality is managed by the ECS engine through the EntityTickingSystem interface implemented by its inner classes. The primary contract is with the game loop, not with user code.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| PreTick.tick(...) | void | O(N) | Engine-invoked entry point for the pre-tick phase. N is the number of blocks in the chunk. |
| Ticking.tick(...) | void | O(K) | Engine-invoked entry point for the main ticking phase. K is the number of blocks scheduled to tick. |

## Integration Patterns

### Standard Usage
A developer does not interact with this system directly. Instead, they provide block-specific logic that this system will execute. The standard pattern is to define a TickProcedure in an asset file and register it via the BlockTickPlugin.

```java
// This code does NOT call ChunkBlockTickSystem.
// It defines the logic that the system will find and execute.

// In your plugin's entry point:
public class MyModPlugin extends JavaPlugin {
    @Override
    public void onEnable() {
        // Register a procedure for a custom block ID
        BlockTickPlugin.get().registerTickProcedure(MY_CUSTOM_BLOCK_ID, this::handleMyBlockTick);
    }

    // The logic to be executed by the ChunkBlockTickSystem.Ticking phase
    public BlockTickStrategy handleMyBlockTick(World world, WorldChunk chunk, int x, int y, int z, int blockId) {
        // Implement game logic, e.g., grow a plant
        if (Math.random() > 0.5) {
            world.setBlock(x, y + 1, z, Block.PUMPKIN);
        }
        // Reschedule the block for another tick later
        return BlockTickStrategy.RANDOM;
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ChunkBlockTickSystem.Ticking()`. The ECS engine is solely responsible for the lifecycle of these systems. Manual instantiation will result in a non-functional object that is not registered with the game loop.
-   **Manual Invocation:** Never call the `tick` method directly. Doing so bypasses the ECS scheduler, dependency ordering, and the CommandBuffer. This will lead to severe concurrency issues, inconsistent world state, and crashes.
-   **Assuming Serial Execution:** Do not write a TickProcedure that depends on the execution order of ticks for blocks in different chunks within the same game tick. The system may process chunks in parallel, and no such ordering is guaranteed.

## Data Pipeline
The system processes data as part of the main server game loop. The flow is orchestrated by the ECS scheduler.

> Flow:
> Game Loop Tick -> ECS Scheduler -> **ChunkBlockTickSystem.PreTick** -> Updates internal state of BlockChunk components -> ECS Scheduler (enforces dependency) -> **ChunkBlockTickSystem.Ticking** -> Reads scheduled ticks from BlockChunk -> Executes registered TickProcedure -> TickProcedure logic modifies World state (via World API)

