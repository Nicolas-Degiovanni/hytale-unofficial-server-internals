---
description: Architectural reference for ConstantIntCondition
---

# ConstantIntCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class ConstantIntCondition implements IIntCondition {
```

## Architecture & Concepts
The ConstantIntCondition is a foundational, concrete implementation of the IIntCondition strategy interface. Its primary architectural role is to provide a condition whose result is invariant—it is determined at construction time and is completely independent of the input value provided during evaluation.

This class serves several key purposes within the procedural generation system:

1.  **Default or Fallback Behavior:** It can act as a terminal node in a complex decision tree, providing a guaranteed true or false outcome when other, more dynamic conditions are not met.
2.  **Configuration Toggles:** It allows procedural rules to be statically enabled or disabled via configuration without altering the logical structure of the condition set. A rule can be effectively "turned off" by injecting a ConstantIntCondition that always returns false.
3.  **Null Object Pattern:** It embodies the Null Object Pattern for the IIntCondition interface, providing a non-null, inert object that satisfies the contract but has no operational logic. This simplifies client code by removing the need for null checks when handling dynamically loaded or optional conditions.

The class provides two pre-allocated, public static instances, DEFAULT_TRUE and DEFAULT_FALSE, which should be preferred over direct instantiation to reduce object allocation.

### Lifecycle & Ownership
-   **Creation:** Instantiated either directly via its constructor or, more commonly, by referencing the static singletons DEFAULT_TRUE and DEFAULT_FALSE. As a simple value object, it is typically created and owned by a higher-level configuration or rule object.
-   **Scope:** The lifetime of a ConstantIntCondition instance is tied to its owner. The static singletons exist for the entire application lifetime.
-   **Destruction:** Instances are lightweight and managed entirely by the Java Garbage Collector. No manual cleanup is necessary.

## Internal State & Concurrency
-   **State:** This object is **deeply immutable**. Its single state field, *result*, is final and is set exclusively during construction. It performs no caching and holds no references to other services.
-   **Thread Safety:** The class is **unconditionally thread-safe**. Its immutable design guarantees that instances, including the static singletons, can be safely shared and evaluated across multiple threads without any form of synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DEFAULT_TRUE | ConstantIntCondition | O(1) | A pre-allocated, reusable instance that always evaluates to true. |
| DEFAULT_FALSE | ConstantIntCondition | O(1) | A pre-allocated, reusable instance that always evaluates to false. |
| ConstantIntCondition(boolean) | constructor | O(1) | Creates a new condition that will always return the given boolean result. |
| eval(int) | boolean | O(1) | Evaluates the condition. Ignores the input integer and returns the configured constant result. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to use the static singletons as stand-ins for simple, unchanging logical conditions within a larger procedural system.

```java
// Preferred usage for enabling or disabling a feature
// based on a static configuration flag.
boolean isFeatureEnabled = getConfig().isFeatureXEnabled();
IIntCondition condition = isFeatureEnabled
    ? ConstantIntCondition.DEFAULT_TRUE
    : ConstantIntCondition.DEFAULT_FALSE;

// The condition can now be evaluated without knowledge of its constant nature.
if (condition.eval(someValue)) {
    // Execute feature X logic...
}
```

### Anti-Patterns (Do NOT do this)
-   **Redundant Instantiation:** Avoid creating new instances when the provided singletons will suffice. This pattern is functionally correct but creates unnecessary object churn.

    ```java
    // AVOID: Creates a new object on the heap.
    IIntCondition condition = new ConstantIntCondition(true);

    // PREFER: Reuses the static, pre-allocated object.
    IIntCondition condition = ConstantIntCondition.DEFAULT_TRUE;
    ```

## Data Pipeline
ConstantIntCondition acts as a *source* or *terminator* in a data flow, not a transformer. It takes an input value, discards it, and produces a new boolean value based on its internal, pre-configured state.

> Flow:
> Procedural Rule Engine -> **ConstantIntCondition.eval(int)** -> Boolean Result

