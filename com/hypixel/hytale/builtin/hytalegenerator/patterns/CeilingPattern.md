---
description: Architectural reference for CeilingPattern
---

# CeilingPattern

**Package:** com.hypixel.hytale.builtin.hytalegenerator.patterns
**Type:** Transient

## Definition
```java
// Signature
public class CeilingPattern extends Pattern {
```

## Architecture & Concepts
The CeilingPattern is a composite decorator within the procedural world generation framework. It does not represent a material itself, but rather enforces a spatial relationship between two other patterns. Its primary function is to verify that a specific *ceiling* pattern exists directly above another *base* pattern at a given world coordinate.

This class is a fundamental building block for creating vertically aware structures. For example, it can be used to ensure that cave interiors are only generated where there is solid ground above them, or to place stalactites only on the underside of a specific block type. It operates as a logical AND condition with a positional offset, effectively combining two simpler rules into a more complex, contextual one.

## Lifecycle & Ownership
- **Creation:** A CeilingPattern is instantiated by a higher-level configuration or factory responsible for assembling the complete set of rules for a world generator. It is constructed by composing two existing Pattern instances.
- **Scope:** The lifetime of a CeilingPattern instance is tied to the world generation preset or ruleset it belongs to. It typically persists as long as that ruleset is active.
- **Destruction:** The object is managed by the Java garbage collector. It is de-referenced and cleaned up when the parent world generation configuration is discarded.

## Internal State & Concurrency
- **State:** This object is **immutable** after construction. The constituent patterns and the pre-calculated `readSpaceSize` are final. This design choice guarantees predictable behavior and eliminates entire classes of bugs related to state corruption.
- **Thread Safety:** The CeilingPattern is unconditionally thread-safe. Its immutability allows it to be safely shared and accessed by multiple world generation worker threads simultaneously without requiring any synchronization or locking mechanisms. The `matches` method is a pure function with no side effects.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| matches(Pattern.Context context) | boolean | O(N+M) | Checks if the base pattern matches at the context position and the ceiling pattern matches at position Y+1. Complexity is the sum of the composed patterns. |
| readSpace() | SpaceSize | O(1) | Returns the bounding box required to perform the match check. This is a pre-calculated value. |

## Integration Patterns

### Standard Usage
The CeilingPattern is used to combine two patterns into a single, vertically-aware rule. The generator's execution engine invokes the `matches` method, passing in the current world context.

```java
// Define the constituent patterns
Pattern stone = new MaterialPattern(STONE_ID);
Pattern air = new MaterialPattern(AIR_ID);

// Create the composite rule: "match air only if stone is above it"
Pattern caveInteriorRule = new CeilingPattern(stone, air);

// The world generator would then use this rule
boolean isCave = caveInteriorRule.matches(currentGeneratorContext);
```

### Anti-Patterns (Do NOT do this)
- **Complex Compositions:** Nesting computationally expensive patterns within a CeilingPattern can create significant performance bottlenecks, as both sub-patterns must be evaluated for every call. This is especially critical for patterns that perform large area lookups.
- **Incorrect Ordering:** The constructor arguments are `(ceiling, base)`. Reversing this order is a common logical error that will produce unexpected generation results, as it will check for the base pattern above the ceiling.

## Data Pipeline
The CeilingPattern acts as a filter or predicate in the world generation data flow. It consumes a world position and state, and produces a boolean result that influences the generator's next action.

> Flow:
> World Generator -> Provides `Pattern.Context` -> **CeilingPattern.matches()** -> Evaluates `basePattern` and `ceilingPattern` -> Returns `boolean` -> World Generator Decision Engine

