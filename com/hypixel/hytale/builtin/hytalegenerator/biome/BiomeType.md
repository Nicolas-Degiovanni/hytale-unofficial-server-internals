---
description: Architectural reference for the BiomeType interface, the core contract for world generation biomes.
---

# BiomeType

**Package:** com.hypixel.hytale.builtin.hytalegenerator.biome
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface BiomeType extends MaterialSource, PropsSource, EnvironmentSource, TintSource {
```

## Architecture & Concepts
The BiomeType interface is a foundational contract within the Hytale world generation system. It does not represent a specific instance of a biome in the game world, but rather the immutable, static definition of a biome's characteristics. It serves as the central point of aggregation for all properties that define a biome, from its physical shape to its aesthetic and content.

By extending multiple source interfaces—MaterialSource, PropsSource, EnvironmentSource, and TintSource—the BiomeType contract mandates that any implementing class must provide a complete recipe for generating a slice of the world. It acts as a data source and rule set for downstream systems in the procedural generation pipeline, such as the Terrain Generator, Decorator, and Block Placer.

The most critical responsibility defined directly on this interface is providing the terrain density function, which is the mathematical basis for the biome's landscape shape.

### Lifecycle & Ownership
As an interface, BiomeType itself has no lifecycle. The following applies to its concrete implementations.

-   **Creation:** Implementations are expected to be instantiated once at application startup by a central registry system, likely a BiomeRegistry. They are loaded from configuration or code as part of the initial asset loading phase.
-   **Scope:** Application-scoped. A biome definition persists for the entire lifetime of the game client or server. It is a read-only template.
-   **Destruction:** Implementations are garbage collected upon final application shutdown when the central biome registry is cleared.

## Internal State & Concurrency
-   **State:** The BiomeType contract implies immutability. Implementations must be treated as read-only data structures. They hold definitional data (e.g., which blocks to use, what the terrain shape is) that must not change at runtime.
-   **Thread Safety:** Implementations of this interface **must be thread-safe**. The world generator operates across multiple threads, and various workers will query the same BiomeType instance concurrently to generate different parts of the world. All methods must be safe to call from any thread without external locking.

## API Surface
The API surface is composed of methods defined directly on the interface and those inherited from its parents. The direct methods are fundamental to a biome's identity and physical form.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBiomeName() | String | O(1) | Returns the unique, human-readable identifier for the biome, such as *forest* or *desert*. |
| getTerrainDensity() | Density | O(1) | Returns the Density function object used to calculate the terrain's shape. **Warning:** While this method is O(1), the returned Density object's evaluation can be computationally expensive. |

## Integration Patterns

### Standard Usage
The BiomeType is not used directly by most game logic. It is a core component consumed by the world generation engine. A generator will typically retrieve a specific implementation from a registry and use it to inform its algorithms.

```java
// A world generator retrieves a biome definition from a central registry
BiomeType forestBiome = biomeRegistry.getBiome("hytale:zone1_forest");

// It then uses the definition to drive generation logic
Density terrainShape = forestBiome.getTerrainDensity();
int blockId = forestBiome.getMaterial(worldPos, noiseValue);
// ... and so on for props, environment colors, etc.
```

### Anti-Patterns (Do NOT do this)
-   **Runtime Modification:** Never attempt to cast a BiomeType to its concrete implementation to modify its state. Biome definitions are assumed to be immutable by the entire world generation pipeline. Modifying them at runtime will lead to unpredictable world generation artifacts and severe concurrency issues.
-   **Incomplete Implementation:** Implementing this interface requires satisfying the contracts of all parent interfaces (MaterialSource, PropsSource, etc.). A partial implementation will cause NullPointerExceptions or other fatal errors deep within the native world generation code.

## Data Pipeline
The BiomeType interface acts as a data source, not a processing stage. It provides the configuration and rules that drive the primary terrain generation pipeline.

> Flow:
> World Generator -> Biome Selector -> **BiomeType** (provides rules) -> Terrain Density Function -> Voxel Buffer -> Block Placer -> World Data

