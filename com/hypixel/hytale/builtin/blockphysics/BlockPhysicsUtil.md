---
description: Architectural reference for BlockPhysicsUtil
---

# BlockPhysicsUtil

**Package:** com.hypixel.hytale.builtin.blockphysics
**Type:** Utility

## Definition
```java
// Signature
public class BlockPhysicsUtil {
```

## Architecture & Concepts
BlockPhysicsUtil is a stateless, static utility class that serves as the core calculation engine for Hytale's block support and stability system. It is the central authority for determining whether a block can exist at its current position based on its neighbors and its own intrinsic properties defined in its asset configuration.

This class does not participate directly in the main game loop. Instead, it is invoked by higher-level systems, primarily the BlockPhysicsSystems, which manage the queue of blocks requiring physics updates. BlockPhysicsUtil operates as a pure function in principle: given a block's context (position, type, surrounding world data), it computes a deterministic result. Its primary responsibilities include:

-   Evaluating the support requirements for a given BlockType against its adjacent blocks.
-   Handling complex physics for multi-block structures by decomposing them into their constituent parts.
-   Calculating "support distance," a mechanism for propagating stability through chains of blocks.
-   Triggering block destruction via the BlockHarvestUtils when support requirements are not met.

It acts as the logical bridge between block asset definitions (the rules) and the in-game world state (the data).

## Lifecycle & Ownership
As a static utility class, BlockPhysicsUtil has no object instances and therefore no traditional lifecycle.

-   **Creation:** The class is loaded by the Java Virtual Machine's classloader when first referenced. It is never instantiated.
-   **Scope:** Its static methods are available globally for the entire duration of the server's runtime.
-   **Destruction:** The class is unloaded when the JVM shuts down. There is no manual cleanup or resource management associated with it.

## Internal State & Concurrency
-   **State:** BlockPhysicsUtil is entirely stateless. It contains no member variables and all calculations are performed exclusively on data passed in as method arguments. All constants are static final.

-   **Thread Safety:** This class is inherently thread-safe. Since it has no internal state, multiple threads can invoke its static methods concurrently without risk of interference or race conditions within the utility itself.

    **WARNING:** While the class is thread-safe, the data structures passed to it (such as ChunkStore, BlockSection, and ComponentAccessor) may not be. The caller is responsible for ensuring that any access to shared world state is properly synchronized. This utility performs read-heavy operations on world data and may dispatch write commands to a command buffer, which must be processed in a thread-safe manner by the calling system.

## API Surface
The public API consists of static methods for calculating and applying block physics.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| applyBlockPhysics(...) | Result | O(N\*M) | The primary entry point. Tests a block's stability and applies the outcome, potentially destroying the block. N is the number of sub-blocks in a complex hitbox, M is the number of faces checked. |
| testBlockPhysics(...) | int | O(M) | The core logic routine. Calculates a block's support status and returns an integer code representing the result (e.g., SATISFIES_SUPPORT, DOESNT_SATISFY, WAITING_CHUNK). |
| doesSatisfyRequirements(...) | boolean | O(1) | A low-level helper that checks if a neighboring block satisfies a specific support requirement rule. |
| doesMatchFaceType(...) | boolean | O(K) | A helper to verify if a specific face of a block provides the required type of support. K is the number of support types on a face. |

## Integration Patterns

### Standard Usage
This utility should be invoked by a world-update system that manages a queue of blocks needing physics checks. The caller must prepare a CachedAccessor to provide efficient, localized chunk data access.

```java
// Standard invocation from a block update system
BlockPhysicsSystems.CachedAccessor accessor = ...;
Ref<ChunkStore> chunkRef = ...;
ComponentAccessor<EntityStore> commands = ...;

// For a specific block at (x, y, z)
BlockPhysicsUtil.Result result = BlockPhysicsUtil.applyBlockPhysics(
    commands,
    chunkRef,
    accessor,
    blockSection,
    blockPhysicsComponent,
    fluidSection,
    x, y, z,
    blockType,
    rotation,
    filler
);

// The calling system must handle cases where a chunk is not yet loaded
if (result == BlockPhysicsUtil.Result.WAITING_CHUNK) {
    // Re-queue the block update for a later tick
}
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring WAITING_CHUNK:** The WAITING_CHUNK result is a critical signal that the physics calculation is incomplete because a neighboring chunk is not loaded. Failure to handle this result by re-queueing the update will cause inconsistent physics behavior at chunk boundaries, such as floating blocks that should have fallen.
-   **Excessive Polling:** Do not call applyBlockPhysics for the same block repeatedly in a tight loop. Physics updates should be event-driven (e.g., when a neighboring block changes) and managed by a queueing system to avoid performance degradation.
-   **Bypassing applyBlockPhysics:** While testBlockPhysics is public, it should not be used to make final decisions about a block's fate. The applyBlockPhysics method contains crucial logic for handling multi-block structures and correctly applying consequences like block destruction or support distance updates. Use the higher-level method unless you are implementing a custom physics simulation.

## Data Pipeline
BlockPhysicsUtil processes data from multiple sources to determine a block's validity. The flow is a read-evaluate-act cycle.

> Flow:
> Block Update Event -> BlockPhysicsSystems -> **BlockPhysicsUtil.applyBlockPhysics**
>
> 1.  **Data Aggregation:** The calling system provides world state accessors (ChunkStore, BlockSection, FluidSection) and asset data (BlockType).
> 2.  **Hitbox Decomposition:** For blocks with complex hitboxes, the utility identifies all unit block spaces occupied by the model.
> 3.  **Iterative Testing:** The utility calls **testBlockPhysics** for each constituent block space.
> 4.  **Neighbor Analysis:** **testBlockPhysics** retrieves the BlockType's support requirements (a map of BlockFace to RequiredBlockFaceSupport).
> 5.  **Requirement Evaluation:** It iterates through all required faces, checking adjacent blocks. For each neighbor, **doesSatisfyRequirements** is called to validate against rules (e.g., BlockSet, specific BlockType, tags, fluid presence).
> 6.  **Result Aggregation:** Results from all constituent spaces are combined based on the BlockType's requirement (Any or All).
> 7.  **Action Dispatch:** If the final result is INVALID, a command to destroy the block is created using BlockHarvestUtils and submitted to the provided command buffer. If VALID, the block's support distance value may be updated.
>
> Output: A Result enum is returned, and a block destruction command may be enqueued.

