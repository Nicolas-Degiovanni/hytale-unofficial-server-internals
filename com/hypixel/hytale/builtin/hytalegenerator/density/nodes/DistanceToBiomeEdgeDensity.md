---
description: Architectural reference for DistanceToBiomeEdgeDensity
---

# DistanceToBiomeEdgeDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class DistanceToBiomeEdgeDensity extends Density {
```

## Architecture & Concepts
The DistanceToBiomeEdgeDensity class is a fundamental node within Hytale's procedural world generation framework, specifically operating within the Density Graph system. It does not perform any calculations itself; instead, it serves as a terminal input node that exposes a critical environmental variable to the graph.

Its sole purpose is to retrieve a pre-calculated value—the distance from the current sample point to the nearest biome boundary—from the active Density.Context. This allows other, more complex density functions within the same graph to modulate their output based on proximity to biome transitions. For example, a terrain smoothing function could apply a stronger effect near biome edges, or a resource placement function could prevent certain ores from spawning too close to a biome change.

This class embodies the principle of separating data calculation from data utilization. The complex logic for determining biome boundaries and distances is handled earlier in the world generation pipeline, while this class provides a clean, efficient access point for that data during the density evaluation phase.

### Lifecycle & Ownership
- **Creation:** Instances are created by the world generator's configuration loader when it parses and constructs a Density Graph from a zone file or other preset. It is not intended for manual instantiation during gameplay.
- **Scope:** The object's lifetime is bound to the Density Graph that contains it. It persists as long as the world generator's configuration is active.
- **Destruction:** The instance is eligible for garbage collection when its parent Density Graph is unloaded or replaced, for example, when a player leaves a zone with a unique generation profile.

## Internal State & Concurrency
- **State:** This class is **stateless and immutable**. It contains no member fields and its behavior is entirely dependent on the arguments passed to its process method.
- **Thread Safety:** The class is inherently **thread-safe**. As a stateless object, concurrent calls to the process method from multiple world generation threads pose no risk of data corruption. This is a critical attribute, as the engine heavily parallelizes chunk generation, and the Density Graph is evaluated concurrently for many different world coordinates.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(1) | Returns the pre-calculated distance to the nearest biome edge, sourced directly from the context object. |

## Integration Patterns

### Standard Usage
A developer does not typically interact with this class directly. It is defined within a world generation preset and invoked automatically by the engine's Density Graph Processor. The processor is responsible for populating the context and traversing the graph.

```java
// CONCEPTUAL: The engine's Density Graph Processor executes nodes like this.
// This code is not intended for direct use by game logic developers.

// 1. The processor prepares a context for a world coordinate (x, y, z).
//    The distanceToBiomeEdge field is populated by an earlier pipeline stage.
Density.Context context = worldGen.createContextFor(x, y, z);

// 2. The processor finds an instance of this node in its graph.
DistanceToBiomeEdgeDensity edgeNode = densityGraph.getNodeOfType(DistanceToBiomeEdgeDensity.class);

// 3. The node is processed to retrieve the value.
double distance = edgeNode.process(context);

// 4. This value is then used as input for other connected nodes.
anotherNode.compute(distance);
```

### Anti-Patterns (Do NOT do this)
- **Assuming Calculation:** Do not operate under the assumption that this class *calculates* the distance. It is a simple accessor. The actual calculation is performed by a different system before graph evaluation begins. Relying on it for real-time calculation will lead to incorrect assumptions about performance and data freshness.
- **Direct Instantiation:** Avoid creating instances with *new DistanceToBiomeEdgeDensity()*. These nodes are designed to be part of a managed graph defined in configuration files. Manual creation bypasses the intended data flow and will likely fail to integrate with the world generator.

## Data Pipeline
This class acts as a simple passthrough for data computed earlier in the world generation pipeline. It bridges the gap between the biome placement stage and the density calculation stage.

> Flow:
> World Generator Identifies Sample Coordinate -> Biome Map Sampler -> **Distance Field Calculator** -> Density.Context Population -> Density Graph Processor -> **DistanceToBiomeEdgeDensity.process()** -> Upstream Consumer Node (e.g., Terrain Smoother)

