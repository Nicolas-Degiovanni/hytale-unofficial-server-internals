---
description: Architectural reference for BlockMask
---

# BlockMask

**Package:** com.hypixel.hytale.builtin.hytalegenerator
**Type:** Transient

## Definition
```java
// Signature
public class BlockMask {
```

## Architecture & Concepts
The BlockMask is a fundamental control structure within the world generation pipeline. It functions as a declarative rule engine that governs the placement and replacement of blocks during procedural generation. Its primary role is to enforce constraints, ensuring that features like ores, vegetation, and structures are placed in valid locations.

The system is built on two core concepts:

1.  **Placement Veto:** A simple, high-performance blocklist (the *skipped blocks* set) that provides a global veto on placing certain materials. This is typically used to prevent generators from placing solid blocks in air or water.
2.  **Contextual Replacement Rules:** A more sophisticated, ordered list of source-to-destination rules. This allows a generator to define complex behaviors, such as "Iron Ore may replace Stone, but not Dirt". If no specific rule matches a given source block, a *default mask* is consulted as a fallback.

This class is not a singleton or a globally managed service. It is a lightweight, stateful object created and configured on-demand for a specific generation task.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly via its constructor (`new BlockMask()`) by a higher-level generator, such as a `GeneratorPass` or a `FeaturePlacer`. It is not managed by a service locator or dependency injection container.
-   **Scope:** The lifetime of a BlockMask instance is tightly coupled to the operation it governs. It is typically created, configured, used for a single generation pass (e.g., placing all ores in a chunk), and then discarded.
-   **Destruction:** The object becomes eligible for garbage collection as soon as the owning generator pass completes and its reference is dropped. It holds no persistent resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The BlockMask is highly **mutable**. Its internal state consists of several `MaterialSet` collections that are populated after instantiation through its public API. The object is effectively a builder for a set of generation rules.

-   **Thread Safety:** This class is **not thread-safe**. The internal collections, such as `ArrayList`, are not synchronized. Concurrent modification of the rule set (e.g., calling `putBlockMaskEntry`) while another thread is reading it (e.g., calling `canReplace`) will lead to `ConcurrentModificationException` or other undefined behavior.

    **WARNING:** A BlockMask instance must be fully configured on a single thread *before* being used by any world generation workers. Once passed into the generation pipeline, it must be treated as an immutable, read-only object.

## API Surface
The public API is designed for configuration followed by repeated querying.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| canPlace(Material) | boolean | O(1) | Checks if the material is in the `skippedBlocks` set. |
| canReplace(Material, Material) | boolean | O(N) | Checks if a source material can be replaced by a destination material. N is the number of rules. |
| setSkippedBlocks(MaterialSet) | void | O(1) | Sets the complete set of materials that cannot be placed. Overwrites any previous set. |
| putBlockMaskEntry(MaterialSet, MaterialSet) | void | O(1) | Adds a new source-to-destination replacement rule. The order of insertion is significant. |
| setDefaultMask(MaterialSet) | void | O(1) | Sets the fallback rule for `canReplace` when no specific source rule matches. |

## Integration Patterns

### Standard Usage
The canonical pattern is to instantiate, configure, and then use the mask as a read-only predicate within a generation loop.

```java
// 1. A generator creates a new mask for a specific task.
BlockMask oreMask = new BlockMask();

// 2. It is configured with specific rules.
// Forbid placing anything in air.
oreMask.setSkippedBlocks(MaterialSets.AIR);

// Allow iron ore to replace stone or deepslate.
oreMask.putBlockMaskEntry(MaterialSets.STONE_LIKE, MaterialSets.IRON_ORE);

// 3. The configured mask is used to validate block operations.
for (BlockPos pos : potentialOreLocations) {
    Material existingBlock = world.getMaterialAt(pos);
    if (oreMask.canReplace(existingBlock, Materials.IRON_ORE)) {
        world.setBlock(pos, Materials.IRON_ORE);
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Modification After Use:** Do not modify a BlockMask instance after it has been passed to a generation algorithm or worker thread. This leads to non-deterministic behavior and race conditions.
-   **Excessive Rule Complexity:** The `canReplace` method has linear time complexity based on the number of rules. Adding thousands of individual rules via `putBlockMaskEntry` will create a performance bottleneck in world generation. Prefer broader rules using `MaterialSet` where possible.
-   **Reusing Instances:** Do not reuse a single BlockMask instance across different, unrelated generator passes without clearing and re-configuring it. This can cause rules from one generator (e.g., tree placement) to leak into and corrupt the behavior of another (e.g., cave carving).

## Data Pipeline
The BlockMask acts as a gate in the control flow of world generation, not as a transformer of data. It validates a proposed state change against its internal rule set.

> Flow:
> Generator Proposes Block Change -> **BlockMask.canReplace(source, dest)** -> [**true**] -> World State is Mutated
>
> **BlockMask.canReplace(source, dest)** -> [**false**] -> Proposed Change is Discarded

