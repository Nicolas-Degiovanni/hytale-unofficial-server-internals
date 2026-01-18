---
description: Architectural reference for ArrayUtli
---

# ArrayUtli

**Package:** com.hypixel.hytale.server.worldgen.util
**Type:** Utility

## Definition
```java
// Signature
public class ArrayUtli {
```

## Architecture & Concepts
ArrayUtli is a stateless, static utility class designed to provide common array manipulation algorithms. Its primary function is to offer a standardized and efficient implementation of the Fisher-Yates shuffle algorithm, a core requirement for procedural generation systems.

Positioned within the `server.worldgen.util` package, its main role is to support the world generation pipeline by introducing controlled randomness. Components responsible for procedural placement of terrain features, structures, or entities rely on this utility to randomize collections of potential candidates, preventing predictable or repetitive patterns in the game world. By centralizing this logic, the engine avoids code duplication and ensures a consistent shuffling implementation across different world generation stages.

## Lifecycle & Ownership
- **Creation:** As a class containing only static methods, ArrayUtli is never instantiated. The Java Virtual Machine's ClassLoader loads the class definition into memory upon its first use.
- **Scope:** The class and its static methods are available globally throughout the application's lifetime once loaded.
- **Destruction:** The class definition is unloaded from memory when its defining ClassLoader is garbage collected, which typically occurs only during a full server shutdown.

## Internal State & Concurrency
- **State:** ArrayUtli is **stateless**. It contains no member fields and does not retain any information between method calls. Its output is exclusively determined by the arguments provided at the time of invocation.

- **Thread Safety:** The class itself is inherently **thread-safe** due to its stateless nature. However, the responsibility for thread safety lies with the caller.

    **WARNING:** The arrays passed to `shuffleArray` are modified in-place. The caller must guarantee that no other thread is reading from or writing to the array while the shuffle operation is in progress. Failure to do so will result in non-deterministic behavior and potential data corruption. Similarly, while the methods accept a `java.util.Random` instance, the `Random` class itself is not thread-safe. Callers in a multi-threaded context must ensure proper synchronization or use a thread-local instance (e.g., `java.util.concurrent.ThreadLocalRandom`).

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shuffleArray(int[] ar, Random rnd) | void | O(n) | Performs an in-place Fisher-Yates shuffle on the provided integer array. Throws NullPointerException if either argument is null. |
| shuffleArray(T[] ar, Random rnd) | void | O(n) | Performs an in-place Fisher-Yates shuffle on the provided generic object array. Throws NullPointerException if either argument is null. |

## Integration Patterns

### Standard Usage
The intended use is to call the static `shuffleArray` method directly to randomize the order of elements within an existing array, typically as part of a larger procedural generation algorithm.

```java
// Example: Randomizing potential ore vein locations before placement
OreVeinLocation[] potentialLocations = getVeinLocationsForChunk();
Random chunkRandom = new Random(world.getSeed() + chunk.getPos().hashCode());

// The array is shuffled in-place
ArrayUtli.shuffleArray(potentialLocations, chunkRandom);

// Now iterate through the randomized locations to place a limited number of veins
for (int i = 0; i < MAX_VEINS_PER_CHUNK; i++) {
    placeOreVein(potentialLocations[i]);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance with `new ArrayUtli()`. This provides no value, as all methods are static. A well-designed utility class would feature a private constructor to prevent this.
- **Unsafe Concurrent Use:** Do not pass a shared `java.util.Random` instance from multiple threads without external locking. This will corrupt the state of the random number generator.
- **Concurrent Array Modification:** Do not allow another thread to access an array while it is being passed to `shuffleArray`. The operation is not atomic and the array will be in an indeterminate state during the shuffle.

## Data Pipeline
ArrayUtli does not act as a stage in a data pipeline; rather, it is a functional tool used to transform data *within* a stage. Its role is to inject entropy into an ordered dataset.

> **Flow within a World Generation Stage:**
>
> 1. A list of candidate positions is generated (e.g., all valid tree spawn locations in a biome).
> 2. The list is converted to an array.
> 3. The array is passed to **ArrayUtli.shuffleArray** to randomize the order.
> 4. The parent system iterates over the now-shuffled array to place a subset of the features, ensuring a natural, non-grid-like distribution.

