---
description: Architectural reference for SneakyThrow
---

# SneakyThrow

**Package:** com.hypixel.hytale.sneakythrow
**Type:** Utility

## Definition
```java
// Signature
public class SneakyThrow {
```

## Architecture & Concepts

The SneakyThrow class is a low-level, engine-wide utility designed to subvert Java's checked exception system. Its primary architectural role is to enable the propagation of checked exceptions through code paths that do not permit them, most notably within standard Java functional interfaces like Consumer, Supplier, and Function used in the Streams API or other modern Java features.

This utility leverages a well-known Java generics trick to "sneak" a checked exception past the compiler's static analysis. The core mechanism, found in the private method sneakyThrow0, uses a generic type parameter constrained to Throwable (`<T extends Throwable>`). By declaring that the method `throws T`, it satisfies the type system, but because the specific type of T is not known at the call site, the compiler does not enforce a try-catch block or a throws declaration on the caller.

**WARNING:** This is a powerful and dangerous tool. Its purpose is to solve specific interoperability problems, not to bypass proper exception handling. Misuse can lead to unhandled exceptions terminating threads unexpectedly.

## Lifecycle & Ownership

-   **Creation:** As a static utility class, SneakyThrow is never instantiated. Its methods are accessed statically (e.g., SneakyThrow.sneakyThrow(e)).
-   **Scope:** Application-level. The class and its methods are available as soon as the SneakyThrow class is loaded by the JVM's ClassLoader.
-   **Destruction:** The class is unloaded when its ClassLoader is garbage collected, which typically occurs at application shutdown. There is no manual destruction or cleanup process.

## Internal State & Concurrency

-   **State:** SneakyThrow is completely stateless. It contains no member variables and all its methods are pure functions that operate solely on their input arguments.
-   **Thread Safety:** This class is inherently thread-safe. Due to its stateless nature, methods can be called concurrently from any number of threads without risk of race conditions or data corruption. No synchronization mechanisms are required or used.

## API Surface

The public API consists of two categories: the core throwing mechanism and a set of wrappers for standard functional interfaces.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| sneakyThrow(Throwable t) | RuntimeException | O(1) | The core function. Throws the provided Throwable without requiring the caller to declare it. The return type is a formality to satisfy language rules; this method never returns normally. |
| sneakyRunnable(ThrowableRunnable) | Runnable | O(1) | Wraps a runnable-like lambda that can throw a checked exception, returning a standard Runnable. |
| sneakySupplier(ThrowableSupplier) | Supplier | O(1) | Wraps a supplier-like lambda that can throw a checked exception, returning a standard Supplier. |
| sneakyConsumer(ThrowableConsumer) | Consumer | O(1) | Wraps a consumer-like lambda that can throw a checked exception, returning a standard Consumer. |
| sneakyFunction(ThrowableFunction) | Function | O(1) | Wraps a function-like lambda that can throw a checked exception, returning a standard Function. |

## Integration Patterns

### Standard Usage

The primary use case is to adapt code that throws checked exceptions for use with APIs that accept standard functional interfaces, such as the Java Streams API.

```java
// A method that throws a checked exception
private void processFile(Path path) throws IOException {
    // ... file processing logic
}

// Using SneakyThrow to integrate with Stream.forEach
List<Path> filePaths = List.of(Path.of("a.txt"), Path.of("b.txt"));

try {
    filePaths.stream().forEach(
        SneakyThrow.sneakyConsumer(this::processFile)
    );
} catch (IOException e) {
    // The checked exception is caught here, outside the lambda
    log.error("Failed to process a file", e);
}
```

### Anti-Patterns (Do NOT do this)

-   **Swallowing Exceptions:** Do not use SneakyThrow to silently ignore exceptions. The goal is to propagate them to a proper handling block, not to eliminate them.
    ```java
    // BAD: The IOException is thrown but never caught, potentially crashing the thread
    paths.stream().forEach(SneakyThrow.sneakyConsumer(path -> Files.delete(path)));
    ```
-   **Overuse:** This utility should not be your default exception handling strategy. It is reserved for bridging incompatible interfaces. Prefer standard try-catch blocks and throws declarations wherever possible.
-   **Direct Instantiation:** The class has no instance methods and a private constructor is implied. Do not attempt to create an instance with `new SneakyThrow()`.

## Data Pipeline

The "data" processed by this utility is not traditional data but rather a Java Throwable object. The pipeline describes how a checked exception is transformed into one that behaves like an unchecked exception at compile time.

> Flow:
> Checked Exception (e.g., IOException) is thrown inside a lambda -> The lambda calls **SneakyThrow.sneakyThrow(e)** -> The generic `sneakyThrow0` method re-throws the exception -> The JVM propagates the original exception up the call stack -> The exception can be caught by a standard `try-catch` block surrounding the entry point (e.g., the `stream()` call).

