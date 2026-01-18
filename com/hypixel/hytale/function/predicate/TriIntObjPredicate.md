---
description: Architectural reference for TriIntObjPredicate
---

# TriIntObjPredicate

**Package:** com.hypixel.hytale.function.predicate
**Type:** Functional Interface

## Definition
```java
// Signature
@FunctionalInterface
public interface TriIntObjPredicate<T> {
```

## Architecture & Concepts
The **TriIntObjPredicate** is a functional interface that defines a behavioral contract for a predicate function. It represents a boolean-valued function that accepts three integer arguments and one generic object argument.

This interface is a fundamental building block for implementing functional programming patterns within the Hytale engine. Its primary role is to provide a standardized type for lambda expressions and method references that perform a test or check. The three integer arguments are commonly used to represent 3D coordinates (x, y, z) in world-space or chunk-space, making this interface prevalent in systems that iterate over or query spatial data, such as world generation, block physics, or entity lookups.

By abstracting the *condition* from the *iteration*, systems can be designed in a highly decoupled and reusable manner. For example, a single world traversal algorithm can be used with dozens of different **TriIntObjPredicate** implementations to find different types of blocks or locations without duplicating the traversal logic.

## Lifecycle & Ownership
As an interface, **TriIntObjPredicate** itself has no lifecycle. The lifecycle described here pertains to the *implementations* of the interface, which are typically lambda expressions or anonymous inner classes.

- **Creation:** Implementations are created inline at the call site where a method requires a **TriIntObjPredicate**. They are instantiated by the Java runtime when the code is executed.
- **Scope:** The lifetime of a lambda instance is tied to its capturing context. If it is passed as a method argument, it is typically eligible for garbage collection after the method returns. If it is assigned to a field of an object, it lives as long as that object.
- **Destruction:** Instances are managed by the Java Garbage Collector. They are destroyed when they are no longer reachable, for example, when the object holding a reference to the lambda is itself collected.

## Internal State & Concurrency
- **State:** The interface is inherently stateless. However, an implementation (a lambda) can become stateful if it *captures* variables from its enclosing scope. Captured state is effectively part of the lambda's internal state.

- **Thread Safety:** The interface contract is thread-agnostic. The thread safety of any given implementation is entirely the responsibility of the developer who writes it.
    - **WARNING:** Lambdas that capture and modify mutable state are **not** thread-safe by default. If a stateful predicate is used across multiple threads, access to the captured state must be synchronized externally or through thread-safe data structures. Stateless implementations (which only operate on their input arguments) are inherently thread-safe.

## API Surface
The public contract consists of a single abstract method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(int, int, int, T) | boolean | Implementation-Defined | Evaluates this predicate on the given arguments. The complexity is determined by the logic within the lambda implementation. |

## Integration Patterns

### Standard Usage
This interface is intended to be used with lambda expressions to provide conditional logic to higher-order functions, often for filtering collections or searching through spatial data.

```java
// Example: A hypothetical method to find blocks matching a condition
public Block findBlock(TriIntObjPredicate<BlockType> condition) {
    for (int x = 0; x < 16; x++) {
        for (int y = 0; y < 256; y++) {
            for (int z = 0; z < 16; z++) {
                Block block = getBlockAt(x, y, z);
                if (condition.test(x, y, z, block.getType())) {
                    return block;
                }
            }
        }
    }
    return null;
}

// Usage of the method with a lambda
Block diamond = findBlock((x, y, z, type) -> type == BlockType.DIAMOND_ORE && y < 16);
```

### Anti-Patterns (Do NOT do this)
- **Complex or Blocking Operations:** Do not perform computationally expensive, I/O-bound, or otherwise blocking operations within the **test** method. Predicates are often executed in tight loops on performance-critical threads (e.g., the main game loop or world-gen threads). A slow predicate can cause severe performance degradation.

- **Stateful Lambdas in Concurrent Scenarios:** Avoid using lambdas that modify shared, non-thread-safe state when the consuming system may be multi-threaded. This is a direct path to race conditions and unpredictable behavior.

    ```java
    // ANTI-PATTERN: A non-atomic counter is modified from a predicate
    // that might be used by a parallel stream or concurrent task executor.
    int foundCount = 0;
    world.findBlocksParallel(
        (x, y, z, type) -> {
            if (type == BlockType.COAL_ORE) {
                foundCount++; // RACE CONDITION!
                return true;
            }
            return false;
        }
    );
    ```

## Data Pipeline
A **TriIntObjPredicate** does not manage a data pipeline itself; rather, it acts as a *gate* or *filter* within a larger pipeline. It consumes data and produces a boolean decision that affects the downstream flow.

> Flow:
> Data Source (e.g., World Block Iterator) -> **TriIntObjPredicate Implementation** (Filter Logic) -> Downstream Consumer (e.g., Filtered Collection, First Match)

