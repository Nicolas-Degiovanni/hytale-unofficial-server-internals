---
description: Architectural reference for TileBiome
---

# TileBiome

**Package:** com.hypixel.hytale.server.worldgen.biome
**Type:** Value Object

## Definition
```java
// Signature
public class TileBiome extends Biome {
```

## Architecture & Concepts
The **TileBiome** class is an immutable data container that represents a complete biome definition for the server-side world generation system. It extends the base **Biome** class, inheriting a comprehensive set of properties that dictate terrain shape, block composition, environmental effects, and decorative elements like prefabs.

Its primary architectural role is to serve as a static, read-only configuration object. The world generation pipeline consumes instances of **TileBiome** to translate procedural noise values into tangible world features.

The key differentiators introduced by **TileBiome** are the **weight** and **sizeModifier** properties. These fields strongly indicate that this class is designed for a weighted, grid-based biome selection algorithm. During world generation, a biome provider likely evaluates multiple candidate biomes for a given region, and the **weight** property influences the probability of this biome being chosen over others. The **sizeModifier** allows for programmatic control over the typical scale or patch size of the biome, enabling fine-tuned control over the world's landscape composition.

## Lifecycle & Ownership
- **Creation:** **TileBiome** instances are not intended for manual instantiation during the game loop. They are created once during server initialization, typically by a deserialization process that reads biome definitions from configuration files (e.g., JSON assets). A central **BiomeRegistry** service is responsible for loading and owning all **TileBiome** objects.

- **Scope:** An instance of **TileBiome** is a global singleton for its specific biome type (e.g., "forest", "desert"). It persists for the entire lifetime of the server process.

- **Destruction:** The objects are garbage collected upon server shutdown when the **BiomeRegistry** is cleared and the JVM exits. There is no manual destruction logic.

## Internal State & Concurrency
- **State:** The **TileBiome** is strictly **immutable**. All its fields, including those inherited from the parent **Biome** class, are initialized once via the constructor and cannot be changed thereafter. This design is critical for predictable and deterministic world generation.

- **Thread Safety:** This class is inherently **thread-safe**. Its immutability guarantees that multiple world generation threads can read from the same **TileBiome** instance concurrently without any risk of data corruption or race conditions. No synchronization mechanisms, such as locks, are required.

## API Surface
The public API is minimal, exposing only read-only access to its unique properties. This reinforces its role as a pure data container.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWeight() | double | O(1) | Returns the selection weight used by the biome provider. |
| getSizeModifier() | double | O(1) | Returns the scalar modifier for the biome's default size. |

## Integration Patterns

### Standard Usage
Developers should never interact with this class directly. The world generation engine retrieves biome definitions from a central registry and uses their properties to drive its algorithms.

```java
// Hypothetical world generator logic
BiomeRegistry registry = serverContext.getBiomeRegistry();
TileBiome forestBiome = registry.getBiome("hytale:forest", TileBiome.class);

// The generator uses the biome's properties to influence its behavior
double placementChance = calculateChance(baseNoise, forestBiome.getWeight());
int biomeSize = calculateSize(baseSize, forestBiome.getSizeModifier());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call **new TileBiome(...)** in application code. The constructor is complex and its usage is reserved for the initial asset loading and deserialization pipeline. Bypassing the registry will lead to non-functional or inconsistent biomes that are not recognized by the rest of the engine.

- **Stateful Wrappers:** Do not create wrapper classes that attempt to add mutable state to a **TileBiome**. The entire world generation pipeline relies on the immutability of these definitions for thread safety and determinism.

## Data Pipeline
A **TileBiome** does not process data itself; rather, it is a critical source of configuration data that is injected into the world generation pipeline.

> Flow:
> Biome Configuration Asset (JSON) -> Server Bootstrap Deserializer -> **TileBiome Instance** (held in BiomeRegistry) -> World Generator -> Voxel Chunk Data

