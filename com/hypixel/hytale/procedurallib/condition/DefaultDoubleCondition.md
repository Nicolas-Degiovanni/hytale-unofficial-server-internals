---
description: Architectural reference for DefaultDoubleCondition
---

# DefaultDoubleCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class DefaultDoubleCondition implements IDoubleCondition {
```

## Architecture & Concepts
The DefaultDoubleCondition is a foundational, immutable implementation of the IDoubleCondition interface. Its primary role within the procedural generation library is to provide a constant, non-evaluative condition. It completely ignores the input value passed to its evaluation method and returns a pre-configured boolean result.

This component serves several key architectural purposes:
- **Terminal Conditions:** Acts as a default or terminal node in a chain or tree of conditions, providing a guaranteed `true` or `false` outcome.
- **Configuration Toggles:** Allows features within procedural generation to be statically enabled or disabled by injecting `DEFAULT_TRUE` or `DEFAULT_FALSE` where a dynamic condition is expected.
- **Testing & Debugging:** Provides a simple, predictable mock for unit tests or debugging scenarios that require isolating complex conditional logic.

The class exposes two static final instances, `DEFAULT_TRUE` and `DEFAULT_FALSE`, which should be preferred over direct instantiation to reduce object allocation and improve code clarity.

## Lifecycle & Ownership
- **Creation:** Instances are created in two ways:
    1. Statically, by the JVM class loader for the `DEFAULT_TRUE` and `DEFAULT_FALSE` singletons.
    2. Dynamically, via direct constructor invocation (`new DefaultDoubleCondition(...)`) by higher-level components that require a constant condition determined at runtime.
- **Scope:** The static instances are application-scoped and persist for the lifetime of the JVM. Dynamically created instances are transient and their lifetime is managed by the object that created them.
- **Destruction:** Transient instances are eligible for garbage collection once all references are released. The static instances are never destroyed.

## Internal State & Concurrency
- **State:** Immutable. The internal `result` field is `final` and is set only once during construction. The object's state cannot be changed after it is created.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. It can be safely shared and accessed by multiple threads without any external synchronization or locking mechanisms.

## API Surface
The public API is minimal, focusing exclusively on the IDoubleCondition contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(double value) | boolean | O(1) | Returns the pre-configured boolean result, ignoring the input value. |
| getResult() | boolean | O(1) | A simple accessor for the internal boolean result. |

## Integration Patterns

### Standard Usage
For common true or false conditions, always use the provided static instances. This is the most performant and readable approach.

```java
// Example: Using a DefaultDoubleCondition as a fallback in a procedural rule
IDoubleCondition primaryCondition = getSomeComplexCondition();
IDoubleCondition fallback = DefaultDoubleCondition.DEFAULT_FALSE;

// If primaryCondition is null, the system falls back to a constant false
IDoubleCondition finalCondition = (primaryCondition != null) ? primaryCondition : fallback;

// The evaluation is now safe from null pointers
boolean canSpawn = finalCondition.eval(someValue);
```

### Anti-Patterns (Do NOT do this)
- **Redundant Instantiation:** Avoid creating new instances for constant true or false values. This creates unnecessary object churn.
    - **BAD:** `IDoubleCondition condition = new DefaultDoubleCondition(true);`
    - **GOOD:** `IDoubleCondition condition = DefaultDoubleCondition.DEFAULT_TRUE;`
- **Over-complication:** Do not use this class to wrap a boolean that is already known. It is intended for systems that require an object conforming to the IDoubleCondition interface.

## Data Pipeline
This component acts as a simple, constant data source within a conditional evaluation pipeline. It does not transform data; it originates a boolean value.

> Flow:
> Input `double` -> **DefaultDoubleCondition.eval()** -> Constant `boolean` result

