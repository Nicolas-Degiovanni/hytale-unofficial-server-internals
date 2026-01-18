---
description: Architectural reference for ThrowableTriConsumer
---

# ThrowableTriConsumer

**Package:** com.hypixel.hytale.sneakythrow.consumer
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface ThrowableTriConsumer<T, U, V, E extends Throwable> extends TriConsumer<T, U, V> {
```

## Architecture & Concepts
ThrowableTriConsumer is a specialized functional interface designed to bridge the gap between Java's checked exception system and modern functional programming patterns used throughout the engine. Standard functional interfaces, such as Consumer or the engine's own TriConsumer, do not permit implementations that throw checked exceptions, forcing developers into verbose try-catch blocks within lambda bodies.

This interface subverts that restriction. It acts as an adapter, allowing a lambda or method reference that throws a checked exception to be seamlessly used in contexts that expect a standard, non-throwing TriConsumer.

The core mechanism relies on the **SneakyThrow** utility. The default implementation of the `accept` method wraps the user-provided logic (in `acceptNow`) and, upon catching any Throwable, uses SneakyThrow to re-cast and re-throw it as an unchecked exception. This bypasses compile-time exception checking, allowing the exception to propagate up the call stack where it can be handled by a higher-level error boundary.

**WARNING:** This pattern is powerful but must be used with caution. It effectively converts checked exceptions into runtime exceptions. The caller must be aware that an operation using a ThrowableTriConsumer can throw an exception at any time and should be wrapped in appropriate error-handling logic.

## Lifecycle & Ownership
As a functional interface, ThrowableTriConsumer does not have a traditional object lifecycle. Its lifetime is defined by the scope of its implementation.

- **Creation:** An instance is not created via a constructor. It is realized through a lambda expression or a method reference at a call site. For example: `(t, u, v) -> Files.write(...)`.
- **Scope:** The scope is typically ephemeral and tied to its context. When used as a method argument, it exists for the duration of that method call. If assigned to a field, it lives as long as the containing object.
- **Destruction:** The object is eligible for garbage collection as soon as it is no longer referenced. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, a specific implementation (i.e., a lambda) can be stateful if it captures variables from its enclosing scope (forming a closure). Implementations that close over mutable state are a significant source of complexity and potential bugs.
- **Thread Safety:** The interface contract is thread-safe. The thread safety of any given *implementation* is entirely the responsibility of the developer. If a lambda implementation captures and modifies shared mutable state without proper synchronization, it will not be thread-safe.

**WARNING:** Avoid capturing mutable state in lambdas passed to parallel processing systems. Prefer stateless implementations or those that close over immutable data.

## API Surface
The public contract consists of two methods: the standard entry point and the throwing implementation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T t, U u, V v) | void | O(impl) | The standard TriConsumer contract. Executes the underlying logic and re-throws any checked exception as an unchecked one. |
| acceptNow(T t, U u, V v) | void | O(impl) | The method to be implemented by the developer. This is where business logic that may throw a checked exception resides. |

## Integration Patterns

### Standard Usage
The primary use case is to pass a lambda that performs an I/O operation or other action with checked exceptions into a system that expects a standard TriConsumer.

```java
// A system that processes items using a standard TriConsumer
public void processAllItems(TriConsumer<Context, Item, Integer> processor) {
    // ... logic to iterate and call processor.accept(...)
}

// Using ThrowableTriConsumer to pass a method that throws IOException
// The cast to TriConsumer is implicit due to the interface hierarchy.
processAllItems((context, item, count) -> {
    // This call can throw a checked IOException
    Files.writeString(Paths.get(item.getName()), "data");
});
```

### Anti-Patterns (Do NOT do this)
- **Directly Calling acceptNow:** Calling `instance.acceptNow(...)` defeats the purpose of the interface. It forces you to handle the checked exception with a try-catch block, which is precisely what this pattern is designed to avoid. Always call the `accept` method.
- **Swallowing Exceptions:** Do not use this interface to simply ignore checked exceptions. The exception is still thrown and will terminate the thread if not handled at a higher level in the call stack.
- **Stateful Lambdas in Concurrent Code:** Implementing this interface with a lambda that modifies a shared, non-thread-safe collection or object and passing it to a parallel stream or async executor will lead to race conditions and data corruption.

## Data Pipeline
ThrowableTriConsumer is not part of a data pipeline but rather a control-flow and exception-handling pipeline.

> Flow:
> High-Level System invokes `accept` -> Default method wrapper executes `acceptNow` -> Implementation of `acceptNow` throws a checked `Exception` -> The `catch` block intercepts it -> `SneakyThrow.sneakyThrow` is called -> The original `Exception` is re-thrown as an unchecked exception up the call stack.

