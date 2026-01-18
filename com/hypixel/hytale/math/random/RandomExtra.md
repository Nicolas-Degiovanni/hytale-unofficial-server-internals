---
description: Architectural reference for RandomExtra
---

# RandomExtra

**Package:** com.hypixel.hytale.math.random
**Type:** Utility

## Definition
```java
// Signature
public final class RandomExtra {
```

## Architecture & Concepts

RandomExtra is a static utility class that serves as the engine's primary source for stochastic operations. It provides a comprehensive suite of high-performance, thread-safe methods for generating random data, ranging from simple numerical values to complex, weighted collection sampling.

Architecturally, this class is a stateless facade over Java's `java.util.concurrent.ThreadLocalRandom`. This design choice is critical for engine performance. By using a thread-local generator, the system completely avoids the lock contention and performance bottlenecks associated with a single, shared `java.util.Random` instance. Each thread that invokes a RandomExtra method operates on its own isolated random number generator, ensuring maximum throughput in highly concurrent environments such as server-side world simulation or parallel client-side rendering tasks.

Beyond primitive generation, RandomExtra implements foundational game development algorithms like weighted selection and reservoir sampling. These are essential for systems such as loot table processing, procedural content generation, and non-deterministic AI behavior.

## Lifecycle & Ownership

-   **Creation:** As a static utility class with a private constructor, RandomExtra is never instantiated. The Java Virtual Machine loads the class definition into memory at runtime.
-   **Scope:** The class and its static methods are available globally throughout the application's entire lifecycle.
-   **Destruction:** The class is unloaded from memory only when the JVM shuts down. There is no manual cleanup or destruction process.

## Internal State & Concurrency

-   **State:** RandomExtra is **completely stateless**. It contains no member fields and all operations are self-contained within their respective method scopes. The underlying state of the random number generators is managed internally by `ThreadLocalRandom` on a per-thread basis.
-   **Thread Safety:** This class is **inherently thread-safe**. Its stateless design and exclusive reliance on `ThreadLocalRandom` guarantee that calls from different threads cannot interfere with one another. This eliminates the need for external synchronization or locking mechanisms when using this utility, making it safe for use in any part of the engine.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| randomRange(from, to) | int | O(1) | Generates a uniformly distributed integer within the inclusive range. |
| randomElement(List) | T | O(1) | Selects a random element from a List. Assumes list provides O(1) access. |
| jitter(Vector3d, double) | Vector3d | O(1) | Modifies the input vector in-place by adding a random offset to each component. |
| randomWeightedElement(Collection, ToDoubleFunction) | T | O(N) | Selects an element from a collection based on a provided weight function. Requires a full pass to sum weights. |
| reservoirSample(List, Predicate, int, List) | void | O(N) | Populates an output list with a fixed-size random sample from a larger input list, an algorithm ideal for large or streaming data sets. |
| pickWeightedIndex(double[]) | int | O(N) | Selects an array index based on its corresponding weight. |

## Integration Patterns

### Standard Usage

RandomExtra methods should be invoked statically wherever random data is required. It is the canonical source for randomness within the engine.

```java
// Example: Determining a critical hit chance in a combat system.
if (RandomExtra.randomRange(0.0, 1.0) <= CRITICAL_CHANCE) {
    // Apply critical damage
}

// Example: Selecting a monster to spawn from a weighted list.
List<MonsterData> spawnableMonsters = ...;
MonsterData toSpawn = RandomExtra.randomWeightedElement(
    spawnableMonsters, 
    monster -> monster.getSpawnWeight()
);
world.spawnEntity(toSpawn);
```

### Anti-Patterns (Do NOT do this)

-   **Attempted Instantiation:** Do not attempt to create an instance of RandomExtra via reflection. The class is designed to be purely static.
-   **Performance Ignorance:** Calling `randomWeightedElement` repeatedly on a large, unchanging collection is inefficient. If the weights are static, consider pre-calculating the total weight and using the overload that accepts `sumWeights` to avoid redundant calculations.
-   **Security-Sensitive Use:** This class is not suitable for cryptographic operations. `ThreadLocalRandom` is not a cryptographically secure pseudo-random number generator (CSPRNG). Do not use it for generating secrets, tokens, or for any security-related functionality.

## Data Pipeline

RandomExtra typically acts as a data source or a behavioral modifier rather than a step in a processing pipeline. It injects non-determinism into various engine systems.

> **Example Flow 1: AI Behavior Selection**
> AI System Clock Tick -> Query Nearby Entities -> **RandomExtra.randomWeightedElement(possibleActions)** -> Execute Selected Action

> **Example Flow 2: Procedural Decoration**
> World Chunk Generation -> Identify Placement Surfaces -> **RandomExtra.reservoirSample(possibleDecorations)** -> Place Selected Decorations in Chunk

