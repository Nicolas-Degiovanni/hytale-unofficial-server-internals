---
description: Architectural reference for ThrowableBiConsumer
---

# ThrowableBiConsumer

**Package:** com.hypixel.hytale.sneakythrow.consumer
**Type:** Utility Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface ThrowableBiConsumer<T, U, E extends Throwable> extends BiConsumer<T, U> {
```

## Architecture & Concepts
ThrowableBiConsumer is a specialized functional interface designed to bridge the gap between Java's checked exception system and its standard functional interfaces. It extends the native java.util.function.BiConsumer, allowing lambda expressions that perform operations capable of throwing checked exceptions to be used seamlessly where a standard BiConsumer is expected.

This component is a foundational element of the engine's *sneaky throw* pattern. Its primary architectural role is to eliminate boilerplate try-catch blocks at the call site of functional operations, particularly within streams or asynchronous task chains. It achieves this by catching any thrown exception within its default *accept* method and re-throwing it as an unchecked exception via the SneakyThrow utility. This forces exception handling to a higher-level, centralized error management system, promoting cleaner and more readable functional code.

**WARNING:** This utility intentionally subverts Java's checked exception mechanism. It should only be used when exceptions are expected to be handled by a top-level, application-wide uncaught exception handler.

## Lifecycle & Ownership
As a functional interface, ThrowableBiConsumer does not have a traditional object lifecycle. Instead, its lifecycle is tied to the specific implementation, which is almost always a lambda expression or method reference.

- **Creation:** An instance is implicitly created by the Java runtime when a developer provides a lambda expression or method reference to a method that accepts a ThrowableBiConsumer. For example: `(t, u) -> Files.write(t, u)`.
- **Scope:** The scope of a ThrowableBiConsumer instance is ephemeral and is determined by the scope of the lambda that implements it. It may be stack-local for a single method call or exist as a field within a longer-lived object.
- **Destruction:** The instance is eligible for garbage collection as soon as it is no longer referenced, typically after the method it was passed to completes execution.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Any state is derived from the closure of the implementing lambda; that is, the variables from the enclosing scope that the lambda captures.
- **Thread Safety:** The interface contract is inherently thread-safe. However, the thread safety of any specific *implementation* is entirely dependent on the logic defined within the lambda and the state it captures. If the lambda's implementation of *acceptNow* modifies shared, mutable state without proper synchronization, it will not be thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T t, U u) | void | O(N) | Implements the standard BiConsumer contract. Wraps the call to *acceptNow* and re-throws any exception as an unchecked one. Complexity is dependent on the *acceptNow* implementation. |
| acceptNow(T var1, U var2) | void | O(N) | **Abstract method.** This is the contract that developers must implement. It contains the core logic that may throw a checked exception of type E. |

## Integration Patterns

### Standard Usage
This interface is intended for use in higher-order functions that operate on collections or streams, where individual operations might fail with a checked exception (e.g., IO operations).

```java
// Example: Writing multiple files using a stream, where Files.write
// throws a checked IOException.
public void writeAll(Map<Path, byte[]> filesToWrite) {
    // The forEach method expects a BiConsumer. Our lambda, which throws
    // a checked exception, is adapted by ThrowableBiConsumer.
    filesToWrite.forEach((path, bytes) -> {
        // This lambda implicitly creates a ThrowableBiConsumer instance.
        // The potential IOException from Files.write is handled internally.
        Files.write(path, bytes);
    });
}
```

### Anti-Patterns (Do NOT do this)
- **Ignoring the Thrown Exception:** Implementing *acceptNow* with an empty catch block or a block that only logs the error defeats the purpose of the sneaky throw mechanism. The goal is to propagate failure, not swallow it.
- **Incorrect Exception Handling:** Do not attempt to catch the unchecked exception immediately around the call site. The pattern is designed to push error handling to a global or service-level boundary. Catching it locally re-introduces the boilerplate this interface aims to remove.

## Data Pipeline
ThrowableBiConsumer acts as a control-flow component rather than a data-flow one. It transforms a potential checked exception into an unchecked one that propagates up the call stack.

> Flow:
> Input Arguments (T, U) -> **acceptNow** (User-defined logic) -> Throws checked exception **E** -> **accept** (Default method's catch block) -> SneakyThrow.sneakyThrow(E) -> Unchecked exception propagates up the call stack.

