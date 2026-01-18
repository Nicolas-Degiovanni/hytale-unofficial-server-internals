---
description: Architectural reference for CustomBiome
---

# CustomBiome

**Package:** com.hypixel.hytale.server.worldgen.biome
**Type:** Model / Data Object

## Definition
```java
// Signature
public class CustomBiome extends Biome {
```

## Architecture & Concepts
The CustomBiome class is a specialized data model that represents a single, fully-defined biome within the server's world generation system. It extends the base Biome class, inheriting fundamental biome properties like terrain layers, cover, and environmental tinting.

Its primary architectural role is to act as a configuration container. It encapsulates all the procedural rules and asset references required for the world generator to construct a specific type of terrain. The key distinction of CustomBiome is its direct association with a CustomBiomeGenerator, allowing for more complex and unique procedural logic compared to standard biomes.

This class is a cornerstone of Hytale's data-driven world generation. Instead of hard-coding biome logic, the engine instantiates these objects from external definitions (e.g., JSON files), allowing designers and modders to create new biomes without altering core engine code.

## Lifecycle & Ownership
- **Creation:** CustomBiome instances are not created directly by game logic. They are instantiated by a biome definition parser during the server's initial boot sequence. This system reads biome configuration files from game assets and constructs the corresponding CustomBiome objects.
- **Scope:** An instance of CustomBiome persists for the entire server session. Once loaded, it is stored in a central BiomeRegistry and is treated as a read-only singleton for that specific biome type.
- **Destruction:** The object is eligible for garbage collection only when the server shuts down and the BiomeRegistry is cleared.

## Internal State & Concurrency
- **State:** The CustomBiome object is **effectively immutable**. Its state, including the reference to its CustomBiomeGenerator and all inherited properties, is set exclusively at construction time. There are no public methods to alter its internal state post-instantiation.
- **Thread Safety:** This class is **inherently thread-safe**. Its immutable nature is critical for performance, as it allows multiple world generation threads to safely access its configuration data concurrently without requiring any locks or synchronization primitives.

## API Surface
The public contract is minimal, primarily exposing the specialized generator. Most functionality is inherited from the parent Biome class.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getCustomBiomeGenerator() | CustomBiomeGenerator | O(1) | Returns the procedural generator instance associated with this biome. |

## Integration Patterns

### Standard Usage
Developers should never instantiate this class directly. It must be retrieved from the central biome registry, typically by its unique identifier. The world generation pipeline then consumes this object to access its properties.

```java
// Correctly retrieve a biome configuration during world generation
BiomeRegistry registry = serverContext.getBiomeRegistry();
CustomBiome forestBiome = (CustomBiome) registry.getBiomeByName("hytale:emerald_grove");

// Use its properties to generate a chunk column
CustomBiomeGenerator generator = forestBiome.getCustomBiomeGenerator();
generator.applyToChunk(chunk, seed);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new CustomBiome(...)`. This bypasses the central registry, leading to an unmanaged and unrecognized biome that will cause lookup failures and unpredictable world generation behavior. All biomes must be loaded and registered at server startup.
- **State Assumption:** Do not assume the contents of a CustomBiome. Always query its properties. While the object is immutable, its configuration is loaded from external files that can be changed by server operators or modders.

## Data Pipeline
The CustomBiome class does not process data itself; it is the *source* of configuration data for the world generation pipeline.

> Flow:
> World Generator (requests biome for coordinates) -> Biome Registry (provides instance) -> **CustomBiome** -> Generator reads properties (Layers, Prefabs, Generator Logic) -> Final Chunk Data

