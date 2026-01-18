---
description: Architectural reference for IntTriObjectConsumer
---

# IntTriObjectConsumer<T, J, K>

**Package:** com.hypixel.hytale.function.consumer
**Type:** Functional Contract

## Definition
```java
// Signature
@FunctionalInterface
public interface IntTriObjectConsumer<T, J, K> {
```

## Architecture & Concepts
The IntTriObjectConsumer is a specialized functional interface designed for high-performance callback operations. It serves as a contract for any operation that accepts four arguments—a primitive integer and three generic objects—and returns no result.

Its primary architectural purpose is to provide a standardized, allocation-free mechanism for lambda expressions and method references in performance-critical code paths. By accepting a primitive **int** directly, it avoids the overhead of autoboxing the value into an **Integer** object, which is a common performance bottleneck in tight loops found in game logic, rendering, and networking systems.

This interface is a key component of the engine's functional programming paradigm, enabling developers to write clean, expressive, and efficient code for iterating or processing complex data structures without incurring garbage collection pressure. It is intended to be used in APIs where an action needs to be performed for each element of a collection, along with its index and associated metadata.

## Lifecycle & Ownership
As a functional interface, IntTriObjectConsumer itself has no lifecycle. The lifecycle pertains to its *implementations*, which are typically lambda expressions or anonymous inner classes.

- **Creation:** Instances are almost exclusively created inline as lambda expressions or method references passed as arguments to higher-order functions. They are not meant to be instantiated in a traditional manner.
- **Scope:** The lifetime of an IntTriObjectConsumer implementation is tied to the object that holds a reference to it. It can be ephemeral, existing only for the duration of a method call (e.g., passed to a forEach method), or it can be long-lived if stored as a field in another object (e.g., a registered event listener).
- **Destruction:** Implementations are subject to standard Java garbage collection. An instance is eligible for cleanup once no more strong references to it exist.

**Warning:** Be mindful of capturing variables from the enclosing scope (creating a closure). If a lambda captures a reference to a long-lived object, it may prevent that object from being garbage collected, potentially causing memory leaks.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, an implementation (a lambda) can be stateful if it closes over mutable variables from its enclosing scope. This practice is strongly discouraged in concurrent contexts.
- **Thread Safety:** The contract is thread-agnostic. **Thread safety is the absolute responsibility of the implementation.** If a lambda expression implementing this interface modifies shared state from multiple threads, it must be protected by external synchronization mechanisms. The engine provides no implicit thread safety for these operations.

## API Surface
The interface defines a single abstract method, which is its entire public contract.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(int, T, J, K) | void | Implementation-Defined | Executes the consumer's logic. The complexity is entirely dependent on the code provided in the lambda body. |

## Integration Patterns

### Standard Usage
This interface is designed to be implemented with a lambda expression and passed to methods that process collections or event streams. The typical use case involves iterating over data where an index, a primary object, and two secondary related objects are required for the operation.

```java
// A hypothetical system that processes all active entities in a region.
// The consumer receives the entity ID, the Entity object, its Position, and the World context.
IntTriObjectConsumer<Entity, Position, World> entityProcessor = (id, entity, pos, world) -> {
    if (pos.isWithinBounds(someArea)) {
        entity.applyEffect(Effects.FROST);
        world.log("Applied FROST to entity " + id);
    }
};

entityManager.forEachActiveEntity(entityProcessor);
```

### Anti-Patterns (Do NOT do this)
- **Blocking Operations:** Never implement this interface with long-running or blocking code (e.g., file I/O, synchronous network requests) if it will be executed on a critical thread like the main game loop or a rendering thread. This will cause severe performance degradation and application stalls.
- **Stateful Lambdas in Parallel Streams:** Avoid using lambdas that modify shared state when passed to parallel processing systems. This is a direct path to race conditions and non-deterministic behavior.
- **Ignoring Primitive Specialization:** Do not use a more generic but less performant alternative like Consumer<Object[]> when this specialized interface is available. Using the correct primitive specialization is critical for maintaining engine performance.

## Data Pipeline
IntTriObjectConsumer does not process a pipeline of data itself; rather, it acts as a **terminal sink** or a **processing stage** within a larger data flow. It consumes data and produces side effects, such as modifying game state, logging, or dispatching commands.

> Flow:
> Data Source (e.g., EntityManager, WorldState) -> Iteration Logic -> **IntTriObjectConsumer Implementation** -> Side Effect (e.g., Game State Mutation, Render Command Queue, Network Packet Dispatch)

