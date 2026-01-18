---
description: Architectural reference for ThrowableIntConsumer
---

# ThrowableIntConsumer

**Package:** com.hypixel.hytale.sneakythrow.consumer
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface ThrowableIntConsumer<E extends Throwable> extends IntConsumer {
```

## Architecture & Concepts
ThrowableIntConsumer is a specialized functional interface that serves as a critical component in Hytale's exception handling strategy, particularly within functional programming paradigms like Java Streams. It extends the standard Java IntConsumer, enhancing it with the ability to handle checked exceptions without verbose try-catch blocks inside lambda expressions.

The core architectural purpose of this interface is to bridge the gap between Java's strict checked exception system and the design of its functional interfaces, which do not declare throwing checked exceptions in their method signatures. It achieves this by leveraging the **SneakyThrow** utility. When the implemented logic in the acceptNow method throws any Throwable, the default accept method catches it and re-throws it as an unchecked exception. This "sneaky" throw propagates up the call stack, bypassing compiler checks and allowing for cleaner, more readable functional code pipelines, especially when dealing with I/O or other exception-prone operations.

This component is fundamental to maintaining code clarity in systems that process collections or streams of data where each element's processing might fail.

## Lifecycle & Ownership
As a functional interface, ThrowableIntConsumer does not have a managed lifecycle in the same way a service or component does. Its existence is typically ephemeral and lexically scoped.

- **Creation:** Instances are created on-the-fly, most commonly as a lambda expression or a method reference passed as an argument to a higher-order function (e.g., a stream operator). It is not instantiated via dependency injection or a factory.
- **Scope:** The instance exists for the duration of the method call it is passed to. It is a short-lived object whose scope is tied to the execution of the operation that consumes it.
- **Destruction:** The object is eligible for garbage collection as soon as the operation completes and no other references to it exist.

## Internal State & Concurrency
- **State:** This interface is inherently stateless. It defines a behavior, not data. Any state it operates on is provided via its arguments or captured from its enclosing lexical scope (i.e., it can be a closure).
- **Thread Safety:** The interface itself is thread-safe. However, the thread safety of a specific *implementation* (the lambda body) is entirely the responsibility of the developer. If the lambda captures and modifies shared mutable state without proper synchronization, it will not be thread-safe.

**WARNING:** When using this interface in parallel streams or multi-threaded contexts, ensure the implementation logic is re-entrant and does not cause race conditions on shared data.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(int t) | void | O(N) | Default method implementing the IntConsumer contract. Wraps the call to acceptNow, catching and re-throwing any exception via SneakyThrow. Complexity is determined by the acceptNow implementation. |
| acceptNow(int var1) | void | O(N) | The abstract method to be implemented by the developer. This is where business logic that may throw a checked exception resides. |

## Integration Patterns

### Standard Usage
This interface is intended for use in places where a standard IntConsumer is required, but the operation may throw a checked exception. A common use case is within a stream processing pipeline.

```java
// A method that can throw a checked exception
void writeToIntFile(int number) throws java.io.IOException {
    // ... I/O logic
}

// Using the interface to adapt the method for a stream
public void processNumbers() {
    try {
        java.util.stream.IntStream.of(1, 2, 3)
            .forEach((ThrowableIntConsumer<java.io.IOException>) this::writeToIntFile);
    } catch (Exception e) {
        // The sneaky exception is caught here
        log.error("Failed to process numbers", e);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Directly Calling acceptNow:** Do not invoke the `acceptNow` method directly. This bypasses the sneaky-throw mechanism, forcing you to handle the checked exception with a standard try-catch block and defeating the entire purpose of the interface.
- **Swallowing Exceptions:** The sneaky-throw mechanism does not eliminate exceptions; it merely changes how they are propagated. Failing to have a higher-level catch block to handle these runtime-wrapped exceptions will result in silent failures or application crashes. Always wrap stream operations that use this interface in a try-catch block.

## Data Pipeline
ThrowableIntConsumer acts as a control-flow adapter rather than a data-flow component. Its primary role is to manage the exception propagation path.

> Exception Flow:
> 1. A method (e.g., `IntStream.forEach`) invokes the `accept(int)` method on the provided lambda.
> 2. The default `accept` method immediately calls the user-implemented `acceptNow(int)`.
> 3. An operation inside `acceptNow` throws a checked exception (e.g., IOException).
> 4. The `try-catch` block within the `accept` method catches the Throwable.
> 5. The caught exception is passed to `SneakyThrow.sneakyThrow()`.
> 6. `SneakyThrow` re-throws the exception as an unchecked one.
> 7. The unchecked exception propagates up the call stack, terminating the stream operation and unwinding to the nearest `catch` block capable of handling it.

