---
description: Architectural reference for LayerContainer
---

# LayerContainer

**Package:** com.hypixel.hytale.server.worldgen.container
**Type:** Transient Model / Data Transfer Object

## Definition
```java
// Signature
public class LayerContainer {
    // Inner classes for StaticLayer, DynamicLayer, etc. are defined within.
}
```

## Architecture & Concepts

The LayerContainer is a foundational data structure within the server-side procedural world generation system. It serves as an immutable blueprint that defines the vertical composition of blocks for a specific region or biome. It does not generate blocks itself; rather, it encapsulates the *rules* and *materials* that a higher-level generator, such as a ChunkGenerator, will use to populate a world column.

Architecturally, it implements a **Composition pattern** through its nested classes. A single LayerContainer is composed of two primary collections:
1.  **Static Layers:** These represent strata of blocks with defined vertical boundaries (min and max height), often driven by noise functions. They are used for foundational geology like bedrock, stone, and deepslate.
2.  **Dynamic Layers:** These represent surface-level or feature layers whose vertical position is determined by a noise-driven offset. This is critical for creating terrain with variable height, such as grass, dirt, or sand surfaces.

The system operates on a **priority-based evaluation model**. When determining the topmost block for a coordinate, the `dynamicLayers` array is iterated in order. The first layer that evaluates as active for that coordinate dictates the result, effectively occluding any subsequent layers. This allows for a declarative and powerful way to blend features, where more specific or important features are placed earlier in the collection.

## Lifecycle & Ownership

-   **Creation:** LayerContainer instances are not meant to be manually constructed in gameplay logic. They are typically deserialized from asset files (e.g., biome definitions in JSON) by a loading manager at server startup or during world initialization. This process populates the container with the complex hierarchy of layers, conditions, and noise suppliers that define a biome's structure.
-   **Scope:** The scope of a LayerContainer is transient and context-dependent. A single instance representing a biome's definition may be held by a BiomeRegistry and referenced by many generation tasks concurrently. However, it is fundamentally a stateless data holder. Its lifetime is tied to the biome data it represents, which typically persists for the entire server session.
-   **Destruction:** Instances are managed by the Java garbage collector. As they are simple data objects with no external resources, no explicit cleanup is required.

## Internal State & Concurrency

-   **State:** The LayerContainer is designed to be **effectively immutable**. All of its primary fields are declared as `final`. While the arrays they reference are technically mutable, the class provides no methods to modify them post-construction. This immutability is a critical design choice.
-   **Thread Safety:** The class is **thread-safe**. Its immutable nature allows a single LayerContainer instance to be safely shared and read by multiple world generation threads simultaneously without any need for locks or synchronization. All computational methods, such as `getTopBlockAt`, are pure functions that depend only on their arguments and the object's immutable state.

**WARNING:** Modifying the internal layer arrays via reflection or other unsafe mechanisms will break the concurrency guarantees and lead to unpredictable world generation artifacts. These arrays should be treated as strictly read-only after instantiation.

## API Surface

The public API is minimal, focusing on querying the generation rules for a specific world coordinate.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getTopBlockAt(seed, x, z) | BlockFluidEntry | O(N) | Calculates the topmost block for a given (x, z) column. Iterates through N dynamic layers, returning the result from the first active layer. Returns the default filling block if no dynamic layer is active. |
| getFilling() | BlockFluidEntry | O(1) | Returns the default block type used when no other layer applies. |
| getStaticLayers() | StaticLayer[] | O(1) | Provides access to the static geological layers. |
| getDynamicLayers() | DynamicLayer[] | O(1) | Provides access to the prioritized surface layers. |

## Integration Patterns

### Standard Usage

The LayerContainer is a dependency for chunk or column generation tasks. The generator retrieves the appropriate container for a given biome and uses it to determine block placement.

```java
// Pseudo-code for a world generator
LayerContainer biomeLayers = biomeRegistry.getLayersFor(biomeId);
BlockFluidEntry filling = biomeLayers.getFilling();

// For each (x, z) in a chunk...
for (int x = 0; x < 16; x++) {
    for (int z = 0; z < 16; z++) {
        // 1. Determine the surface block
        BlockFluidEntry topBlock = biomeLayers.getTopBlockAt(worldSeed, worldX + x, worldZ + z);
        
        // 2. Use static layers and other logic to fill blocks below the surface
        // ... generator logic ...
    }
}
```

### Anti-Patterns (Do NOT do this)

-   **Manual Instantiation:** Do not use `new LayerContainer(...)` in procedural logic. These objects are complex and intended to be configured and loaded from data assets. Manual creation is error-prone and bypasses the content pipeline.
-   **State Mutation:** Do not attempt to modify the contents of the arrays returned by `getStaticLayers()` or `getDynamicLayers()`. The entire world generation system relies on these data structures being immutable and thread-safe.
-   **Ignoring Layer Order:** The order of entries in the `dynamicLayers` array is critical. Placing a broadly-applicable layer (like grass) before a more specific feature layer (like a path) will cause the feature to be hidden.

## Data Pipeline

The LayerContainer acts as a structured, in-memory representation of world generation rules that are authored in external data files.

> Flow:
> Biome Definition (JSON Asset) -> Asset Deserializer -> **LayerContainer** Instance -> Biome Registry -> Chunk Generator -> Final Chunk Block Data

