---
description: Architectural reference for Biome
---

# Biome

**Package:** com.hypixel.hytale.server.worldgen.biome
**Type:** Configuration Data Model

## Definition
```java
// Signature
public abstract class Biome {
```

## Architecture & Concepts
The **Biome** class is a foundational, data-centric component of the server-side world generation system. It does not contain active logic; instead, it serves as an immutable configuration blueprint that defines the rules and properties for a specific geographical region within the game world.

Architecturally, it functions as a high-level container for a collection of more specialized configuration objects, known as "Containers" (e.g., **LayerContainer**, **CoverContainer**, **PrefabContainer**). Each container governs a distinct aspect of the generation process for that biome, such as the strata of rock, the surface-level blocks, or the structures that can spawn.

The world generation pipeline consumes **Biome** objects to make procedural decisions. A **BiomeSelector** first determines which **Biome** applies to a given world coordinate, and then subsequent generation stages query that **Biome** instance to retrieve the specific rules they need to execute their task. This design decouples the generation algorithms from the biome-specific data, allowing for high modularity and data-driven world design.

### Lifecycle & Ownership
- **Creation:** Concrete subclasses of **Biome** are not instantiated directly in game logic. They are defined as assets (likely in JSON or a similar format) and are deserialized and loaded into a central **BiomeRegistry** during server initialization.
- **Scope:** A **Biome** object is a stateless, flyweight-style object. Once loaded, it persists for the entire server session and is shared across all world generation threads.
- **Destruction:** The object is garbage collected when the **BiomeRegistry** is cleared, typically during server shutdown.

## Internal State & Concurrency
- **State:** The **Biome** class is strictly immutable. All of its fields are declared as final and are populated exclusively through its constructor. It holds no mutable state and performs no caching.
- **Thread Safety:** This class is inherently thread-safe due to its immutability. It can be safely read by any number of concurrent world generation threads without requiring locks or other synchronization primitives. This is a critical design feature for a high-performance, multi-threaded world generator.

## API Surface
The public API consists entirely of simple accessors for retrieving the configuration data and containers associated with the biome.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getId() | int | O(1) | Returns the unique numerical identifier for this biome. |
| getName() | String | O(1) | Returns the unique string identifier for this biome. |
| getLayerContainer() | LayerContainer | O(1) | Retrieves the configuration for block layers (e.g., stone, dirt). |
| getCoverContainer() | CoverContainer | O(1) | Retrieves the configuration for surface-level blocks (e.g., grass, sand). |
| getPrefabContainer() | PrefabContainer | O(1) | Retrieves the rules for spawning structures and large objects. May be null. |
| getHeightmapNoise() | NoiseProperty | O(1) | Returns the noise function parameters used to generate the biome's terrain height. |
| getHeightmapInterpreter() | IHeightThresholdInterpreter | O(1) | Returns the interpreter that translates heightmap values into block placement rules. |

## Integration Patterns

### Standard Usage
A **Biome** object should never be instantiated directly. It must be retrieved from a central registry, which is responsible for its lifecycle. Generation systems then query the object for the specific configuration they require.

```java
// Example from a hypothetical WorldGenerator
BiomeRegistry registry = serverContext.getBiomeRegistry();
Biome targetBiome = registry.getBiomeById(biomeId);

// A LayerGenerator system consumes the LayerContainer
LayerContainer layers = targetBiome.getLayerContainer();
layerGenerator.apply(chunk, layers);

// A CoverGenerator system consumes the CoverContainer
CoverContainer cover = targetBiome.getCoverContainer();
coverGenerator.apply(chunk, cover);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call a **Biome** subclass constructor directly (e.g., `new ForestBiome(...)`). This bypasses the registry, breaks asset management, and will lead to inconsistent world generation.
- **Stateful Subclassing:** Do not create subclasses of **Biome** that introduce mutable state. The entire world generation pipeline relies on the assumption that these objects are immutable and thread-safe.
- **Null Container Handling:** Systems consuming containers like **PrefabContainer** must be robust to null return values, as not all biomes will define all possible generation features.

## Data Pipeline
The **Biome** object is not part of a data processing flow itself. Instead, it acts as a static data source that informs multiple stages of the chunk generation pipeline.

> Flow:
> BiomeSelector (using noise maps) → Determines **Biome** ID → **Biome** object retrieved from Registry → Generation Stages (Layer, Cover, Prefab) read configuration from **Biome** → Chunk Voxel Data is produced

