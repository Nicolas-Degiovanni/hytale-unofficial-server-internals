---
description: Architectural reference for ThrowableBiFunction
---

# ThrowableBiFunction

**Package:** com.hypixel.hytale.sneakythrow.function
**Type:** Utility Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface ThrowableBiFunction<T, U, R, E extends Throwable> extends BiFunction<T, U, R> {
```

## Architecture & Concepts
ThrowableBiFunction is a specialized functional interface designed to bridge the gap between functional programming paradigms and Java's checked exception system. It extends the standard `java.util.function.BiFunction` but modifies its contract to permit the throwing of checked exceptions from within lambda expressions.

This is achieved through a technique known as a "sneaky throw". The interface provides a default implementation for the standard `apply` method. This implementation wraps a call to a new, exception-declaring method, `applyNow`. If `applyNow` throws a checked exception, the `catch` block intercepts it and uses the `SneakyThrow` utility to re-throw it as an unchecked exception.

This pattern is a low-level, cross-cutting utility. Its primary purpose is to improve developer ergonomics by allowing methods that throw checked exceptions (e.g., file I/O, network operations) to be used seamlessly within higher-order functions and streams that expect standard, non-throwing functional interfaces.

**WARNING:** This mechanism bypasses compile-time exception checking. While this simplifies code, it shifts the responsibility of exception handling entirely to the runtime and the developer's awareness of the underlying implementation.

### Lifecycle & Ownership
-   **Creation:** This interface is not instantiated via a constructor. It is implemented by a lambda expression or method reference at the point of use. The Java runtime synthesizes an anonymous class that implements the interface.
-   **Scope:** The lifetime of a ThrowableBiFunction instance is tied to the scope of the lambda that defines it. It is typically ephemeral, created to be passed as an argument to a method and eligible for garbage collection once that method call completes, unless captured by a longer-lived object.
-   **Destruction:** Managed by the Java Garbage Collector. There is no manual cleanup required.

## Internal State & Concurrency
-   **State:** The interface itself is stateless. However, any lambda expression implementing this interface can be stateful if it captures variables from its enclosing scope. The statefulness and immutability are determined entirely by the implementation.
-   **Thread Safety:** The interface contract is inherently thread-safe. The thread safety of a specific *implementation* is not guaranteed. If a lambda implementing ThrowableBiFunction captures and modifies shared, mutable state without proper synchronization, it will not be thread-safe.

**WARNING:** Extreme caution is advised when using implementations of this interface in a multi-threaded context. The developer is solely responsible for ensuring the thread safety of the lambda's logic and any captured state.

## API Surface
The public contract is composed of the standard `BiFunction` method and the new exception-aware variant.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(T t, U u) | R | O(1) + Impl | Default method. Executes `applyNow` and re-throws any checked exception as an unchecked one. This is the intended entry point. |
| applyNow(T t, U u) | R | Impl | Abstract method to be implemented by a lambda. Declared with `throws E`, allowing operations that throw checked exceptions. |

## Integration Patterns

### Standard Usage
This interface is intended for use with higher-order functions that expect a standard `BiFunction`, but where the underlying logic needs to perform an operation that throws a checked exception.

```java
// Assume processData expects a standard BiFunction<String, String, Integer>
public void processData(BiFunction<String, String, Integer> processor) {
    // ... logic
    int result = processor.apply("data1", "data2");
    // ... logic
}

// The lambda implements ThrowableBiFunction, which can be passed where a
// BiFunction is expected. The IOException is handled by the sneaky throw mechanism.
public void execute() {
    ThrowableBiFunction<String, String, Integer, IOException> fileProcessor = (s1, s2) -> {
        // This operation can throw a checked IOException
        return Files.readAllBytes(Paths.get(s1)).length;
    };

    // No try-catch or 'throws' declaration is needed here
    processData(fileProcessor);
}
```

### Anti-Patterns (Do NOT do this)
-   **Directly Calling applyNow:** Calling `fileProcessor.applyNow(...)` is an anti-pattern. This bypasses the sneaky-throw wrapper in the `apply` method, defeating the purpose of the interface and forcing the caller to handle the checked exception with a `try-catch` block or a `throws` clause.
-   **Ignoring Potential Exceptions:** Do not treat the absence of a compile-time error as a guarantee that no exception will be thrown. The underlying `IOException` in the example above will still be thrown at runtime. Higher-level application logic must be prepared to handle these unchecked exceptions.

## Data Pipeline
This component acts as a control-flow adapter rather than a data processor. Its "pipeline" describes how it transforms a checked exception into an unchecked one.

> Flow:
> Caller invokes `apply(t, u)` -> Default `apply` implementation calls `applyNow(t, u)` -> Lambda logic executes -> **(Path A: Success)** -> Returns result `R` -> Caller receives result.
> **(Path B: Failure)** -> Lambda logic throws checked exception `E` -> `catch (Throwable)` block in `apply` intercepts it -> `SneakyThrow.sneakyThrow(e)` is called -> An equivalent unchecked exception is thrown -> The exception propagates up the call stack to the original caller.

