---
description: Architectural reference for ObjectPositionBlockFunction
---

# ObjectPositionBlockFunction

**Package:** com.hypixel.hytale.function.predicate
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface ObjectPositionBlockFunction<T, V, K> {
   K accept(T var1, V var2, int var3, int var4, int var5, int var6);
}
```

## Architecture & Concepts
The ObjectPositionBlockFunction is a generic, stateless behavioral contract that defines a function signature for operations involving two objects and a specific 6-integer coordinate set. Within the Hytale engine, this pattern is fundamental for implementing the Strategy Pattern in world-processing and entity-interaction systems.

Its primary role is to decouple high-level iteration logic (e.g., scanning a region of the world) from the specific action performed at each coordinate. By accepting an implementation of this interface, a system like a WorldManager can execute arbitrary, developer-defined logic without needing to know its details.

The six integer parameters strongly suggest a dual coordinate system, most likely a combination of a world-space block position (x, y, z) and a secondary context-dependent position, such as chunk coordinates or a relative offset. The generic types T, V, and K provide maximum flexibility:
- **T:** Typically the primary context object, such as a World or a Region.
- **V:** Typically a secondary context object, such as an Entity or a query parameter object.
- **K:** The return type, often a Boolean to indicate success/failure, a status enum, or a computed value.

This interface is a cornerstone for creating efficient and reusable game logic for tasks like block queries, structure generation, AI environmental awareness, and physics calculations.

## Lifecycle & Ownership
As a functional interface, ObjectPositionBlockFunction itself has no lifecycle. It is a compile-time contract. The lifecycle and ownership concerns apply to its *implementations*, which are typically lambda expressions or anonymous inner classes.

- **Creation:** An implementation is created dynamically at the point where it is passed as an argument to a method. For example, `world.queryBlocks(..., (world, entity, x, y, z, flags) -> { /* logic */ });`.
- **Scope:** The lifetime of the implementation is transient. It exists for the duration of the method call it is passed into. If it is stored in a field by the receiving system, its lifetime is extended to match the lifetime of that system, but this is an atypical use case.
- **Destruction:** The implementation object is eligible for garbage collection as soon as the method it was passed to completes and no other references to it exist.

**WARNING:** Implementations that capture variables from their enclosing scope (creating a closure) will hold references to those variables, potentially extending their lifetime beyond expectation. Be mindful of memory leaks when capturing large objects.

## Internal State & Concurrency
- **State:** The interface is inherently stateless. However, its implementations can be stateful if they are defined as closures that capture and modify variables from their surrounding scope. Pure functions, which only operate on input parameters, are the preferred implementation style.
- **Thread Safety:** The interface itself is thread-safe. The thread safety of an *implementation* is entirely the responsibility of the developer.
    - **Safe:** Lambdas that are pure functions (no side effects, no access to shared mutable state) are inherently thread-safe and can be used freely in parallel world-processing tasks.
    - **Unsafe:** Lambdas that capture and modify shared, non-thread-safe collections or objects from an outer scope are **not thread-safe**. Accessing such an implementation from multiple threads without explicit synchronization will lead to race conditions, data corruption, and unpredictable behavior.

**WARNING:** Never assume an implementation is thread-safe. When passing a function to a system that may operate concurrently, such as a parallel chunk processor, you MUST ensure your implementation is free of side effects or properly synchronized.

## API Surface
The public contract consists of a single method to be implemented.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(T, V, int, int, int, int) | K | O(impl) | Executes the defined logic. The complexity is entirely dependent on the implementation provided. |

## Integration Patterns

### Standard Usage
This interface is intended to be implemented using a concise lambda expression, passed as a callback to a system that iterates over world positions or entities.

```java
// Example: Finding a specific block type around an entity
World world = ...;
Entity player = ...;
Block targetBlock = world.findBlockInRadius(
    player.getPosition(),
    10,
    // Lambda implementation of ObjectPositionBlockFunction
    (w, e, x, y, z, unusedFlags) -> {
        Block currentBlock = w.getBlockAt(x, y, z);
        if (currentBlock.isType(BlockTypes.DIAMOND_ORE)) {
            return currentBlock; // Return type K is Block
        }
        return null;
    }
);
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations in Parallel Systems:** Providing a lambda that modifies a shared external list to a parallel processing system without using a concurrent collection. This is a classic race condition.
    ```java
    // ANTI-PATTERN: Unsafe concurrent modification
    List<Block> foundBlocks = new ArrayList<>();
    world.forEachBlockInRegionParallel(region, (w, e, x, y, z, f) -> {
        if (w.getBlockAt(x, y, z).isValuable()) {
            foundBlocks.add(w.getBlockAt(x, y, z)); // Race condition!
        }
    });
    ```
- **Complex Class Implementations:** Creating a dedicated, named class that implements ObjectPositionBlockFunction for a simple, one-off operation. This adds unnecessary boilerplate and verbosity where a lambda would be clearer.

## Data Pipeline
ObjectPositionBlockFunction acts as a processing node or a filter within a larger data flow. It does not manage a pipeline itself but is a critical component used by pipeline managers.

> Flow:
> System (e.g., WorldIterator) -> Provides (T, V, int, int, int, int) -> **ObjectPositionBlockFunction Implementation** -> Processes Data -> Returns Result (K) -> System (consumes result)

