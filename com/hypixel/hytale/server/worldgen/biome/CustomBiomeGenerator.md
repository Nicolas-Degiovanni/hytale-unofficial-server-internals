---
description: Architectural reference for CustomBiomeGenerator
---

# CustomBiomeGenerator

**Package:** com.hypixel.hytale.server.worldgen.biome
**Type:** Transient

## Definition
```java
// Signature
public class CustomBiomeGenerator {
```

## Architecture & Concepts
The CustomBiomeGenerator is a fundamental component within the server-side procedural world generation pipeline. It does not generate block data itself; rather, it functions as a rule-based decision engine that determines *if* a specific custom biome should be placed at a given world coordinate.

Each instance of CustomBiomeGenerator encapsulates a complete set of conditions for a single biome variant. These conditions typically combine a noise function, a value threshold, and contextual rules about the underlying biome. The world generator evaluates a collection of these objects, ordered by priority, to layer custom biomes on top of the base world terrain. This class is the bridge between raw procedural noise and concrete biome placement logic.

### Lifecycle & Ownership
- **Creation:** Instances are not created manually. They are instantiated by the server's configuration loading system when parsing biome definition files, typically at server startup or during a hot-reload. Each custom biome defined in the game's assets will have one or more corresponding CustomBiomeGenerator objects created in memory.
- **Scope:** An instance persists for the entire server session. It is part of the immutable, in-memory representation of the world's generation rules. It is designed to be reused for every chunk generation request.
- **Destruction:** The object is marked for garbage collection when the server shuts down or the world configuration is unloaded.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields are final and are injected via the constructor. Once an instance is created, its state cannot be modified. This design is critical for predictable and reproducible world generation.
- **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, a single CustomBiomeGenerator instance can be safely shared and executed by multiple, concurrent chunk generation threads without any need for locks or synchronization primitives. This is a key enabler for high-performance, parallel world generation.

## API Surface
The public contract is focused on evaluating placement conditions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| shouldGenerateAt(seed, x, z, zoneResult, customBiome) | boolean | O(1) | The primary evaluation method. Returns true if the configured conditions are met for the given world coordinate. |
| isValidParentBiome(index) | boolean | O(1) | Checks if the biome identified by *index* is a valid base for this custom biome to generate on. |
| getPriority() | int | O(1) | Returns the generator's priority, used by the world generator to resolve conflicts when multiple custom biomes could be placed at the same location. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by most systems. It is invoked exclusively by the core biome placement service during chunk generation. The service retrieves a list of all registered generators and evaluates them for each column in a chunk.

```java
// Simplified conceptual example from a hypothetical BiomePlacementService

for (CustomBiomeGenerator generator : registeredGenerators) {
    if (generator.isValidParentBiome(parentBiomeId)) {
        if (generator.shouldGenerateAt(seed, x, z, zone, biome)) {
            // This generator wins. Place the custom biome.
            // ...
            break; // Assuming generators are pre-sorted by priority
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use **new CustomBiomeGenerator()** in game logic. The world generation system relies on instances loaded from configuration files to ensure consistency. Manual creation will lead to biomes that are not part of the canonical world definition.
- **State Caching:** Do not cache the result of **shouldGenerateAt**. The method is computationally inexpensive, and improper caching could lead to severe visual artifacts or chunk borders if the context (like **zoneResult**) is not part of the cache key.

## Data Pipeline
The CustomBiomeGenerator acts as a filter and decision point in the biome generation data flow. It consumes spatial and contextual data and produces a simple boolean output.

> Flow:
> World Coordinates & Seed -> NoiseProperty -> Raw Noise Value -> **CustomBiomeGenerator** -> Boolean Decision -> Biome Placement Service -> Final Chunk Biome Data

