---
description: Architectural reference for IWeightedMap
---

# IWeightedMap<T>

**Package:** com.hypixel.hytale.common.map
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface IWeightedMap<T> {
```

## Architecture & Concepts
The IWeightedMap interface defines a contract for a read-only collection that facilitates probabilistic, weighted selection of its elements. It is a foundational component for any system requiring non-uniform random choice, such as procedural content generation, loot table resolution, or AI decision-making.

The core design principle is the decoupling of the selection logic from the source of randomness. The interface provides multiple overloaded get methods that accept various inputs—from a simple normalized double value to a multi-dimensional coordinate seed. This allows a single, pre-compiled weighted map to be used in diverse contexts, including both purely random and deterministic, coordinate-based procedural systems.

By abstracting the underlying data structure, IWeightedMap allows for different implementations optimized for specific use cases (e.g., small vs. large collections, frequency of access) without altering the consuming systems. The contract is implicitly immutable; it provides no methods for adding, removing, or modifying elements or their weights after creation. This design choice is critical for ensuring thread safety and predictable behavior in concurrent environments like world generation.

## Lifecycle & Ownership
- **Creation:** Objects implementing this interface are not created directly. They are typically constructed by factory or builder classes that take a collection of items and their corresponding weights. For example, a BiomeManager might build an IWeightedMap of biomes during server initialization.
- **Scope:** The lifetime of an IWeightedMap implementation is tied to its owner. A map defining global biome distribution will persist for the entire server session. A temporary map used for a single procedural calculation (e.g., selecting decorative features within a chunk) may be scoped to that specific operation and garbage collected shortly after.
- **Destruction:** Managed by the Java garbage collector. An IWeightedMap is eligible for cleanup when its owning system (e.g., a WorldGenerator instance) is de-referenced and no other references remain.

## Internal State & Concurrency
- **State:** An IWeightedMap is a stateful contract, representing a fixed collection of elements and their associated weights. Implementations are expected to be **immutable** or at least behave as such from a public API perspective. The lack of mutation methods in the interface enforces this. Internal data structures often involve pre-calculated cumulative weight arrays to enable efficient O(log n) or O(1) lookups.
- **Thread Safety:** The interface itself does not guarantee thread safety, but the immutable design strongly implies that implementations **must be thread-safe for read operations**. It is a common and expected pattern to build an IWeightedMap on a main thread and then share it for concurrent sampling by multiple worker threads (e.g., parallel chunk generation).

**WARNING:** Developers implementing this interface must ensure that all get methods are re-entrant and free of side effects to support safe concurrent access.

## API Surface
The public contract is focused entirely on querying and transforming the collection.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(double) | T | O(log N) | Selects an item using a normalized random value [0,1). |
| get(Random) | T | O(log N) | Selects an item using the provided Random instance. |
| get(int, int, BiIntToDoubleFunction) | T | O(log N) | Selects an item deterministically based on 2D integer coordinates. |
| get(long, long, BiLongToDoubleFunction) | T | O(log N) | Selects an item deterministically based on 2D long coordinates. |
| get(int, int, int, SeedCoordinateFunction, K) | T | O(log N) | Selects an item deterministically using a 3D seed and a context object. |
| forEachEntry(ObjDoubleConsumer) | void | O(N) | Iterates over all entries, providing the item and its weight. |
| resolveKeys(Function, IntFunction) | IWeightedMap<K> | O(N) | Creates a new IWeightedMap by applying a transformation function to each item. |

## Integration Patterns

### Standard Usage
The primary pattern is to retrieve a pre-configured IWeightedMap from a central manager or registry and use it for deterministic, coordinate-based selection in procedural generation.

```java
// In a world generation context
// biomeMap is an IWeightedMap<Biome> provided by the engine
IWeightedMap<Biome> biomeMap = worldConfig.getBiomeMap();

// Deterministically select a biome for a specific world column
// The same (x, z) will always yield the same biome
Biome biome = biomeMap.get(chunkX, chunkZ, NoiseFunctions.SIMPLEX_2D);
```

### Anti-Patterns (Do NOT do this)
- **Re-creation in Loops:** Implementations can be computationally expensive to build. Do not construct a new IWeightedMap inside a performance-critical loop. They are designed to be created once and reused many times.
- **Dependence on Implementation:** Your code should depend on the IWeightedMap interface, not a concrete class like ArrayBackedWeightedMap. This allows the underlying engine to substitute more efficient implementations without breaking game logic.
- **Assuming Uniform Distribution:** The entire purpose of this class is to handle *non-uniform* distributions. Do not use it if you simply need to pick a random item from a list with equal probability; use a standard List and Random for that.

## Data Pipeline
IWeightedMap acts as a transformation node in a data pipeline, converting a source of randomness or a deterministic seed into a discrete categorical outcome.

> Flow:
> Coordinate Seed (x, y, z) -> Noise Function -> Normalized Value [0,1) -> **IWeightedMap.get()** -> Selected Item (e.g., Biome, BlockType, LootItem) -> Downstream System (e.g., ChunkGenerator, LootProcessor)

