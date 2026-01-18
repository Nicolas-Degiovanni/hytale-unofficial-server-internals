---
description: Architectural reference for AndCondition
---

# AndCondition

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders.spaceanddepth.conditions
**Type:** Transient

## Definition
```java
// Signature
public class AndCondition implements SpaceAndDepthMaterialProvider.Condition {
```

## Architecture & Concepts
The AndCondition class is a foundational component within the procedural world generation system, specifically for the SpaceAndDepthMaterialProvider. It functions as a **Composite Predicate**, implementing a logical AND operation over a collection of other Condition objects.

Architecturally, this class embodies the **Composite design pattern**. It allows developers to construct complex, layered material placement rules from simple, reusable Condition primitives. Instead of hard-coding deeply nested conditional logic within the generator, rules can be defined declaratively as a tree of Condition objects. AndCondition acts as a non-leaf node in this tree, aggregating the results of its children.

Its primary role is to determine if a specific block coordinate `(x, y, z)` satisfies *all* specified criteria simultaneously, enabling rules such as "place granite if depth is greater than 10 AND space above is less than 3".

## Lifecycle & Ownership
- **Creation:** AndCondition is not a managed service. It is instantiated directly during the configuration phase of a SpaceAndDepthMaterialProvider. This typically occurs when world generation rules are loaded from configuration files or constructed programmatically.
- **Scope:** The lifetime of an AndCondition instance is strictly tied to its parent SpaceAndDepthMaterialProvider. It exists only as part of the provider's rule set.
- **Destruction:** The object is eligible for garbage collection as soon as its owning SpaceAndDepthMaterialProvider is dereferenced. No explicit cleanup methods are required or provided.

## Internal State & Concurrency
- **State:** The internal state consists of a single final array of Condition objects. This array is populated once in the constructor from the provided List and is never modified thereafter. The object is therefore **effectively immutable** after construction.

- **Thread Safety:** This class is **conditionally thread-safe**. The `qualifies` method performs no writes and accesses only its immutable internal array. Concurrency safety is therefore dependent on the thread safety of the contained Condition objects. As world generation is a highly parallelized process, all Condition implementations passed to this class must themselves be thread-safe and stateless.

**WARNING:** Passing a Condition with mutable internal state to an AndCondition can introduce severe, difficult-to-debug race conditions in a multi-threaded world generator.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| qualifies(int, int, int, int, int, int, int) | boolean | O(N) | Evaluates each contained Condition in order. Returns `false` immediately if any Condition fails (short-circuiting). Returns `true` only if all Conditions pass. N is the number of contained conditions. |

## Integration Patterns

### Standard Usage
AndCondition is used to combine multiple checks into a single rule. It is almost always instantiated as part of a larger provider configuration.

```java
// Assume DepthCondition and SpaceCondition are other Condition implementations
List<SpaceAndDepthMaterialProvider.Condition> rules = new ArrayList<>();
rules.add(new DepthCondition(10)); // Must be at least 10 blocks deep
rules.add(new SpaceCondition(3));  // Must have at least 3 blocks of space above

// The AndCondition combines these rules
SpaceAndDepthMaterialProvider.Condition combinedRule = new AndCondition(rules);

// The world generator would then invoke this rule for a given block
boolean canPlaceMaterial = combinedRule.qualifies(x, y, z, ...);
```

### Anti-Patterns (Do NOT do this)
- **Passing Null Elements:** The constructor is defensive and will throw an `IllegalArgumentException` if the input list contains any null elements. Do not rely on this for flow control; ensure input data is clean.
- **Empty Condition List:** Constructing an AndCondition with an empty list of conditions will cause it to *always* return true. While not an error, this is logically redundant and can indicate a configuration problem.
- **Nesting Single Conditions:** Do not wrap a single Condition inside an AndCondition. This adds unnecessary object overhead and computational cost for no logical benefit.

## Data Pipeline
AndCondition acts as a filter or gate within the material selection data flow for a single block. It does not transform data but rather produces a binary decision that controls the pipeline's subsequent steps.

> Flow:
> World Generator queries block at (x, y, z) -> SpaceAndDepthMaterialProvider calculates spatial metrics -> Provider invokes **AndCondition.qualifies()** with metrics -> Boolean result returned -> Provider uses result to select or reject a material.

