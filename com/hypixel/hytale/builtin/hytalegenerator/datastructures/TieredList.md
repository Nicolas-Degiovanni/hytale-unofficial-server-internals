---
description: Architectural reference for TieredList
---

# TieredList

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures
**Type:** Utility

## Definition
```java
// Signature
public class TieredList<E> {
```

## Architecture & Concepts
The TieredList is a specialized collection designed to group elements into distinct, integer-indexed "tiers". Its primary architectural purpose is to function as a priority queue where priority is determined by the tier index; lower-numbered tiers are considered higher priority.

Internally, it is implemented as a map from an integer tier to a list of elements (`Map<Integer, ArrayList<E>>`). This structure is optimized for algorithms that need to process items in a specific, ordered sequence of batches. For example, in world generation, foundational terrain features (tier 0) must be processed before decorative elements like foliage (tier 10).

Unlike a standard PriorityQueue which orders individual elements, TieredList preserves the insertion order of elements *within* the same tier. The `peek` and `remove` methods operate on the lowest-numbered, non-empty tier, enforcing a strict "First-In, First-Out" (FIFO) behavior at the tier level.

## Lifecycle & Ownership
-   **Creation:** An instance is created directly via its constructor: `new TieredList()`. It is a standard Java object, not managed by a dependency injection framework or service locator. Its creator is its sole owner.
-   **Scope:** The lifecycle of a TieredList instance is bound to the object that creates it. It is typically used as a short-lived data structure within a single complex method or algorithmic pass.
-   **Destruction:** The object is eligible for garbage collection once all references to it are released. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
-   **State:** The internal state is **mutable**. The core state is held in a HashMap, which stores the lists of elements for each tier. It also maintains a cached, sorted list of tier keys (`sortedTierList`) to optimize iteration. This cache is rebuilt whenever a tier is added or removed.

-   **Thread Safety:** This class is **not thread-safe**. All internal collections (`HashMap`, `ArrayList`) are unsynchronized. Concurrent modification from multiple threads will result in undefined behavior, including `ConcurrentModificationException` and data corruption.

    **WARNING:** Any use of TieredList in a multi-threaded context requires external synchronization (e.g., a `synchronized` block) around all read and write operations.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add(E element, int tier) | void | Amortized O(1) | Adds an element to the specified tier. If the tier does not exist, it is created dynamically. |
| addTier(int tier) | TieredList | O(N log N) | Explicitly creates a new, empty tier. N is the number of existing tiers. |
| removeTier(int tier) | TieredList | O(N log N) | Removes a tier and all elements within it. N is the number of existing tiers. |
| peek() | E | O(T) | Returns the first element from the lowest-numbered, non-empty tier without removing it. T is the `tiers` value from the constructor. |
| remove() | E | O(T) | Removes and returns the first element from the lowest-numbered, non-empty tier. T is the `tiers` value from the constructor. |
| forEach(Consumer) | TieredList | O(N + M) | Iterates over all elements in tier-sorted order. N is the number of tiers, M is the total number of elements. |
| getTiers() | List<Integer> | O(1) | Returns a cached, sorted, unmodifiable list of all tier indices. |
| listOf(int tier) | List<E> | O(1) | Returns an unmodifiable view of the elements in a specific tier. |

## Integration Patterns

### Standard Usage
The intended use is to populate the list with data across various tiers and then process it in a single, ordered pass using the `forEach` method.

```java
// How a developer should normally use this
TieredList<GenerationTask> tasks = new TieredList<>();

tasks.add(new PlaceRiverTask(), 1);
tasks.add(new PlaceMountainTask(), 0);
tasks.add(new PlaceTreeTask(), 2);
tasks.add(new PlaceBedrockTask(), 0);

// Process tasks in guaranteed order: Bedrock, Mountain, River, Tree
tasks.forEach(task -> {
    task.execute();
});
```

### Anti-Patterns (Do NOT do this)
-   **Concurrent Access:** Do not share a TieredList instance across multiple threads without external locking. The internal state is not protected.

-   **Reliance on `peek`/`remove` with Dynamic Tiers:** The `peek` and `remove` methods iterate from tier 0 up to the `tiers` value provided in the constructor. They will **fail to find elements** in tiers added dynamically with an index greater than or equal to this initial value. This is a critical implementation detail that can lead to unexpected behavior and infinite loops.

    **WARNING:** Prefer the `forEach` iterator for processing, as it correctly iterates over all dynamically added tiers. Use `peek` and `remove` only when the set of tiers is fixed and known at construction time.

-   **Modification of Returned Collections:** The `listOf` method correctly returns an unmodifiable list, preventing external state corruption. Do not attempt to bypass this protection.

## Data Pipeline
TieredList does not represent a data processing pipeline itself, but rather serves as an ordered staging area or buffer within a larger algorithmic pipeline. It allows a system to collect and sort work items before batch execution.

> Flow:
> Generation Algorithm -> **TieredList.add(task, priority)** -> Staging Buffer -> **TieredList.forEach(processTask)** -> World State Update

