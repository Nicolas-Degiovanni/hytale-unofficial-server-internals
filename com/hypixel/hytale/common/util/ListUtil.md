---
description: Architectural reference for ListUtil
---

# ListUtil

**Package:** com.hypixel.hytale.common.util
**Type:** Utility

## Definition
```java
// Signature
public class ListUtil {
```

## Architecture & Concepts
ListUtil is a stateless, static utility class designed to provide high-performance, specialized list manipulation algorithms. It complements the standard Java Collections Framework by offering methods tailored for specific engine use cases, often with performance characteristics superior to naive implementations.

The primary design goals of this class are:
1.  **Performance:** Methods are implemented with an awareness of the underlying data structures. For example, the `removeIf` methods iterate backward to avoid the costly re-indexing that occurs when removing from the start or middle of an `ArrayList`.
2.  **Functionality:** It provides complex operations not available in the standard library, such as the `binarySearch` method which operates on a derived key via a `Function`, preventing the need to create intermediate collections for searching.
3.  **Convenience:** It centralizes common but verbose list operations, such as partitioning a list into fixed-size chunks.

This class should be considered a low-level toolkit for any system that performs intensive list processing, such as entity management, data serialization, or chunk processing.

### Lifecycle & Ownership
-   **Creation:** The ListUtil class is never instantiated. Its constructor is implicitly private, and all members are static.
-   **Scope:** Its methods are globally available throughout the application's runtime, for as long as the class is loaded by the JVM.
-   **Destruction:** The class is unloaded by the JVM upon application termination. There is no manual cleanup required.

## Internal State & Concurrency
-   **State:** ListUtil is entirely stateless. It contains no member fields and all operations are pure functions of their inputs. The behavior of a method depends solely on the arguments provided at the time of the call.

-   **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, the methods operate on `List` instances provided by the caller. **CRITICAL WARNING:** The thread safety of any operation is entirely dependent on the thread safety of the list being mutated or read. Passing a non-thread-safe collection, such as a standard `java.util.ArrayList`, to ListUtil methods from multiple threads without external synchronization will result in undefined behavior, including `ConcurrentModificationException` and data corruption. Callers are responsible for ensuring exclusive access to the list during the operation.

## API Surface
The public API consists exclusively of static methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| partition(List<T> list, int sectionSize) | List<List<T>> | O(N) | Divides a list into consecutive sublists. Returns a list of views; does not copy data. |
| removeIf(List<T> list, Predicate<T> predicate) | void | O(N) | Mutates the list, removing elements that match the predicate. Iterates backward for performance. |
| removeIf(List<T> list, BiPredicate<T, U> predicate, U obj) | void | O(N) | Overload of removeIf using a two-argument predicate. |
| emptyOrAllNull(List<T> list) | boolean | O(N) | Checks if a list is empty or if every element it contains is null. Short-circuits on the first non-null element. |
| binarySearch(List<? extends T> l, Function<T, V> func, V key, Comparator<? super V> c) | int | O(log N) | Performs a binary search on a sorted list, comparing against a key extracted from each element by the provided function. |

## Integration Patterns

### Standard Usage
ListUtil methods should be called statically. They are common tools for preparing or filtering data before it is passed between major engine systems.

```java
// Example: Partitioning a list of tasks for parallel processing
List<Task> allTasks = getPendingTasks();
List<List<Task>> taskPartitions = ListUtil.partition(allTasks, 100);

for (List<Task> partition : taskPartitions) {
    taskExecutor.submit(partition);
}

// Example: Efficiently removing inactive entities from a game world list
List<Entity> entities = world.getEntities();
ListUtil.removeIf(entities, entity -> !entity.isActive());
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Modification:** Never call a mutating method like `removeIf` on a list that is being read or modified by another thread without explicit locking. This is the most common source of errors when using this utility.

    ```java
    // BAD: Unsynchronized access from multiple threads
    List<GameObject> objects = new ArrayList<>();
    // Thread 1:
    ListUtil.removeIf(objects, obj -> obj.isExpired());
    // Thread 2:
    for (GameObject obj : objects) { /* ... */ } // Throws ConcurrentModificationException
    ```

-   **Incorrect `binarySearch` Usage:** The `binarySearch` method requires that the input list is already sorted according to the same logic that the provided `Comparator` uses. Failure to pre-sort the list will result in incorrect and unpredictable return values.

## Data Pipeline
ListUtil does not define a data pipeline itself but rather provides the tools to implement stages within one. It acts as a **Transformer** or **Filter** component on collections of data.

> **Flow: Filtering a Game State**
> Raw Entity List -> **ListUtil.removeIf** (predicate: !isActive) -> Active Entity List -> Rendering System

> **Flow: Data Batching**
> Large Data Set (List) -> **ListUtil.partition** -> Batched Data (List of Lists) -> Network Sender / Worker Threads

