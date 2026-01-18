---
description: Architectural reference for BlockMountAPI
---

# BlockMountAPI

**Package:** com.hypixel.hytale.builtin.mounts
**Type:** Utility

## Definition
```java
// Signature
public final class BlockMountAPI {
```

## Architecture & Concepts

The BlockMountAPI is a high-level, static utility class that serves as the primary entry point for the block mounting system. It acts as a transactional orchestrator, bridging user-level actions (like interacting with a chair) with the underlying Entity Component System (ECS). Its sole purpose is to manage the complex state changes required to attach an entity to a specific mount point on a block, such as a seat or a bed.

This API is not a data container; it is a stateless service. Its core function, mountOnBlock, encapsulates a multi-step process that involves:
1.  **State Validation:** Verifying that the entity is not already mounted and that the target block and chunk are valid and loaded in memory.
2.  **Lazy Entity Promotion:** A key architectural pattern in Hytale. A standard block does not have an associated entity by default. When an entity first mounts a block, this API "promotes" the block by creating a corresponding entity in the ChunkStore. This new block-entity then holds the BlockMountComponent, which tracks all mounted entities. This on-demand creation is a significant performance optimization, avoiding the overhead of an entity for every block in the world.
3.  **Transactional Operations:** All modifications to the ECS are funneled through a CommandBuffer. This ensures that the entire mounting operation is atomic. Either all component changes succeed (e.g., the entity's TransformComponent is updated, and a MountedComponent is added), or none do. This is critical for maintaining world state consistency during server ticks.

The API is designed to be the definitive authority for initiating a block mount, abstracting away the intricate details of chunk lookups, component management, and coordinate space transformations.

## Lifecycle & Ownership

-   **Creation:** As a final class with a private constructor, BlockMountAPI is never instantiated. It exists only as a collection of static methods.
-   **Scope:** The class and its static methods are available for the entire lifetime of the server's Java Virtual Machine.
-   **Destruction:** Not applicable. The class is unloaded when the server shuts down.

## Internal State & Concurrency

-   **State:** The BlockMountAPI is entirely **stateless**. It contains no member variables and all necessary data is passed as arguments to its methods. Each call to mountOnBlock is an independent, self-contained operation that reads from and writes to the world state passed into it.

-   **Thread Safety:** **CRITICAL WARNING:** This API is **not thread-safe** and is fundamentally tied to the server's main game loop. It performs direct, unsynchronized reads and writes on shared world state objects like World, WorldChunk, and various component stores.

    Invoking mountOnBlock from any thread other than the main server thread will lead to catastrophic race conditions, data corruption, and server instability. All calls must be synchronized with the server's tick processing cycle.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| mountOnBlock(entity, commandBuffer, targetBlock, interactPos) | BlockMountResult | O(N) | Atomically mounts an entity onto a target block. Complexity is O(N) where N is the number of mount points on the block asset. The operation is transactional via the CommandBuffer. Returns a sealed result indicating success or a specific failure reason. |

## Integration Patterns

### Standard Usage

The mountOnBlock method should be called from a higher-level game system that processes player actions or AI behaviors within the main server tick. The CommandBuffer must be the one provided for the current tick.

```java
// Example from within a server-side interaction system
void handleInteraction(PlayerEntity player, Block targetBlock, Vector3f interactPos) {
    // The CommandBuffer is provided by the server's tick-processing context
    CommandBuffer<EntityStore> cmdBuffer = world.getTickCommandBuffer();
    Ref<EntityStore> playerRef = player.getReference();

    BlockMountAPI.BlockMountResult result = BlockMountAPI.mountOnBlock(
        playerRef,
        cmdBuffer,
        targetBlock.getPosition(),
        interactPos
    );

    if (result instanceof BlockMountAPI.DidNotMount failure) {
        // Handle the failure case, e.g., send a message to the player
        log.warn("Failed to mount entity: " + failure.name());
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Asynchronous Execution:** Never call mountOnBlock from a network handler thread, a separate worker thread, or within a CompletableFuture that is not synchronized with the main game loop. This will bypass thread-safety mechanisms and corrupt world state.

-   **State Speculation:** Do not check if a block has a BlockMountComponent before calling this API. The API itself handles the lazy creation of this component. Pre-checking is redundant and may introduce race conditions.

-   **Ignoring the Result:** The BlockMountResult provides critical feedback. Ignoring failure cases (like NO_MOUNT_POINT_FOUND) will leave the system in an inconsistent state from the player's perspective, where an interaction appears to do nothing.

## Data Pipeline

The flow of data and control for a successful mounting operation is orchestrated by this API.

> Flow:
> Player Input -> Server Interaction System -> **BlockMountAPI.mountOnBlock()** -> CommandBuffer Queues Changes -> Server Tick Ends -> ECS Processor Applies Commands -> Entity State Updated
>
> **Detailed Breakdown:**
> 1.  The API receives an entity reference and target coordinates.
> 2.  It reads from World and ChunkStore to find the target block's data and asset definition (BlockType).
> 3.  It determines the correct mount point based on the block's rotation and the interaction position.
> 4.  It calculates the final world-space position and rotation for the entity.
> 5.  It **writes** commands to the CommandBuffer:
>     -   Add a **MountedComponent** to the mounting entity.
>     -   Update the entity's **TransformComponent** with the new position/rotation.
>     -   Add or update the **BlockMountComponent** on the block's own entity to register the new occupant.
> 6.  At the end of the tick, the ECS processes the CommandBuffer, atomically applying these changes to the world state.

