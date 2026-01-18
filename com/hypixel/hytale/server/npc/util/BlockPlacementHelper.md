---
description: Architectural reference for BlockPlacementHelper
---

# BlockPlacementHelper

**Package:** com.hypixel.hytale.server.npc.util
**Type:** Utility

## Definition
```java
// Signature
public class BlockPlacementHelper {
```

## Architecture & Concepts
The BlockPlacementHelper is a stateless, server-side utility class designed to centralize the logic for validating block placement within the game world. It serves as a pure function library, encapsulating the set of rules that determine whether a block or a multi-block structure can be placed at a given location.

This class acts as a gatekeeper for programmatic world modification. Systems such as NPC behaviors, quest scripts, or procedural generation routines rely on this helper to ensure their actions comply with the game's physical rules without needing to implement the complex validation logic themselves. Its primary responsibilities are:

1.  **Target Space Validation:** Determining if the block(s) at the destination coordinates can be replaced.
2.  **Support Validation:** Ensuring the block beneath the placement location is solid and can physically support the new block.

By providing a static, side-effect-free interface, it decouples world-mutating logic from world-validation logic, promoting cleaner and more maintainable server code.

### Lifecycle & Ownership
-   **Creation:** This class is never instantiated. As a utility class containing only static methods, its functionality is accessed directly via the class name.
-   **Scope:** The BlockPlacementHelper class is loaded by the JVM and persists for the entire lifetime of the server process.
-   **Destruction:** The class is unloaded when the server application terminates.

## Internal State & Concurrency
-   **State:** The BlockPlacementHelper is completely stateless. It contains no member or static fields, and its methods' outputs depend solely on their inputs. It is therefore immutable by design.
-   **Thread Safety:** This class is inherently thread-safe. Since it holds no state, concurrent calls to its methods from different threads cannot interfere with each other.

    **WARNING:** While the helper itself is thread-safe, the World object passed into its methods is not. Callers are responsible for ensuring that any access to the World or its constituent WorldChunk objects is properly synchronized to prevent race conditions or reads of inconsistent world state. The methods in this class perform read-only operations on the world.

## API Surface
The public API consists of static methods for performing placement checks.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canPlaceUnitBlock(world, placedBlockType, allowEmptyMaterials, x, y, z) | boolean | O(1) | Checks if a single 1x1x1 block can be placed. Critically, this validates both the target cell and the supporting block directly beneath it. |
| canPlaceBlock(world, placedBlockType, rotationIndex, allowEmptyMaterials, x, y, z) | boolean | O(N) | Checks if a potentially multi-block structure can be placed. Delegates to the world to test all cells within the block's bounding box. N is the number of cells. |
| testBlock(placedBlockType, blockType, allowEmptyMaterials) | boolean | O(1) | A low-level predicate used by other methods to determine if a single existing block can be replaced. |
| testSupportingBlock(blockType, rotation, filler) | boolean | O(1) | A low-level predicate that validates if a block is a valid support surface. It must be solid, have a unit hitbox, and not be a filler block. |

## Integration Patterns

### Standard Usage
The intended use is for any server-side system to perform a pre-emptive check before attempting to modify the world. This prevents invalid operations and ensures game rules are respected.

```java
// A server system checks if it's valid to place a stone block for a quest
BlockType stoneBlock = BlockType.getAssetMap().getAsset("hytale:stone");
World world = server.getUniverse().getWorld("my_world");
int targetX = 100, targetY = 64, targetZ = 250;

// Validate the placement before committing the change
boolean canPlace = BlockPlacementHelper.canPlaceUnitBlock(world, stoneBlock, false, targetX, targetY, targetZ);

if (canPlace) {
    world.setBlock(targetX, targetY, targetZ, stoneBlock.getBlockId());
} else {
    // Handle the failure case, e.g., find a new location
}
```

### Anti-Patterns (Do NOT do this)
-   **Ignoring Return Value:** Calling a validation method without using its boolean result to gate a subsequent world modification is a critical error. The entire purpose of the helper is to prevent invalid state changes.
-   **Assuming Success:** Never assume a placement will be valid. Always check first, as the world state can be changed by other actors (players, NPCs, physics) between ticks.
-   **Re-implementing Logic:** Do not write custom placement validation logic elsewhere. This helper is the single source of truth for these rules. Bypassing it can lead to inconsistencies, floating blocks, or corrupted chunk data.

## Data Pipeline
BlockPlacementHelper functions as a validation node in a larger data flow. It does not transform data but rather produces a binary decision that controls the flow.

> Flow:
> Server System Logic (e.g., NPC AI, Quest Script) -> **BlockPlacementHelper.canPlace...()** -> World State Read (Chunk Data) -> Asset Data Read (Block Properties, Hitboxes) -> Boolean Result -> Conditional World Write Operation

