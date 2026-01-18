---
description: Architectural reference for MaterialSetPattern
---

# MaterialSetPattern

**Package:** com.hypixel.hytale.builtin.hytalegenerator.patterns
**Type:** Transient

## Definition
```java
// Signature
public class MaterialSetPattern extends Pattern {
```

## Architecture & Concepts
The MaterialSetPattern is a foundational component within the procedural world generation engine. It functions as a highly specific and efficient *predicate*, designed to answer a single question: "Does the block at a given coordinate belong to a predefined collection of materials?".

Architecturally, this class is an implementation of the **Strategy Pattern**, where `Pattern` is the abstract strategy. The world generator uses various `Pattern` implementations to define complex rules for feature placement, terrain modification, and biome definition. The MaterialSetPattern is one of the most common strategies, used for identifying specific material types like "any stone", "any dirt", or "any log".

Its operation is atomic and localized; it only ever considers the single block at the context's target position, as defined by its `readSpace` method. This simplicity is critical for performance, allowing the generator to rapidly evaluate conditions across vast regions of the world.

### Lifecycle & Ownership
- **Creation:** Instantiated by higher-level generator constructs, such as a `Rule` or `FeaturePlacer`. It is configured at creation time with a `MaterialSet` object that defines the materials it should match.
- **Scope:** The lifetime of a MaterialSetPattern instance is typically bound to the lifetime of the rule that defines it. It is not a global or session-scoped object. It is a lightweight, disposable object created to execute a specific part of a generation algorithm.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup. It becomes eligible for collection as soon as its owning rule is no longer in use.

## Internal State & Concurrency
- **State:** The internal state consists of a single final field, `materialSet`. This state is provided at construction and is **immutable** for the lifetime of the object. The class itself can be considered a stateless predicate after its initial configuration.
- **Thread Safety:** This class is **inherently thread-safe**. Its immutable state and lack of side effects in the `matches` method make it perfectly suited for use in a multi-threaded world generation environment. Multiple worker threads can safely and concurrently invoke `matches` on a shared instance of MaterialSetPattern without requiring any locks or synchronization, which is a critical design choice for scalable world generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(Pattern.Context context) | boolean | O(1) | Evaluates the material at the context's position. Returns true if the material's hash is present in the configured MaterialSet. Returns false if out of bounds. |
| readSpace() | SpaceSize | O(1) | Returns the spatial footprint required by the pattern, which is always a 1x1x1 volume centered on the origin. This signifies the pattern does not need to inspect neighboring blocks. |

## Integration Patterns

### Standard Usage
A MaterialSetPattern is almost never used in isolation. It is constructed as part of a larger rule definition within the world generator and passed to a matching engine.

```java
// In a hypothetical rule definition class
MaterialSet allStoneTypes = materialRegistry.getSet("core:all_stone");
Pattern stoneMatcher = new MaterialSetPattern(allStoneTypes);

// The generator's rule engine would then use this instance
// to check thousands of positions.
if (stoneMatcher.matches(currentContext)) {
    // Logic to execute when stone is found, e.g., attempt to place an ore vein.
}
```

### Anti-Patterns (Do NOT do this)
- **Instantiation in a Loop:** Avoid creating `new MaterialSetPattern(...)` inside a tight generation loop that iterates over blocks. This is highly inefficient and generates excessive garbage. A pattern should be created once per rule and reused for all evaluations of that rule.
- **Misunderstanding `readSpace`:** Do not attempt to modify the `SpaceSize` returned by `readSpace`. It is a declaration of the pattern's required input area, not a configurable parameter.

## Data Pipeline
The MaterialSetPattern acts as a filter or gate in the procedural generation data flow. It consumes positional context and produces a simple boolean decision that influences subsequent generation stages.

> Flow:
> World Generator selects a coordinate -> `Pattern.Context` is created with world data -> **MaterialSetPattern.matches(context)** -> Boolean result is passed to a Rule Engine -> Rule Engine triggers further actions (e.g., Place Feature, Modify Terrain)

