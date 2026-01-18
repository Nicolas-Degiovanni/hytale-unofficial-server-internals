---
description: Architectural reference for TriFunction
---

# TriFunction

**Package:** com.hypixel.hytale.function.function
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface TriFunction<T, U, V, R> {
```

## Architecture & Concepts
TriFunction is a foundational utility interface that codifies a core functional programming concept: a function that accepts three arguments and produces a result. It is analogous to the standard Java Development Kit interfaces like Function and BiFunction, but extended for three parameters.

Within the Hytale engine, this interface is not tied to a specific subsystem like rendering or networking. Instead, it serves as a generic, reusable contract for callbacks, strategy patterns, and data transformation pipelines across the entire codebase. Its primary purpose is to enable the use of lambda expressions and method references, which significantly reduces boilerplate code for single-method implementations and improves code readability.

By providing a standardized signature for three-argument functions, it allows disparate systems to pass behavior as data, a cornerstone of modern, decoupled software architecture.

## Lifecycle & Ownership
As a functional interface, TriFunction itself has no lifecycle. The concepts of creation, scope, and destruction apply to the *implementations* of this interface, which are typically lambda expressions or anonymous inner classes.

- **Creation:** An implementation of TriFunction is created at the point where a lambda expression or method reference matching its signature is declared. This object is instantiated by the Java Virtual Machine.
- **Scope:** The lifetime of a TriFunction instance is determined by the scope of the variable or field it is assigned to. If assigned to a local variable, it is eligible for garbage collection when the method returns. If assigned to a field of a long-lived service, it will persist for the lifetime of that service.
- **Destruction:** The instance is destroyed by the Java Garbage Collector when no more strong references to it exist.

**WARNING:** Be mindful of lambdas capturing references to other objects (closing over variables). This can unintentionally extend the lifetime of those captured objects, potentially leading to memory leaks if the TriFunction instance is held by a long-lived component.

## Internal State & Concurrency
- **State:** The TriFunction interface is inherently stateless. However, its implementations can be stateful if the lambda expression captures and/or modifies variables from its enclosing scope. Such stateful lambdas behave like a class with instance fields.
- **Thread Safety:** The interface contract is not thread-safe. The thread safety of any given TriFunction instance is entirely dependent on its implementation. A lambda that is purely functional (i.e., has no side effects and does not modify shared state) is inherently thread-safe. Conversely, a lambda that modifies non-local, non-thread-safe state is **not** thread-safe and must be synchronized externally if used in a concurrent context.

**WARNING:** Assume no implementation is thread-safe unless explicitly documented as such. Passing stateful, non-synchronized TriFunction implementations to multi-threaded systems like a task scheduler is a common source of severe concurrency bugs.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(T, U, V) | R | Implementation-Defined | Executes the function. The complexity is entirely dependent on the code provided in the implementation. |

## Integration Patterns

### Standard Usage
TriFunction is most commonly used to pass behavior as a parameter or to store a unit of logic in a field.

```java
// Example: Defining a function to combine three values into a formatted string.
TriFunction<Integer, String, Boolean, String> formatter = 
    (id, name, isActive) -> String.format("User[%d]: %s (Active: %b)", id, name, isActive);

// Invoking the function
String result = formatter.apply(101, "PlayerOne", true);
// result -> "User[101]: PlayerOne (Active: true)"
```

### Anti-Patterns (Do NOT do this)
- **Stateful Lambdas in Concurrent Code:** Never use a lambda that modifies shared, unsynchronized state from a concurrent execution context. This will introduce race conditions.
    ```java
    // ANTI-PATTERN: This is not thread-safe.
    List<String> sharedList = new ArrayList<>();
    TriFunction<String, String, String, Boolean> unsafeAdder = (a, b, c) -> {
        sharedList.add(a); // Race condition if called from multiple threads
        return true;
    };
    ```
- **Ignoring Nulls:** The interface contract does not enforce null-safety on its parameters or return value. Implementations must be defensively coded to handle potential null inputs unless the calling context guarantees non-null values.

## Data Pipeline
TriFunction is a generic component used to define a single step within a larger data pipeline. It does not manage a pipeline itself but acts as a building block.

> Conceptual Flow:
> Data Source -> Transformation Step 1 -> **TriFunction Implementation (e.g., Aggregation)** -> Transformation Step 2 -> Data Sink

