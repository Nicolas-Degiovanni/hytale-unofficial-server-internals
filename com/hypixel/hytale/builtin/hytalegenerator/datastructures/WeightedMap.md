---
description: Architectural reference for WeightedMap
---

# WeightedMap<T>

**Package:** com.hypixel.hytale.builtin.hytalegenerator.datastructures
**Type:** Utility

## Definition
```java
// Signature
public class WeightedMap<T> {
```

## Architecture & Concepts
The WeightedMap is a specialized data structure designed for a single, critical purpose: performing weighted random selections. It is not a general-purpose map implementation. Its primary use case is within procedural generation systems, such as loot table resolution, biome selection, or AI behavior weighting, where outcomes must be random but biased towards certain results.

To achieve both flexibility during construction and performance during selection, the WeightedMap internally employs a composite structure of three distinct collections:
*   An **ArrayList** of elements, which preserves insertion order and allows for an efficient linear scan during the selection process.
*   An **ArrayList** of weights, which directly corresponds to the element list by index.
*   A **HashSet** of elements, providing O(1) complexity for existence checks.
*   A **HashMap** mapping elements to their list index, providing O(1) lookups for retrieving an element's weight.

This design optimizes for a "build once, read many" lifecycle. The `add` operation is O(1) amortized, while the core `pick` operation is O(N), where N is the number of elements. This trade-off is acceptable because these maps are typically small and built infrequently, but `pick` may be called thousands of times per second.

A key architectural feature is the ability to transition the map to an immutable state via `makeImmutable`. This is a critical safeguard for preventing runtime modifications after initialization, especially when the map is shared across different systems.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its constructor, for example `new WeightedMap<>()`. It is typically instantiated by configuration loaders, world generators, or other systems that require probabilistic logic.
- **Scope:** The object's lifetime is bound to its owner. It is a transient data structure, not a persistent service. For example, a WeightedMap for a monster's loot table might be created when the monster type is loaded and exist as long as that configuration is in memory.
- **Destruction:** The object is eligible for garbage collection when all references to it are released. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** The WeightedMap is **mutable** by default. Calling the `add` method modifies its internal collections and recalculates the total weight. It can be permanently transitioned to an **immutable** state by calling `makeImmutable`, after which any attempt to modify it will result in an `IllegalStateException`.

- **Thread Safety:** **WARNING:** This class is **NOT thread-safe**. All internal collections are non-synchronized. Concurrent modification (e.g., one thread calling `add` while another calls `pick`) will lead to data corruption, incorrect weight calculations, and likely a `ConcurrentModificationException`.
    - If a WeightedMap must be shared between threads, it is imperative that it is first fully constructed and then made immutable via `makeImmutable` before being shared. Reads on an immutable instance are safe from any thread.
    - If mutations are required from multiple threads, access must be controlled by external synchronization mechanisms, such as a `synchronized` block or a `ReentrantLock`.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add(T element, double weight) | WeightedMap<T> | O(1) amortized | Adds an element with a positive weight. Throws if immutable or weight is negative. |
| get(T element) | double | O(1) | Retrieves the weight for a given element. Returns 0.0 if the element does not exist. |
| pick(Random rand) | T | O(N) | Selects a random element based on its weight. Throws if the map is empty. |
| makeImmutable() | void | O(1) | Permanently locks the map, preventing any further modifications. |
| size() | int | O(1) | Returns the number of elements in the map. |
| allElements() | List<T> | O(N) | Returns a new list containing all elements. This is a defensive copy. |

## Integration Patterns

### Standard Usage
The intended pattern is to build the map, make it immutable, and then use it for repeated selections. This ensures predictable behavior and thread safety for read operations.

```java
// How a developer should normally use this
// 1. Create and populate the map
WeightedMap<String> lootTable = new WeightedMap<>();
lootTable.add("Sword", 10.0);
lootTable.add("Shield", 15.0);
lootTable.add("Potion", 75.0);

// 2. Lock the map to prevent further changes
lootTable.makeImmutable();

// 3. Use the immutable map for selections
// This instance can now be safely shared across threads.
Random random = new Random();
String droppedItem = lootTable.pick(random);
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Never modify a WeightedMap from one thread while another thread might be reading from it (e.g., calling `pick`). This will cause unpredictable behavior. If sharing is required, call `makeImmutable` first.
- **Repetitive Instantiation:** Do not create a new WeightedMap for every single selection. The cost of building the map is non-trivial. Build it once, cache it, and reuse it.
- **Ignoring Immutability:** Failing to call `makeImmutable` on a map that is passed to other systems can lead to downstream code unexpectedly modifying the probability distribution.

## Data Pipeline
The WeightedMap acts as a runtime representation of configured probabilistic data. It is not a pipeline processor itself, but rather the result of one.

> Flow:
> Configuration Source (e.g., JSON file) -> Deserializer -> **WeightedMap Builder** -> `makeImmutable()` -> **Immutable WeightedMap** -> Game System (e.g., LootSystem) -> `pick()` -> Selected Item

