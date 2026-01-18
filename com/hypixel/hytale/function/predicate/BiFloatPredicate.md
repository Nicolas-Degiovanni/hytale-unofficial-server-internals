---
description: Architectural reference for BiFloatPredicate
---

# BiFloatPredicate

**Package:** com.hypixel.hytale.function.predicate
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface BiFloatPredicate {
   boolean test(float var1, float var2);
}
```

## Architecture & Concepts
BiFloatPredicate is a specialized, performance-oriented functional interface. It serves as a core engine primitive for representing a boolean-valued function that accepts two primitive float arguments.

Its primary architectural purpose is to eliminate the overhead associated with autoboxing and unboxing primitive types. In performance-sensitive domains such as the rendering pipeline, physics engine, or high-frequency game logic, the use of the generic `java.util.function.BiPredicate<Float, Float>` would introduce significant garbage collector pressure and performance degradation due to the creation of wrapper Float objects. BiFloatPredicate operates directly on the primitive stack values, ensuring maximum execution speed.

This interface defines a contract for behavior, allowing systems to be parameterized with custom logic. It is a fundamental building block for creating flexible and efficient APIs that need to filter, query, or test pairs of floating-point values without dictating the specific implementation of the test itself.

## Lifecycle & Ownership
As a functional interface, BiFloatPredicate itself does not have a lifecycle. Instead, the lifecycle pertains to the *implementing instances*, which are typically lambda expressions or method references.

-   **Creation:** An implementation is created at the point where a lambda expression or method reference matching the interface's signature is declared. This can be as a method argument, a local variable, or a class field.
-   **Scope:** The lifetime of a BiFloatPredicate instance is determined by its referencing context.
    -   **Method-Scoped:** An instance passed directly as a method argument is eligible for garbage collection after the method returns, unless it is stored elsewhere.
    -   **Object-Scoped:** An instance assigned to an object's field will live as long as the containing object.
-   **Destruction:** Instances are managed by the Java Garbage Collector. They are destroyed when they are no longer reachable from any GC roots.

## Internal State & Concurrency
-   **State:** The interface is inherently stateless. However, an implementing lambda expression can be stateful if it *captures* variables from its enclosing scope.
    -   **Stateless (Non-capturing):** A lambda like `(f1, f2) -> f1 > f2` has no state and is intrinsically reusable and safe.
    -   **Stateful (Capturing):** A lambda like `(f1, f2) -> f1 > someInstanceField` captures the enclosing `this` reference, making its behavior dependent on the state of that object.

-   **Thread Safety:** The interface contract is thread-safe. The thread safety of a specific *implementation* is entirely dependent on its internal logic and captured state.
    -   **Safe:** Stateless implementations are always thread-safe.
    -   **Unsafe:** Stateful implementations that capture and mutate shared, non-thread-safe state are not safe for concurrent use without external synchronization.

    **WARNING:** Exercise extreme caution when using stateful predicates in parallelized systems. Unsynchronized access to captured state from multiple threads is a common source of race conditions and data corruption.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(float var1, float var2) | boolean | O(N) | Evaluates the predicate logic on the two float arguments. The computational complexity is determined by the user-provided implementation. |

## Integration Patterns

### Standard Usage
BiFloatPredicate is intended to be used as a parameter for methods that need to perform conditional logic on pairs of floats. This decouples the algorithm from the specific condition it tests.

```java
// A system method that accepts a predicate to filter coordinates
public List<Vector2> findPointsWithinThreshold(List<Vector2> points, BiFloatPredicate predicate) {
    List<Vector2> results = new ArrayList<>();
    for (Vector2 p : points) {
        if (predicate.test(p.x, p.y)) {
            results.add(p);
        }
    }
    return results;
}

// Usage: Pass a lambda implementing the desired logic
float maxDistance = 100.0f;
BiFloatPredicate distanceCheck = (x, y) -> (x * x + y * y) < (maxDistance * maxDistance);
List<Vector2> nearbyPoints = findPointsWithinThreshold(allPoints, distanceCheck);
```

### Anti-Patterns (Do NOT do this)
-   **Using Generic BiPredicate:** Never use `BiPredicate<Float, Float>` in performance-critical code paths. The resulting autoboxing will severely impact performance and increase memory churn. This interface exists specifically to prevent that.
-   **Complex or Blocking Logic:** Avoid embedding long-running calculations, I/O operations, or complex business logic within the predicate's implementation. Predicates are often executed in tight loops, and any significant latency will degrade engine performance.
-   **Stateful Predicates in Parallel Operations:** Do not use predicates that modify captured state when operating on parallel streams or within multi-threaded job systems. This will lead to non-deterministic behavior and severe concurrency bugs.

## Data Pipeline
BiFloatPredicate is not a data pipeline stage itself, but rather a behavioral component *injected into* a pipeline to act as a filter or gate. It allows a generic processing stage to make a boolean decision based on incoming data.

> Flow:
> Data Source (e.g., Physics Engine State) -> Filtering Stage (using a **BiFloatPredicate**) -> Consumer (e.g., Renderer or AI System)

