---
description: Architectural reference for ThrowableIntSupplier
---

# ThrowableIntSupplier

**Package:** com.hypixel.hytale.sneakythrow.supplier
**Type:** Utility Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface ThrowableIntSupplier<E extends Throwable> extends IntSupplier {
```

## Architecture & Concepts
ThrowableIntSupplier is a specialized functional interface designed to bridge the gap between code that produces checked exceptions and standard Java APIs that do not support them. It extends Java's native `IntSupplier` but modifies its contract to allow for exception handling via the `SneakyThrow` utility.

The core architectural purpose of this interface is to enable developers to use lambda expressions and method references in contexts that expect an `IntSupplier` (such as in Stream API operations) without being forced to wrap the logic in a verbose try-catch block. It achieves this by providing a new method, `getAsIntNow`, which is permitted to throw a checked exception. The default implementation of the standard `getAsInt` method catches this exception and re-throws it as an unchecked exception, effectively "sneaking" it past the Java compiler's checks.

This component is a key part of the engine's strategy for writing cleaner, more functional code when interacting with I/O, reflection, or other exception-prone operations.

### Lifecycle & Ownership
As a functional interface, ThrowableIntSupplier does not have a managed lifecycle in the same way a service or component does. Its lifetime is ephemeral and dictated by standard Java object scoping and garbage collection.

- **Creation:** Instances are created implicitly through lambda expressions or method references. They are not instantiated directly via a constructor. A developer creates one when they need to supply an integer-producing function that can throw a checked exception to an API that only accepts a standard `IntSupplier`.
- **Scope:** The scope of a ThrowableIntSupplier instance is tied to the scope of the lambda that defines it. It may be stack-local, or if it is a non-static method reference or captures variables, it may be tied to the lifetime of a parent object.
- **Destruction:** The instance is eligible for garbage collection as soon as it is no longer referenced.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, an instance created from a lambda expression can be stateful if the lambda captures mutable variables from its enclosing scope. The statefulness is entirely determined by the implementation.
- **Thread Safety:** **Not intrinsically thread-safe.** The thread safety of a ThrowableIntSupplier instance is wholly dependent on the code provided in the implementation of the `getAsIntNow` method. If the implementation accesses shared mutable state without proper synchronization, it will not be thread-safe. The interface provides no concurrency control mechanisms.

## API Surface
The public contract consists of two methods: the standard `IntSupplier` method and the exception-aware variant.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAsInt() | int | O(N) | Implements the `IntSupplier` contract. Internally calls `getAsIntNow` and uses `SneakyThrow` to propagate any thrown `Throwable` as an unchecked exception. Complexity is dependent on the `getAsIntNow` implementation. |
| getAsIntNow() | int | O(N) | **Abstract method to be implemented.** Contains the core logic that produces an integer. This is the method that is permitted to throw a checked exception of type E. |

## Integration Patterns

### Standard Usage
This interface should be used to adapt exception-throwing methods for use with standard Java functional APIs, particularly streams.

```java
// How a developer should normally use this
// Assume readIntFromFile() throws IOException
public int readIntFromFile() throws IOException {
    // ... I/O logic
    return 42;
}

// Use the interface to pass the method to an API expecting an IntSupplier
OptionalInt result = OptionalInt.of(0)
    .stream()
    .map(i -> ((ThrowableIntSupplier<IOException>) this::readIntFromFile).getAsInt())
    .findFirst();
```

### Anti-Patterns (Do NOT do this)
- **Ignoring Recoverable Exceptions:** This utility is designed to propagate exceptions, not to swallow them. Using it to bypass compiler checks for exceptions that should be handled locally (e.g., a `FileNotFoundException` where a default value could be used) is an anti-pattern.
- **Obscuring Control Flow:** Overusing this pattern can make it difficult to reason about where exceptions originate. For complex logic with multiple potential failure points, a traditional try-catch block is often more readable and maintainable.
- **Casting without Context:** The standard usage example requires a cast. This is a signal that you are deliberately opting into this behavior. Avoid creating complex chains of these suppliers where the underlying exception types become obscured.

## Data Pipeline
This component acts as a control-flow adapter rather than a data processor. Its flow is centered on exception handling.

> Flow:
> Caller invokes `getAsInt()` -> **ThrowableIntSupplier** invokes `getAsIntNow()` -> `getAsIntNow()` implementation either returns an `int` OR throws a `Throwable` -> The `catch` block in `getAsInt()` intercepts the `Throwable` -> `SneakyThrow.sneakyThrow()` is called -> The original exception is re-thrown as an unchecked exception, unwinding the stack.

