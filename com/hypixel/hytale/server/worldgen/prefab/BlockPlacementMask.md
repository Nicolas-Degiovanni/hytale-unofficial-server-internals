---
description: Architectural reference for BlockPlacementMask
---

# BlockPlacementMask

**Package:** com.hypixel.hytale.server.worldgen.prefab
**Type:** Transient

## Definition
```java
// Signature
public class BlockPlacementMask implements BlockMaskCondition {
```

## Architecture & Concepts
The BlockPlacementMask is a specialized, configurable rule engine that governs block replacement logic during server-side world generation. It acts as a predicate, answering a single, critical question for a prefab placer or other world generation feature: "Given the block that currently exists at a target coordinate, am I allowed to replace it with this new block?"

This component implements the BlockMaskCondition interface, signaling its role as a conditional check within the broader world generation pipeline. Its design is based on a highly specific override system:

1.  **Default Behavior:** A global `defaultMask` defines the baseline behavior for all block replacements. The standard implementation, DefaultMask, permits placement only if the target location is air (block ID 0, fluid ID 0).

2.  **Specific Overrides:** A map of `specificMasks` allows for fine-grained control. This map is keyed by a packed long representing the *new* block and fluid ID being placed. If a specific mask exists for the block being placed, its rules are used instead of the default.

3.  **Rule Chaining:** The specific Mask implementation contains an array of IEntry rules. It evaluates these rules in order, using the first one that handles the *current* block at the target location. This creates a chain-of-responsibility pattern, allowing for complex rules like "replace grass and dirt, but not stone, and allow replacing anything else".

This system provides a declarative and data-driven way to define complex placement logic for structures and features, decoupling the placement algorithm from the rules themselves.

## Lifecycle & Ownership
- **Creation:** An instance of BlockPlacementMask is typically instantiated by a higher-level world generation service when it loads a prefab or a set of procedural generation rules. It is created as a plain object and must be configured via the `set` method.

- **Scope:** The object's lifetime is transient and is strictly bound to the scope of a single, discrete world generation operation, such as the placement of one prefab.

- **Destruction:** It holds no external resources and is eligible for garbage collection as soon as the world generation operation that owns it completes.

## Internal State & Concurrency
- **State:** The BlockPlacementMask is a mutable state container. Its entire logical state is defined by the `defaultMask` and `specificMasks` fields. These are typically loaded from asset definitions and injected post-construction.

- **Thread Safety:** **This class is not thread-safe.** The internal fields are accessed without any synchronization primitives. The `set` method is not atomic and will cause severe race conditions if called while the `eval` method is being used by another thread.

    **WARNING:** An instance of BlockPlacementMask must be fully configured on a single thread *before* being passed to multi-threaded world generation workers. Once passed to workers, it must be treated as immutable for the duration of the generation task.

## API Surface
The public contract is minimal, focusing on configuration and evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| set(IMask, Long2ObjectMap) | void | O(1) | Configures the mask. Replaces all existing rules. Not thread-safe. |
| eval(int, int, BlockFluidEntry) | boolean | O(N) | Evaluates the placement condition. Complexity is O(N) where N is the number of entries in the most complex specific mask. Map lookup is near O(1). |

## Integration Patterns

### Standard Usage
The BlockPlacementMask is used as a predicate function within a world generation loop. The generator provides the context (the block to be placed and the block that already exists), and the mask returns a boolean decision.

```java
// A world generation service prepares a mask based on prefab data
BlockPlacementMask placementRules = new BlockPlacementMask();
// ... logic to load default and specific masks from assets ...
placementRules.set(defaultMask, specificMasks);

// During chunk generation, the placer uses the mask
for (BlockPlacementOperation op : prefab.getOperations()) {
    BlockFluidEntry existingBlock = world.getBlockAt(op.targetPosition);
    
    // The mask determines if the operation should proceed
    boolean canPlace = placementRules.eval(
        existingBlock.blockId(),
        existingBlock.fluidId(),
        op.newBlockEntry()
    );

    if (canPlace) {
        world.setBlockAt(op.targetPosition, op.newBlockEntry());
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never call the `set` method on an instance that is actively being used by world generation workers. This will lead to non-deterministic behavior and potential crashes.

- **Use Before Configuration:** Using a default-constructed BlockPlacementMask without calling `set` will result in a NullPointerException when `eval` is invoked, as its internal fields will be null.

## Data Pipeline
The BlockPlacementMask is a decision point in the data flow of block placement. It consumes world state and proposed changes, and produces a simple boolean control signal.

> Flow:
> World Gen Placer → **BlockPlacementMask.eval**(currentBlockState, proposedBlockState) → Boolean Decision → Conditional Block Write Operation

