---
description: Architectural reference for ShortObjectConsumer
---

# ShortObjectConsumer

**Package:** com.hypixel.hytale.function.consumer
**Type:** Utility

## Definition
```java
// Signature
@FunctionalInterface
public interface ShortObjectConsumer<T> {
```

## Architecture & Concepts
ShortObjectConsumer is a specialized functional interface designed for high-performance scenarios. It serves as a contract for operations that consume a primitive short and a generic object.

Its primary architectural purpose is to **avoid primitive boxing**. The standard Java BiConsumer requires object types, which would force the boxing of a primitive `short` into a `Short` wrapper object. This boxing operation introduces memory allocation and garbage collector pressure, which can be detrimental in performance-critical code paths such as rendering loops, network packet processing, or entity update cycles.

By defining a signature with a primitive `short`, this interface allows for the direct, allocation-free passing of short values into lambda expressions and method references, making it a foundational component for performance-sensitive systems.

## Lifecycle & Ownership
As a functional interface, ShortObjectConsumer itself does not have a managed lifecycle. Instead, the lifecycle applies to the *implementing instances* (typically lambda expressions or anonymous classes).

- **Creation:** An instance is created inline at the point of use, often as a lambda expression passed as an argument to a method. For example, `(id, entity) -> entity.setHealth(id)`.
- **Scope:** The scope of an instance is tied to the object that holds a reference to it. It may be transient (existing only for the duration of a method call) or it may be stored as a field in a longer-lived object.
- **Destruction:** Instances are managed by the Java Garbage Collector. They become eligible for collection when they are no longer reachable, such as when the holding object is destroyed or the method call completes.

## Internal State & Concurrency
- **State:** The interface itself is stateless. Concrete implementations (lambdas) can be stateful if they close over non-final variables from their enclosing scope (a practice known as creating a closure). However, the idiomatic use is for stateless, pure functions.
- **Thread Safety:** The interface is inherently thread-safe. The thread safety of a specific *implementation* is the responsibility of the developer. Lambdas that only operate on their input arguments and do not modify shared state are thread-safe.

**WARNING:** If a lambda implementation of this interface modifies shared, mutable state without proper synchronization, it will not be thread-safe. This is a common source of concurrency bugs.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(short var1, T var2) | void | O(N) | Performs this operation on the given arguments. Complexity is dependent on the implementation. |

## Integration Patterns

### Standard Usage
This interface is intended to be used with lambda expressions to provide a concise, performant implementation for methods that iterate over or process pairs of shorts and objects.

```java
// A system that processes entities by their short ID
public void processEntities(ShortObjectConsumer<Entity> processor) {
    for (short i = 0; i < entityCount; i++) {
        Entity entity = entities[i];
        if (entity != null) {
            processor.accept(i, entity);
        }
    }
}

// Usage:
entitySystem.processEntities((id, entity) -> {
    System.out.println("Processing entity with ID: " + id);
    entity.update();
});
```

### Anti-Patterns (Do NOT do this)
- **Unnecessary Class Implementation:** Avoid creating a full, named class that implements this interface for a simple, single-use operation. This defeats the purpose of a lightweight functional interface. Use a lambda instead.
- **Stateful Lambdas in Concurrent Code:** Do not create lambda implementations that modify shared state from multiple threads without explicit synchronization. This will lead to race conditions.

## Data Pipeline
ShortObjectConsumer is not a data pipeline itself, but rather a fundamental processing block *within* a pipeline. It represents a terminal operation or a processing step that acts on data but does not forward it.

> **Example Flow:**
> Data Source (e.g., Entity Array) -> Iteration Logic -> **ShortObjectConsumer (as lambda)** -> Side Effect (e.g., Component Update)

