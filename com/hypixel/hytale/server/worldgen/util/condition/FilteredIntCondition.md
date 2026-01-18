---
description: Architectural reference for FilteredIntCondition
---

# FilteredIntCondition

**Package:** com.hypixel.hytale.server.worldgen.util.condition
**Type:** Transient

## Definition
```java
// Signature
public class FilteredIntCondition implements IIntCondition {
```

## Architecture & Concepts
The FilteredIntCondition is a composite conditional component used extensively within the server-side world generation pipeline. It functions as a specialized implementation of the Decorator pattern, wrapping and modifying the behavior of another IIntCondition.

Its primary architectural role is to create exclusionary rules. It combines two distinct conditions: a **filter** and a **condition**. The logic is designed to first consult the filter. If a given integer value satisfies the filter, it is immediately rejected. Only if the value passes the filter (i.e., the filter evaluates to false) is the primary condition evaluated.

This pattern is critical for layering world generation rules, allowing developers to define broad conditions (e.g., "place granite between Y=10 and Y=40") and then apply specific exceptions (e.g., "but not if the biome is 'river'").

## Lifecycle & Ownership
- **Creation:** Instantiated on-the-fly by higher-level world generation services or procedural rule assemblers. It is not a managed service and should not be treated as one.
- **Scope:** The object's lifetime is ephemeral, typically scoped to a single procedural evaluation or chunk generation task. It is created, used for one or more evaluations, and then becomes eligible for garbage collection.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no native resources or explicit cleanup methods.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal filter and condition references are declared as final and are assigned only during construction. The object holds no mutable state post-instantiation.
- **Thread Safety:** This class is inherently thread-safe. Its immutability guarantees that it can be safely shared and executed by multiple world generation threads concurrently without locks or synchronization. This is a crucial property for achieving high-performance, parallelized world generation.

## API Surface
The public contract is minimal, consisting of the core evaluation method inherited from the IIntCondition interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(int value) | boolean | O(filter) + O(condition) | Evaluates the integer. Returns false if the filter matches; otherwise, returns the result of the primary condition. |

## Integration Patterns

### Standard Usage
FilteredIntCondition is used to compose complex rule sets by layering an exclusion rule on top of a base condition. It is almost always instantiated as part of a larger chain of procedural logic.

```java
// Example: Place a feature if a condition is met, but NOT if the biome ID is 2 (Ocean).
IIntCondition isOcean = new HashSetIntCondition(IntSets.singleton(2));
IIntCondition primaryCheck = new SomeOtherCondition();

// The filter rejects biome ID 2. The condition defers to the primary check.
IIntCondition placementRule = new FilteredIntCondition(isOcean, primaryCheck);

// In the worldgen loop for a given biome ID...
if (placementRule.eval(currentBiomeId)) {
    // This block will not execute if currentBiomeId is 2.
    // For other biomes, its execution depends on the primaryCheck.
}
```

### Anti-Patterns (Do NOT do this)
- **Logic Inversion:** Do not mistake the filter for an inclusionary check. The filter's purpose is to *reject* values. If the filter condition is met, the evaluation short-circuits and returns false, regardless of the main condition's logic.
- **Redundant Filtering:** Avoid creating a FilteredIntCondition where the filter logic is already mutually exclusive with the main condition. This adds unnecessary overhead and complexity.
- **Deep Nesting:** While possible, deeply nesting FilteredIntCondition instances within each other can create logic that is extremely difficult to reason about and debug. Prefer a flatter composition of conditions where possible.

## Data Pipeline
The data flow for this component is simple and linear. It acts as a gate within a larger data processing stream.

> Flow:
> World Generator provides an integer (e.g., Biome ID, Block ID, Height Value) -> **FilteredIntCondition.eval()** -> The filter condition is checked -> If false, the main condition is checked -> A final boolean result is returned -> The World Generator makes a decision (e.g., Place Block, Spawn Entity).

