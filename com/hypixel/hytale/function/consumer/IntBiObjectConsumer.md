---
description: Architectural reference for IntBiObjectConsumer
---

# IntBiObjectConsumer<T, J>

**Package:** com.hypixel.hytale.function.consumer
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface IntBiObjectConsumer<T, J> {
```

## Architecture & Concepts
IntBiObjectConsumer is a functional interface that defines a behavioral contract for an operation that accepts three arguments and returns no result. It is a specialized consumer designed to operate on a primitive integer and two generic objects.

This interface is a cornerstone of the engine's functional programming patterns, particularly in systems that require iterating over collections with an index. By defining a specific arity (three arguments: `int`, `T`, `J`), it avoids the overhead and potential ambiguity of using more generic structures like `Object[]` or wrapper classes. Its primary role is to allow systems to pass behavior—the *what to do*—as a parameter, decoupling the iteration logic from the operation performed during iteration.

This pattern is frequently used in high-performance loops within entity-component systems, rendering pipelines, or network packet processing where both an index and multiple related objects are required for an operation.

## Lifecycle & Ownership
As a functional interface, IntBiObjectConsumer does not have a traditional object lifecycle. Instead, the lifecycle pertains to its *implementations*, which are typically lambda expressions or method references.

- **Creation:** Instances are not created directly via a constructor. They are defined inline as lambda expressions or created via method references. The Java runtime synthesizes an anonymous class that implements the interface.
- **Scope:** The scope of an IntBiObjectConsumer instance is determined by the object that holds a reference to it. It can be ephemeral, existing only for the duration of a method call, or it can be long-lived if stored as a field in a service or component (e.g., as a callback handler).
- **Destruction:** The synthesized instance is subject to standard Java garbage collection. It is destroyed when no more strong references to it exist.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, an implementation (a lambda expression) can become stateful if it *captures* variables from its enclosing scope. This is known as forming a closure.

- **Thread Safety:** The interface contract is inherently thread-safe. The thread safety of a specific *implementation* is entirely dependent on the code within the lambda and the nature of any captured state.

    **WARNING:** Stateful implementations that capture and modify non-thread-safe objects are a significant source of concurrency bugs. If an IntBiObjectConsumer is used in a parallel stream or a multi-threaded task scheduler, any captured state must be synchronized or be inherently thread-safe (e.g., final or effectively final variables, concurrent collections).

## API Surface
The public contract consists of a single abstract method, as required for a functional interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(int, T, J) | void | O(N) | Executes the defined operation. The complexity is determined by the implementation provided by the caller. |

## Integration Patterns

### Standard Usage
The primary integration pattern is providing a lambda expression to a method that accepts an IntBiObjectConsumer. This is common in systems that iterate over indexed data structures.

```java
// A hypothetical system that processes entities and their physics components
// The lambda (index, entity, physics) -> { ... } is an implementation of IntBiObjectConsumer
entitySystem.forEachEntityWithComponent((index, entity, physics) -> {
    if (physics.isMoving()) {
        System.out.println("Entity " + index + " is moving.");
    }
    entity.update();
});
```

### Anti-Patterns (Do NOT do this)
- **Stateful Lambdas in Parallel Systems:** Never use a lambda that modifies shared, unsynchronized state when passed to a parallel processing system. This will lead to race conditions and non-deterministic behavior.

    ```java
    // ANTI-PATTERN: Unsafe shared state modification
    List<Entity> unsafeList = new ArrayList<>();
    parallelEntityProcessor.process((index, entity, component) -> {
        if (component.isActive()) {
            unsafeList.add(entity); // Race condition! Multiple threads writing to ArrayList.
        }
    });
    ```

- **Blocking Operations:** Avoid performing long-running or blocking operations (e.g., file I/O, network requests) within an IntBiObjectConsumer implementation if it is being executed on a critical engine thread, such as the main game loop or a rendering thread. This will cause stalls and performance degradation.

## Data Pipeline
IntBiObjectConsumer is not a data pipeline component itself; rather, it represents a processing stage *within* a pipeline. It is a sink, consuming data to produce a side-effect.

> Flow:
> Data Source (e.g., Entity List, Network Buffer) -> Iteration Logic (e.g., for-loop, stream) -> **IntBiObjectConsumer.accept()** -> Side-Effect (e.g., Component State Mutation, Rendering Call, Logging)

