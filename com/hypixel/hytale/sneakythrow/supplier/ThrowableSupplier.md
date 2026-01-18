---
description: Architectural reference for ThrowableSupplier
---

# ThrowableSupplier<T, E extends Throwable>

**Package:** com.hypixel.hytale.sneakythrow.supplier
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface ThrowableSupplier<T, E extends Throwable> extends Supplier<T> {
```

## Architecture & Concepts
The ThrowableSupplier interface is a specialized functional interface designed to bridge the gap between Java's checked exceptions and its functional programming constructs, particularly those expecting a standard java.util.function.Supplier.

In the Hytale engine, many operations within functional pipelines (such as data processing streams or asynchronous task chains) may involve methods that throw checked exceptions, like IOException or SQLException. Standard functional interfaces, including Supplier, do not declare checked exceptions in their method signatures. This forces developers to wrap lambda bodies in cumbersome try-catch blocks, which harms code readability and conciseness.

ThrowableSupplier solves this by extending Supplier and providing a default implementation of the get method. This implementation calls an abstract method, getNow, which *is* permitted to throw a checked exception. If an exception is caught, it is re-thrown as an unchecked exception using the engine's SneakyThrow utility. This pattern allows lambdas that perform exception-prone work to be seamlessly integrated into APIs that expect a standard Supplier, effectively "sneaking" the checked exception past the compiler.

This component is a cornerstone of the engine's error-handling philosophy in functional contexts, prioritizing clean, expressive code while centralizing the mechanism for bypassing checked exception constraints.

### Lifecycle & Ownership
As a functional interface, a ThrowableSupplier does not have a managed lifecycle in the same way a service or component does. Its lifetime is determined by standard Java garbage collection rules.

- **Creation:** An instance is typically created implicitly via a lambda expression or a method reference. It is not instantiated directly via a constructor. For example: `ThrowableSupplier<String, IOException> supplier = () -> Files.readString(path);`.
- **Scope:** The scope is lexical. The instance exists as long as it is referenced, often ephemerally for the duration of a single method call (e.g., as an argument to a stream operator). If assigned to a field, its lifetime is tied to the containing object.
- **Destruction:** The instance is eligible for garbage collection once it is no longer reachable. There is no explicit destruction or cleanup method.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Any statefulness of a particular ThrowableSupplier instance is derived from the variables it captures from its enclosing lexical scope (its closure).
- **Thread Safety:** **Not intrinsically thread-safe.** The thread safety of a ThrowableSupplier is entirely dependent on the implementation of the getNow method provided by the developer. If the lambda or method reference accesses shared mutable state without proper synchronization, it will not be thread-safe. The interface provides no internal locking or synchronization mechanisms.

## API Surface
The public contract is defined by the Supplier interface and the single abstract method required for implementation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | T | Varies | Default method. Executes getNow and re-throws any checked exception as an unchecked one. This is the primary entry point for consumers. |
| getNow() | T | Varies | Abstract method to be implemented by the developer. Contains the core logic that may produce a value of type T or throw an exception of type E. |

## Integration Patterns

### Standard Usage
This interface should be used when passing a method that throws a checked exception into a higher-order function that expects a standard Supplier. This avoids boilerplate try-catch blocks.

```java
// A method that expects a standard Supplier
public void processData(Supplier<String> dataSupplier) {
    String data = dataSupplier.get();
    // ... process data
}

// A method that can throw a checked exception
public String loadFile() throws IOException {
    return new String(Files.readAllBytes(Paths.get("config.json")));
}

// Use ThrowableSupplier to bridge the two
public void run() {
    // The lambda () -> loadFile() is an instance of ThrowableSupplier
    // It can be passed directly to processData because ThrowableSupplier
    // is a subtype of Supplier.
    processData(this::loadFile);
}
```

### Anti-Patterns (Do NOT do this)
- **Swallowing Critical Exceptions:** Do not use this utility to bypass handling for exceptions that indicate a critical, unrecoverable program error. It is intended for checked exceptions that are inconvenient in a functional context, not for ignoring fundamental logic flaws.
- **Directly Calling getNow:** Avoid calling the getNow method directly unless you are in a context where you explicitly want to handle the checked exception with a traditional try-catch block. The primary value of the interface is its default get method.
- **Ignoring the Underlying Cause:** When an exception is thrown by get, it will be an unchecked wrapper. It is critical that higher-level exception handlers inspect the cause of the exception to understand the original checked exception that was thrown.

## Data Pipeline
The primary flow is one of control, not data. It transforms a potential checked exception into an unchecked one.

> Flow:
> Consumer calls `get()` -> **ThrowableSupplier** invokes `getNow()` -> `getNow()` implementation executes -> (If `IOException` is thrown) -> `get()` catches it -> `SneakyThrow.sneakyThrow()` is called -> Unchecked exception propagates to the consumer.

