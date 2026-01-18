---
description: Architectural reference for OffsetPattern
---

# OffsetPattern

**Package:** com.hypixel.hytale.builtin.hytalegenerator.patterns
**Type:** Transient

## Definition
```java
// Signature
public class OffsetPattern extends Pattern {
```

## Architecture & Concepts
The OffsetPattern is a fundamental component within the procedural world generation framework. It embodies the **Decorator** design pattern, acting as a lightweight, immutable wrapper that modifies the spatial context of another Pattern without altering its underlying logic.

Its primary function is to apply a three-dimensional translation (an offset) to a child Pattern. This enables a single, complex pattern defined relative to an origin point (0, 0, 0) to be reused and positioned anywhere within the world. For example, a generator can define a "tree" pattern once and then use multiple OffsetPattern instances to place that same tree at various coordinates.

This class is a key enabler of compositional world generation, allowing developers to build complex scenes by arranging and transforming simpler, reusable components.

### Lifecycle & Ownership
- **Creation:** OffsetPattern instances are created on-demand by higher-level generator logic or composition utilities. They are not managed by a central registry or service locator.
- **Scope:** The lifetime of an OffsetPattern is typically short and tied directly to the lifecycle of the composite pattern that contains it. It exists only for the duration of a specific generation task.
- **Destruction:** As a simple, stateless object, it is reclaimed by the Java garbage collector when no longer referenced. It requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The OffsetPattern is **immutable**. Its core state, consisting of the wrapped Pattern and the Vector3i offset, is established at construction and cannot be modified. It contains one piece of pre-computed state: the `readSpaceSize` field. This field caches the translated bounding box of the child pattern, a performance optimization to avoid repeated calculations.

- **Thread Safety:** This class is inherently **thread-safe**. Its immutability guarantees that it can be safely shared and accessed by multiple world generation threads simultaneously without any need for locks, synchronization, or other concurrency controls. All methods are pure and free of side effects.

## API Surface
The public contract is minimal, adhering to the base Pattern interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| OffsetPattern(pattern, offset) | constructor | O(N) | Constructs the decorator. Complexity depends on the child pattern's `readSpace` calculation. |
| matches(context) | boolean | O(1) + Child | Delegates the match query to the child pattern after translating the position in the context. |
| readSpace() | SpaceSize | O(1) | Returns a clone of the pre-computed, translated bounding box for the child pattern. |

## Integration Patterns

### Standard Usage
The canonical use case is to wrap an existing Pattern to displace it within the world. This is a core technique for procedural placement.

```java
// 1. Define a base pattern, such as a structure or feature.
Pattern featurePattern = new DungeonEntrancePattern();

// 2. Define the world coordinates where the feature should be placed.
Vector3i placementOffset = new Vector3i(1204, 75, -850);

// 3. Wrap the base pattern with an OffsetPattern to create the translated version.
Pattern placedFeature = new OffsetPattern(featurePattern, placementOffset);

// 4. Use the translated pattern in the generator. The context's position
//    will be adjusted before being passed to DungeonEntrancePattern.
boolean canPlace = placedFeature.matches(worldGenerationContext);
```

### Anti-Patterns (Do NOT do this)
- **Null Inputs:** The constructor is annotated with Nonnull. Providing a null Pattern or a null Vector3i is a contract violation and will result in a runtime exception.
- **Redundant Nesting:** While functionally correct, nesting multiple OffsetPatterns is inefficient. It creates unnecessary object allocations and call stack depth.
    ```java
    // BAD: Inefficient and hard to read
    Vector3i offsetA = new Vector3i(10, 0, 0);
    Vector3i offsetB = new Vector3i(0, 5, 0);
    Pattern p = new OffsetPattern(new OffsetPattern(basePattern, offsetA), offsetB);

    // GOOD: Pre-calculate the final offset
    Vector3i combinedOffset = new Vector3i(10, 5, 0);
    Pattern p = new OffsetPattern(basePattern, combinedOffset);
    ```

## Data Pipeline
The OffsetPattern acts as a transformation node within a generation query pipeline. It does not process data itself but rather modifies the metadata of the query (the context) before forwarding it to the next stage.

> Flow:
> Generator Query with `Pattern.Context` -> **OffsetPattern.matches(context)** -> New `Pattern.Context` created with `position` translated by `offset` -> Child `Pattern.matches(newContext)` -> Boolean Result

