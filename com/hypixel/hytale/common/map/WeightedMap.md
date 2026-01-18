---
description: Architectural reference for WeightedMap
---

# WeightedMap

**Package:** com.hypixel.hytale.common.map
**Type:** Transient

## Definition
```java
// Signature
public class WeightedMap<T> implements IWeightedMap<T> {
```

## Architecture & Concepts

The WeightedMap is an immutable, high-performance data structure designed for weighted random selection. It is a foundational component in systems requiring probabilistic outcomes, such as procedural world generation, loot table resolution, and AI decision-making.

Its core design prioritizes read performance and thread safety after an initial construction phase. This is achieved through a strict separation of a mutable **Builder** and the final immutable **WeightedMap** instance.

Architecturally, the system relies on a few key principles:

1.  **Immutability:** Once a WeightedMap is created via its Builder, its internal state cannot be changed. This guarantees predictable behavior and makes it inherently thread-safe for concurrent reads.
2.  **Builder Pattern:** Construction is handled exclusively by the nested `WeightedMap.Builder` class. This provides a fluent API for populating the map and isolates all mutable operations to the pre-build phase.
3.  **Cumulative Weight Algorithm:** The selection logic in the `get` method does not require pre-calculating a cumulative distribution table. Instead, it performs a linear scan, decrementing a scaled input value by each item's weight. This approach is efficient for moderately sized collections and simplifies the construction logic.
4.  **Parallel Primitive Arrays:** Internally, it stores keys (objects) and values (weights) in separate `T[]` and `double[]` arrays. This avoids the memory overhead of `Map.Entry` objects and improves data locality, which can be beneficial for CPU cache performance during the linear scan of the `get` operation.
5.  **Singleton Optimization:** A specialized internal class, `SingletonWeightedMap`, is used when the final map contains zero or one element. This avoids the overhead of the main selection algorithm when the outcome is deterministic.

## Lifecycle & Ownership

A WeightedMap instance is a transient data-transfer object. Its lifecycle is simple and managed entirely by the Java Garbage Collector.

-   **Creation:** An instance is *exclusively* created by invoking the `build()` method on a `WeightedMap.Builder`. The builder itself is instantiated via the static factory `WeightedMap.builder()`. Direct instantiation of WeightedMap is prohibited by its private constructor.
-   **Scope:** The object's lifetime is determined by its holder. In world generation, a WeightedMap for biome selection might be created, used for a single chunk, and then immediately become eligible for garbage collection. Conversely, a server-wide loot table map might be created at startup and persist for the entire server session.
-   **Destruction:** There are no manual cleanup or `close` methods. The object and its internal arrays are reclaimed by the garbage collector once they are no longer referenced.

## Internal State & Concurrency

-   **State:** The WeightedMap class is **immutable**. Its internal fields, including the `keys` and `values` arrays, are populated once at construction and never modified thereafter. The total weight `sum` is also pre-calculated and stored as a final field.

-   **Thread Safety:**
    -   **WeightedMap:** **Fully thread-safe.** Due to its immutable nature, a single instance can be safely shared and read by multiple threads concurrently without any external synchronization.
    -   **WeightedMap.Builder:** **Not thread-safe.** The builder is a mutable state machine. It must not be shared across threads. All population operations (`put`, `putAll`) and the final `build()` call must be confined to a single thread.

    **WARNING:** Sharing a Builder instance across threads will lead to race conditions, resulting in a corrupted internal state and unpredictable `build()` outcomes.

## API Surface

The public contract is focused on construction via the Builder and selection via the `get` methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| builder(T[] emptyKeys) | static Builder | O(1) | Creates a new builder instance. |
| get(double value) | T | O(N) | The core selection method. Returns an item based on the input value (0.0 to 1.0). |
| get(Random random) | T | O(N) | Convenience method for selection using a standard Random instance. |
| size() | int | O(1) | Returns the number of elements in the map. |
| contains(T obj) | boolean | O(1) | Checks for the existence of a key. Uses an internal HashSet for fast lookups. |
| resolveKeys(...) | IWeightedMap | O(N) | Creates a new WeightedMap by applying a mapping function to the keys. |

## Integration Patterns

### Standard Usage

The standard pattern is to configure a Builder, create an immutable map, and then reuse that map for subsequent selections. This is highly efficient if the weights do not change.

```java
// How a developer should normally use this
// 1. Create a builder. The empty array provides the component type.
WeightedMap.Builder<String> builder = WeightedMap.builder(new String[0]);

// 2. Populate the builder with items and their weights.
builder.put("Iron Ore", 50.0);
builder.put("Coal Ore", 40.0);
builder.put("Diamond Ore", 1.0);

// 3. Build the immutable, thread-safe map.
IWeightedMap<String> oreDistribution = builder.build();

// 4. Use the map for selection, typically with a random source.
Random random = new Random();
String selectedOre = oreDistribution.get(random);
```

### Anti-Patterns (Do NOT do this)

-   **Rebuilding in a Loop:** Do not repeatedly create a builder and build a new map inside a performance-critical loop if the weights are static. This is extremely inefficient and generates significant garbage. Build once, cache the result, and reuse it.
-   **Concurrent Builder Modification:** Never share a single `Builder` instance across multiple threads. All `put` operations on a given builder must be performed from the same thread.
-   **Ignoring the Builder:** Attempting to manage weighted collections manually is error-prone. The `WeightedMap.Builder` -> `IWeightedMap` pattern is the only supported and safe construction method.

## Data Pipeline

WeightedMap is a component that transforms a normalized floating-point value into a discrete object selection. It is often a terminal step in a procedural generation pipeline.

> **Flow (World Generation Example):**
> 2D Noise Function (e.g., Perlin) -> Generates value for (x, z) -> Normalized to [0.0, 1.0) -> **WeightedMap.get(value)** -> Selected Biome Type -> World Chunk Assembler

