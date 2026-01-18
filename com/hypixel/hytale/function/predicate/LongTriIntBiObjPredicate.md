---
description: Architectural reference for LongTriIntBiObjPredicate
---

# LongTriIntBiObjPredicate

**Package:** com.hypixel.hytale.function.predicate
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface LongTriIntBiObjPredicate<T, V> {
   boolean test(long var1, int var3, int var4, int var5, T var6, V var7);
}
```

## Architecture & Concepts
LongTriIntBiObjPredicate is a highly specialized functional interface designed to represent a boolean-returning test condition (a predicate) that accepts a specific, primitive-heavy set of arguments: one long, three integers, and two generic objects.

This interface is a core component of the engine's functional programming toolkit, specifically tailored for performance-critical systems. Its design directly addresses the need to avoid object allocation and autoboxing overhead in tight loops, such as those found in rendering, physics, or world generation systems. By enforcing a contract with primitive types (long, int), it allows engine subsystems to pass raw data directly into filtering or validation logic without the performance penalty of wrapping them in objects like Long or Integer.

It serves as a behavioral contract, allowing methods to accept complex filtering logic as a parameter without being coupled to a specific implementation.

## Lifecycle & Ownership
As a functional interface, LongTriIntBiObjPredicate does not have a traditional object lifecycle. Its instances are typically lambda expressions or method references.

- **Creation:** An instance is created implicitly whenever a lambda expression or method reference matching its signature is supplied to a method that requires it. The Java runtime synthesizes an object that implements the interface.
- **Scope:** The lifetime of an instance is tied to the object that holds a reference to it. It may be ephemeral (scoped to a single method call) or long-lived (stored as a field in a service or component).
- **Destruction:** The instance is eligible for garbage collection as soon as it is no longer referenced.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, an implementing lambda expression can become stateful if it forms a closure by capturing variables from its enclosing scope.

- **Thread Safety:** The interface contract is thread-agnostic. The thread safety of any given implementation is entirely the responsibility of the developer providing the lambda.

    **WARNING:** Stateful lambdas that capture and modify shared, mutable state are **not** thread-safe and must not be used in concurrent contexts without explicit synchronization. Doing so will introduce severe and difficult-to-diagnose race conditions. It is strongly recommended that implementations remain pure functions.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(long, int, int, int, T, V) | boolean | *Implementation-dependent* | Executes the predicate logic against the provided arguments. Returns true if the condition is met, false otherwise. |

## Integration Patterns

### Standard Usage
This interface is intended to be used as a method parameter, allowing callers to inject custom filtering logic into a generic algorithm.

```java
// A system can define a method that accepts the predicate
public class WorldQuerySystem {
    public List<Entity> findEntities(LongTriIntBiObjPredicate<World, Chunk> filter) {
        // ... logic to iterate through entities
        // The predicate is used as a filter condition
        if (filter.test(entityId, x, y, z, currentWorld, currentChunk)) {
            // ... add entity to results
        }
        // ... return results
    }
}

// A developer provides the logic via a lambda
WorldQuerySystem querySystem = context.getService(WorldQuerySystem.class);
List<Entity> results = querySystem.findEntities(
    (id, x, y, z, world, chunk) -> x > 100 && world.isDaylight()
);
```

### Anti-Patterns (Do NOT do this)
- **Complex Operations:** Do not embed long-running or blocking operations (e.g., file I/O, network requests) inside the predicate. It is often invoked hundreds or thousands of times per second in performance-sensitive loops. The implementation must be fast.
- **Stateful Closures in Parallel Streams:** Avoid using lambdas that modify captured state when working with parallel streams or multi-threaded systems. This is a classic concurrency bug.

    ```java
    // BAD: This will fail unpredictably in a concurrent environment
    List<Integer> counter = new ArrayList<>();
    someParallelSystem.filter((id, x, y, z, world, chunk) -> {
        if (x > 0) {
            counter.add(1); // Race condition: multiple threads modifying list
            return true;
        }
        return false;
    });
    ```

## Data Pipeline
This interface does not manage a data pipeline itself; rather, it acts as a **gate** or **filter** component within a larger data flow. It is a processing stage that conditionally allows data to proceed.

> Flow:
> Data Source (e.g., Voxel Iterator, Entity Stream) -> **LongTriIntBiObjPredicate (Filter)** -> Data Consumer (e.g., Renderer, Logic System)

