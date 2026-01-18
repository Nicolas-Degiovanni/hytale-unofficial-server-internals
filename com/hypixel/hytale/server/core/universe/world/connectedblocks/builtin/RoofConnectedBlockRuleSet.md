---
description: Architectural reference for RoofConnectedBlockRuleSet
---

# RoofConnectedBlockRuleSet

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks.builtin
**Type:** Data Model / Transient

## Definition
```java
// Signature
public class RoofConnectedBlockRuleSet extends ConnectedBlockRuleSet implements StairLikeConnectedBlockRuleSet {
```

## Architecture & Concepts
The RoofConnectedBlockRuleSet is a specialized implementation of the connected blocks strategy, designed to automate the creation of complex and aesthetically pleasing roof structures. It functions as a high-level rule engine that determines the appropriate block model and orientation for a roof block based on its spatial relationship with adjacent blocks.

Architecturally, this class acts as a composite object. It does not contain the low-level connection logic itself; instead, it composes and orchestrates two instances of StairConnectedBlockRuleSet: one for **regular** (solidly supported) roof pieces and one for **hollow** (unsupported) pieces. This composition allows it to handle different visual styles for eaves versus the main roof body.

Its primary responsibility is to analyze the local block environment and decide:
1.  Whether the block should be a standard stair shape, a corner, an inverted corner, or a special "topper" piece.
2.  Whether to use the regular or hollow model set, determined by inspecting the block directly beneath it for structural support.

The connection logic relies heavily on a shared **materialName**, which acts as a grouping key. This ensures that, for example, an "oak" roof corner will only connect to other "oak" roof pieces, preventing unintended connections between different material types.

### Lifecycle & Ownership
-   **Creation:** Instances are not created programmatically via a constructor. They are deserialized and instantiated by the server's asset loading system using the static **CODEC** field. This process occurs during server bootstrap when block definition files are parsed.
-   **Scope:** The lifecycle of a RoofConnectedBlockRuleSet instance is tied to the lifecycle of the BlockType asset it defines. It persists in memory for the entire server session as part of the global asset registry.
-   **Destruction:** The object is eligible for garbage collection only when the server's asset registry is cleared, typically during a full server shutdown or a complete asset reload.

## Internal State & Concurrency
-   **State:** The object's state is effectively **immutable** after its initial creation and hydration by the asset loader. Fields such as *regular*, *hollow*, *topper*, *materialName*, and *width* are configured once and are not intended for runtime modification. The method *updateCachedBlockTypes* resolves string identifiers to object references post-deserialization, but this is considered part of the final initialization phase, not a runtime state change.

-   **Thread Safety:** This class is **conditionally thread-safe**. It contains no mutable internal state and performs no locking. Its methods are pure functions with respect to its own state. However, methods like *getConnectedBlockType* perform read operations on the World object. Therefore, safe concurrent execution is entirely dependent on the thread safety guarantees of the World and WorldChunk objects passed into it. It is safe to call from any thread as long as the underlying world data structures are synchronized for reads.

## API Surface
The public API is minimal and primarily serves the internal connected blocks system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getConnectedBlockType(world, coord, ...) | Optional<ConnectedBlockResult> | O(k) | The core evaluation function. Determines the correct block ID and rotation by querying a constant number of neighboring blocks. Complexity is constant relative to world size but involves multiple chunk lookups. |
| getStairType(blockId) | StairType | O(1) | Resolves a given block ID to its corresponding stair shape (e.g., STRAIGHT, CORNER_LEFT). Delegates to the internal regular and hollow rule sets. |
| getMaterialName() | String | O(1) | Returns the material identifier used for connection grouping. |
| toPacket(...) | ConnectedBlockRuleSet | O(1) | Serializes the rule set into a network packet for client-side prediction and rendering. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use in gameplay logic. It is exclusively configured within block asset files and invoked automatically by the world engine when a block is placed or a neighboring block is updated. The engine's connected blocks processor retrieves the appropriate rule set for a given block and executes it.

A simplified view of the engine's internal usage:
```java
// Engine-level code (conceptual)
void onBlockUpdate(World world, Vector3i position) {
    BlockType blockType = world.getBlockTypeAt(position);
    ConnectedBlockRuleSet ruleSet = blockType.getConnectedBlockRuleSet();

    if (ruleSet instanceof RoofConnectedBlockRuleSet) {
        Optional<ConnectedBlockResult> result = ruleSet.getConnectedBlockType(world, position, ...);
        result.ifPresent(res -> world.setBlock(position, res.getBlockId(), res.getRotation()));
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new RoofConnectedBlockRuleSet()`. The object will be in an invalid state without its internal fields being populated by the asset loader's CODEC. This will lead to NullPointerExceptions.
-   **State Mutation:** Do not attempt to modify the object's fields after it has been loaded. This would violate the integrity of the asset system and cause unpredictable behavior across the server.
-   **Manual Invocation:** Avoid calling *getConnectedBlockType* from general gameplay code. This logic is owned by the world's block update system and should not be triggered manually.

## Data Pipeline
The RoofConnectedBlockRuleSet is a processing node in the server's block update pipeline. It does not handle network data directly but consumes world state to produce a new world state.

> Flow:
> World Event (Block Placement/Update) -> Connected Blocks Processor -> **RoofConnectedBlockRuleSet.getConnectedBlockType** -> World State Read (Neighboring Blocks) -> Logic Evaluation -> ConnectedBlockResult (New Block ID/Rotation) -> World State Write (Update Block in Chunk)

