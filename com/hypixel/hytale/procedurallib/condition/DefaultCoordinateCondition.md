---
description: Architectural reference for DefaultCoordinateCondition
---

# DefaultCoordinateCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Utility / Flyweight

## Definition
```java
// Signature
public class DefaultCoordinateCondition implements ICoordinateCondition {
```

## Architecture & Concepts
The DefaultCoordinateCondition class provides a trivial, constant-value implementation of the ICoordinateCondition interface. It is a foundational component within the procedural generation library, designed to act as a **terminating node** or a **default case** in complex procedural logic chains.

Its primary architectural purpose is to satisfy interface contracts that require a condition, even when the outcome is predetermined (always true or always false). The design employs the **Flyweight pattern** by exposing two pre-allocated, public, static instances: DEFAULT_TRUE and DEFAULT_FALSE. This strategy is critical for performance, as it prevents the unnecessary allocation and garbage collection of millions of identical, stateless condition objects during intensive world generation processes.

## Lifecycle & Ownership
- **Creation:** The two public instances, DEFAULT_TRUE and DEFAULT_FALSE, are instantiated by the JVM during static class initialization. The constructor is private, which strictly prohibits any further instantiation at runtime.
- **Scope:** These static instances are global and persist for the entire lifetime of the application. They are effectively permanent, shared objects.
- **Destruction:** The instances are managed by the JVM and are only eligible for cleanup when the application shuts down. No manual destruction is required or possible.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state consists of a single final boolean field, *result*, which is set at creation time and can never be modified.
- **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, instances of DefaultCoordinateCondition can be safely shared and accessed by any number of threads without locks or other synchronization primitives. The eval methods are pure functions with no side effects.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(seed, x, y) | boolean | O(1) | Evaluates the condition for a 2D coordinate. This method ignores all input parameters and returns the pre-configured boolean result. |
| eval(seed, x, y, z) | boolean | O(1) | Evaluates the condition for a 3D coordinate. This method ignores all input parameters and returns the pre-configured boolean result. |

## Integration Patterns

### Standard Usage
This class should not be instantiated. Developers must use the provided static instances to represent unconditional logic within the procedural generation system.

```java
// Example: Using the static flyweight instance for a condition that must always pass.
ICoordinateCondition alwaysTrueCondition = DefaultCoordinateCondition.DEFAULT_TRUE;

if (alwaysTrueCondition.eval(seed, x, y, z)) {
    // This block is guaranteed to execute.
    // Place unconditional generation logic here.
}

// Example: Using the static flyweight instance for a disabled feature.
ICoordinateCondition featureDisabledCondition = DefaultCoordinateCondition.DEFAULT_FALSE;

if (featureDisabledCondition.eval(seed, x, y, z)) {
    // This block will never execute.
}
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** The constructor is private. Attempting to bypass this restriction using reflection violates the core design principles of the class, negates the memory optimization of the Flyweight pattern, and is strictly forbidden.
- **Redundant Comparison:** Avoid comparing an object reference to the static instances. The purpose of the ICoordinateCondition interface is to provide polymorphic behavior. Rely on the `eval` method to get the result.

```java
// BAD: Unnecessary and brittle reference check.
if (myCondition == DefaultCoordinateCondition.DEFAULT_TRUE) { ... }

// GOOD: Correct polymorphic usage.
if (myCondition.eval(seed, x, y)) { ... }
```

## Data Pipeline
DefaultCoordinateCondition acts as a **data source** of a constant boolean value within a larger data flow. It does not process or transform input data; it simply injects a fixed result into the evaluation chain.

> Flow:
> **DefaultCoordinateCondition.DEFAULT_TRUE** -> `eval()` call -> **Boolean `true`** -> Procedural Logic Gate -> World Generation Action

