---
description: Architectural reference for DoubleRangeMap
---

# DoubleRangeMap

**Package:** com.hypixel.hytale.builtin.hytalegenerator.rangemaps
**Type:** Transient

## Definition
```java
// Signature
public class DoubleRangeMap<T> {
```

## Architecture & Concepts
The DoubleRangeMap is a specialized data structure that maps a continuous range of double-precision floating-point numbers to a discrete value of a generic type T. Unlike a standard Map which associates a single key with a value, this class associates an entire interval, defined by a DoubleRange object, with a value.

Its primary role is within data-driven systems like world generation, where it can be used to translate continuous noise values into categorical data. For example, it can map an elevation range of 0.5 to 0.8 to a "Mountain" biome type, or a temperature range of -10.0 to 0.0 to an "Ice" block type.

The internal implementation is based on two parallel ArrayLists: one for DoubleRange objects and one for their corresponding values. This design is simple but has significant performance implications. Lookups are performed via a linear scan, resulting in O(N) time complexity.

**WARNING:** The class does not enforce or manage overlapping ranges. If multiple ranges contain the lookup key, the `get` method will return the value associated with the *first* matching range found in the internal list. This behavior is entirely dependent on the insertion order of the `put` operations.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its default constructor (`new DoubleRangeMap<>()`). It is a Plain Old Java Object (POJO) and is not managed by any service locator or dependency injection framework. It is typically created on-demand by a higher-level system, such as a BiomeGenerator or a MaterialSelector.
- **Scope:** The object's lifetime is bound to its owner. It is designed to be a short-lived, task-specific data container. For instance, a generator might build a DoubleRangeMap, use it to process a large set of data points, and then allow it to be garbage collected once the task is complete.
- **Destruction:** The object is managed by the Java garbage collector. It holds no native resources and requires no explicit cleanup. It becomes eligible for collection as soon as it is no longer referenced.

## Internal State & Concurrency
- **State:** The internal state is mutable and consists of two parallel ArrayLists, `ranges` and `values`. Each call to the `put` method appends elements to these lists, modifying the object's state.
- **Thread Safety:** **This class is not thread-safe.** The underlying use of unsynchronized ArrayLists makes it vulnerable to race conditions and inconsistent state if accessed concurrently. Any and all access from multiple threads must be protected by external synchronization mechanisms, such as a `synchronized` block or a ReentrantLock. Failure to do so will lead to undefined behavior, including potential ConcurrentModificationExceptions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(double k) | T | O(N) | Scans for the first range containing k and returns the associated value, or null if no range matches. |
| put(DoubleRange range, T value) | void | O(1) | Appends a new range-value pair. Amortized constant time. |
| ranges() | List<DoubleRange> | O(N) | Returns a defensive shallow copy of the internal list of ranges. |
| values() | List<T> | O(N) | Returns a defensive shallow copy of the internal list of values. |
| size() | int | O(1) | Returns the number of range-value pairs stored. |

## Integration Patterns

### Standard Usage
The class is intended to be used as a simple lookup table for translating continuous data into categories within a single-threaded context, such as a procedural generation algorithm.

```java
// Example: Mapping elevation noise to a BiomeType enum
DoubleRangeMap<BiomeType> biomeMap = new DoubleRangeMap<>();

// Populate the map based on generator configuration
biomeMap.put(new DoubleRange(0.0, 0.4), BiomeType.OCEAN);
biomeMap.put(new DoubleRange(0.4, 0.6), BiomeType.PLAINS);
biomeMap.put(new DoubleRange(0.6, 1.0), BiomeType.MOUNTAINS);

// During world generation, look up the biome for a given elevation
double currentElevation = noise.get(x, z); // Assume this returns 0.75
BiomeType result = biomeMap.get(currentElevation); // result will be BiomeType.MOUNTAINS
```

### Anti-Patterns (Do NOT do this)
- **Concurrent Modification:** Sharing a single DoubleRangeMap instance across multiple worker threads without external locking is a critical error and will lead to data corruption or application crashes.
- **Performance-Critical Lookups:** Do not use this class for very large datasets where lookup performance is a bottleneck. The O(N) linear scan becomes prohibitively slow as the number of ranges increases. For such use cases, an Interval Tree or a similar spatial partitioning data structure is required.
- **Assuming Insertion Order Invariance:** Adding overlapping ranges and expecting a specific outcome is fragile. The behavior depends entirely on insertion order, which can be an obscure source of bugs. Ranges should ideally be non-overlapping.

## Data Pipeline
DoubleRangeMap acts as a transformation or classification stage within a larger data processing pipeline. It takes a continuous value as input and outputs a discrete, categorical value.

> Flow:
> Noise Function Output (double) -> **DoubleRangeMap** -> Categorical ID (e.g., BiomeType, BlockID) -> Chunk Builder

