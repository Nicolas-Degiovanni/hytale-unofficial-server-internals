---
description: Architectural reference for BackwardIntIterator
---

# BackwardIntIterator

**Package:** com.hypixel.hytale.builtin.hytalegenerator.iterators
**Type:** Transient Utility

## Definition
```java
// Signature
public class BackwardIntIterator implements IntIterator, Iterator<Integer> {
```

## Architecture & Concepts
The BackwardIntIterator is a low-level, performance-oriented utility component designed for efficient, reverse iteration over a range of primitive integers. It is a fundamental building block, likely employed within tight loops in procedural generation algorithms, such as world generation or biome placement, where iterating from a high value to a low value is required.

By implementing the fastutil IntIterator interface, this class avoids the performance overhead associated with Java's standard boxing and unboxing of Integer objects. This makes it suitable for performance-critical code paths that execute millions of times per second. Its design is simple and stateful, intended for single-threaded, ephemeral use.

## Lifecycle & Ownership
- **Creation:** Instances are created directly via the public constructor: new BackwardIntIterator(min, maxExclusive). It is not managed by a service locator or dependency injection container. Its creation is entirely on-demand by the consuming algorithm.
- **Scope:** The lifecycle of a BackwardIntIterator is ephemeral and strictly tied to the scope of the loop or method in which it was created. It is designed to be used immediately and then discarded.
- **Destruction:** The object is managed by the Java Garbage Collector. It becomes eligible for collection as soon as it falls out of scope and no outstanding references exist. No manual cleanup is required.

## Internal State & Concurrency
- **State:** The iterator maintains a mutable internal state consisting of two primitive integers: *min* and *current*. The *current* field is decremented upon each call to next() or nextInt(), representing the iterator's progress.
- **Thread Safety:** This class is **not thread-safe**. It is a stateful object designed exclusively for single-threaded access. Concurrent calls to next() or nextInt() from multiple threads will result in a race condition, leading to unpredictable behavior, incorrect values, and potential data corruption.

**WARNING:** Instances of BackwardIntIterator must be confined to the thread that created them. Do not share instances across threads.

## API Surface
The public contract is minimal, adhering to the standard iterator pattern.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| BackwardIntIterator(min, max) | constructor | O(1) | Constructs an iterator for the range [min, maxExclusive). Throws IllegalArgumentException if min > maxExclusive. |
| hasNext() | boolean | O(1) | Returns true if the iteration has more elements. |
| nextInt() | int | O(1) | Returns the next element in the iteration as a primitive int. |
| next() | Integer | O(1) | Returns the next element as a boxed Integer. Incurs boxing overhead. |
| getCurrent() | Integer | O(1) | Returns the current value of the iterator without advancing it. |
| clone() | BackwardIntIterator | O(1) | Creates a shallow copy of the iterator, preserving its current state. |

## Integration Patterns

### Standard Usage
The canonical use case is to instantiate the iterator and immediately consume it in a while loop. This pattern ensures performance by leveraging the primitive-specific nextInt method.

```java
// Iterate backwards from 99 down to 0
BackwardIntIterator iter = new BackwardIntIterator(0, 100);

while (iter.hasNext()) {
    int y = iter.nextInt();
    // Perform work with the y coordinate
}
```

### Anti-Patterns (Do NOT do this)
- **Multi-threaded Access:** Never share a single BackwardIntIterator instance between multiple threads. The internal state is not protected by locks or atomic operations.
- **Post-iteration Reuse:** Do not attempt to reuse an iterator after it has been fully consumed (i.e., after hasNext() returns false). It must be re-instantiated.
- **Incorrect Range:** Providing a *min* value greater than *maxExclusive* to the constructor will immediately throw an IllegalArgumentException. Always ensure the range is valid before instantiation.

## Data Pipeline
As a data source, the BackwardIntIterator's role in a data pipeline is to generate a sequence of integers. It typically sits at the very beginning of a processing chain.

> Flow:
> Algorithm Logic -> **new BackwardIntIterator(min, max)** -> Consumer Loop -> Further Processing

