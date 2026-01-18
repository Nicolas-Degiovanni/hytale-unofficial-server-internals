---
description: Architectural reference for ThrowableConsumer
---

# ThrowableConsumer

**Package:** com.hypixel.hytale.sneakythrow.consumer
**Type:** Utility Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface ThrowableConsumer<T, E extends Throwable> extends Consumer<T> {
```

## Architecture & Concepts
ThrowableConsumer is a specialized functional interface designed to bridge the gap between modern Java functional programming paradigms (like Streams and lambdas) and legacy or I/O-bound code that throws checked exceptions.

Standard Java functional interfaces, such as java.util.function.Consumer, do not permit checked exceptions in their method signatures. This forces developers to wrap exception-throwing logic in cumbersome try-catch blocks within the lambda body, often breaking the fluency of the code.

This interface elegantly solves this problem by extending the standard Consumer and providing a new abstract method, acceptNow, which *is* permitted to throw a checked exception. The default implementation of the standard accept method wraps the call to acceptNow, catching any thrown Throwable and re-throwing it as an unchecked exception via the Hytale SneakyThrow utility. This technique, known as "sneaky throw", bypasses the compiler's checked exception verification, allowing for cleaner integration with APIs that expect a standard Consumer.

It is a foundational component of Hytale's error-handling strategy, promoting a more expressive and less verbose coding style when dealing with operations that can fail.

## Lifecycle & Ownership
As a functional interface, ThrowableConsumer does not have a managed lifecycle in the same way a service or component does. Its instances are typically ephemeral.

- **Creation:** An instance is created implicitly whenever a lambda expression or method reference matching its signature is declared. This is common in stream operations or when passing behavior as a parameter.
- **Scope:** The lifetime of an instance is tied to the scope in which it is defined. Most commonly, this is method-local, meaning the instance is eligible for garbage collection once the method execution completes.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The interface itself is stateless. An implementing lambda, however, can be stateful if it forms a closure over variables from its enclosing scope. The statefulness and its implications are the responsibility of the implementer.
- **Thread Safety:** The interface contract is inherently thread-safe. The thread safety of a specific implementation depends entirely on the logic contained within the lambda and whether it safely manages access to any shared, mutable state. The SneakyThrow mechanism itself is thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T t) | void | O(N) | Implements the standard Consumer interface. Wraps the call to acceptNow and re-throws any checked exception as an unchecked one. Complexity is determined by the acceptNow implementation. |
| acceptNow(T t) | void | O(N) | The method to be implemented by a lambda. Contains the core logic and is permitted to throw a checked exception of type E. |

## Integration Patterns

### Standard Usage
The primary use case is within Java Stream operations or any other API that accepts a standard Consumer, where the operation itself may throw a checked exception.

```java
// Process a list of files, where processing might throw an IOException
List<Path> filePaths = ...;

// The lambda (path -> processFile(path)) is an instance of ThrowableConsumer
// It is passed to forEach, which expects a standard Consumer<Path>
filePaths.stream().forEach((ThrowableConsumer<Path, IOException>) path -> {
    // This code can now throw a checked exception without a local try-catch
    Files.copy(path, new FileOutputStream("..."));
});
```

### Anti-Patterns (Do NOT do this)
- **Direct Invocation of acceptNow:** Avoid calling the acceptNow method directly unless you are inside a context that can handle the declared checked exception. The primary value of this interface is the wrapping behavior provided by the default accept method.
- **Wrapping Unchecked Exceptions:** Do not use this interface for logic that only throws RuntimeExceptions. The mechanism is specifically designed to handle checked exceptions. Using it for unchecked exceptions is redundant and adds unnecessary complexity.
- **Swallowing Exceptions:** The purpose of this utility is to propagate exceptions, not to hide them. Do not implement acceptNow with an empty catch block.

## Data Pipeline
ThrowableConsumer is not part of a data pipeline but rather a control-flow and exception-propagation mechanism. It alters the path of exception handling.

> Flow:
> API expecting Consumer -> **ThrowableConsumer.accept(T)** -> try...catch block -> **Implementation.acceptNow(T)** -> (Checked Exception Thrown) -> catch (Throwable) -> SneakyThrow.sneakyThrow() -> (Unchecked Exception Propagates Up the Stack)

