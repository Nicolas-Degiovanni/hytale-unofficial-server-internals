---
description: Architectural reference for NotPattern
---

# NotPattern

**Package:** com.hypixel.hytale.builtin.hytalegenerator.patterns
**Type:** Transient

## Definition
```java
// Signature
public class NotPattern extends Pattern {
```

## Architecture & Concepts
The NotPattern class is a fundamental structural component within the procedural world generation engine. It embodies the **Decorator** design pattern, acting as a logical inverter for other Pattern objects. Its primary role is to enable more complex and exclusionary generation rules by negating the result of a wrapped pattern.

In the context of the generator's composite pattern system, NotPattern serves as a logical NOT operator. This allows designers to construct sophisticated rules such as "place entity X, but NOT if condition Y is met". It is a foundational building block for creating negative constraints in a declarative, rule-based system. The evaluation flows downward through the composite tree, with this class inverting the boolean result from its child.

## Lifecycle & Ownership
- **Creation:** Instantiated dynamically by higher-level pattern assemblers or parsers during the construction of a generation rule set. It is never a managed service or singleton.
- **Scope:** Short-lived. Its lifetime is strictly bound to the composite Pattern that contains it. It exists only for the duration of a specific generation pass.
- **Destruction:** Becomes eligible for garbage collection immediately after the root Pattern that references it is no longer in use. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** **Deeply immutable**. The internal reference to the wrapped Pattern and the cached SpaceSize are both declared `final` and are assigned exclusively at construction time. This design guarantees that an instance of NotPattern cannot be modified after creation.
- **Thread Safety:** Inherently **thread-safe**. Its immutable nature ensures that a single NotPattern instance can be safely read and evaluated by multiple world-generation threads concurrently without any need for locks or synchronization primitives. This is a critical attribute for achieving high-performance, parallelized world generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(Context context) | boolean | O(inner) | Evaluates the wrapped Pattern. Returns the logical inverse of the result. Throws NullPointerException if context is null. |
| readSpace() | SpaceSize | O(1) | Returns a clone of the spatial footprint required by the wrapped pattern. The value is cached at construction for high performance. |

## Integration Patterns

### Standard Usage
NotPattern is never used in isolation. It must wrap another, more concrete Pattern to provide a meaningful rule. Its purpose is to invert the logic of its child in a larger composite structure.

```java
// 1. Define a base condition, for example, a pattern that matches water blocks.
Pattern isWater = new BlockTypePattern(BlockTypes.WATER);

// 2. Wrap the base condition in a NotPattern to invert its logic.
Pattern isNotWater = new NotPattern(isWater);

// 3. Use the inverted pattern within a larger generation rule.
// This rule will now only succeed on dry land.
if (isNotWater.matches(generatorContext)) {
   // Place a structure or entity that should not appear in water.
   placeTree(generatorContext.getPosition());
}
```

### Anti-Patterns (Do NOT do this)
- **Null Wrapping:** Do not construct a NotPattern with a null argument. The constructor is annotated with Nonnull and the runtime will throw an exception, causing the generation task to fail catastrophically.
- **Redundant Negation:** Avoid double-wrapping, such as `new NotPattern(new NotPattern(somePattern))`. This is functionally equivalent to `somePattern` but introduces unnecessary object allocation and method call overhead, degrading generator performance.

## Data Pipeline
NotPattern acts as a transformation node in the boolean evaluation pipeline of the pattern matching system. It does not process data streams but rather alters the logical flow of a boolean check.

> Flow:
> Generator Rule Evaluation -> `matches()` on Composite Pattern -> **`NotPattern.matches()`** -> `matches()` on Inner Pattern -> `boolean` result is inverted -> Returned to Caller

