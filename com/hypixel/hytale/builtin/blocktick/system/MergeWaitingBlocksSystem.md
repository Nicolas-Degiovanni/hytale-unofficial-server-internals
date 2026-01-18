---
description: Architectural reference for MergeWaitingBlocksSystem
---

# MergeWaitingBlocksSystem

**Package:** com.hypixel.hytale.builtin.blocktick.system
**Type:** System Component

## Definition
```java
// Signature
public class MergeWaitingBlocksSystem extends RefSystem<ChunkStore> {
```

## Architecture & Concepts
The MergeWaitingBlocksSystem is a reactive, event-driven component within the server-side Entity Component System (ECS) framework. Its sole responsibility is to ensure the continuity and correctness of the block ticking scheduler across chunk boundaries.

In Hytale's world architecture, each WorldChunk is an independent entity. When a block in one chunk needs to schedule an update (a "tick") for a block in an adjacent chunk, a potential consistency issue arises, especially if the adjacent chunk is not yet loaded. This system resolves this by acting as a "neighbor notification" service.

When a new WorldChunk is loaded and added to the world, this system is automatically triggered. It does not process the new chunk itself. Instead, it identifies the four cardinal neighbors of the new chunk and instructs them to re-evaluate their pending block ticks. This allows a neighboring chunk to "merge" any scheduled ticks that were waiting for the new chunk to become available. This design elegantly handles edge-case updates and ensures the block update mechanism is robust and spatially aware.

### Lifecycle & Ownership
- **Creation:** Instantiated by the central ECS System Manager during the server's world initialization sequence. The system is then registered to receive notifications for specific entity and component lifecycle events.
- **Scope:** The system's lifecycle is bound to the world session. It persists as a stateless, long-lived object for the entire time the world is active.
- **Destruction:** De-registered and marked for garbage collection when the world is shut down and the ECS System Manager is torn down.

## Internal State & Concurrency
- **State:** This system is **entirely stateless**. It maintains no instance fields and does not cache data between invocations. All required context, such as the ChunkStore and entity references, is provided as method arguments by the ECS framework.
- **Thread Safety:** The system is inherently thread-safe due to its stateless design. However, it operates under the assumption that the ECS framework invokes its methods, such as onEntityAdded, from a single, main world-update thread. The static utility method, mergeTickingBlocks, is also thread-safe, but callers are responsible for ensuring the provided ChunkStore is accessed in a thread-safe manner. Direct concurrent modification of the ChunkStore while this system is operating is not supported and will lead to undefined behavior.

## API Surface
The primary interaction with this system is through ECS event callbacks. The static utility method is exposed for specialized, low-level world manipulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| mergeTickingBlocks(store, x, z) | static void | O(N) | Retrieves a chunk at coordinates (x, z) and triggers its internal block tick merge process. N is the number of pending block ticks in the target chunk. |

## Integration Patterns

### Standard Usage
This system is not designed for direct invocation. Its logic is triggered automatically by the game engine as a side effect of loading chunks. A developer implicitly uses this system by allowing the standard chunk loading pipeline to execute.

The following example demonstrates the action that *triggers* the system, not how to use the system itself.
```java
// This action, performed by the chunk loading pipeline,
// will cause the ECS framework to invoke MergeWaitingBlocksSystem.
WorldChunk newChunk = createNewChunkAt(x, z);
commandBuffer.addComponent(chunkEntityRef, newChunk);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never create an instance using `new MergeWaitingBlocksSystem()`. The ECS framework is solely responsible for its lifecycle.
- **Manual Callback Invocation:** Do not call the onEntityAdded method directly. It is an event handler managed by the engine and requires a specific state context that cannot be replicated manually.
- **Unsynchronized `mergeTickingBlocks` Calls:** Calling the static mergeTickingBlocks method from an asynchronous thread without proper locking on the world or ChunkStore can corrupt the block tick scheduler, leading to missed or duplicated block updates.

## Data Pipeline
The system operates as a step in a larger, event-driven data flow related to world streaming.

> Flow:
> Chunk Loader Pipeline -> Adds WorldChunk Component to Entity -> ECS Framework Event -> **MergeWaitingBlocksSystem.onEntityAdded** -> Issues 4 calls to `mergeTickingBlocks` on adjacent chunks -> Neighboring BlockChunk instances -> Internal Tick Scheduler is updated.

