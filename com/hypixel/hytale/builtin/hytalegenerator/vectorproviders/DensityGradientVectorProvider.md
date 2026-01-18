---
description: Architectural reference for DensityGradientVectorProvider
---

# DensityGradientVectorProvider

**Package:** com.hypixel.hytale.builtin.hytalegenerator.vectorproviders
**Type:** Transient

## Definition
```java
// Signature
public class DensityGradientVectorProvider extends VectorProvider {
```

## Architecture & Concepts
The DensityGradientVectorProvider is a foundational component within the procedural world generation engine. Its primary role is to transform a scalar field, represented by a Density object, into a vector field representing the gradient of that scalar field.

In procedural generation, the gradient indicates the direction and magnitude of the steepest ascent of a density function (such as Perlin noise or a signed distance field). This vector information is critical for calculating surface normals, defining material flows, orienting foliage, and shaping terrain features.

This class implements a numerical approximation of the gradient using the **finite difference method**. For a given input position, it samples the injected Density function at four points: the origin and three nearby points offset along the cardinal axes (X, Y, Z). The differences in density values between the origin and these offset points are used to construct the final gradient vector. It effectively acts as a mathematical operator, converting scalar potential fields into directional vector fields.

### Lifecycle & Ownership
- **Creation:** Instances are created and configured during the initialization of a world generator. They are typically defined within data-driven configuration files (e.g., JSON or HOCON) that describe the entire generation pipeline. An instance is "baked" with a specific Density function and a sampleDistance, making it a specialized, non-reconfigurable component for its lifetime.
- **Scope:** The object's lifetime is bound to the world generator configuration it is part of. It is not a global singleton; many distinct instances can and will exist simultaneously, each configured for a different part of the generation algorithm.
- **Destruction:** The object is marked for garbage collection when its parent world generator configuration is discarded. This typically occurs when the server shuts down or a new world with a different ruleset is loaded.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, consisting of the Density function reference and the sampleDistance value, is established at construction and stored in final fields. The object holds no mutable state across invocations of its process method.
- **Thread Safety:** **Conditionally Thread-Safe**. The thread safety of this class is entirely dependent on the thread safety of the injected Density object. Given that world generation is a highly parallelized task, it is imperative that any Density function passed to this provider is itself thread-safe. The DensityGradientVectorProvider performs no internal locking and introduces no state of its own, delegating all concurrency concerns to its dependencies.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| process(Context context) | Vector3d | O(1) | Computes the approximate gradient vector of the configured density field at the context's position. The true cost is four times the cost of the injected Density function's process method. |

## Integration Patterns

### Standard Usage
This provider should be instantiated once as part of a larger generator definition and reused for all subsequent calculations within that context. It is a building block used to feed vector data into other procedural systems.

```java
// Within a world generator's configuration phase
Density terrainDensityField = new PerlinNoiseDensity(/*...params...*/);
double gradientSampleDistance = 0.25;

// The provider is configured once and stored for later use
VectorProvider terrainNormalProvider = new DensityGradientVectorProvider(
    terrainDensityField,
    gradientSampleDistance
);

// During a chunk generation task, the provider is invoked many times
VectorProvider.Context generationContext = new VectorProvider.Context(blockPosition);
Vector3d surfaceGradient = terrainNormalProvider.process(generationContext);
```

### Anti-Patterns (Do NOT do this)
- **Instantiation in a Loop:** Never create a new DensityGradientVectorProvider inside a tight loop (e.g., for each block). This is extremely inefficient and generates significant garbage collector pressure. Define providers once and reuse them.
- **Stateful Density Injection:** Passing a Density object that is not thread-safe or relies on mutable state will cause unpredictable behavior and race conditions during parallel chunk generation. All dependencies must be stateless or properly synchronized.
- **Zero Sample Distance:** While the constructor prevents negative values, providing a sampleDistance of 0.0 will always produce a zero vector. This yields no useful information and wastes computation.

## Data Pipeline
The primary data flow involves receiving a spatial context and outputting a derived vector. It acts as a transformer in the world generation pipeline.

> Flow:
> World Generator Configuration -> **DensityGradientVectorProvider** (Instance created with a specific Density function) -> Chunk Generation Task -> `process(Context)` -> Four samples of the injected Density function -> `Vector3d` (Gradient) -> Consumer (e.g., Surface Normal Calculator, Biome Placer)

