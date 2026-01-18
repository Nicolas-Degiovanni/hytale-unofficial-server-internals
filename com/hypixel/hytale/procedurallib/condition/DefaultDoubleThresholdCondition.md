---
description: Architectural reference for DefaultDoubleThresholdCondition
---

# DefaultDoubleThresholdCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class DefaultDoubleThresholdCondition implements IDoubleThreshold {
```

## Architecture & Concepts
The DefaultDoubleThresholdCondition is a foundational, immutable implementation of the IDoubleThreshold interface. Its primary architectural role is to provide a constant, pre-determined boolean result within systems that require a conditional check, such as procedural generation rule sets.

This class represents a terminal condition—a leaf node in a condition evaluation tree that does not depend on runtime input. It is used where an algorithm requires an IDoubleThreshold object, but the outcome is fixed, either for testing, debugging, or to simplify a generation rule by providing a permanent "pass" or "fail" state. It embodies the Null Object pattern for conditional logic, providing a non-null, inert object that satisfies an interface contract without performing any real computation.

The class provides two static, pre-allocated instances, DEFAULT_TRUE and DEFAULT_FALSE, for maximum performance and memory efficiency in common use cases.

## Lifecycle & Ownership
- **Creation:** The static instances DEFAULT_TRUE and DEFAULT_FALSE are instantiated once at class-loading time by the JVM. Dynamic instances are created via the public constructor, typically by a factory or configuration loader that assembles procedural generation rules from data.
- **Scope:** The static instances are global and persist for the entire lifetime of the application. The scope of a dynamically created instance is bound to the lifecycle of the object that holds a reference to it, such as a parent rule or configuration object.
- **Destruction:** Dynamically created instances are eligible for garbage collection when they are no longer referenced. The static instances are never collected.

## Internal State & Concurrency
- **State:** Immutable. The internal state consists of a single final boolean field, *result*, which is set exclusively at construction time. The object's state cannot be modified after creation.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. Instances can be safely shared and accessed by multiple threads without any need for external synchronization or locking mechanisms.

## API Surface
The public contract is minimal, focusing entirely on satisfying the IDoubleThreshold interface with a constant value.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| DefaultDoubleThresholdCondition(boolean) | constructor | O(1) | Creates a new instance with a fixed boolean result. |
| eval(double d) | boolean | O(1) | Ignores the input and returns the pre-configured boolean result. |
| eval(double d, double factor) | boolean | O(1) | Ignores all inputs and returns the pre-configured boolean result. |
| isResult() | boolean | O(1) | Returns the internal boolean state. |

## Integration Patterns

### Standard Usage
This component is intended to be used as a placeholder or a fixed condition within a larger system. The static instances should be preferred for common true/false cases.

```java
// Preferred usage for a condition that must always pass
IDoubleThreshold condition = DefaultDoubleThresholdCondition.DEFAULT_TRUE;

// The procedural engine can now evaluate this without any real computation
boolean outcome = condition.eval(0.75); // outcome is always true
```

### Anti-Patterns (Do NOT do this)
- **Redundant Instantiation:** Do not create new instances for constant true or false conditions. This is inefficient and defeats the purpose of the provided static singletons.
  - **BAD:** `IDoubleThreshold condition = new DefaultDoubleThresholdCondition(true);`
  - **GOOD:** `IDoubleThreshold condition = DefaultDoubleThresholdCondition.DEFAULT_TRUE;`
- **Expecting Dynamic Evaluation:** Do not use this class with the expectation that the *eval* methods will perform any logic based on their input parameters. This class is designed to be static and unresponsive to runtime values.

## Data Pipeline
The data flow for this component is trivial, as it intentionally short-circuits any computational pipeline.

> Flow:
> Input Value -> **DefaultDoubleThresholdCondition.eval()** -> Constant Boolean Result

