---
description: Architectural reference for CustomTemplateConnectedBlockRuleSet
---

# CustomTemplateConnectedBlockRuleSet

**Package:** com.hypixel.hytale.server.core.universe.world.connectedblocks
**Type:** Configuration Model

## Definition
```java
// Signature
public class CustomTemplateConnectedBlockRuleSet extends ConnectedBlockRuleSet {
```

## Architecture & Concepts
The CustomTemplateConnectedBlockRuleSet is a data-driven configuration object that defines how a family of blocks should visually connect to one another. It does not implement the connection logic itself; instead, it acts as a declarative bridge between a block's properties and a reusable, logic-heavy template asset.

Its primary architectural purpose is to decouple specific block assets (like oak planks) from the generalized connection behavior (like how all wooden plank types should form corners, edges, and centers). This is achieved through a delegation pattern:

1.  **Rule Set Definition:** This class holds a map, shapeNameToBlockPatternMap, which links abstract shape names (e.g., "corner_inner", "pillar_top") to concrete BlockPattern definitions. A BlockPattern specifies the required arrangement of neighboring blocks to satisfy the shape.
2.  **Template Reference:** It holds a string reference, shapeAssetId, to a CustomConnectedBlockTemplateAsset. This referenced asset contains the core algorithm for evaluating neighboring blocks against a set of shapes and determining the correct final block state.
3.  **Runtime Evaluation:** When the game engine needs to determine the correct visual for a block, it invokes getConnectedBlockType on this ruleset. The ruleset then retrieves the referenced template asset and delegates the evaluation call to it, passing along its own shape-to-pattern mappings and the current world state.

This design allows many different RuleSet assets (e.g., "OakPlankRules", "SprucePlankRules") to share a single, complex TemplateAsset (e.g., "StandardPlankConnectionLogic"), promoting high reusability and simplifying the definition of new connected block types.

## Lifecycle & Ownership
-   **Creation:** Instances are exclusively created by the Hytale Asset Pipeline during server initialization or asset hot-reloading. The static CODEC field defines the deserialization contract from asset source files (e.g., JSON). Direct instantiation via code is a critical anti-pattern.
-   **Scope:** The object's lifetime is bound to the server's asset registry. Once loaded, it persists in memory for the entire server session.
-   **Destruction:** The object is marked for garbage collection when the asset registry is cleared, typically during a server shutdown or a full asset reload.

## Internal State & Concurrency
-   **State:** The class is **conditionally mutable**. Upon deserialization, its state is incomplete. The asset system subsequently calls the updateCachedBlockTypes method, which populates the internal shapesPerBlockType cache. After this post-load initialization step, the object's state should be considered immutable for the remainder of its lifecycle.
-   **Thread Safety:** This class is **not thread-safe** and is designed for single-threaded access. The internal maps are non-concurrent. All initialization, including the population of its cache via updateCachedBlockTypes, must occur synchronously within the asset loading thread. Subsequent reads from the main game thread are safe, provided no external mutations occur.

## API Surface
The public API is primarily for internal engine use, facilitating the connection between the asset system and the world simulation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getConnectedBlockType(...) | Optional | O(N) | Primary evaluation entry point. Delegates to the referenced template asset. Complexity is dependent on the template's logic. |
| updateCachedBlockTypes(...) | void | O(S*B) | **Lifecycle Hook.** Populates an internal cache mapping block types to shape names. S is the number of shapes, B is the number of blocks per pattern. |
| getShapeTemplateAsset() | CustomConnectedBlockTemplateAsset | O(1) | Retrieves the referenced template asset from the global asset map. Returns null if the asset is not found. |
| getShapesForBlockType(int) | Set | O(1) | Performs a fast, cached lookup to find all shape names a given block type can participate in. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay systems. It is automatically invoked by the world's block update logic. The following example is a conceptual representation of how the engine interacts with it.

```java
// Conceptual engine code during a block update at 'targetPos'
BlockType placedBlockType = world.getBlockTypeAt(targetPos);
ConnectedBlockRuleSet rules = placedBlockType.getConnectedBlockRuleSet();

// The engine checks if the block has rules and they are of the correct type
if (rules instanceof CustomTemplateConnectedBlockRuleSet) {
    // The engine invokes the evaluation, passing in world context
    Optional<ConnectedBlockResult> result = rules.getConnectedBlockType(
        world,
        targetPos,
        placedBlockType,
        rotation,
        placementNormal,
        true
    );

    // If a result is found, the engine applies the new block state
    result.ifPresent(blockResult -> world.setBlockState(targetPos, blockResult.getBlockState()));
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never use `new CustomTemplateConnectedBlockRuleSet()`. The object will be uninitialized, lack a valid asset reference, and its internal cache will be empty, leading to NullPointerExceptions and incorrect game behavior.
-   **Manual Cache Management:** Do not call `updateCachedBlockTypes` manually. This method is a lifecycle hook managed exclusively by the asset loading system. Calling it at the wrong time or with incomplete data can corrupt the object's state.
-   **State Mutation:** Do not modify the `shapeNameToBlockPatternMap` after the object has been loaded. This will break the internal cache and lead to unpredictable block connection behavior.

## Data Pipeline
The CustomTemplateConnectedBlockRuleSet is a key component in the block update pipeline. It acts as a configuration provider for the connection logic.

> Flow:
> Block Update Event (Place/Break) -> World Engine -> Retrieve BlockType Asset -> Get **CustomTemplateConnectedBlockRuleSet** -> Get CustomConnectedBlockTemplateAsset -> Evaluate Neighbors -> Return ConnectedBlockResult -> World Engine Applies New Block State

