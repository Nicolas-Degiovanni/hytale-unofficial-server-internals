---
description: Architectural reference for TriIntConsumer
---

# TriIntConsumer

**Package:** com.hypixel.hytale.function.consumer
**Type:** Utility

## Definition
```java
// Signature
@FunctionalInterface
public interface TriIntConsumer {
   void accept(int var1, int var2, int var3);
}
```

## Architecture & Concepts
TriIntConsumer is a specialized functional interface designed for high-performance operations involving three primitive integer arguments. Its primary architectural purpose is to provide a standardized, allocation-free callback mechanism for systems that operate on integer-based data, such as 3D coordinates (x, y, z), color components, or grid indices.

Unlike a generic `Consumer` from the standard Java libraries which would require boxing integers into `Integer` objects or using an `int[]` array, TriIntConsumer operates directly on the primitive stack values. This design is critical in performance-sensitive code, such as rendering loops, world generation, or particle systems, where avoiding heap allocations and garbage collection pressure is paramount. It serves as a fundamental building block for creating efficient, lambda-based APIs throughout the engine.

## Lifecycle & Ownership
As a functional interface, TriIntConsumer does not possess its own lifecycle in the same way a managed service does. Its lifecycle is entirely dependent on the object that implements it, which is typically a lambda expression or a method reference.

- **Creation:** An instance is created implicitly whenever a lambda expression or method reference matching its signature is declared. For example: `TriIntConsumer myConsumer = (x, y, z) -> { ... };`.
- **Scope:** The scope is determined by the context in which the implementing lambda is defined. It can be a short-lived local variable within a method or a long-lived member of a class.
- **Destruction:** The instance is eligible for garbage collection when the lambda or the object holding a reference to it goes out of scope.

**WARNING:** Be mindful of lambdas that capture state from their enclosing scope (closures). If a long-lived system holds a reference to a TriIntConsumer that captures a short-lived object, it can lead to memory leaks.

## Internal State & Concurrency
- **State:** The interface itself is stateless. However, any lambda expression that implements TriIntConsumer can be stateful if it captures variables from its surrounding scope. The state is owned by the lambda's closure, not the interface.
- **Thread Safety:** The interface imposes no concurrency constraints. Thread safety is **entirely the responsibility of the implementation**. If the lambda's `accept` method modifies shared, mutable state without proper synchronization, it is not thread-safe. Primitives passed to the `accept` method are passed by value, making the arguments themselves inherently safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(int, int, int) | void | O(N) | Executes the consumer's logic. Complexity is dependent on the implementation. |

## Integration Patterns

### Standard Usage
TriIntConsumer is intended to be used as a parameter for methods that need to iterate or process sets of three integers, allowing the caller to inject custom logic efficiently.

```java
// A hypothetical system that iterates over voxel blocks in a region
// and invokes a callback for each one.
public void forEachBlock(int startX, int startY, int startZ, TriIntConsumer action) {
    for (int x = 0; x < 16; x++) {
        for (int y = 0; y < 16; y++) {
            for (int z = 0; z < 16; z++) {
                action.accept(startX + x, startY + y, startZ + z);
            }
        }
    }
}

// Usage:
VoxelSystem system = ...;
system.forEachBlock(0, 64, 0, (x, y, z) -> {
    System.out.println("Processing block at: " + x + ", " + y + ", " + z);
});
```

### Anti-Patterns (Do NOT do this)
- **Unnecessary Instantiation:** Do not use a full anonymous inner class when a lambda or method reference is sufficient. The former is more verbose and can be less performant.
- **Stateful Lambdas in Parallel Streams:** Avoid using stateful lambdas that modify shared state from multiple threads without synchronization. This is a classic race condition.

```java
// BAD: This lambda is stateful and not thread-safe if used in parallel.
List<Integer> results = new ArrayList<>();
TriIntConsumer unsafeConsumer = (x, y, z) -> results.add(x + y + z);
```

## Data Pipeline
This component does not manage a data pipeline itself; rather, it acts as a processing node *within* a larger pipeline. It is a functional sink that terminates a flow or triggers a side effect.

> **Example Flow:**
> Voxel World Iterator -> **TriIntConsumer (Block Processor)** -> Side Effect (e.g., Logging, State Mutation)

