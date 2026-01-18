---
description: Architectural reference for StairConnectedBlockRuleSet
---

# StairConnectedBlockRuleSet

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks.builtin
**Type:** Transient

## Definition
```java
// Signature
public class StairConnectedBlockRuleSet extends ConnectedBlockRuleSet implements StairLikeConnectedBlockRuleSet {
```

## Architecture & Concepts

The StairConnectedBlockRuleSet is a specialized implementation of the ConnectedBlockRuleSet system, designed exclusively to manage the geometric connectivity of stair blocks within the game world. It is not a live entity within the world simulation but rather a static data and logic container that defines *how* a specific type of stair should behave when placed near other compatible stairs.

This class acts as the brain for complex stair construction, enabling the automatic transformation of a simple stair block into inner corners, outer corners, or straight sections based on its neighbors. Its core responsibility is to inspect the blocks adjacent to a given stair and determine the correct BlockType and rotation that should be displayed, creating seamless and aesthetically pleasing structures.

The central mechanism for determining connectivity is the **MaterialName**. Stairs will only attempt to connect to other stair blocks that share the identical MaterialName, preventing, for example, wood stairs from connecting to stone stairs.

This class is designed to be deserialized directly from asset definition files using its static **CODEC** field. This pattern indicates it is a data-driven component, where game designers can define new stair behaviors in data files without changing engine code.

## Lifecycle & Ownership

-   **Creation:** Instances are not created manually via the *new* keyword. They are instantiated by the Hytale asset loading system during server bootstrap. The static CODEC field is responsible for deserializing the rule set from a block's asset definition file.
-   **Scope:** An instance of StairConnectedBlockRuleSet is owned by the BlockType it is defined for. It is loaded once when assets are processed and persists in memory for the entire server session, effectively acting as a singleton for that specific block type.
-   **Destruction:** The object is eligible for garbage collection only when the server shuts down and the central BlockType asset registries are cleared.

## Internal State & Concurrency

-   **State:** The class maintains two categories of state:
    1.  **Configuration State:** Fields like straight, cornerLeft, and materialName are loaded from asset files. This state is considered immutable after deserialization.
    2.  **Cached State:** The blockIdToStairType and stairTypeToBlockId maps are mutable but populated only once during a post-load initialization step via the updateCachedBlockTypes method. This pre-computes expensive lookups for high-performance access during gameplay.

-   **Thread Safety:** This class is **conditionally thread-safe**.
    -   The initialization method, updateCachedBlockTypes, is **not thread-safe** and must be called from a single-threaded context during the asset loading phase.
    -   After initialization, all public methods that perform calculations, such as getConnectedBlockType, are thread-safe for read-only operations. They do not modify internal state and can be safely called by multiple world-update threads concurrently, provided the passed World object guarantees its own thread safety.

    **WARNING:** Calling any lookup or calculation method before updateCachedBlockTypes has completed will result in a NullPointerException.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| updateCachedBlockTypes(base, assetMap) | void | O(1) | **Critical Initialization.** Populates internal lookup maps. Must be called once after asset loading. |
| getConnectedBlockType(world, coord, ...) | Optional | O(1) | The primary logic entry point. Evaluates neighbors and returns the resulting block state. |
| getStairData(world, coord, material) | static ObjectIntPair | O(1) | A static utility to query a block's stair properties from the world. Returns null if not a stair. |
| getStairType(blockId) | StairType | O(1) | Performs a fast reverse lookup from a numeric block ID to its logical stair shape. |
| getMaterialName() | String | O(1) | Returns the material identifier used for connection logic. |
| toProtocol(assetMap) | protocol.Stair... | O(1) | Serializes the rule set into a network packet for client synchronization. |

## Integration Patterns

### Standard Usage

This class is not intended for direct use by typical gameplay logic developers. It is invoked by lower-level world systems responsible for block updates. The engine retrieves the rule set from a block's type definition and calls it to resolve connectivity.

```java
// Hypothetical engine code for a block update
void onBlockUpdate(World world, Vector3i position) {
    BlockType type = world.getBlockTypeAt(position);
    ConnectedBlockRuleSet rules = type.getConnectedBlockRuleSet();

    if (rules instanceof StairConnectedBlockRuleSet stairRules) {
        // The core logic call to determine the correct block variant
        Optional<ConnectedBlocksUtil.ConnectedBlockResult> result;
        result = stairRules.getConnectedBlockType(world, position, type, /*...other args*/);

        if (result.isPresent()) {
            // Apply the calculated block state back to the world
            world.setBlock(position, result.get().getBlockTypeId(), result.get().getRotation());
        }
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never use `new StairConnectedBlockRuleSet()`. The object is incomplete without being populated by the asset loader via its CODEC. The public constructor exists only for the reflection used by the codec system.
-   **Premature Access:** Do not call getConnectedBlockType or other lookup methods before the engine has called updateCachedBlockTypes. This will lead to fatal runtime exceptions.
-   **State Mutation:** Do not attempt to modify the internal maps or configuration fields after initialization. The class is designed as a read-only data object post-load.

## Data Pipeline

The class is involved in two distinct data flows: an initial configuration pipeline and a runtime logic pipeline.

**Configuration Pipeline (Server Start):**
> Block Asset File (e.g., JSON) -> Hytale Asset Loader -> **StairConnectedBlockRuleSet.CODEC** -> In-Memory `StairConnectedBlockRuleSet` Instance -> `updateCachedBlockTypes()` Call -> Populated & Cached Lookup Maps

**Runtime Logic Pipeline (During Gameplay):**
> Player Places Block -> World Block Update Event -> Engine retrieves `StairConnectedBlockRuleSet` for the block -> **`getConnectedBlockType()`** -> Scans Neighboring Blocks -> Calculates Final Shape & Rotation -> Returns `ConnectedBlockResult` -> Engine updates the block in the `WorldChunk`

