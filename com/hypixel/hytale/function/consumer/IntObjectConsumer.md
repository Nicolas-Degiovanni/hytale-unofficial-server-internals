---
description: Architectural reference for IntObjectConsumer
---

# IntObjectConsumer<T>

**Package:** com.hypixel.hytale.function.consumer
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface IntObjectConsumer<T> {
```

## Architecture & Concepts
IntObjectConsumer is a functional interface that defines a contract for a consumer operation accepting two arguments: a primitive `int` and a generic object of type `T`. It is a specialized variant of the standard Java `BiConsumer<Integer, T>`.

The primary architectural driver for this interface is **performance**. By operating on a primitive `int` instead of its boxed `Integer` counterpart, it completely avoids the overhead of object allocation and unboxing. In performance-critical systems like game engines, where operations may be executed millions of times per second across thousands of entities, this seemingly minor optimization has a significant cumulative impact on memory pressure and garbage collection pauses.

This interface embodies the "pass behavior as data" pattern. It allows systems to define a generic algorithm (e.g., iterating over a collection with an index) while allowing the caller to supply the specific logic to be executed for each element. It is frequently used in entity-component systems, rendering loops, and network packet processing where indexed access is common.

### Lifecycle & Ownership
As a functional interface, IntObjectConsumer does not have a traditional object lifecycle managed by a container. Its lifecycle is defined by the scope of its concrete implementation, which is typically a lambda expression or method reference.

- **Creation:** Instances are created ad-hoc at the call site where a method requires an IntObjectConsumer. They are not registered as services or managed by a central factory.
- **Scope:** The lifetime of an IntObjectConsumer instance is bound to the object that holds a reference to it. It may be a short-lived argument on the stack or a long-lived member of a service class (e.g., a cached event handler).
- **Destruction:** Instances are subject to standard Java garbage collection. Once all references to the lambda or method reference object are released, it becomes eligible for cleanup.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, an implementation (i.e., a lambda expression) can become stateful if it captures variables from its enclosing scope, forming a closure. Stateful consumers can introduce side effects and are more complex to reason about.

- **Thread Safety:** The interface contract is inherently thread-safe. The thread safety of a specific *implementation* is entirely dependent on the code it executes. If a lambda implementation captures and mutates shared state from an enclosing scope without proper synchronization, it is **not thread-safe**.

**WARNING:** Extreme caution is required when using stateful IntObjectConsumer implementations in a concurrent or parallel context. Prefer stateless functions or ensure that any captured state is immutable or thread-safe.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(int, T) | void | O(N) | Executes the defined operation. Complexity is determined by the implementation (N), not the call itself. |

## Integration Patterns

### Standard Usage
The most common pattern is to provide an implementation as a lambda expression to a method that iterates over a data structure, providing both the index and the value to the consumer.

```java
// A hypothetical system that processes a list of entities
List<Entity> entities = world.getEntities();
EntityProcessor processor = context.getService(EntityProcessor.class);

// The lambda (i, entity) -> ... is the implementation of IntObjectConsumer
processor.forEachEntityWithIndex((i, entity) -> {
    if (entity.isDirty()) {
        System.out.println("Processing dirty entity at index: " + i);
        entity.update();
    }
});
```

### Anti-Patterns (Do NOT do this)
- **Stateful Lambdas in Parallel Streams:** Do not use a consumer that modifies a shared, non-thread-safe collection or variable from within a parallel stream or multi-threaded task executor. This will lead to non-deterministic behavior and race conditions.
- **Blocking Operations:** Implementations of `accept` should be designed to be fast and non-blocking. Performing file I/O, network requests, or other long-running tasks within a consumer that is part of a critical game loop will cause severe performance degradation.

## Data Pipeline
This interface does not represent a data pipeline itself, but rather a functional stage *within* one. It is a terminal operation that consumes data and produces a side effect.

> Flow:
> Data Source (e.g., Array, List) -> Iteration Logic -> **IntObjectConsumer.accept(index, element)** -> Side Effect (e.g., State Mutation, Logging, Rendering Command)

