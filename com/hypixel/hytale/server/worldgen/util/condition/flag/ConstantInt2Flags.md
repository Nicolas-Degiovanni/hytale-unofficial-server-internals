---
description: Architectural reference for ConstantInt2Flags
---

# ConstantInt2Flags

**Package:** com.hypixel.hytale.server.worldgen.util.condition.flag
**Type:** Utility / Strategy Object

## Definition
```java
// Signature
public class ConstantInt2Flags implements Int2FlagsCondition {
```

## Architecture & Concepts
The ConstantInt2Flags class is a specific, immutable implementation of the Int2FlagsCondition strategy interface. Within the server-side world generation system, it represents the simplest possible conditional logic: one that always returns a pre-configured, constant integer value representing a set of flags.

Its primary role is to serve as a terminal node or a default case in a more complex tree of world generation rules. Where other conditions might evaluate biome temperature, height, or proximity to other features, ConstantInt2Flags provides a fixed, predictable outcome. This is essential for establishing baseline rules, defining non-negotiable terrain features, or for testing and debugging generation pipelines by injecting a known value.

By conforming to the Int2FlagsCondition interface, it allows the world generation engine to remain agnostic of the condition's complexity, treating this simple constant provider and a complex procedural calculator interchangeably.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly via its constructor, typically during the assembly of a world generation ruleset. It is not managed by a dependency injection framework and is considered a simple value object.
-   **Scope:** Transient. Its lifetime is strictly bound to the parent configuration or rule object that holds a reference to it. It does not persist beyond the world generation process for a given region.
-   **Destruction:** The object is lightweight and requires no explicit cleanup. It becomes eligible for garbage collection as soon as its owning rule is discarded.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal state consists of a single final integer, *result*, which is set at construction time. The object's state cannot be modified after instantiation.
-   **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, an instance of ConstantInt2Flags can be safely shared and accessed concurrently by multiple world generation threads without any external synchronization or locking mechanisms. This makes it highly suitable for use in parallelized chunk generation tasks.

## API Surface
The public contract is minimal, focused entirely on fulfilling the Int2FlagsCondition interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(int input) | int | O(1) | Returns the constant integer value configured at construction. The *input* parameter is completely ignored. |

## Integration Patterns

### Standard Usage
This class is used as a concrete strategy for the Int2FlagsCondition interface. It is instantiated with the desired flag value and passed to a system that evaluates these conditions.

```java
// Example: A worldgen rule is configured to always apply forest flags.
// The value for 'input' passed to eval() will have no effect on the outcome.
Int2FlagsCondition alwaysForest = new ConstantInt2Flags(WorldGenFlags.FOREST);

// The rule engine later evaluates the condition
int generatedFlags = alwaysForest.eval(currentNoiseValue);
// generatedFlags will always equal WorldGenFlags.FOREST
```

### Anti-Patterns (Do NOT do this)
-   **Conditional Logic:** Do not select this implementation when the resulting flags must change based on world data like noise, height, or temperature. Using this class where a dynamic evaluation is needed will lead to monolithic, non-procedural terrain.
-   **Over-Composition:** Avoid wrapping a ConstantInt2Flags instance in further conditional logic. Its purpose is to be a simple, terminal value. If you find yourself checking a condition to decide whether to use ConstantInt2Flags, your logic should likely be a different, more comprehensive Int2FlagsCondition implementation.

## Data Pipeline
ConstantInt2Flags acts as a simple, static data source within the world generation pipeline. It ignores any upstream data and injects a fixed value into the stream.

> Flow:
> WorldGen Rule Engine -> **ConstantInt2Flags.eval(ignored_input)** -> Constant Flag Set -> Biome/Feature Placement System

