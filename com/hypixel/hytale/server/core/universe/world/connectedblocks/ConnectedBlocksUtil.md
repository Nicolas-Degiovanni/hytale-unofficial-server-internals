---
description: Architectural reference for ConnectedBlocksUtil
---

# ConnectedBlocksUtil

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks
**Type:** Utility

## Definition
```java
// Signature
public class ConnectedBlocksUtil {
```

## Architecture & Concepts
The ConnectedBlocksUtil is a static utility class that serves as the central processing system for blocks that change their appearance based on their neighbors. This includes blocks like fences, walls, and glass panes, a feature often referred to as "autotiling" or "connected textures".

Its primary function is to evaluate the local 3x3x3 neighborhood of a modified block and determine the correct block state (model and rotation) for both the initial block and its affected neighbors. This process is not a single operation but a cascading update that propagates outwards from an initial change.

The core algorithm operates in two phases:
1.  **Initial Placement Evaluation:** When a block is placed, the system first calls `getDesiredConnectedBlockType`. This method consults the block's associated **ConnectedBlockRuleSet** asset to determine the correct block type and rotation for its specific location and orientation relative to its new neighbors.
2.  **Cascading Neighbor Update:** After the initial block is set, the system triggers `updateNeighborsWithDepth`. This initiates a breadth-first search (BFS) starting from the origin point. It recursively notifies adjacent blocks of the change, causing them to re-evaluate their own state. To prevent performance degradation from large-scale chain reactions, this update propagation is hard-limited to a depth of **MAX_UPDATE_DEPTH** (3 blocks).

This utility is the bridge between raw block placement actions and the complex, rule-based visual logic that makes the world feel cohesive and detailed. It directly reads from and writes to the core world data structures, specifically **WorldChunk** and **BlockSection**.

## Lifecycle & Ownership
-   **Creation:** As a static utility class, ConnectedBlocksUtil is never instantiated. Its methods are available globally once the class is loaded by the Java Virtual Machine.
-   **Scope:** Application-wide. It is a stateless service that can be invoked from any server-side logic that modifies the world.
-   **Destruction:** The class is unloaded when the server application terminates.

## Internal State & Concurrency
-   **State:** This class is entirely stateless. It maintains no internal fields or caches between method invocations. All necessary state, such as the **World** reference and block coordinates, is passed in as method arguments. All internal data structures, like queues or sets for the BFS algorithm, are created and destroyed within a single method call.

-   **Thread Safety:** **WARNING:** This utility is **NOT THREAD-SAFE**. Its methods perform direct, unsynchronized reads and writes on **World** and **WorldChunk** objects, which are not designed for concurrent access. All invocations of methods within this class **MUST** be performed on the main server thread, which is responsible for the world tick. Calling this utility from an asynchronous task or a different thread will lead to world data corruption, race conditions, and server instability.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| setConnectedBlockAndNotifyNeighbors(...) | static void | O(N) | The primary entry point. Sets a block and triggers a cascading update of its neighbors. N is the number of affected blocks within the update depth limit. |
| notifyNeighborsAndCollectChanges(...) | static void | O(1) | Scans the 26 adjacent blocks around an origin, evaluates their required state, and collects necessary changes into the output map. |
| getDesiredConnectedBlockType(...) | static Optional | O(1) | Evaluates a single block against its rule set to determine the optimal block type and rotation. The complexity is constant as it depends on a fixed set of rules. |

## Integration Patterns

### Standard Usage
The utility should be called immediately after a block has been placed in the world by a player or a world generation system. It is a critical step in the block placement pipeline.

```java
// Example from a hypothetical block placement handler
void onBlockPlaced(Player player, Vector3i position, int blockId, RotationTuple rotation) {
    World world = player.getWorld();
    WorldChunk chunk = world.getChunkAtBlock(position.x, position.z);
    BlockChunk blockChunk = chunk.getBlockChunk();

    // The core integration: set the block and immediately notify neighbors
    ConnectedBlocksUtil.setConnectedBlockAndNotifyNeighbors(
        blockId,
        rotation,
        player.getPlacementNormal(), // The surface normal vector
        position,
        chunk,
        blockChunk
    );
}
```

### Anti-Patterns (Do NOT do this)
-   **Asynchronous Invocation:** Never call any method in this class from a separate thread or an asynchronous task without first queuing the operation to be executed on the main world thread. This is the most critical anti-pattern and will corrupt world data.
-   **Manual Iteration:** Do not manually iterate neighbors and call `getDesiredConnectedBlockType` for each one. The `setConnectedBlockAndNotifyNeighbors` method is designed to handle the entire cascading update process atomically and efficiently.
-   **Ignoring the Result:** Placing a block and failing to call this utility will result in blocks that do not connect properly to their neighbors, appearing visually incorrect (e.g., a fence post that does not connect to an adjacent fence).

## Data Pipeline
The flow of data for a single connected block update is a read-evaluate-write-propagate cycle.

> Flow:
> Block Placement Event -> **ConnectedBlocksUtil.setConnectedBlockAndNotifyNeighbors** -> World State Read (Neighboring Blocks) -> Asset Data Read (ConnectedBlockRuleSet) -> State Evaluation -> World State Write (Set Initial Block) -> Begin Cascading Update (BFS) -> Repeat Read/Evaluate/Write for Neighbors -> End of Update

