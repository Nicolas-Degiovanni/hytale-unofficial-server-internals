---
description: Architectural reference for BiDoubleToDoubleFunction
---

# BiDoubleToDoubleFunction

**Package:** com.hypixel.hytale.function.function
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface BiDoubleToDoubleFunction {
   double apply(double var1, double var3);
}
```

## Architecture & Concepts
The BiDoubleToDoubleFunction is a specialized functional interface designed for high-performance mathematical and logical operations. It serves as a contract for any function that accepts two primitive double arguments and produces a primitive double result.

Its primary architectural role is to eliminate the performance overhead associated with autoboxing and unboxing. The standard Java `java.util.function.BiFunction<Double, Double, Double>` operates on object wrappers (Double), which can lead to significant garbage collection pressure in tight loops, such as those found in the physics engine, rendering pipeline, or complex game logic calculations. By operating directly on primitives, this interface ensures that calculations remain on the stack and avoid heap allocations.

This interface is a foundational building block, intended to be implemented by stateless lambda expressions or method references at the point of use.

### Lifecycle & Ownership
As a functional interface, BiDoubleToDoubleFunction does not have a lifecycle in the traditional sense of a managed service. Instead, the lifecycle pertains to its *implementations*.

- **Creation:** Implementations are typically created ad-hoc via lambda expressions (e.g., `(x, y) -> x * y`) or method references (e.g., `Math::pow`). They are instantiated on-demand by the system that requires the specific logic.
- **Scope:** The lifetime of an implementation is bound to the object that holds a reference to it. It may be a short-lived local variable within a method or a long-lived field in a configuration object.
- **Destruction:** An implementation is eligible for garbage collection as soon as it is no longer referenced. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Implementations should, by convention, also be stateless and represent pure functions. However, it is possible for a lambda to be stateful by capturing variables from its enclosing scope (creating a closure). This is strongly discouraged as it can lead to unpredictable behavior.
- **Thread Safety:** The interface contract is inherently thread-safe. The thread safety of a given implementation is entirely dependent on its logic. A stateless implementation, such as one performing a simple mathematical operation, is fully thread-safe and can be shared across threads without synchronization.

**WARNING:** Implementations that capture and modify shared mutable state are **not** thread-safe and must be properly synchronized by the developer. This practice violates the intended use of this interface.

## API Surface
The public contract consists of a single method to be implemented.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(double, double) | double | O(N) | Executes the defined function. The complexity N is determined entirely by the implementation provided. |

## Integration Patterns

### Standard Usage
This interface is intended to be used with lambda expressions for providing custom, high-performance logic to other systems.

```java
// A system that combines two values using a provided strategy
public class PhysicsUtil {
    public static double combineVelocities(double v1, double v2, BiDoubleToDoubleFunction strategy) {
        return strategy.apply(v1, v2);
    }
}

// Usage: Provide the combination logic via a lambda
double resultantVelocity = PhysicsUtil.combineVelocities(10.5, 22.1, (a, b) -> a + b);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Avoid creating implementations that rely on or modify external state. This breaks the functional paradigm, introduces side effects, and creates significant challenges for concurrency and debugging.
- **Null References:** Passing a null reference where a BiDoubleToDoubleFunction is expected will result in a NullPointerException at the call site. Always ensure a valid function implementation is provided.

## Data Pipeline
This interface is not a component in a data pipeline but rather the logic that drives a single transformation step within one. It represents a pure processing node.

> Flow:
> Two `double` values -> **BiDoubleToDoubleFunction.apply()** -> Resulting `double` value

