---
description: Architectural reference for DoubleThresholdCondition
---

# DoubleThresholdCondition

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Transient

## Definition
```java
// Signature
public class DoubleThresholdCondition implements IDoubleCondition {
```

## Architecture & Concepts
The DoubleThresholdCondition class is a fundamental component within the procedural generation library, acting as an adapter that implements the IDoubleCondition interface. Its sole architectural purpose is to delegate a boolean evaluation to a specialized thresholding object.

This class embodies the **Strategy Pattern**. It decouples the generic concept of a "condition" from the specific logic of a "threshold". By accepting any object implementing the IDoubleThreshold interface, it allows developers to inject various evaluation strategies (e.g., greater than, less than, within a range) without altering the condition-checking system itself. This promotes a highly composable and extensible design for building complex procedural rules.

It is a lightweight, value-like object intended to be composed into larger, more complex rule sets.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its public constructor, typically by a factory or builder responsible for assembling a procedural generation rule graph. It is not a managed service.
- **Scope:** The lifetime of a DoubleThresholdCondition instance is strictly bound to its owner, such as a parent rule or a generator configuration. These are expected to be short-lived objects.
- **Destruction:** The object is reclaimed by the Java garbage collector when its owning object is dereferenced. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state is defined by the injected IDoubleThreshold object, which is stored in a final field. This reference cannot be changed after construction, guaranteeing that the condition's behavior is fixed for its entire lifetime.
- **Thread Safety:** **Conditionally Thread-Safe**. The class itself introduces no state and is inherently thread-safe. However, its overall thread safety is entirely dependent on the injected IDoubleThreshold implementation. If the provided threshold object is thread-safe, this condition can be safely evaluated by multiple threads concurrently.

**WARNING:** Injecting a stateful, non-thread-safe IDoubleThreshold implementation is a critical anti-pattern and will lead to unpredictable behavior in a multi-threaded generation environment.

## API Surface
The public contract is minimal, focusing exclusively on the evaluation logic defined by the IDoubleCondition interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| eval(double value) | boolean | O(1) | Evaluates the input value against the configured threshold. The complexity is technically dependent on the injected threshold, but is constant for standard comparisons. |

## Integration Patterns

### Standard Usage
The class is designed to be instantiated with a concrete threshold implementation and used as part of a larger evaluation chain.

```java
// Assume GreaterThanThreshold is an implementation of IDoubleThreshold
IDoubleThreshold minHeight = new GreaterThanThreshold(64.0);
IDoubleCondition condition = new DoubleThresholdCondition(minHeight);

// This condition can now be used by a procedural generator
boolean isHighEnough = condition.eval(currentHeight);
```

### Anti-Patterns (Do NOT do this)
- **Null Injection:** The constructor does not perform null checks. Passing a null IDoubleThreshold will not fail at construction but will cause a NullPointerException during the first call to eval. Always ensure a valid threshold is provided.
- **Stateful Thresholds:** Do not inject an IDoubleThreshold implementation that modifies its internal state during an eval call. This violates the principle of referential transparency and can cause severe, difficult-to-diagnose bugs, especially in concurrent environments.

## Data Pipeline
The data flow through this component is direct and stateless. It acts as a simple pass-through evaluator.

> Flow:
> Input Value (double) -> **DoubleThresholdCondition.eval()** -> Injected IDoubleThreshold.eval() -> Boolean Result

