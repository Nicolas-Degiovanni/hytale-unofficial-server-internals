---
description: Architectural reference for GradientDensity
---

# GradientDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Component

## Definition
```java
// Signature
public class GradientDensity extends Density {
```

## Architecture & Concepts
The GradientDensity class is a specialized node within the procedural world generation framework. It functions as a higher-order density operator, transforming an incoming density field into a new field that represents its local gradient, or steepness.

Its primary role is to analyze the rate of change of an input Density function. By sampling the input at a given point and several nearby points along the cardinal axes, it computes a vector representing the direction of the steepest ascent. It then calculates the angle between this gradient vector and a pre-configured reference axis. The final output is this angle, scaled to a range of 0 to 90.

This component is critical for creating sophisticated, feature-aware terrain. For example, it can be used to:
*   Restrict tree or structure placement to flat ground (low gradient).
*   Generate scree or cliff-face textures on steep slopes (high gradient).
*   Align geological strata or erosion features with the terrain's contour.

It is a fundamental building block in a composable density graph, allowing world designers to make decisions based not just on a value (like elevation or temperature) but on the *rate of change* of that value.

### Lifecycle & Ownership
- **Creation:** GradientDensity nodes are not intended for direct, ad-hoc instantiation. They are constructed by a higher-level density graph builder or parser, typically when loading world generation presets from configuration files. The constructor wires its dependencies, such as the input Density source and its operational parameters.
- **Scope:** The lifetime of a GradientDensity instance is tied to the density graph to which it belongs. It persists as long as the graph is held in memory for generating a world region and is discarded afterward.
- **Destruction:** Management is delegated to the Java Garbage Collector. There are no native resources or explicit cleanup methods.

## Internal State & Concurrency
- **State:** The internal state consists of its configuration: the input Density node, the slopeRange, and the comparison axis. This state is established at creation and is considered effectively immutable during processing. The `setInputs` method allows for late-binding but is not intended for use during an active generation process.
- **Thread Safety:** This class is **conditionally thread-safe**. The `process` method is re-entrant and does not mutate the object's own state. It operates on a context object, creating a new, temporary context for its internal sampling calls. However, its overall thread safety is entirely dependent on the thread safety of the `input` Density it invokes. If the input node is thread-safe, then GradientDensity can be safely processed by multiple worker threads simultaneously.

**WARNING:** Modifying the input node via `setInputs` while the density graph is being concurrently processed will lead to severe and unpredictable race conditions. This method should only be called during the graph's initial assembly phase.

## API Surface
The public contract is minimal, focusing exclusively on graph processing and configuration.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context context) | double | O(C_input) | Computes the local gradient angle of the input density field. Complexity is dependent on the input node, which is processed four times. |
| setInputs(Density[] inputs) | void | O(1) | Sets the upstream input node for this density function. Expects an array with at least one element. |

## Integration Patterns

### Standard Usage
GradientDensity is used as an intermediate node in a density graph. It takes the output of one node (e.g., a Perlin noise function) and feeds its own output (the slope) into another node that makes a decision based on it.

```java
// Assume a graph builder context
// 1. Create a base noise function for terrain elevation
Density elevation = new PerlinNoiseDensity(/*...params...*/);

// 2. Create the GradientDensity to analyze the elevation's slope
//    It will measure the slope against the world's vertical axis (0,1,0)
Vector3d verticalAxis = new Vector3d(0.0, 1.0, 0.0);
GradientDensity slopeAnalyzer = new GradientDensity(elevation, 2.0, verticalAxis);

// 3. Use the output of the slope analyzer in another function
//    For example, a selector that chooses rock for steep slopes.
Density terrainMaterial = new SelectorDensity(slopeAnalyzer, /*...params...*/);

// The final value is determined by processing the end of the chain
double materialType = terrainMaterial.process(context);
```

### Anti-Patterns (Do NOT do this)
- **Invalid Configuration:** Do not construct with a `slopeRange` that is zero or negative. The constructor will throw an `IllegalArgumentException`, but bypassing this check would lead to undefined mathematical behavior.
- **Dynamic Re-Configuration:** Do not call `setInputs` on a GradientDensity node after the world generation process has begun. Density graphs are assumed to be structurally immutable during processing to ensure thread safety and deterministic output.
- **Ignoring The Input:** If `setInputs` is called with an empty array, the `input` field becomes null. Subsequent calls to `process` will return a constant 0.0, silently producing flat terrain and masking configuration errors.

## Data Pipeline
The flow of data through this component involves multiple samples of its input to derive a single output value. It acts as a spatial derivative operator within the larger density pipeline.

> Flow:
> Upstream Density Field (e.g., Elevation) -> **GradientDensity** (4x Sampling) -> Gradient Vector Calculation -> Angle Comparison -> Downstream Density Field (e.g., Slope in Degrees)

