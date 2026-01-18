---
description: Architectural reference for DefaultCoordinateRndCondition
---

# DefaultCoordinateRndCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class DefaultCoordinateRndCondition implements ICoordinateRndCondition {
```

## Architecture & Concepts
The DefaultCoordinateRndCondition is a foundational implementation of the ICoordinateRndCondition interface. Its primary architectural role is to provide a constant, non-dynamic boolean result within the procedural generation framework. It serves as a "null object" or a base case for conditional logic that would otherwise require complex, coordinate-dependent evaluation.

This class decouples procedural rules from the need for complex spatial logic. By providing a condition that is guaranteed to be always *true* or always *false*, it allows the system to configure rules that are universally applied or universally disabled without requiring special case handling in the evaluation engine. The static instances, DEFAULT_TRUE and DEFAULT_FALSE, are heavily optimized for this purpose.

## Lifecycle & Ownership
- **Creation:** The static instances DEFAULT_TRUE and DEFAULT_FALSE are instantiated once at class-loading time by the JVM. Custom instances can be created at any time via the public constructor, typically during the parsing of procedural configuration files.
- **Scope:** The static instances are global and persist for the entire application lifetime. Custom instances are transient and their lifetime is tied to the procedural rule or configuration object that holds a reference to them.
- **Destruction:** Custom instances are eligible for garbage collection once they are no longer referenced. The static instances are never destroyed.

## Internal State & Concurrency
- **State:** Immutable. The internal boolean *result* is a final field, set exclusively during construction. The object's state cannot be modified after instantiation.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. Instances can be safely shared and accessed across multiple threads without any external synchronization or locking mechanisms. This is critical for parallelized world generation tasks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DefaultCoordinateRndCondition(boolean) | constructor | O(1) | Creates a new instance with a fixed result. |
| eval(seed, x, z, y, random) | boolean | O(1) | Returns the pre-configured boolean result. **Warning:** All input parameters are ignored. |
| getResult() | boolean | O(1) | A simple accessor for the internal result state. |

## Integration Patterns

### Standard Usage
This class should be used when a procedural rule requires a condition that is constant. The static singletons are the preferred mechanism for obtaining a universally true or false condition.

```java
// In a procedural rule definition, use the static instance for a rule that should always apply.
ICoordinateRndCondition condition = DefaultCoordinateRndCondition.DEFAULT_TRUE;

// The generation engine evaluates the condition, which will always return true.
if (condition.eval(seed, x, y, z, random)) {
    // ...execute rule action
}
```

### Anti-Patterns (Do NOT do this)
- **Redundant Instantiation:** Do not create new instances when the static singletons suffice. This creates unnecessary object churn and defeats the purpose of the shared instances.
    - **BAD:** `new DefaultCoordinateRndCondition(true)`
    - **GOOD:** `DefaultCoordinateRndCondition.DEFAULT_TRUE`
- **Expecting Dynamic Evaluation:** Do not use this class with the expectation that it will evaluate the coordinate or random parameters passed to the *eval* method. It is designed to be static and will always ignore them.

## Data Pipeline
This component acts as a terminal node in a conditional data flow. It does not transform data but instead produces a constant signal that controls the subsequent flow of logic within the broader procedural generation system.

> Flow:
> Procedural Rule Engine -> Request for Condition Evaluation (with seed, coordinates) -> **DefaultCoordinateRndCondition.eval()** -> Constant Boolean Result -> Engine Executes or Skips Rule Action

