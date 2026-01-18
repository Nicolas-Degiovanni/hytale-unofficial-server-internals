---
description: Architectural reference for SimpleBiomeType
---

# SimpleBiomeType

**Package:** com.hypixel.hytale.builtin.hytalegenerator.biome
**Type:** Transient Data Model

## Definition
```java
// Signature
public class SimpleBiomeType implements BiomeType {
```

## Architecture & Concepts
The SimpleBiomeType class is a concrete implementation of the BiomeType interface. It functions as a foundational data container that aggregates all the defining characteristics of a specific biome within the world generation system. It does not perform any generation logic itself; rather, it serves as a descriptive blueprint or configuration object.

This class encapsulates the core components that dictate a biome's appearance and structure:
*   **Terrain Shape:** Defined by a Density object, which controls the elevation and form of the landscape.
*   **Material Composition:** Governed by a MaterialProvider, which determines the types of blocks (e.g., grass, stone, sand) that constitute the terrain.
*   **Decorative Elements:** Managed through a list of PropFields, which specify the placement of environmental features like trees, rocks, and foliage.
*   **Atmospherics:** Controlled by an EnvironmentProvider and a TintProvider, which influence visual aspects like sky color, water color, and lighting.

Higher-level systems, such as a BiomeSelector or ChunkGenerator, query this object to retrieve the necessary rules for constructing a specific region of the world. It is the canonical source of truth for "what" a biome is, which is then interpreted by other systems to determine "how" it is built.

## Lifecycle & Ownership
-   **Creation:** SimpleBiomeType instances are typically created during the server's initialization or world-loading phase. A central factory or registry, responsible for parsing biome definition files (e.g., JSON assets), instantiates this class. The object is configured immediately after construction, primarily through calls to the addPropFieldTo method.
-   **Scope:** An instance of SimpleBiomeType persists for the entire lifetime of the world generator's configuration. Once fully configured during the setup phase, it is treated as a read-only data source by the rest of the engine.
-   **Destruction:** The object is eligible for garbage collection only when the server shuts down or the world it belongs to is unloaded, and all references from the world generation configuration are released.

## Internal State & Concurrency
-   **State:** The object is **conditionally mutable**. The majority of its fields are final and set in the constructor. However, the internal list of PropFields is mutable and is populated via the addPropFieldTo method. This design implies a two-phase lifecycle: a writable configuration phase followed by a read-only operational phase.

-   **Thread Safety:** This class is **not thread-safe**. The addPropFieldTo method modifies an internal ArrayList without any synchronization. Accessing this object from multiple threads during its configuration phase will result in race conditions and unpredictable state.

    **WARNING:** The design assumes that all mutations occur within a single-threaded context during server initialization. Once the world generator begins its work, the object must be treated as immutable and can be safely read by multiple worker threads.

## API Surface
The public API is designed for configuration and subsequent data retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| addPropFieldTo(propField) | void | O(1) amortized | Adds a prop definition to the biome. This method mutates the internal state and is not thread-safe. |
| getTerrainDensity() | Density | O(1) | Retrieves the density function used to generate the biome's terrain shape. |
| getMaterialProvider() | MaterialProvider | O(1) | Retrieves the provider responsible for selecting block materials for the terrain. |
| getPropFields() | List<PropField> | O(1) | Returns a **direct reference** to the internal list of prop fields. |
| getAllPropDistributions() | List<Assignments> | O(N) | Creates and returns a new list of prop distribution assignments from all contained PropFields. |
| getEnvironmentProvider() | EnvironmentProvider | O(1) | Retrieves the provider for environmental effects like sky color and fog. |
| getTintProvider() | TintProvider | O(1) | Retrieves the provider for biome-specific color tints on blocks and foliage. |

## Integration Patterns

### Standard Usage
The intended pattern involves a central system creating and configuring the SimpleBiomeType instance before it is used by the world generator. This ensures state is stable before concurrent access occurs.

```java
// During a single-threaded initialization phase
SimpleBiomeType forestBiome = new SimpleBiomeType(
    "zone1_forest",
    terrainDensity,
    materialProvider,
    environmentProvider,
    tintProvider
);

// Configure the biome with props
forestBiome.addPropFieldTo(oakTreePropField);
forestBiome.addPropFieldTo(boulderPropField);

// The fully configured object is then registered and used by the generator
biomeRegistry.register(forestBiome);
```

### Anti-Patterns (Do NOT do this)
-   **Post-Initialization Mutation:** Calling addPropFieldTo after the biome has been registered and the world generator has started is a critical error. This will lead to non-deterministic generation and severe concurrency bugs, as worker threads may read the prop list while it is being modified.
-   **External Modification of Returned Collections:** The getPropFields method returns a direct reference to the internal list. Code that retrieves this list and then modifies it (e.g., by adding or removing elements) violates the object's encapsulation and will cause unpredictable behavior across the world generator.
-   **Concurrent Configuration:** Attempting to configure a single SimpleBiomeType instance from multiple threads is not supported and will corrupt its internal state.

## Data Pipeline
SimpleBiomeType acts as a data source within the world generation pipeline. It does not process data itself but provides the configuration that drives downstream processes.

> Flow:
> Biome Definition Asset (JSON) -> Asset Loader -> **SimpleBiomeType (Instantiation & Configuration)** -> BiomeSelector -> ChunkGenerator -> Final World Data

