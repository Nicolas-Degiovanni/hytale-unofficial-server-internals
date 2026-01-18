---
description: Architectural reference for BaseHeightDensity
---

# BaseHeightDensity

**Package:** com.hypixel.hytale.builtin.hytalegenerator.density.nodes
**Type:** Transient Component

## Definition
```java
// Signature
public class BaseHeightDensity extends Density {
```

## Architecture & Concepts
BaseHeightDensity is a foundational component within Hytale's procedural world generation framework. It operates as a *Density Function Node* in a larger computational graph responsible for defining the shape of terrain.

Its primary role is to translate a 2D heightmap function into a 3D density field. In procedural generation, a density field is a function that returns a value for any point in 3D space; this value typically represents whether the point is inside solid material (e.g., positive value) or in open air (e.g., negative value). The surface of the terrain exists where the density value is zero.

This class acts as an adapter. It takes a two-dimensional function, a `BiDouble2DoubleFunction` that defines surface elevation based on X and Z coordinates, and uses it to calculate a density value at a three-dimensional (X, Y, Z) point. It can operate in two modes:

1.  **Absolute Height Mode:** The node returns the raw height value calculated by the underlying height function. This is less common for final terrain generation but can be useful for sampling heightmaps or for intermediate calculations.
2.  **Distance Mode:** The node calculates the vertical distance from the given Y coordinate to the terrain surface height (`y - height`). This is the standard mode for generating terrain geometry, as it directly produces the signed distance field required by algorithms like Marching Cubes to construct a mesh.

## Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level generator system, such as a `ZoneGraphBuilder` or a world preset loader. It is configured at creation time with a specific height function (e.g., a Perlin noise function) and a boolean flag to set its mode. It is a building block, not a standalone service.
-   **Scope:** The object's lifetime is tied to the generator graph it belongs to. It persists as long as that specific world generation configuration is active.
-   **Destruction:** The object is eligible for garbage collection when the parent generator graph is discarded, for example, when a server shuts down or a different world preset is loaded.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal fields `heightFunction` and `isDistance` are private, final, and set exclusively by the constructor. The class holds no mutable state and performs no caching.
-   **Thread Safety:** **Thread-safe**. The `process` method is a pure function of its inputs and its immutable configuration. The same instance can be safely and concurrently invoked by multiple world-generation worker threads without locks or race conditions.

    **WARNING:** The thread safety of this class is contingent on the provided `BiDouble2DoubleFunction` also being thread-safe. Injecting a stateful or non-thread-safe function will break this guarantee and lead to severe generation artifacts.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Density.Context) | double | O(F) | Core method. Calculates the density at the context's position. Complexity is dependent on the injected height function, denoted as O(F). |
| skipInputs(double) | boolean | O(1) | Optimization hook. Always returns true, indicating to the graph evaluator that its inputs are not needed for its own computation. |

## Integration Patterns

### Standard Usage
This class is not intended to be used directly. It is designed to be composed into a larger `Density` graph that is then processed by the generation engine.

```java
// A conceptual builder for a world generation graph
// NOTE: This is a hypothetical example for illustration.

// 1. Define a height function, e.g., using a noise generator
BiDouble2DoubleFunction perlinHeight = (x, z) -> Noise.perlin(x * 0.01, z * 0.01) * 50 + 64;

// 2. Create the node in distance mode for terrain generation
Density terrainSurface = new BaseHeightDensity(perlinHeight, true);

// 3. Add the node to a larger graph for further processing
//    (e.g., adding caves by subtracting another density field)
Density finalTerrain = DensityGraph.subtract(terrainSurface, caveSystemDensity);

// 4. The engine evaluates the final graph
double densityValue = finalTerrain.process(new Density.Context(100, 70, 250));
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Functions:** Do not inject a `BiDouble2DoubleFunction` that relies on or modifies external state. This will produce non-deterministic and incorrect results when the generator runs in a multithreaded environment.
-   **Manual Processing:** Avoid calling the `process` method in a loop for a large volume of points. The world generation engine is optimized to traverse and evaluate the entire density graph efficiently; manual iteration bypasses these optimizations.

## Data Pipeline
The data flow for this component is simple and linear, acting as a single step in a much larger generation pipeline.

> Flow:
> World Generator Request (for point x,y,z) -> Density.Context created -> **BaseHeightDensity.process(context)** -> `heightFunction` is called with (x,z) -> Result is used to compute `y - height` -> Density (double) is returned -> World Generator uses value to determine material.

