---
description: Architectural reference for OrPattern
---

# OrPattern

**Package:** com.hypixel.hytale.builtin.hytalegenerator.patterns
**Type:** Transient

## Definition
```java
// Signature
public class OrPattern extends Pattern {
```

## Architecture & Concepts
The OrPattern is a structural component within the world generation framework that implements the **Composite Pattern**. It functions as a container, treating a collection of individual Pattern objects as a single, unified Pattern. Its primary role is to provide logical OR evaluation, succeeding if *any* of its contained sub-patterns successfully match the given context.

This class is fundamental for creating procedural variety and non-deterministic outcomes in world generation. For example, it can be used to define that a specific location can spawn *either* a cluster of rocks *or* a small pond.

A critical performance optimization is performed during construction. The OrPattern pre-calculates the union of the spatial boundaries (SpaceSize) of all its child patterns. This merged SpaceSize is then cached. This avoids expensive, repeated boundary calculations during the matching phase of world generation, where the readSpace method may be called frequently.

Evaluation is performed using short-circuiting logic. The matching process halts and returns true immediately upon the first successful match from a child pattern, preventing unnecessary computation.

## Lifecycle & Ownership
- **Creation:** An OrPattern is instantiated directly by higher-level generation logic, often within a factory or builder that assembles a complex pattern tree from configuration files or procedural rules. It is not a managed service.
- **Scope:** The object's lifetime is ephemeral, typically scoped to a single generation task or the lifetime of its parent composite pattern. It does not persist between generation sessions.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as all references to it are dropped, which usually occurs after the relevant section of the world has been generated. No explicit cleanup is necessary.

## Internal State & Concurrency
- **State:** The OrPattern is **effectively immutable**. Its internal state, consisting of an array of child patterns and a pre-calculated SpaceSize, is established at construction and is not modified thereafter. Both internal fields are marked as final.
- **Thread Safety:** This class is **conditionally thread-safe**. Its immutable nature ensures that it can be safely read by multiple threads simultaneously. However, overall thread safety depends on the guarantees provided by the Pattern.Context object passed into the matches method and the thread safety of the contained child Pattern objects. The OrPattern class itself introduces no synchronization mechanisms.

**WARNING:** Sharing mutable child Pattern objects across different threads can lead to race conditions and non-deterministic generation bugs. It is recommended to use immutable patterns or ensure that each generation thread operates on a distinct instance of a pattern tree.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(Pattern.Context context) | boolean | O(N) | Iterates through child patterns, returning true on the first successful match. N is the number of children. |
| readSpace() | SpaceSize | O(1) | Returns a clone of the pre-calculated spatial boundary. The operation is constant time due to caching. |

## Integration Patterns

### Standard Usage
The OrPattern is used to combine multiple patterns into a single choice. The generator engine then treats this composite as a single unit.

```java
// Assume patternForForest and patternForClearing are pre-existing
List<Pattern> biomeChoices = List.of(patternForForest, patternForClearing);

// The OrPattern encapsulates the choice
Pattern eitherForestOrClearing = new OrPattern(biomeChoices);

// The generator engine can now use the composite pattern
// to decide which feature to place.
if (eitherForestOrClearing.matches(generationContext)) {
    // A valid placement was found for one of the choices.
}
```

### Anti-Patterns (Do NOT do this)
- **Empty Composition:** Do not construct an OrPattern with an empty list of patterns. While the constructor handles this case gracefully, the resulting pattern is useless as its matches method will always return false. This represents dead code and should be avoided.
- **Over-Composition:** Nesting many layers of OrPattern objects can make the generation logic difficult to debug and reason about. Where possible, prefer a flatter structure by combining choices into a single, larger OrPattern.

## Data Pipeline
The OrPattern acts as a routing and decision-making node within the data flow of the world generator. It does not source or sink data itself but directs the flow of control based on the success of its children.

> Flow:
> Generator Engine -> `matches(context)` -> **OrPattern** -> `matches(context)` on Child 1 -> `matches(context)` on Child 2 ... -> Boolean Result -> Generator Engine

