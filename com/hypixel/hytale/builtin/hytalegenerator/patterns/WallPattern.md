---
description: Architectural reference for WallPattern
---

# WallPattern

**Package:** com.hypixel.hytale.builtin.hytalegenerator.patterns
**Type:** Transient

## Definition
```java
// Signature
public class WallPattern extends Pattern {
```

## Architecture & Concepts
The WallPattern is a composite `Pattern` used within the world generation system to evaluate adjacency conditions. It acts as a logical predicate that combines two subordinate patterns: an *originPattern* and a *wallPattern*. Its primary function is to determine if a specific block configuration exists, where the origin pattern matches at a central point and the wall pattern matches at one or more adjacent cardinal locations (North, South, East, West).

This class is a fundamental building block for creating context-aware procedural generation rules. For example, it can be used to define rules such as "place a lily pad only if the origin is an air block and an adjacent block is water" or "a fence post should have a connecting fence piece if the adjacent block is also a fence post".

The behavior is controlled by two key parameters:
1.  **directions**: A list of `WallDirection` enums specifying which adjacent cells to check.
2.  **matchAll**: A boolean flag that dictates the logical operator. If true, the wallPattern must match in **all** specified directions (logical AND). If false, it must match in **at least one** direction (logical OR).

Upon instantiation, WallPattern pre-calculates its total spatial footprint, or `readSpace`, by merging the space of the origin pattern with the translated spaces of the wall pattern for each specified direction. This allows the generator to efficiently determine the bounding box required for evaluation.

## Lifecycle & Ownership
-   **Creation:** WallPattern instances are not managed by a central service or registry. They are instantiated directly by higher-level world generation configurations, often during the deserialization of generator rule sets from asset files. The presence of a `Codec` on the inner `WallDirection` enum strongly implies this data-driven creation process.
-   **Scope:** The lifetime of a WallPattern object is tied to the world generator configuration that defines it. As it is stateless after construction, a single instance can be safely reused for millions of evaluations across the world grid. It persists as long as its parent rule set is loaded in memory.
-   **Destruction:** The object is subject to standard Java garbage collection. It is cleaned up when the generator configuration that holds a reference to it is unloaded or replaced. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
-   **State:** The WallPattern is **effectively immutable**. Its constructor defensively copies the incoming `directions` list into a new `ArrayList`, and all other fields are final. The internal state (the two sub-patterns, directions, and matchAll flag) cannot be modified after the object is created.
-   **Thread Safety:** This class is **inherently thread-safe**. Due to its immutable nature, the `matches` method is a pure function with no side effects. Multiple worker threads within the world generator can invoke `matches` on a shared WallPattern instance concurrently without any need for external synchronization or locks.

## API Surface
The public contract is minimal, focusing exclusively on the `Pattern` interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(Pattern.Context context) | boolean | O(N) | Evaluates the pattern at the given context. N is the number of directions. Returns true if the origin and wall patterns match according to the `matchAll` rule. |
| readSpace() | SpaceSize | O(1) | Returns a clone of the pre-calculated bounding box required to evaluate this pattern. |

## Integration Patterns

### Standard Usage
WallPattern is designed to be composed with other `Pattern` implementations to form a complex rule. The typical use case involves defining simple block patterns and combining them.

```java
// Example: Define a rule that matches a grass block next to a water block on its north side.

// Assume BlockPattern is a simple Pattern that matches a specific block type.
Pattern grass = new BlockPattern(GRASS_BLOCK);
Pattern water = new BlockPattern(WATER_BLOCK);

List<WallPattern.WallDirection> directions = List.of(WallPattern.WallDirection.N);

// The matchAll flag is irrelevant for a single direction.
WallPattern grassNextToWater = new WallPattern(water, grass, directions, false);

// The generator would then use this pattern.
if (grassNextToWater.matches(generatorContext)) {
    // Place a special plant or feature here.
}
```

### Anti-Patterns (Do NOT do this)
-   **Deep Nesting:** Avoid creating deeply nested structures where a WallPattern contains another WallPattern as its origin or wall. While functionally possible, this can lead to significant performance degradation due to the recursive creation of `Pattern.Context` objects and repeated position cloning during evaluation.
-   **Empty Direction List:** Instantiating a WallPattern with an empty list of directions is a logical error. The `matches` method will always return `false` in this configuration, as its evaluation loop will never execute.

## Data Pipeline
The data flow for a single evaluation is linear and self-contained within the world generator's evaluation loop.

> Flow:
> World Generator provides `Pattern.Context` -> `WallPattern.matches(context)` is invoked -> Loop begins over `directions` -> For each direction, a new `wallPosition` is calculated -> A new `wallContext` is created -> `originPattern.matches()` and `wallPattern.matches()` are called -> Boolean result is aggregated and returned.

