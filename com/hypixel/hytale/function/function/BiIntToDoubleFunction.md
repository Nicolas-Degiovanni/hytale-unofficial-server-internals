---
description: Architectural reference for BiIntToDoubleFunction
---

# BiIntToDoubleFunction

**Package:** com.hypixel.hytale.function.function
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface BiIntToDoubleFunction {
   double apply(int var1, int var2);
}
```

## Architecture & Concepts
BiIntToDoubleFunction is a specialized functional interface that serves as a high-performance contract for operations that consume two primitive integer inputs and produce a primitive double output. Its primary architectural role is to enable the implementation of the **Strategy Pattern** for mathematical or logical operations without the performance overhead associated with autoboxing and unboxing standard Java generic functional interfaces like `BiFunction<Integer, Integer, Double>`.

This interface is a foundational primitive, likely used in performance-critical systems such as the physics engine, rendering pipeline calculations, or complex game logic where heap allocations and type conversions in tight loops must be aggressively avoided. By operating directly on primitives, it ensures minimal overhead and predictable performance characteristics.

## Lifecycle & Ownership
As an interface, BiIntToDoubleFunction itself has no lifecycle. The lifecycle pertains to its *implementations*, which are typically provided as lambda expressions or method references.

- **Creation:** Implementations are created ad-hoc at the call site, usually via a lambda expression (`(x, y) -> ...`) or a method reference (`Math::pow`). They are not managed by a dependency injection container or a central factory.
- **Scope:** The lifetime of an implementation instance is bound to the object that holds a reference to it. If passed as a method argument, its scope is ephemeral and limited to that method's execution. If stored in a field, it lives as long as the containing object.
- **Destruction:** Implementations are subject to standard Java Garbage Collection. They are destroyed once they are no longer reachable from any GC roots.

## Internal State & Concurrency
- **State:** The interface contract is inherently stateless. However, implementations can be stateful if they are defined as a class that holds member variables or as a lambda that closes over (captures) mutable variables from its enclosing scope.

    **Warning:** Creating stateful implementations, especially those that capture mutable state, is a significant code smell. It violates the principle of functional purity and can lead to unpredictable behavior and severe concurrency issues.

- **Thread Safety:** The interface itself is thread-agnostic. The thread safety of any given implementation is the sole responsibility of the developer providing it.
    - **Safe:** Pure functions that rely only on their inputs are inherently thread-safe. Example: `(x, y) -> (double)x / y;`
    - **Unsafe:** Implementations that modify shared state (e.g., incrementing a shared counter, modifying a collection) are not thread-safe and require external synchronization, which defeats the performance purpose of this interface.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(int var1, int var2) | double | O(N) | Executes the encapsulated logic. Complexity is entirely dependent on the specific implementation provided by the developer. |

## Integration Patterns

### Standard Usage
This interface is intended to be used as a method parameter, allowing callers to inject custom, high-performance logic into a system.

```java
// A system that uses the interface to calculate distance
public class PhysicsEngine {
    public double calculate(int x1, int y1, BiIntToDoubleFunction formula) {
        // ... complex setup
        return formula.apply(x1, y1);
    }
}

// Usage:
PhysicsEngine engine = new PhysicsEngine();
// Provide a lambda implementation for Euclidean distance
double distance = engine.calculate(10, 20, (x, y) -> Math.sqrt(x*x + y*y));
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not write lambdas that modify external state. This creates hidden side effects that are difficult to debug and paralyze concurrent execution.
    ```java
    // BAD: This lambda has a side effect and is not thread-safe
    AtomicInteger callCounter = new AtomicInteger(0);
    engine.calculate(10, 20, (x, y) -> {
        callCounter.incrementAndGet(); // Modifies external state
        return (double) x / y;
    });
    ```
- **Null Arguments:** Passing a null reference where a BiIntToDoubleFunction is expected will result in a `NullPointerException` when the `apply` method is invoked. Systems using this interface should perform null-checks if the function is optional.

## Data Pipeline
This interface acts as a pure transformation node within a data flow. It does not orchestrate a pipeline but rather serves as a single, atomic step within one.

> Flow:
> `int`, `int` -> **BiIntToDoubleFunction.apply()** -> `double`

