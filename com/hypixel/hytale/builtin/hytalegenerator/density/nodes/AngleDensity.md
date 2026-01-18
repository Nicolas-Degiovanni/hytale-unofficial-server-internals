---
description: Architectural reference for AngleDensity
---

# AngleDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient

## Definition
```java
// Signature
public class AngleDensity extends Density {
```

## Architecture & Concepts
The AngleDensity class is a specialized node within the procedural world generation framework. It functions as a mathematical operator that produces a density value based on the geometric angle between two vectors. This component is fundamental for creating terrain features that are dependent on surface orientation or slope.

Architecturally, AngleDensity is a "generator" or "leaf" node in a directed acyclic graph of Density operations. It does not consume the output of other Density nodes (as evidenced by its empty setInputs method). Instead, it generates a value based on its internal configuration and a dynamic vector supplied by a VectorProvider at runtime.

Common use cases include:
- **Slope Analysis:** By comparing a surface normal vector (from a VectorProvider) to a fixed "up" vector (e.g., 0,1,0), this node can determine the steepness of terrain. This output can then be used by other nodes to prevent trees from growing on cliffs or to place snow only on flatter surfaces.
- **Directional Masking:** Creating effects that are biased towards a certain direction, such as wind-swept snow dunes or rock striations.

The output is not a direct angle in degrees or radians, but a normalized value scaled by 180. This transforms the angular relationship into a standardized density value suitable for consumption by other parts of the generation pipeline.

### Lifecycle & Ownership
- **Creation:** AngleDensity instances are not intended for direct manual instantiation in game logic. They are constructed by a higher-level world generator configuration loader, which parses a preset (e.g., a JSON definition) and assembles the complete Density graph.
- **Scope:** The object's lifetime is bound to the world generator instance it is a part of. It is created once during the generator's initialization and persists until the generator is discarded.
- **Destruction:** The object is managed by the Java Garbage Collector. It is eligible for collection once the world generator configuration that owns it is no longer referenced.

## Internal State & Concurrency
- **State:** The class holds configuration state (vectorProvider, vector, toAxis) that is set at construction and is not modified thereafter. The input vector is defensively cloned, ensuring the internal state is isolated from external mutation. This design makes the object *effectively immutable* after its creation.
- **Thread Safety:** The process method is fully re-entrant and does not modify any instance fields. Consequently, a single AngleDensity instance is thread-safe and can be processed by multiple world generation threads concurrently without locks or synchronization. This is a critical feature for achieving high-performance, parallelized world generation. The overall thread safety, however, depends on the injected VectorProvider also being thread-safe.

## API Surface
The public contract is minimal, focusing exclusively on the processing of density values.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context) | double | O(k) | Calculates density based on the angle between its configured vector and one from the provider. Complexity is O(k) where k is the complexity of the injected VectorProvider. |
| setInputs(Density[]) | void | O(1) | No-op. This node does not accept inputs from other Density nodes, reinforcing its role as a generator node. |

## Integration Patterns

### Standard Usage
AngleDensity is designed to be a component within a larger Density graph. It is configured once and then its process method is invoked repeatedly by the generation engine for different points in space.

```java
// A hypothetical generator using an AngleDensity node
// Note: This is a conceptual example. Direct instantiation is an anti-pattern.

// 1. Configuration (Done by a worldgen loader)
VectorProvider surfaceNormalProvider = new SurfaceNormalProvider(...);
Vector3d upVector = new Vector3d(0, 1, 0);
AngleDensity slopeNode = new AngleDensity(surfaceNormalProvider, upVector, true);

// 2. Execution (Inside the worldgen engine for a specific block)
Density.Context generationContext = new Density.Context(x, y, z);
double slopeDensity = slopeNode.process(generationContext);

// Use the density to influence material placement
if (slopeDensity > 160) { // 160 corresponds to a very flat surface
    setBlock(x, y, z, GRASS);
} else {
    setBlock(x, y, z, STONE);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid `new AngleDensity()` in gameplay code. These nodes should only be defined as part of a static world generation configuration and managed by the generator's lifecycle.
- **State Mutation:** Do not attempt to modify the internal state of an AngleDensity instance after it has been constructed. The system relies on these nodes being effectively immutable for thread safety.
- **Misinterpreting Output:** The return value of process is not an angle in degrees. It is a normalized value (0.0 to 1.0, representing 0 to PI radians) scaled by 180.0. Using this value directly in trigonometric functions will produce incorrect results.

## Data Pipeline
The flow of data for a single process call is linear and self-contained. The node transforms positional context into a final density value.

> Flow:
> Density.Context -> VectorProvider.process -> Dynamic Vector3d -> **AngleDensity** (Angle Calculation & Normalization) -> Final double (Density Value)

