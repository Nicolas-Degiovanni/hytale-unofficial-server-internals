---
description: Architectural reference for AndPattern
---

# AndPattern

**Package:** com.hypixel.hytale.builtin.hytalegenerator.patterns
**Type:** Transient

## Definition
```java
// Signature
public class AndPattern extends Pattern {
```

## Architecture & Concepts
The AndPattern is a structural component within the procedural world generation framework, implementing the **Composite** design pattern. It functions as a logical AND operator, aggregating multiple child Pattern objects and treating them as a single, unified condition. For an AndPattern to be considered a match, *all* of its constituent child patterns must successfully match against the provided context.

This class is a fundamental building block for creating complex, layered, and rule-based generation logic. It allows designers to combine simple, reusable patterns—such as checking for a specific block type or elevation—into sophisticated composite rules that define the placement of structures, biomes, or features.

A key architectural feature is its constructor-time optimization. Upon instantiation, the AndPattern pre-calculates and caches the union of the spatial boundaries (the SpaceSize) of all its child patterns. This prevents redundant and potentially expensive boundary calculations during the highly repetitive matching phase of world generation, significantly improving performance.

## Lifecycle & Ownership
- **Creation:** AndPattern instances are not managed services. They are instantiated dynamically by higher-level generator logic, typically when a pattern tree is constructed from a data-driven configuration (e.g., a JSON or script file). They are created on-the-fly as part of a specific generation task.
- **Scope:** The lifetime of an AndPattern is ephemeral and tied directly to the scope of the generation operation it is part of. It exists only as a node within a larger pattern tree and is discarded once that tree is no longer needed.
- **Destruction:** The object is managed by the Java Garbage Collector. No manual cleanup is necessary. It becomes eligible for collection as soon as the root of its pattern tree is dereferenced.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, consisting of the array of child patterns and the pre-computed SpaceSize, is established exclusively within the constructor and marked as final. The object cannot be modified after creation. The readSpace method defensively returns a clone of its internal SpaceSize to prevent external mutation.
- **Thread Safety:** **Fully thread-safe**. Its immutable nature guarantees that an AndPattern instance can be safely read and executed by multiple generator threads concurrently without locks or any other synchronization primitives. The matches method is a pure function, free of side effects.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(Pattern.Context context) | boolean | O(N) | Evaluates each child pattern in sequence. Returns false immediately upon the first failure (short-circuiting). Returns true only if all N child patterns match. |
| readSpace() | SpaceSize | O(1) | Returns a clone of the pre-computed spatial boundary that encompasses all child patterns. This is a fast cache lookup. |

## Integration Patterns

### Standard Usage
The AndPattern is used to combine multiple conditions. It is constructed with a list of other Pattern objects and then evaluated as part of a larger generation algorithm.

```java
// Example: Define a rule that only matches if the location is both
// above a certain height AND in a specific biome.
Pattern heightCheck = new HeightPattern(100, 255);
Pattern biomeCheck = new BiomePattern("forest");

// Combine the two conditions using AndPattern
Pattern forestMountainPattern = new AndPattern(List.of(heightCheck, biomeCheck));

// Later, in the generator loop...
if (forestMountainPattern.matches(generationContext)) {
   // Place a specific feature, like a mountain pine tree
}
```

### Anti-Patterns (Do NOT do this)
- **Redundant Nesting:** Avoid nesting AndPattern instances within other AndPattern instances. This adds unnecessary object overhead and complexity. The constructor accepts a list, so a flat structure is always possible and preferred.
- **Performance-Intensive Ordering:** The evaluation short-circuits. For optimal performance, place the computationally cheapest or most likely to fail patterns at the beginning of the list provided to the constructor.
- **Empty Composition:** Constructing an AndPattern with an empty list of patterns is syntactically valid but logically ambiguous. The current implementation will always return true. For clarity, a dedicated AlwaysTruePattern or similar explicit construct should be used instead.

## Data Pipeline
The AndPattern acts as a conditional gate or predicate within the world generation pipeline. It does not transform data but rather determines which logic path to follow based on world state.

> Flow:
> Generator Configuration -> Pattern Tree Construction -> **AndPattern Instantiation** -> World Generator evaluates tree -> **AndPattern.matches(context)** -> Boolean Result -> Voxel Placement / Feature Generation Decision

