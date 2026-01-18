---
description: Architectural reference for OrderPriority
---

# OrderPriority

**Package:** com.hypixel.hytale.component.dependency
**Type:** Utility

## Definition
```java
// Signature
public enum OrderPriority {
```

## Architecture & Concepts
OrderPriority is a type-safe enumeration that defines a fixed set of priority levels for ordering operations within the component system. Its primary purpose is to provide a stable, readable, and compile-time-checked contract for specifying the sequence of component initialization, dependency resolution, or event listener execution.

This enum is a foundational element of the dependency framework, replacing fragile "magic number" or string-based priority schemes. By providing named constants like CLOSEST, NORMAL, and FURTHEST, it establishes a clear and unambiguous vocabulary for developers. The underlying integer values are an internal implementation detail used by sorting algorithms to arrange components or tasks into a predictable execution chain. The symmetric negative and positive values around a central NORMAL (0) suggest a system designed for sorting relative to a default baseline.

## Lifecycle & Ownership
As a Java enumeration, OrderPriority has a lifecycle strictly managed by the JVM, distinct from standard objects.

- **Creation:** The five instances (CLOSEST, CLOSE, NORMAL, FURTHER, FURTHEST) are constructed automatically and exactly once by the JVM when the OrderPriority class is first loaded.
- **Scope:** These instances are static, final, and globally accessible. They persist for the entire lifetime of the application.
- **Destruction:** The instances are eligible for garbage collection only when the defining ClassLoader is unloaded, which typically occurs at application shutdown.

## Internal State & Concurrency
- **State:** Each enum constant holds a single, private, final integer field named value. This state is immutable and is assigned only during the JVM's static initialization phase.
- **Thread Safety:** OrderPriority is inherently thread-safe. Its instances are effectively global constants, and their internal state cannot be modified after creation. They can be safely read and passed between any number of threads without requiring synchronization.

## API Surface
The primary API consists of the enum constants themselves.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue() | int | O(1) | Returns the low-level integer representation of the priority. This is intended for use by internal sorting mechanisms. |

## Integration Patterns
OrderPriority is designed to be used declaratively, often within annotations or configuration methods, to influence the behavior of a higher-level system.

### Standard Usage
The most common pattern is to use the enum constants directly when registering a component or service that requires ordered initialization or processing.

```java
// Example: Using OrderPriority in a hypothetical component registration
// The framework would read this annotation to determine execution order.
@RegisterComponent(priority = OrderPriority.CLOSE)
public class HighPriorityService implements IService {
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Relying on Integer Values:** Avoid comparing the raw integer values in application logic. The specific numbers are an implementation detail of the sorting system and may change. Always compare the enum constants directly.

    ```java
    // BAD: Brittle code that relies on implementation details
    if (component.getPriority().getValue() == 0) {
        // ... logic for normal priority
    }

    // GOOD: Robust, readable, and type-safe
    if (component.getPriority() == OrderPriority.NORMAL) {
        // ... logic for normal priority
    }
    ```
- **Using ordinal():** Do not use the built-in `ordinal()` method for sorting logic. It is extremely fragile, as its value depends entirely on the declaration order of the constants in the source file. The `getValue()` method provides a stable and explicit integer for this purpose.

## Data Pipeline
This component does not process data; it is a static data definition. It serves as metadata that influences the flow of control in other systems, such as a component loader or event dispatcher, but it does not have an inbound or outbound data pipeline itself.

