---
description: Architectural reference for TriConsumer
---

# TriConsumer

**Package:** com.hypixel.hytale.function.consumer
**Type:** Utility

## Definition
```java
// Signature
@FunctionalInterface
public interface TriConsumer<T, U, R> {
```

## Architecture & Concepts
TriConsumer is a functional interface that serves as a foundational utility within the engine's functional programming paradigm. It represents an operation that accepts three input arguments and returns no result. Its primary architectural role is to provide a standardized contract for callback functions, event handlers, and iteration logic that require three distinct parameters.

Unlike stateful services, a TriConsumer is a stateless behavior definition. It acts as a "sink" or a terminal operation, consuming data to produce a side effect, such as modifying game state, logging information, or dispatching a subsequent event. Its inclusion in the codebase promotes a more declarative style, allowing developers to pass behavior as data, which is critical for systems like event buses, command processors, and complex data iterators.

## Lifecycle & Ownership
As a functional interface, TriConsumer does not possess a lifecycle in the same manner as a managed service or entity. Its instances are value-like objects whose lifetime is determined by standard Java garbage collection rules.

- **Creation:** Instances are typically created ad-hoc via lambda expressions or method references at the call site. They are not instantiated by a dependency injection container or a central factory.
- **Scope:** The scope is transient and local. An instance of a TriConsumer exists only as long as it is referenced by the object or method that created it. For example, if passed as an argument to a method, it is eligible for garbage collection after the method returns, unless captured by a more persistent object.
- **Destruction:** Cleanup is managed automatically by the JVM garbage collector. There are no explicit destruction or teardown methods.

## Internal State & Concurrency
- **State:** The interface itself is stateless. It defines a contract for behavior, not data.
- **Thread Safety:** The interface contract is inherently thread-safe. However, the thread safety of any specific *implementation* is the responsibility of the developer. Implementations that mutate shared state from multiple threads without proper synchronization will introduce race conditions.

**WARNING:** Implementations of TriConsumer, especially those defined as lambdas, should be stateless or operate on thread-local data whenever possible. Capturing and modifying shared, mutable state is a significant concurrency risk and should be avoided unless protected by explicit locking mechanisms.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T, U, R) | void | O(N) | Performs this operation on the given arguments. Complexity is dependent on the implementation. |

## Integration Patterns

### Standard Usage
The primary integration pattern is providing an implementation via a lambda expression to a higher-order function. This is common for iterating over data structures that yield three values per element, such as a map with an index.

```java
// Example: A hypothetical system that processes entity properties
Map<String, Object> properties = entity.getProperties();
int index = 0;

// The forEachProperty method accepts a TriConsumer
entity.forEachProperty((String key, Object value, Integer idx) -> {
    System.out.println("Processing property " + idx + ": " + key + " = " + value);
});
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Avoid creating implementations that rely on mutable instance fields. This breaks the functional paradigm and can lead to unpredictable behavior and severe concurrency bugs if the consumer is used in a multi-threaded context.

    ```java
    // ANTI-PATTERN: This consumer is not reusable and is not thread-safe
    class UnsafeProcessor implements TriConsumer<Entity, World, Float> {
        private int invocationCount = 0; // Unsafe shared state

        @Override
        public void accept(Entity e, World w, Float deltaTime) {
            this.invocationCount++; // Race condition if used concurrently
            // ...
        }
    }
    ```
- **Complex Lambda Bodies:** Do not embed complex, multi-line business logic directly into a lambda. This harms readability and testability. Extract the logic into a well-named private method and use a method reference.

    ```java
    // ANTI-PATTERN
    system.registerHandler((Player p, Packet packet, Connection c) -> {
        // 20 lines of complex logic here...
    });

    // PREFERRED PATTERN
    system.registerHandler(this::handlePlayerPacket);
    // ...
    private void handlePlayerPacket(Player p, Packet packet, Connection c) {
        // 20 lines of complex logic here...
    }
    ```

## Data Pipeline
TriConsumer acts as a terminal point or a processing stage in a data flow. It does not produce an output value, so it does not chain into subsequent functional operations like a Function or a Predicate would.

> Conceptual Flow:
> Data Source (e.g., Event, Collection, Game State) -> Higher-Order Function -> **TriConsumer Implementation** -> Side Effect (e.g., State Mutation, I/O, Logging)

