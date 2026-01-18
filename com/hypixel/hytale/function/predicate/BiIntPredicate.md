---
description: Architectural reference for BiIntPredicate
---

# BiIntPredicate

**Package:** com.hypixel.hytale.function.predicate
**Type:** Functional Interface / Contract

## Definition
```java
// Signature
@FunctionalInterface
public interface BiIntPredicate {
   boolean test(int var1, int var2);
}
```

## Architecture & Concepts
BiIntPredicate is a specialized functional interface designed for high-performance predicate logic involving two primitive integer arguments. It serves as a direct, allocation-free replacement for the standard Java `BiPredicate<Integer, Integer>`.

In the engine's architecture, this interface is a fundamental building block for performance-critical systems. Its primary purpose is to eliminate the overhead of autoboxing and unboxing `int` primitives into `Integer` objects, which can cause significant garbage collector pressure when executed in tight loops. It is heavily utilized in systems that perform frequent checks on integer pairs, such as:
- **World Generation:** Comparing chunk or block coordinates.
- **Entity Systems:** Evaluating relationships between two entity IDs.
- **Pathfinding:** Testing grid positions during A* or other algorithm executions.
- **UI & Inventory:** Validating slot indices or grid layouts.

By providing a contract for a function that takes two integers and returns a boolean, the engine can pass around behavior (the predicate logic) as if it were data, a core tenet of functional programming.

## Lifecycle & Ownership
As a functional interface, BiIntPredicate itself has no lifecycle. The lifecycle described here pertains to its *implementations*, which are most often lambda expressions or method references.

- **Creation:** Implementations are typically instantiated inline at the call site where a method requires a BiIntPredicate. They are lightweight objects created on the fly.
- **Scope:** The lifetime of a lambda instance is generally ephemeral. It is scoped to the execution of the method that accepts it and is eligible for garbage collection as soon as it is no longer referenced.
- **Destruction:** Managed entirely by the Java Garbage Collector. Due to their short-lived nature, they are almost always collected quickly in the young generation heap space.

## Internal State & Concurrency
- **State:** The interface is inherently stateless. However, an implementation (a lambda) can become stateful if it forms a closure by capturing variables from its enclosing scope.

- **Thread Safety:** The interface contract is thread-safe. The thread safety of a specific *implementation* is **not guaranteed** and is a critical developer responsibility.
    - **Stateless Lambdas:** Lambdas that do not capture any external state are intrinsically thread-safe. Example: `(x, y) -> x > y`.
    - **Stateful Lambdas (Closures):** If a lambda captures a reference to a mutable object from a surrounding scope, its thread safety is determined by the thread safety of that captured object. Accessing such a lambda from multiple threads without external synchronization can lead to severe race conditions.

    **WARNING:** Exercise extreme caution when using stateful BiIntPredicate implementations in parallel streams or multi-threaded game systems.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(int var1, int var2) | boolean | O(N) | Evaluates this predicate on the given arguments. Complexity is determined by the lambda implementation. |

## Integration Patterns

### Standard Usage
The primary integration pattern is to provide a lambda expression to any method that accepts a BiIntPredicate. This allows for concise, inline definitions of conditional logic.

```java
// Example: Finding a position in a grid where coordinates are equal
public Position findDiagonalPosition(Grid grid) {
    // The lambda (x, y) -> x == y is an implementation of BiIntPredicate
    return grid.findFirst( (x, y) -> x == y );
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Primitive Specialization:** Do not use the generic `BiPredicate<Integer, Integer>` in performance-sensitive code. The resulting object boxing and unboxing will degrade performance and increase GC load. Always prefer BiIntPredicate for primitive integers.
- **Overly Complex Lambdas:** Avoid embedding multi-line, complex business logic directly within a lambda. This harms readability, debuggability, and testability. If the logic is more than a single expression, refactor it into a private helper method and use a method reference.
- **Unsynchronized Stateful Closures:** Never pass a lambda that captures and modifies shared, mutable state into a parallel or asynchronous system without explicit synchronization. This is a direct path to data corruption and non-deterministic bugs.

## Data Pipeline
BiIntPredicate is not a data processing component in a pipeline; it is a **control flow component**. It acts as a gate or a decision point, directing the flow of logic based on its boolean result.

> Flow:
> Input Integers (e.g., Coordinates, IDs) -> **BiIntPredicate.test()** -> Boolean Result -> Conditional Execution Path (e.g., If/Else Block, Stream Filter)

