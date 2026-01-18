---
description: Architectural reference for ThrowableRunnable
---

# ThrowableRunnable

**Package:** com.hypixel.hytale.sneakythrow
**Type:** Utility Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface ThrowableRunnable<E extends Throwable> extends Runnable {
```

## Architecture & Concepts
ThrowableRunnable is a specialized functional interface that extends Java's standard `Runnable`. Its primary architectural purpose is to bridge the gap between code that throws checked exceptions and APIs that only accept standard `Runnable` instances, which cannot declare checked exceptions in their `run` method signature.

This interface is a core component of the engine's low-level error handling and concurrency framework. It leverages a technique known as "sneaky throw" to bypass compile-time checks for checked exceptions. The default `run` method catches any `Throwable` from the user-implemented `runNow` method and re-throws it as an unchecked exception via the `SneakyThrow` utility. This allows developers to use lambdas and method references for operations that might throw exceptions (like IO or parsing) within standard Java concurrency APIs (e.g., `ExecutorService.submit(Runnable)`) without cumbersome boilerplate `try-catch` blocks that wrap exceptions in a `RuntimeException`.

**WARNING:** This pattern is powerful but subverts the Java compiler's safety guarantees. Exceptions thrown in this manner are effectively unchecked and will propagate up the call stack, potentially terminating the thread if not handled by an `UncaughtExceptionHandler`.

## Lifecycle & Ownership
As a functional interface, ThrowableRunnable itself does not have a lifecycle. The following describes the lifecycle of its *implementations*, which are typically lambda expressions or anonymous classes.

- **Creation:** An instance is created ad-hoc at the call site where a `Runnable` is required but the underlying logic throws a checked exception. It is typically passed directly as an argument to a method.
- **Scope:** The scope is transient and ephemeral. The object exists for the duration of its use, usually within a task queue or as a direct argument to a thread constructor.
- **Destruction:** The implementation object is eligible for garbage collection as soon as the method consuming it (e.g., `Thread.run()` or an executor's worker) completes and no other references to it exist.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, implementations (especially lambdas that form a closure) can capture and modify state from their enclosing scope. Therefore, any implementation may be stateful.
- **Thread Safety:** The interface provides no thread safety guarantees. The thread safety of any implementation is the sole responsibility of the developer. If a lambda implementation closes over shared, mutable state, it must be synchronized externally.

## API Surface
The public contract is designed for implementing exception-throwing logic and executing it through a standard `Runnable` interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| run() | void | O(N) | Default method. Executes `runNow` and re-throws any exception as an unchecked one. This is the intended entry point. |
| runNow() | void | O(N) | Abstract method. Must be implemented by the developer to contain the core logic that may throw a checked exception of type E. |

*Complexity O(N) is dependent on the user-provided implementation of `runNow`.*

## Integration Patterns

### Standard Usage
The primary use case is to pass a block of code that throws a checked exception to a consumer that only accepts a standard `Runnable`.

```java
// Example: Submitting a file-writing task that throws IOException
// to an ExecutorService which expects a standard Runnable.

ExecutorService executor = Executors.newSingleThreadExecutor();

// The lambda () -> writeToFile() implements ThrowableRunnable<IOException>
// The run() method handles the sneaky throw, satisfying the submit(Runnable) signature.
executor.submit((ThrowableRunnable<IOException>) () -> {
    Files.writeString(Paths.get("output.txt"), "Hytale");
});
```

### Anti-Patterns (Do NOT do this)
- **Directly Calling runNow:** Calling `runNow()` directly bypasses the sneaky throw mechanism. This forces the caller to handle the checked exception with a standard `try-catch` block, defeating the entire purpose of the interface.

    ```java
    // INCORRECT: This forces a try-catch, negating the interface's value.
    ThrowableRunnable<IOException> task = () -> { /* ... */ };
    try {
        task.runNow();
    } catch (IOException e) {
        // ... boilerplate handling
    }
    ```

- **Swallowing All Exceptions:** Relying on this interface to simply ignore exceptions is a dangerous practice. The exception is still thrown at runtime and will likely terminate the executing thread if not handled at a higher level.

## Data Pipeline
This component does not process a traditional data pipeline. Instead, it transforms the control flow of exception handling.

> Flow:
> Checked Exception (e.g., IOException) -> `runNow()` Implementation -> `run()` Default Method -> `SneakyThrow.sneakyThrow()` -> Unchecked `RuntimeException` -> Propagates up the call stack

