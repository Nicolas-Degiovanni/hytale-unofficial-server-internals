---
description: Architectural reference for ThrowableFunction
---

# ThrowableFunction

**Package:** com.hypixel.hytale.sneakythrow.function
**Type:** Utility Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface ThrowableFunction<T, R, E extends Throwable> extends Function<T, R> {
```

## Architecture & Concepts
ThrowableFunction is a specialized functional interface designed to bridge the gap between modern functional programming patterns (like Java Streams) and legacy or I/O-bound code that throws checked exceptions.

Standard Java functional interfaces, such as `java.util.function.Function`, do not have a `throws` clause in their method signatures. This forces developers to wrap any lambda body that calls an exception-throwing method in a verbose `try-catch` block, often breaking the fluency of stream pipelines.

This interface elegantly solves this problem by extending the standard `Function` interface and providing a default implementation of the `apply` method. This implementation delegates to a new abstract method, `applyNow`, which *is* permitted to throw a checked exception. The default `apply` method catches any `Throwable` from `applyNow` and re-throws it as an unchecked exception using the `SneakyThrow` utility. This technique preserves the original stack trace and exception type while satisfying the compiler, allowing exception-throwing methods to be used seamlessly in contexts that expect a standard `Function`.

It is a foundational, low-level component for promoting cleaner code and reducing boilerplate exception handling.

### Lifecycle & Ownership
As a functional interface, ThrowableFunction does not have a managed lifecycle in the same way a service or component does. Its instances are typically implemented as lambdas or method references.

- **Creation:** An instance is created ad-hoc by the developer, either as a lambda expression or a method reference. It is not instantiated by a dependency injection container or a factory.
- **Scope:** The lifetime of a ThrowableFunction instance is tied to the scope in which it is defined. It may be a local variable, a class field, or an anonymous object passed as a method argument.
- **Destruction:** The instance is eligible for garbage collection as soon as it is no longer referenced, following standard Java memory management rules.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, an implementing lambda can be stateful if it closes over (captures) non-final or mutable variables from its enclosing scope. Stateful implementations can introduce significant complexity and are discouraged unless absolutely necessary.

- **Thread Safety:** The interface contract is inherently thread-safe. The thread safety of a specific *implementation* is entirely dependent on the code provided within the lambda. If a lambda accesses shared, mutable state, it is the developer's responsibility to ensure proper synchronization. The `SneakyThrow` mechanism itself is thread-safe.

## API Surface
The public contract is designed for implementing the sneaky throw pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(T t) | R | O(1) + Impl | **(Entry Point)** Implements `java.util.function.Function`. Wraps the call to `applyNow` and re-throws any checked exception as an unchecked one. |
| applyNow(T var1) | R | Impl | **(Implementation Target)** The abstract method to be implemented by a lambda. This is where the business logic that may throw a checked exception resides. |

## Integration Patterns

### Standard Usage
The primary use case is to adapt a method that throws a checked exception for use within a Java Stream pipeline or any other API that expects a standard `Function`.

```java
// How a developer should normally use this
List<String> filePaths = List.of("file1.txt", "file2.txt");

// Without ThrowableFunction, this would require a messy try-catch block
List<String> contents = filePaths.stream()
    .map((ThrowableFunction<String, String, IOException>) path -> Files.readString(Path.of(path)))
    .collect(Collectors.toList());
```

### Anti-Patterns (Do NOT do this)
- **Directly Calling applyNow:** Calling `instance.applyNow(t)` is possible, but it completely bypasses the sneaky throw mechanism. This forces you to handle the checked exception with a `try-catch` block, defeating the entire purpose of the interface. Always call the `apply` method.
- **Wrapping Unchecked Exceptions:** This interface is designed for checked exceptions. While it will work for runtime exceptions, it is redundant and adds unnecessary overhead. Standard lambdas can already throw unchecked exceptions without issue.

## Data Pipeline
The flow of data and control through this component is centered on its exception-handling strategy.

> Flow:
> Input `T` -> `apply(T)` method call -> `applyNow(T)` implementation -> (Success) -> Return `R`
>
> Flow (Exception):
> Input `T` -> `apply(T)` method call -> `applyNow(T)` implementation -> Throws `E` -> `catch (Throwable)` block -> `SneakyThrow.sneakyThrow(E)` -> Unchecked exception propagates up the call stack.

