---
description: Architectural reference for QuadConsumer
---

# QuadConsumer

**Package:** com.hypixel.hytale.function.consumer
**Type:** Utility

## Definition
```java
// Signature
@FunctionalInterface
public interface QuadConsumer<T, U, R, V> {
```

## Architecture & Concepts
The QuadConsumer is a functional interface that serves as a fundamental building block within the engine's event and data processing systems. Its purpose is to define a behavioral contract for any operation that accepts four input arguments and produces no result, focusing solely on performing a side-effect.

This interface extends the standard Java functional programming paradigm (like Consumer and BiConsumer) to handle scenarios with four distinct parameters. It is heavily utilized for defining callbacks, event listeners, and terminal operations in data streams where multiple pieces of context are required to perform an action. By being a functional interface, it allows developers to provide implementations concisely using lambda expressions or method references, promoting a declarative and less verbose coding style.

Architecturally, QuadConsumer decouples the invoker of an operation from the concrete implementation of that operation. A system can define an event that provides four pieces of data without needing to know *how* listeners will consume that data.

## Lifecycle & Ownership
As a functional interface, QuadConsumer itself does not have a lifecycle. Instead, the lifecycle pertains to its *implementations* (i.e., the lambda expressions or anonymous classes that implement it).

-   **Creation:** Implementations are typically created inline as lambda expressions or method references at the point where a callback or listener is registered. For example, `(a, b, c, d) -> System.out.println(a)`.
-   **Scope:** The scope and lifetime of a QuadConsumer implementation are bound to the object that holds a reference to it. If it is registered as a listener with a session-scoped service, it will live for the duration of the session. If it is a local variable in a method, it will be eligible for garbage collection once the method scope is exited, provided it is not captured by a longer-lived object.
-   **Destruction:** An implementation is marked for garbage collection when no more strong references point to it.

## Internal State & Concurrency
-   **State:** The QuadConsumer interface is inherently stateless. However, its implementations can be stateful if they are defined as closures that capture variables from their enclosing scope.

    **WARNING:** Stateful implementations can introduce significant complexity and potential for memory leaks if the captured state has a longer lifecycle than intended. Prefer stateless implementations where possible.

-   **Thread Safety:** The interface itself is thread-safe. The thread safety of a specific *implementation* is the responsibility of the developer. If the code within the lambda's `accept` method modifies shared, mutable state, it must be properly synchronized.

    **WARNING:** Implementations of QuadConsumer used in multi-threaded contexts, such as the networking or asset loading subsystems, **must be thread-safe**. Failure to ensure this will lead to race conditions and data corruption.

## API Surface
The public contract consists of a single abstract method to be implemented.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T, U, R, V) | void | O(N) | Executes the defined operation. Complexity is entirely dependent on the implementation (N). |

## Integration Patterns

### Standard Usage
QuadConsumer is most commonly used to subscribe to events or provide a callback that requires four pieces of contextual information.

```java
// Example: Registering a listener for a block modification event
BlockEventBus bus = context.getService(BlockEventBus.class);

// The lambda is a concrete implementation of QuadConsumer
bus.onBlockModified((World world, Position pos, Block oldBlock, Block newBlock) -> {
    log.info("Block at " + pos + " changed from " + oldBlock.id + " to " + newBlock.id);
});
```

### Anti-Patterns (Do NOT do this)
-   **Complex Logic:** Avoid embedding significant or complex business logic directly within a lambda implementation. This harms readability and testability. If an operation is more than a few lines, encapsulate it in a dedicated, well-named method and provide a method reference.
    ```java
    // BAD: Complex logic in lambda
    bus.onEvent((a, b, c, d) -> {
        // 20 lines of complex state changes...
    });

    // GOOD: Logic encapsulated in a method
    bus.onEvent(this::handleComplexEvent);
    ```
-   **Blocking Operations:** Never implement a QuadConsumer with a long-running or blocking operation if it is to be executed on a performance-critical thread, such as the main game loop or a Netty event loop. Doing so will cause severe performance degradation, stalls, or deadlocks.

## Data Pipeline
QuadConsumer acts as a terminal "sink" in a data flow. It consumes data from up to four sources to produce a side-effect, concluding a particular branch of a data pipeline.

> Flow:
> World State -> **QuadConsumer Implementation** -> Log Output
>
> Network Packet -> **QuadConsumer Implementation** -> Entity State Update
>
> UI Input -> **QuadConsumer Implementation** -> Game Command Execution

