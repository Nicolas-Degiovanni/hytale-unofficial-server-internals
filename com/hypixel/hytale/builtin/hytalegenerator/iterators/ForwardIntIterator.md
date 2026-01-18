---
description: Architectural reference for ForwardIntIterator
---

# ForwardIntIterator

**Package:** com.hypixel.hytale.builtin.hytalegenerator.iterators
**Type:** Transient

## Definition
```java
// Signature
public class ForwardIntIterator implements IntIterator, Iterator<Integer> {
```

## Architecture & Concepts
The ForwardIntIterator is a low-level, performance-critical utility class designed for efficient iteration over a range of primitive integers. It is a foundational component within the world generation system, where iterating over coordinates or indices in tight loops is a common and computationally expensive operation.

By implementing the fastutil IntIterator interface, this class avoids the overhead of autoboxing primitive integers into Integer objects on each step of an iteration. This design choice significantly reduces garbage collection pressure and improves throughput in algorithms that perform millions of iterations, such as procedural terrain generation, biome placement, and structure population.

This class should be considered a specialized tool, not a general-purpose iterator. Its existence is a direct consequence of performance requirements in the Hytale world generator.

## Lifecycle & Ownership
- **Creation:** An instance is created directly by the calling code using the public constructor: new ForwardIntIterator(min, max). It is not managed by a service locator or dependency injection container. The creator is the sole owner.
- **Scope:** The object's lifetime is ephemeral and is typically confined to the scope of a single method or algorithmic pass. It is designed to be short-lived.
- **Destruction:** The object is eligible for garbage collection as soon as it falls out of scope. There are no manual cleanup or disposal methods required.

## Internal State & Concurrency
- **State:** The internal state is **mutable**, consisting of the current iteration value and the maximum boundary. Each call to next or nextInt modifies this internal state by advancing the current value.
- **Thread Safety:** This class is **not thread-safe** and is fundamentally unsafe for concurrent use. It contains no locks or synchronization primitives. A single instance must be confined to the thread that created it. Sharing an instance across multiple threads will lead to race conditions and non-deterministic behavior.

**WARNING:** Do not share instances of ForwardIntIterator between threads. If multiple threads need to iterate over the same range, each thread must create its own instance.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ForwardIntIterator(min, maxExclusive) | constructor | O(1) | Creates an iterator for the range [min, maxExclusive). Throws IllegalArgumentException if min > maxExclusive. |
| hasNext() | boolean | O(1) | Returns true if the iteration has more elements. |
| nextInt() | int | O(1) | Returns the next element as a primitive int. This is the preferred, high-performance method. |
| next() | Integer | O(1) | Returns the next element as a boxed Integer. Use only for compatibility with standard Java APIs. |
| clone() | ForwardIntIterator | O(1) | Creates a new iterator instance with identical internal state. |

## Integration Patterns

### Standard Usage
The primary use case is within a standard while loop for procedural generation tasks. Always prefer the primitive-based nextInt method in performance-sensitive code.

```java
// Example: Iterating through a vertical chunk column
ForwardIntIterator yIterator = new ForwardIntIterator(0, 256);
while (yIterator.hasNext()) {
    int currentY = yIterator.nextInt();
    // ... perform generation logic for the block at currentY
}
```

### Anti-Patterns (Do NOT do this)
- **Shared Instances:** Never pass a single iterator instance to multiple worker threads or parallel tasks. The internal state is not protected against concurrent modification.
- **Using next() in Hot Paths:** Avoid calling the `next()` method that returns a boxed Integer inside tight loops. The resulting object allocations can trigger expensive garbage collection cycles and degrade performance.
- **Modification after Cloning:** The clone method creates a new iterator with the same state. Be aware that the original and the clone are independent; advancing one will not affect the other.

## Data Pipeline
This class acts as a data source, not a processing stage. It generates a sequence of primitive integers that are consumed by downstream algorithms.

> Flow:
> **ForwardIntIterator** -> Primitive Integer Sequence -> World Generation Algorithm (e.g., Terrain Carver, Ore Placer)

