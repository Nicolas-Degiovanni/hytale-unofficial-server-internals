---
description: Architectural reference for ClimateMaskProvider
---

# ClimateMaskProvider

**Package:** com.hypixel.hytale.server.worldgen.climate
**Type:** Immutable Strategy Object

## Definition
```java
// Signature
public class ClimateMaskProvider extends MaskProvider {
```

## Architecture & Concepts
The ClimateMaskProvider is a foundational component of the procedural world generation engine. Its primary responsibility is to act as the definitive source of truth for the climate type at any given world coordinate. It is a specialized implementation of the abstract MaskProvider, which defines a generic interface for querying two-dimensional spatial data.

This provider implements a sophisticated, two-layer strategy to balance procedural infinity with authored design:

1.  **Unique Zone Prioritization:** It first queries a UniqueClimateGenerator to determine if the coordinate falls within a specially designated, non-procedural climate zone. This allows designers to override the noise-based generation for specific world features, such as a capital city, a cursed forest, or a volcanic caldera.

2.  **Procedural Noise Fallback:** If the coordinate does not belong to a unique zone, the provider falls back to a multi-layered noise algorithm, managed by the ClimateNoise service. This noise function, in conjunction with a ClimateGraph, ensures that the vast majority of the world is filled with a seamless, varied, and deterministic distribution of climates.

The class is designed to be immutable. Operations that modify the set of unique zones, such as `generateUniqueZones`, do not alter the existing instance but instead return a new, derivative ClimateMaskProvider. This pattern is critical for maintaining determinism and avoiding side effects during different stages of world generation.

## Lifecycle & Ownership
-   **Creation:** An initial, base ClimateMaskProvider is instantiated by a top-level world generation orchestrator during the server's bootstrap sequence. This initial instance is typically configured with a base ClimateNoise profile but an empty UniqueClimateGenerator.
-   **Scope:** Instances are transient and their lifetime is tied to specific stages of world generation. The immutable nature of the class means that as unique zones are defined and applied, a chain of ClimateMaskProvider instances is created, with each new instance inheriting the state of the previous and adding its own modifications. An instance is held as long as it is needed for generating a specific set of world chunks.
-   **Destruction:** Objects are managed by the Java garbage collector. There are no native resources or explicit cleanup methods. An instance is eligible for collection as soon as the world generation stage that created it is complete and no longer holds a reference to it.

## Internal State & Concurrency
-   **State:** The ClimateMaskProvider is **strictly immutable**. All of its core dependencies (randomizer, noise, graph, uniqueGenerator) are final fields injected via the constructor. This design guarantees that a given provider instance will produce identical output for the same input coordinates throughout its lifetime.

-   **Thread Safety:** The class is **thread-safe** for concurrent invocation from multiple chunk generation threads. This safety is achieved through two mechanisms:
    1.  **Immutability:** Since the object's internal state cannot change after construction, multiple threads can read from it without risk of data corruption.
    2.  **Thread-Local Dependencies:** The most performance-critical operation, `get`, relies on a `ClimateNoise.Buffer` to avoid repeated memory allocations. This buffer is retrieved from `ChunkGenerator.getResource()`, which provides a thread-local instance. This ensures that parallel calls to `get` do not create race conditions by overwriting a shared buffer.

    **WARNING:** A critical, non-obvious contract exists between the `get` and `distance` methods. The `distance` method reads a value that is calculated and stored in the thread-local buffer as a side effect of calling `get`. Calling `distance` without a preceding `get` call on the same thread will result in reading stale, incorrect data.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | int | O(N) | Returns the final climate ID for a world coordinate. Complexity depends on the underlying noise function. This is the primary method of the class. |
| distance(x, y) | double | O(1) | Returns the climate fade value from the most recent `get` operation on the current thread. **Must only be called immediately after `get`.** |
| generateUniqueZones(...) | MaskProvider | O(C) | Factory method. Returns a **new** ClimateMaskProvider instance that incorporates a set of unique climate zones. C is the number of candidates. |
| getUniqueZoneCandidates(...) | Zone.UniqueCandidate[] | O(1) | Delegates to the UniqueClimateGenerator to retrieve a list of potential locations for special zones based on world features. |

## Integration Patterns

### Standard Usage
The intended pattern is to start with a base provider and progressively create more specialized providers as unique world features are placed.

```java
// 1. Obtain the base provider from the world generation context.
ClimateMaskProvider baseProvider = worldGenContext.getClimateProvider();

// 2. Find locations for a new, unique zone (e.g., a magic forest).
Map<String, Zone> zoneLookup = worldGenContext.getZoneLookup();
Zone.UniqueCandidate[] candidates = baseProvider.getUniqueZoneCandidates(zoneLookup);

// 3. Create a new, specialized provider that includes the magic forest.
// The original baseProvider remains unchanged.
MaskProvider specializedProvider = baseProvider.generateUniqueZones(seed, candidates, random, collector);

// 4. Use the new provider to generate chunks in that region.
int climateAtFeature = specializedProvider.get(seed, 1530, -450);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new ClimateMaskProvider()` in gameplay or feature code. The root provider is constructed and configured by the core engine. Subsequent providers must be created via `generateUniqueZones`.
-   **Ignoring Return Value:** The `generateUniqueZones` method is a factory, not a mutator. The original instance is not modified. Failure to capture and use the returned `MaskProvider` will result in the unique zones having no effect.
-   **Invalid Call Order:** Do not call `distance(x, y)` without an immediately preceding call to `get(seed, x, y)` on the same thread. This will lead to unpredictable behavior and subtle world generation bugs.

## Data Pipeline
The `get` method follows a clear decision-making flow to determine the final climate value.

> Flow:
> World Coordinate (x, y) -> **ClimateMaskProvider.get()** -> UniqueClimateGenerator.generate()
> > &nbsp;&nbsp;&nbsp;&nbsp;`?` Is a unique climate found?
> > &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`YES` -> Return Unique Climate ID
> > &nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;`NO` -> ClimateNoise.generate() -> ClimateGraph Lookup -> Return Procedural Climate ID

