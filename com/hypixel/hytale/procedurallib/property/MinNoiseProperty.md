---
description: Architectural reference for MinNoiseProperty
---

# MinNoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient Component

## Definition
```java
// Signature
public class MinNoiseProperty implements NoiseProperty {
```

## Architecture & Concepts
The MinNoiseProperty class is a compositor component within the procedural generation framework. It implements the standard NoiseProperty interface, allowing it to be seamlessly integrated into any system that consumes noise data, such as terrain generators or texture placement algorithms.

Its core architectural function is to aggregate multiple NoiseProperty sources and return the single minimum value from the collection at any given coordinate. This is a fundamental operation for blending and carving features in world generation. For example, a MinNoiseProperty can be used to combine a mountain noise function with a river valley noise function, ensuring the river correctly carves into the mountain by taking the lower of the two height values at every point.

This class follows the Composite design pattern, acting as a non-terminal node in a tree of NoiseProperty objects. It treats all its child properties uniformly, abstracting the complexity of multiple noise sources behind a single, unified interface.

## Lifecycle & Ownership
- **Creation:** MinNoiseProperty is instantiated directly via its constructor, typically during the setup phase of a world generator. It is configured by passing an array of other NoiseProperty instances that will be used for the minimum value calculation. It is not managed by a dependency injection container or a central service registry.
- **Scope:** The lifetime of a MinNoiseProperty instance is bound to the parent configuration object that created it, such as a biome definition or a specific procedural layer. It persists as long as that configuration is active.
- **Destruction:** The object is managed by the Java garbage collector. There are no native resources or explicit cleanup methods. It is eligible for garbage collection as soon as its parent configuration is no longer referenced.

## Internal State & Concurrency
- **State:** The primary internal state is the `noiseProperties` array, which holds references to the composed noise sources. This field is declared `final`, meaning the reference to the array is immutable after construction. The class does not modify the array or the objects within it during its operation.
- **Thread Safety:** This class is conditionally thread-safe. It contains no internal locks and its own state is immutable post-construction. Its thread safety is therefore entirely dependent on the thread safety of the `NoiseProperty` objects provided in its constructor. If all composed `NoiseProperty` instances are themselves thread-safe, then MinNoiseProperty can be safely invoked from multiple threads, which is a critical requirement for parallelized world generation.

**WARNING:** The caller is responsible for ensuring that all `NoiseProperty` objects passed to the constructor are thread-safe. Failure to do so will result in unpredictable behavior and race conditions in a multi-threaded environment.

## API Surface
The public API is minimal, focusing exclusively on the contract defined by the NoiseProperty interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(N) | Returns the minimum noise value from N sources at a 2D coordinate. N is the number of properties in the internal array. |
| get(seed, x, y, z) | double | O(N) | Returns the minimum noise value from N sources at a 3D coordinate. N is the number of properties in the internal array. |
| getNoiseProperties() | NoiseProperty[] | O(1) | Provides read-only access to the underlying array of composed noise properties. |

## Integration Patterns

### Standard Usage
The class is intended to be used as a building block for creating complex procedural generation logic by combining simpler noise sources.

```java
// Assume mountainNoise and valleyNoise are pre-configured NoiseProperty objects
NoiseProperty[] terrainFeatures = { mountainNoise, valleyNoise };

// Create the compositor to ensure valleys carve into mountains
NoiseProperty finalTerrain = new MinNoiseProperty(terrainFeatures);

// The world generator can now use finalTerrain as a single source for height data
double height = finalTerrain.get(worldSeed, 150.7, -432.1);
```

### Anti-Patterns (Do NOT do this)
- **Empty or Null Array:** Constructing a MinNoiseProperty with a null or empty array will result in a runtime exception (either NullPointerException or ArrayIndexOutOfBoundsException) when any `get` method is called. The constructor should always be provided with at least one NoiseProperty.
- **Single-Element Array:** While technically functional, creating a MinNoiseProperty with a single noise source is redundant and adds a minor performance overhead. In such cases, the original NoiseProperty should be used directly.
- **Unsafe Publication:** Do not modify the contents of the `noiseProperties` array after passing it to the constructor if that array is referenced elsewhere. While this class treats it as immutable, external modification could lead to concurrency issues.

## Data Pipeline
MinNoiseProperty acts as a stateless processing and aggregation node in the procedural data flow. It does not cache results.

> Flow:
> World Generator Request -> **MinNoiseProperty.get(seed, x, y, z)** -> Iterates internal array -> Dispatches get() to child NoiseProperty[0], NoiseProperty[1], ... -> Aggregates results -> Returns single minimum double value

