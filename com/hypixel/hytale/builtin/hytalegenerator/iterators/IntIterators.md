---
description: Architectural reference for IntIterators
---

# IntIterators

**Package:** com.hypixel.hytale.builtin.hytalegenerator.iterators
**Type:** Utility

## Definition
```java
// Signature
public class IntIterators {
```

## Architecture & Concepts
The IntIterators class is a stateless factory utility designed for high-performance, primitive-based integer iteration. Its primary role is to provide efficient iterators over numerical ranges, abstracting the direction of iteration (forwards or backwards) from the caller.

This component is a foundational element within the world generation system, where iterating over coordinate ranges (e.g., block heights, chunk sections) is a frequent and performance-critical operation. By leveraging the `fastutil` library's IntIterator, it completely avoids the memory and performance overhead of Java's standard `Integer` object boxing, which is crucial in tight loops that may run millions of times during world generation.

The class acts as a single, clear entry point for creating range-based iterators, ensuring that all calling systems use a consistent, optimized implementation.

### Lifecycle & Ownership
- **Creation:** As a utility class with only static methods, IntIterators is never instantiated. Its bytecode is loaded by the JVM ClassLoader at runtime. The iterator objects it *produces* are created on-demand each time the `range` method is invoked.
- **Scope:** The IntIterators class is static and exists for the lifetime of the application's ClassLoader. The returned IntIterator instances are transient, short-lived objects whose lifecycle is tied to the scope of the calling code, such as a `for` loop or a single method execution.
- **Destruction:** The created iterator instances are eligible for garbage collection as soon as they are no longer referenced by the caller.

## Internal State & Concurrency
- **State:** The IntIterators class is entirely stateless. It holds no member variables and its methods operate only on their input arguments. The IntIterator objects it returns are stateful, as they must track the current position within their range.
- **Thread Safety:** The `range` factory method is inherently thread-safe as it is a pure function. However, the returned IntIterator instances are **not thread-safe**. They are stateful and contain no internal synchronization. An iterator instance must be confined to the thread that created it. Sharing an iterator across threads will result in undefined behavior.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| range(int start, int end) | IntIterator | O(1) | Creates and returns an iterator for a closed integer range. The returned object is highly optimized for primitive int iteration. |

## Integration Patterns

### Standard Usage
The `range` method is intended to be used within standard Java iteration constructs, such as an enhanced-for loop (if adaptable) or a `while` loop. This provides a clean and highly performant way to iterate over a numerical sequence.

```java
// Standard iteration from a lower to a higher bound.
IntIterator yIterator = IntIterators.range(0, 255);
while (yIterator.hasNext()) {
    int currentY = yIterator.nextInt();
    // Perform work for each Y-level in a chunk column
}

// The factory automatically handles reverse iteration.
IntIterator reverseIterator = IntIterators.range(10, 0);
while (reverseIterator.hasNext()) {
    int countdown = reverseIterator.nextInt();
    // ...
}
```

### Anti-Patterns (Do NOT do this)
- **Sharing Iterators:** Never pass an IntIterator instance returned by this class to another thread. Iterators maintain internal state (e.g., the current index) and are not designed for concurrent access.
- **Unnecessary Recreation:** Avoid calling `IntIterators.range()` repeatedly within a loop if the range is constant. The factory call is cheap, but it still involves object allocation.

    ```java
    // BAD: A new iterator is allocated on every outer loop iteration.
    for (int x = 0; x < 16; x++) {
        IntIterator yIter = IntIterators.range(0, 255); // Wasted allocations
        while (yIter.hasNext()) {
            // ...
        }
    }

    // GOOD: Create the iterator once if the range is reusable.
    IntIterator yIter = IntIterators.range(0, 255);
    // ... use the iterator ...
    ```

## Data Pipeline
The data flow for this utility is simple and direct, acting as a producer of iterator objects.

> Flow:
> World Generation Algorithm provides `start` and `end` integers -> **IntIterators.range()** -> Returns `ForwardIntIterator` or `BackwardIntIterator` instance -> Algorithm consumes primitive integers from the iterator.

