---
description: Architectural reference for WaterPopulator
---

# WaterPopulator

**Package:** com.hypixel.hytale.server.worldgen.chunk.populator
**Type:** Utility

## Definition
```java
// Signature
public class WaterPopulator {
```

## Architecture & Concepts
The WaterPopulator is a stateless, single-responsibility component within the server's world generation pipeline. Its exclusive function is to add bodies of water, such as oceans, lakes, and rivers, to a newly generated chunk *after* the primary terrain has been sculpted.

It operates as a "populator", a common pattern in procedural generation where distinct features are layered onto a base landscape in sequential passes. This class reads biome-specific water definitions and modifies the chunk data contained within a `ChunkGeneratorExecution` context. It decides where to place water by evaluating block placement priorities, ensuring that water does not overwrite critical, high-priority features generated in earlier stages.

This component is designed to be deterministic; for a given seed and world-generation input, it will always produce the identical water placement.

### Lifecycle & Ownership
- **Creation:** As a static utility class, WaterPopulator is never instantiated. The Java Virtual Machine loads the class definition into memory at runtime.
- **Scope:** Application-level. Its static methods are available throughout the server's lifetime.
- **Destruction:** The class is unloaded only when the application's class loader is garbage collected, which for practical purposes is at server shutdown.

## Internal State & Concurrency
- **State:** The WaterPopulator is entirely stateless. It contains no member variables or static fields, and all required data is passed as arguments to its methods. Its behavior is purely functional, transforming input data without retaining any information between calls.

- **Thread Safety:** The class itself is inherently thread-safe due to its stateless nature. However, the caller is responsible for ensuring the thread safety of the objects passed to it.

    **WARNING:** The `ChunkGeneratorExecution` object passed to the `populate` method is stateful and is **not** guaranteed to be thread-safe. Concurrent calls to `populate` using the *same* `ChunkGeneratorExecution` instance will result in race conditions and catastrophic world corruption. The world generation engine must enforce a concurrency model where each worker thread operates on a unique `ChunkGeneratorExecution` instance.

## API Surface
The public contract consists of a single static method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| populate(seed, execution) | void | O(N) | Iterates through a 32x32 chunk column, placing water blocks and fluids based on biome rules. N is the number of blocks within the chunk. Modifies the provided `ChunkGeneratorExecution` in-place. |

## Integration Patterns

### Standard Usage
The WaterPopulator is intended to be invoked by a higher-level world generation orchestrator. It should be called as one of the final steps in the chunk generation sequence, after terrain shaping and priority map generation but before entity or structure population.

```java
// Within a world generation orchestrator
// 'execution' is a context object for the current chunk being generated
int worldSeed = ...;
ChunkGeneratorExecution execution = ...;

// After base terrain is generated, apply water
WaterPopulator.populate(worldSeed, execution);

// Proceed to the next populator (e.g., TreePopulator)
```

### Anti-Patterns (Do NOT do this)
- **Incorrect Ordering:** Do not call `populate` before the biome map and priority chunk have been fully generated and set within the `ChunkGeneratorExecution`. Doing so will lead to water being placed incorrectly or not at all.
- **Shared Execution Context:** Never share a single `ChunkGeneratorExecution` instance across multiple threads that are simultaneously calling this or any other populator. This will lead to severe data corruption.
- **Direct Instantiation:** The class has no public constructor and contains only static methods. Do not attempt to create an instance of `WaterPopulator`.

## Data Pipeline
The WaterPopulator acts as a transformation stage within the larger chunk generation data flow. It consumes a chunk context object, reads specific data from it, and writes modifications back into the same object.

> Flow:
> Terrain Heightmap -> Biome Assignment -> Priority Map Generation -> **WaterPopulator** -> (Modified Chunk Data) -> TreePopulator -> OrePopulator -> Final Chunk Data

