---
description: Architectural reference for WeighedShader
---

# WeighedShader

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.shaders
**Type:** Transient

## Definition
```java
// Signature
public class WeighedShader<T> implements Shader<T> {
```

## Architecture & Concepts
The WeighedShader is a composite shader that does not perform a transformation itself. Instead, it acts as a probabilistic router, delegating the shading operation to one of its contained child shaders based on a weighted random selection. This component is fundamental to procedural generation systems where controlled, deterministic variation is required.

By implementing the Shader interface, a WeighedShader can be used interchangeably with any other concrete shader, allowing complex generative behaviors to be built by composing simpler parts. For example, a terrain generator could use a WeighedShader to decide whether a block should be grass (90% chance) or a flower (10% chance).

The selection process is driven by a combination of a `java.util.Random` instance and an internal `WeightedMap`. Crucially, the randomness is made deterministic by the `SeedGenerator`, which combines world coordinates (seedA, seedB, etc.) into a single, stable seed. This ensures that for a given world seed and location, the "random" choice is always the same, guaranteeing reproducible world generation.

## Lifecycle & Ownership
- **Creation:** An instance is created directly via its constructor: `new WeighedShader<>(initialChild, weight)`. It is typically instantiated and configured during the setup phase of a larger system, such as a biome or feature generator.
- **Scope:** The object's lifetime is bound to its owner. It is not a global or session-scoped service. It persists as long as the generator configuration that defines it is in memory.
- **Destruction:** The object is eligible for garbage collection once its owner is no longer referenced. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** This class is **highly mutable**. Its primary state, the collection of child shaders and their weights, is modified via the `add` method. The `setSeed` method also mutates the internal `SeedGenerator`. This design follows a builder pattern, where the object is configured post-instantiation.

- **Thread Safety:** This class is **not thread-safe**. The internal `childrenWeightedMap` is not a concurrent collection. Modifying the map via the `add` method while another thread is calling a `shade` method will result in undefined behavior, likely a `ConcurrentModificationException`.

    **WARNING:** All configuration using `add` or `setSeed` must be completed before the shader is used in a multi-threaded generation context. The object should be treated as immutable after its initial setup phase.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| add(Shader child, double weight) | WeighedShader | O(1) | Adds a child shader to the selection pool with a given weight. Throws IllegalArgumentException for non-positive weights. |
| setSeed(long seed) | WeighedShader | O(1) | Overrides the default seed generator with one based on the provided seed. Essential for deterministic output. |
| shade(T current, long seed) | T | O(log N) + O(child) | Selects a child shader using the provided seed and delegates the shade operation to it. N is the number of children. |
| shade(T current, long seedA, long seedB) | T | O(log N) + O(child) | Combines coordinate seeds into a single seed before performing the weighted selection and delegation. |
| shade(T current, long seedA, long seedB, long seedC) | T | O(log N) + O(child) | Combines three coordinate seeds into a single seed before performing the weighted selection and delegation. |

## Integration Patterns

### Standard Usage
The typical pattern is to instantiate the shader, configure it with multiple children using the builder-style `add` method, and then use it within a generation loop.

```java
// 1. Create with an initial shader
Shader<Block> grassShader = new BlockShader(GRASS);
Shader<Block> flowerShader = new BlockShader(FLOWER);
WeighedShader<Block> terrainShader = new WeighedShader<>(grassShader, 90.0);

// 2. Configure with additional children
terrainShader.add(flowerShader, 10.0);

// 3. Set a deterministic seed for the world
terrainShader.setSeed(world.getSeed());

// 4. Use in a generation loop
for (int x = 0; x < 100; x++) {
    for (int z = 0; z < 100; z++) {
        Block currentBlock = getBlockAt(x, z);
        // The shade call will deterministically choose grass or flower
        Block newBlock = terrainShader.shade(currentBlock, x, z);
        setBlockAt(x, z, newBlock);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Modification During Generation:** Never call the `add` method after the shader has been passed to a generator and is actively being used. This is a severe race condition in multi-threaded environments.
- **Non-Deterministic Seeding:** Failing to call `setSeed` will cause the shader to use `System.nanoTime()` as its base seed. This will result in different world generation outputs on every run, which is almost always undesirable.
- **Invalid Weights:** Passing a weight that is less than or equal to zero will throw an `IllegalArgumentException`. All weights must be positive values.

## Data Pipeline
The WeighedShader transforms coordinate-based seeds and an input value into a final, transformed value by delegating to a probabilistically chosen child.

> Flow:
> (Input Value T, Coordinates) -> **WeighedShader.shade(T, x, y, z)** -> SeedGenerator.seedAt(x, y, z) -> Combined Seed -> new Random(Combined Seed) -> WeightedMap.pick(Random) -> *Selected Child Shader* -> ChildShader.shade(T, Combined Seed) -> Output Value T

