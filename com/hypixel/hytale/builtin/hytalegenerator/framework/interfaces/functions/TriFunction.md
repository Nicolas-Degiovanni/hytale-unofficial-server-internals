---
description: Architectural reference for TriFunction
---

# TriFunction

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.interfaces.functions
**Type:** Utility

## Definition
```java
// Signature
@FunctionalInterface
public interface TriFunction<A, B, C, R> {
```

## Architecture & Concepts
The TriFunction interface is a foundational component within the Hytale framework, designed to standardize and enable functional programming patterns. It serves as a generic contract for any operation that accepts three input arguments and produces a single result.

Its primary architectural purpose is to allow behavior to be passed as data. Instead of requiring a concrete class implementation for a simple operation, developers can provide a lambda expression or method reference that conforms to the TriFunction signature. This significantly reduces boilerplate code and promotes a more declarative style, especially in systems dealing with callbacks, strategy patterns, or data transformation pipelines where a three-parameter context is necessary.

This interface is analogous to the standard Java `Function` and `BiFunction` interfaces but extends the concept to three arguments, filling a common gap in the standard library.

## Lifecycle & Ownership
As a functional interface, TriFunction itself has no lifecycle. However, the *instances* (implementations, typically lambdas) that conform to it have a clearly defined scope and ownership.

- **Creation:** An instance of TriFunction is created at the point a lambda expression or method reference is defined. For example, `(a, b, c) -> a + b + c` creates a new TriFunction instance.
- **Scope:** The lifetime of a TriFunction instance is tied to the object that holds a reference to it. If it is a lambda passed as a method argument, it is typically short-lived and eligible for garbage collection after the method returns, unless stored elsewhere. If assigned to a field, it lives as long as the containing object.
- **Destruction:** The instance is destroyed by the Java Garbage Collector when it is no longer reachable. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The TriFunction interface is inherently stateless. However, implementations can be stateful. A lambda expression can capture variables from its enclosing scope (a closure), making its behavior dependent on that external state.

- **Thread Safety:** The interface itself imposes no concurrency constraints. The thread safety of a TriFunction instance is entirely dependent on its implementation.
    - **Stateless Lambdas:** Lambdas that operate solely on their input arguments without accessing shared, mutable state are inherently thread-safe.
    - **Stateful Lambdas (Closures):** If a lambda captures and modifies a non-thread-safe object from its enclosing scope, it is **not** thread-safe. Access to such an implementation from multiple threads must be externally synchronized.

**WARNING:** Exercise extreme caution when using TriFunction implementations that capture mutable state in a multi-threaded environment. The framework does not provide any implicit synchronization.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(A var1, B var2, C var3) | R | O(n) | Executes the function. Complexity is entirely dependent on the implementation. |

## Integration Patterns

### Standard Usage
TriFunction is intended to be used with lambda expressions to provide inline implementations for methods that expect a three-argument function.

```java
// Example: A method that uses TriFunction as a strategy
public <A, B, C, R> R processData(A a, B b, C c, TriFunction<A, B, C, R> strategy) {
    // ...validation or setup...
    return strategy.apply(a, b, c);
}

// Usage with a lambda
Integer result = processData(10, 20, 30, (x, y, z) -> x * y + z);
// result will be 230
```

### Anti-Patterns (Do NOT do this)
- **Complex Implementations:** Avoid creating large, stateful, named classes that implement TriFunction. If an operation is complex, it likely deserves its own dedicated service or class rather than being hidden behind this generic functional interface.
- **Blocking Operations:** Do not implement a TriFunction with long-running or blocking operations (e.g., file I/O, network requests) unless the calling system is explicitly designed for asynchronous execution. This can lead to performance bottlenecks and deadlocks.
- **Ignoring Type Erasure:** Be mindful of Java's type erasure. Creating complex generic hierarchies around TriFunction can lead to unreadable code and unexpected runtime casting errors.

## Data Pipeline
TriFunction does not represent a data pipeline on its own. Instead, it acts as a single, stateless **transformation node** or **processing step** within a larger data flow.

> Flow:
> Input Source -> ... -> **TriFunction (apply)** -> ... -> Output Sink

