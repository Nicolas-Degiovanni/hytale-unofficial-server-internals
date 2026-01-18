---
description: Architectural reference for TerrainDensity
---

# TerrainDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class TerrainDensity extends Density {
```

## Architecture & Concepts
The TerrainDensity class is a fundamental node within Hytale's procedural world generation framework. It operates as a terminal or "leaf" node in a density function graph. Its primary role is not to compute a value algorithmically (like a noise function), but to act as a bridge, fetching a pre-computed or externally defined base terrain value.

Within the world generator, terrain is often built in layers. A foundational, low-frequency density map might be generated first. Subsequent, more detailed density graphs then sample from this base map to add features like caves, overhangs, and fine details. TerrainDensity is the node used within these detail-oriented graphs to access the value from that foundational map.

This design effectively decouples the high-level terrain shape from the fine-grained details, allowing different systems to operate on the terrain data at different stages of the generation pipeline. It serves as a critical injection point for foundational terrain data into more complex density calculations.

### Lifecycle & Ownership
- **Creation:** Instances are typically created during the initialization of a world generator. They are defined as part of a static density graph, often deserialized from a world generation preset file (e.g., a JSON or HOCON configuration).
- **Scope:** The object's lifetime is tied to the world generator configuration it is a part of. It is stateless and can be considered a singleton within the scope of a specific generator graph definition.
- **Destruction:** The object is garbage collected when the parent world generator is discarded, for example, upon server shutdown or when changing worlds.

## Internal State & Concurrency
- **State:** **Immutable and Stateless**. This class holds no internal state. All information required for its operation is provided via the `Density.Context` object passed to the `process` method.
- **Thread Safety:** **Inherently Thread-Safe**. As a stateless object, a single TerrainDensity instance can be safely and concurrently invoked by multiple world generation worker threads without synchronization. Any and all concurrency concerns are deferred to the implementation of the `terrainDensityProvider` supplied in the context.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(provider) | Fetches the base terrain density value from the provider within the context. The performance is entirely dependent on the provider's implementation. Returns 0.0 if no provider is configured. |

## Integration Patterns

### Standard Usage
A developer or designer typically does not interact with this class directly in Java code. Instead, it is declared within a data-driven density graph configuration. The world generation engine is responsible for instantiating and processing the node.

The primary integration point is ensuring that the `Density.Context` passed by the engine is correctly populated with a valid `terrainDensityProvider`.

```java
// The following is a conceptual example of how the engine uses this class.
// This code is not intended for direct use by developers.

// 1. Engine creates a context for a specific world position and worker.
Density.Context context = buildContextForPosition(x, y, z, workerId);
context.terrainDensityProvider = myGlobalTerrainProvider;

// 2. The node is retrieved from the pre-compiled density graph.
Density densityNode = generatorGraph.getNode("base_terrain_sampler"); // This would be a TerrainDensity instance

// 3. The engine invokes the node.
double baseDensity = densityNode.process(context);
```

### Anti-Patterns (Do NOT do this)
- **Missing Provider:** The most common failure mode is executing a density graph containing this node without setting the `terrainDensityProvider` on the `Density.Context`. The node will not throw an error but will silently return 0.0. This can lead to large, unexpected sections of the world being generated as empty space or solid material, depending on how the value is used.
- **Direct Instantiation:** Do not instantiate this class manually for one-off calculations. It is designed to be a component within a larger, engine-managed system.

## Data Pipeline
The TerrainDensity node acts as a data source or input tap within the larger density calculation pipeline. It injects an external value into the graph's execution flow.

> Flow:
> World Generator Request (at position) -> Density Graph Traversal -> **TerrainDensity.process(context)** -> Accesses `context.terrainDensityProvider` -> Provider returns `double` -> Value is passed to parent nodes in the graph

