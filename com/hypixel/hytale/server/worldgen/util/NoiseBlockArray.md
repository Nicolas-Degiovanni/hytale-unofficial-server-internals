---
description: Architectural reference for NoiseBlockArray
---

# NoiseBlockArray

**Package:** com.hypixel.hytale.server.worldgen.util
**Type:** Immutable Value Object

## Definition
```java
// Signature
public class NoiseBlockArray {
```

## Architecture & Concepts
The NoiseBlockArray is a fundamental data structure within the server-side procedural world generation engine. It does not represent a physical column of blocks in the world, but rather the *generative definition* for a vertical stack of material layers at any given (X, Z) coordinate.

Architecturally, it serves as a data-driven blueprint for terrain composition. It is composed of an ordered list of NoiseBlockArray.Entry objects, where each entry defines a potential material (e.g., grass, dirt, stone) and the noise-driven rules for determining its thickness.

The order of entries is critical, as it defines the top-to-bottom layering of the world. The `getTopBlockAt` and `getBottomBlockAt` methods traverse this ordered list to find the first non-zero thickness layer from the sky downwards or from the void upwards, respectively. This mechanism is central to how biomes and terrain features define their surface, subsurface, and bedrock layers.

## Lifecycle & Ownership
- **Creation:** NoiseBlockArray instances are not intended for manual instantiation in game logic. They are created by a higher-level configuration service during server startup, typically by deserializing world generation asset files (e.g., JSON definitions for biomes).
- **Scope:** These objects are stateless definitions. Once loaded, they persist for the entire server session and are shared across all world generation threads. A single instance, representing a specific biome's material stack, will be reused for generating every single block column within that biome.
- **Destruction:** Instances are eligible for garbage collection only when the server shuts down or if a hot-reload of world generation assets occurs.

## Internal State & Concurrency
- **State:** **Deeply Immutable**. The internal `entries` array is final. The nested NoiseBlockArray.Entry class is also designed to be immutable. Once a NoiseBlockArray is constructed, its state cannot be altered. This is a deliberate design choice to ensure predictable and deterministic world generation.
- **Thread Safety:** **Inherently Thread-Safe**. Due to its complete immutability, a NoiseBlockArray instance can be safely read by numerous concurrent world generation threads without any requirement for locks, synchronization, or other concurrency control mechanisms. This makes it a highly performant component in a parallelized generation pipeline.

## API Surface
The public API is minimal, focusing exclusively on querying the defined block stack at a given coordinate.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTopBlockAt(seed, x, z) | BlockFluidEntry | O(N) | Scans entries from first to last to find the highest block layer with a non-zero thickness. |
| getBottomBlockAt(seed, x, z) | BlockFluidEntry | O(N) | Scans entries from last to first to find the lowest block layer with a non-zero thickness. |
| getEntries() | NoiseBlockArray.Entry[] | O(1) | Returns a direct reference to the internal definition array. **Warning:** The returned array should be treated as read-only. |

## Integration Patterns

### Standard Usage
A NoiseBlockArray is typically held by a larger world generation controller, such as a Biome object. During chunk generation, this object is queried to determine the material composition for a specific column.

```java
// Within a hypothetical BiomeGenerator service
NoiseBlockArray materialStack = currentBiome.getMaterialStackDefinition();
BlockFluidEntry surfaceBlock = materialStack.getTopBlockAt(worldSeed, blockX, blockZ);

// Logic to place the actual block into the chunk data
chunk.setBlock(blockX, someY, blockZ, surfaceBlock.getBlockId());
```

### Anti-Patterns (Do NOT do this)
- **Manual Instantiation:** Do not use `new NoiseBlockArray()`. These objects are data, not logic, and must be loaded from the server's asset configuration to ensure world consistency.
- **Stateful Wrapping:** Do not wrap this class in a mutable container. Its immutability is key to its thread safety and performance.
- **Misinterpreting Order:** The order of entries provided during construction is paramount. An incorrect order will lead to inverted or nonsensical terrain, such as stone appearing above grass.

## Data Pipeline
The NoiseBlockArray acts as a compiled, in-memory representation of a configuration asset. It sits between the asset loading system and the core block placement logic.

> Flow:
> Biome Definition (JSON Asset) -> Asset Deserializer -> **NoiseBlockArray Instance** -> BiomeGenerator -> `getTopBlockAt(x,z)` -> BlockFluidEntry -> Chunk Voxel Buffer

---

