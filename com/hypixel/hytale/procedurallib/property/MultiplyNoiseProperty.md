---
description: Architectural reference for MultiplyNoiseProperty
---

# MultiplyNoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient

## Definition
```java
// Signature
public class MultiplyNoiseProperty implements NoiseProperty {
```

## Architecture & Concepts
The MultiplyNoiseProperty class is a concrete implementation of the NoiseProperty strategy interface. It functions as a *Composite Operator* within the procedural generation framework, designed to combine the outputs of multiple other NoiseProperty instances.

Architecturally, this class is a fundamental building block for creating complex, layered noise patterns. It enables designers to modulate one noise function with another. For example, a primary noise function might define continental shapes, while a secondary, finer-grained noise function defines mountainous regions. By multiplying these two, the mountainous features are constrained to appear only on the continents. This multiplicative blending is essential for creating sophisticated and believable world features.

This component is a pure data transformer; it accepts a coordinate and a seed, queries its child properties, and returns a single, combined floating-point value.

### Lifecycle & Ownership
- **Creation:** MultiplyNoiseProperty is not a globally managed service. It is instantiated on-demand by higher-level systems, typically a procedural generation graph builder that interprets a world generation preset (e.g., from a JSON definition). It is constructed with an array of other NoiseProperty objects that it will operate on.
- **Scope:** The lifetime of a MultiplyNoiseProperty instance is tied to the specific procedural generation task it supports. It is a configuration-dependent, short-lived object that exists only as long as the generator configuration is held in memory.
- **Destruction:** The object is managed by the Java garbage collector. It is eligible for cleanup as soon as the procedural generation graph it belongs to is no longer referenced, typically after a world chunk has been generated. No explicit destruction or cleanup methods are required.

## Internal State & Concurrency
- **State:** The internal state consists of a final array of NoiseProperty objects. The reference to this array is immutable and established at construction time. The class itself does not mutate its state during its operational methods. It should be treated as a stateless calculator from a caller's perspective.
- **Thread Safety:** This class is inherently thread-safe. The core get methods are pure functions with no side effects and do not modify any instance fields. This design is critical for the engine's ability to perform parallel world generation, as multiple worker threads can safely invoke get methods on a shared MultiplyNoiseProperty instance without locks or synchronization.

    **WARNING:** The overall thread safety is contingent on the thread safety of the child NoiseProperty objects provided during construction. All standard engine NoiseProperty implementations are designed to be thread-safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(N) | Calculates the combined 2D noise value by multiplying the results of all contained noise properties. N is the number of properties. |
| get(seed, x, y, z) | double | O(N) | Calculates the combined 3D noise value by multiplying the results of all contained noise properties. N is the number of properties. |
| getNoiseProperties() | NoiseProperty[] | O(1) | Returns the internal array of child noise properties. |

## Integration Patterns

### Standard Usage
This class is intended to be used as a node in a larger procedural generation graph. It is constructed with its dependencies and then used as a standard NoiseProperty.

```java
// Assume noiseA and noiseB are other configured NoiseProperty instances
NoiseProperty[] propertiesToCombine = new NoiseProperty[] { noiseA, noiseB };

// Create the composite operator
NoiseProperty multiplier = new MultiplyNoiseProperty(propertiesToCombine);

// Use the combined property to get a value for a specific world coordinate
double worldValue = multiplier.get(worldSeed, 123.4, 567.8);
```

### Anti-Patterns (Do NOT do this)
- **Invalid Array Construction:** Do not construct this class with a null, empty, or single-element array. An empty array will cause an ArrayIndexOutOfBoundsException at runtime. A single-element array is functionally correct but introduces unnecessary overhead; use the single NoiseProperty directly instead.
- **Post-Construction Mutation:** Do not retrieve the internal array via getNoiseProperties and modify its contents. The procedural system assumes these configurations are immutable after creation. Modifying the array at runtime can lead to severe and difficult-to-debug race conditions in a multi-threaded generator.

## Data Pipeline
MultiplyNoiseProperty acts as a processing node in a data flow. It does not orchestrate a pipeline but rather participates in one.

> Flow:
> Procedural Generator Request -> **MultiplyNoiseProperty.get(seed, x, y)** -> [Recursive calls to child NoiseProperty.get()] -> Final Multiplied Value -> Voxel Generation Stage

