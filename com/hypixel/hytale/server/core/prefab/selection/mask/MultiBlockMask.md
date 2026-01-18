---
description: Architectural reference for MultiBlockMask
---

# MultiBlockMask

**Package:** com.hypixel.hytale.server.core.prefab.selection.mask
**Type:** Transient

## Definition
```java
// Signature
public class MultiBlockMask extends BlockMask {
```

## Architecture & Concepts
The MultiBlockMask is a composite implementation of the BlockMask contract. It acts as a container that aggregates multiple child BlockMask instances, combining their individual filtering logic into a single, more complex rule. This class is a cornerstone of the server's world modification and prefab systems, enabling the definition of sophisticated, non-trivial block selections.

The core operational logic is a logical **OR** on the exclusion results of its children. If *any* contained mask determines a block should be excluded, the MultiBlockMask, by default, will also exclude that block.

This behavior can be reversed via the parent BlockMask's inversion flag. When inverted, the MultiBlockMask performs a logical **NAND** on the exclusion results. This means it will exclude a block *only if all* of its child masks would have included it. This powerful feature allows for creating "intersection" style selections where a block must satisfy multiple positive conditions simultaneously.

## Lifecycle & Ownership
- **Creation:** A MultiBlockMask is typically instantiated by a parser or factory responsible for interpreting a serialized mask definition (e.g., from a command argument or a prefab asset file). It is not intended for direct, widespread instantiation in game logic; it is the result of a build or parse step.
- **Scope:** The object's lifetime is ephemeral and tied directly to the scope of the operation that requires it. For example, it may exist only for the duration of a single world edit command or a single prefab placement check.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup. It becomes eligible for collection as soon as the owning operation completes and drops its reference.

## Internal State & Concurrency
- **State:** The primary internal state is the final array of child BlockMasks provided during construction. As this array reference is final and the contained masks are themselves expected to be immutable, the MultiBlockMask is considered **Immutable**. Its filtering behavior is permanently fixed upon instantiation.
- **Thread Safety:** As an immutable object, MultiBlockMask is inherently **Thread-Safe**. Its methods are pure functions whose results depend solely on the input arguments and its immutable internal state. It can be safely passed between and used by multiple threads without locks or other synchronization primitives, making it ideal for use in parallelized chunk processing and world generation tasks.

## API Surface
The public contract is focused on evaluation and serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| isExcluded(accessor, x, y, z, min, max, blockId, fluidId) | boolean | O(N) | Evaluates if a block at the given coordinates should be excluded. N is the number of child masks. The operation short-circuits, providing O(1) best-case performance. |
| toString() | String | O(N) | Generates a compact, machine-readable string representation of the combined mask rules, suitable for serialization. |
| informativeToString() | String | O(N) | Generates a human-readable string representation of the logic, such as NOT(mask1 AND mask2), for debugging and logging. |

## Integration Patterns

### Standard Usage
The MultiBlockMask is not used directly but is provided to a system that iterates over blocks, using the mask as a predicate to filter which blocks are affected by an operation.

```java
// A system receives a pre-configured mask (e.g., from a command parser)
// This mask could be a MultiBlockMask under the hood.
BlockMask selectionMask = context.getMaskFromDefinition("stone;!grass_block");

// The mask is then used by an iterator or processor
for (Vector3i pos : region) {
    int blockId = world.getBlock(pos);
    if (!selectionMask.isExcluded(world, pos.x, pos.y, pos.z, ..., blockId)) {
        // Apply operation to the selected block
        world.setBlock(pos, BlockTypes.DIAMOND_BLOCK);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Composition:** Avoid manually creating deep trees of MultiBlockMasks in high-performance game logic. Prefer defining complex masks in data (asset files, command strings) and using a centralized parser to build them.
- **Empty Child Array:** Constructing a MultiBlockMask with an empty array of children is logically flawed. While the code handles this gracefully (it will never exclude a block unless inverted), it signifies an upstream error in the logic that generated the mask rules.
- **Misinterpreting Inversion:** Do not assume an inverted MultiBlockMask simply negates each child rule. The `!A;B` mask is a NAND operation, not `(!A);(!B)`. It means "exclude a block only if it is NOT excluded by A AND it is NOT excluded by B".

## Data Pipeline
The MultiBlockMask acts as a predicate or filter within a larger data processing flow, rather than a pipeline stage that transforms data. It is a decision point that gates whether a subsequent operation proceeds.

> Flow:
> Command String -> Mask Parser -> **MultiBlockMask Instantiation** -> World Region Iterator -> For each block: **isExcluded() check** -> [FILTERED] / [PASSED] -> Block Operation (e.g., SetBlock)

