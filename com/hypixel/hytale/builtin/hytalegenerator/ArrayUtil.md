---
description: Architectural reference for ArrayUtil
---

# ArrayUtil

**Package:** com.hypixel.hytale.builtin.hytalegenerator
**Type:** Utility

## Definition
```java
// Signature
public class ArrayUtil {
```

## Architecture & Concepts

ArrayUtil is a stateless, static utility class designed for high-performance, low-level manipulation of arrays and lists. It centralizes common operations that are frequently required in performance-sensitive domains such as world generation, network serialization, and data processing.

The class intentionally operates on primitive arrays and lists to avoid the overhead associated with more complex collection types. Its design philosophy favors raw performance and memory efficiency over idiomatic Java collections framework patterns.

A key architectural feature is the hybrid search algorithm implemented in the sortedSearch method. It dynamically switches between a linear scan and a binary search based on the input size. This reflects a pragmatic optimization, acknowledging that for small datasets (under 250 elements), the overhead of a binary search setup can be less efficient than a direct, cache-friendly linear traversal.

**WARNING:** This class contains the method brokenCopyOf, which is fundamentally unsafe due to Java's type erasure. Its usage is strongly discouraged and can lead to runtime ClassCastExceptions. It likely exists for a highly specific, legacy use case and should not be used in new code.

## Lifecycle & Ownership

-   **Creation:** As a static utility class, ArrayUtil is never instantiated. Its methods are accessed statically.
-   **Scope:** The class and its methods are available for the entire application lifetime once the class is loaded by the JVM's ClassLoader.
-   **Destruction:** The class is unloaded when its defining ClassLoader is garbage collected. This typically occurs at application shutdown.

## Internal State & Concurrency

-   **State:** ArrayUtil is completely stateless. It contains no member fields and all methods are pure functions whose output depends solely on their inputs.
-   **Thread Safety:** This class is inherently thread-safe. As there is no internal state, multiple threads can invoke its methods concurrently without risk of race conditions or data corruption. The caller is still responsible for ensuring that the collections or arrays passed as arguments are not mutated by other threads during the execution of an ArrayUtil method.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| brokenCopyOf(T[] a) | T[] | O(N) | **UNSAFE**. Creates a shallow copy of an array. Throws ClassCastException at runtime if the returned array is used as its generic type. |
| copy(T[] source, T[] dest) | void | O(N) | Copies elements from a source to a destination array. Throws IllegalArgumentException if array sizes differ. |
| append(T[] a, T e) | T[] | O(N) | Creates a new array with a new element appended to the end. Involves a full array copy. |
| split(List<T> list, int parts) | List<List<T>> | O(N) | Partitions a list into a specified number of sublists, distributing elements as evenly as possible. |
| getPartSizes(int total, int parts) | int[] | O(P) | Calculates the size of each partition for a given total and part count. Helper for the split method. |
| sortedSearch(List<T> list, G gauge, BiFunction comp) | int | O(N) or O(log N) | Finds an element in a sorted list. Uses a linear scan for lists <= 250 elements, otherwise uses a binary search. |

## Integration Patterns

### Standard Usage

ArrayUtil methods should be invoked statically. They are common building blocks for algorithms that partition data for parallel processing or perform efficient lookups.

```java
// Example: Splitting a list of tasks for a worker pool
List<Task> allTasks = getPendingTasks();
int workerCount = 4;

// Partition the main list into a list of sub-lists
List<List<Task>> taskPartitions = ArrayUtil.split(allTasks, workerCount);

// Distribute each partition to a worker thread
for (int i = 0; i < workerCount; i++) {
    workerPool.submit(new TaskProcessor(taskPartitions.get(i)));
}
```

### Anti-Patterns (Do NOT do this)

-   **Usage of brokenCopyOf:** Never use the brokenCopyOf method expecting type safety. The returned array is of type Object[] and will cause a runtime crash if cast to a more specific type.

    ```java
    // CRASHES AT RUNTIME
    String[] original = new String[]{"a", "b"};
    String[] copy = ArrayUtil.brokenCopyOf(original); // Throws ClassCastException
    ```

-   **Searching Unsorted Data:** The sortedSearch method requires the input list to be pre-sorted according to the provided comparator. Passing an unsorted list will result in incorrect or unpredictable results (e.g., returning -1 for an element that exists).

-   **Ignoring Append Performance:** The append method creates a new array and copies all existing elements on every call. Using it in a tight loop is highly inefficient and generates significant garbage collector pressure. For building collections, use an ArrayList instead.

## Data Pipeline

ArrayUtil is not a data pipeline component itself. Rather, it is a foundational toolkit used *within* the stages of a data pipeline to perform low-level data manipulation. It does not manage or route data flow.

> Example Flow:
> Data Source -> Deserializer -> **ArrayUtil.split** (for work distribution) -> Parallel Processor -> Aggregator

