---
description: Architectural reference for SingleNoiseProperty
---

# SingleNoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient

## Definition
```java
// Signature
public class SingleNoiseProperty implements NoiseProperty {
```

## Architecture & Concepts
The SingleNoiseProperty class is a fundamental component within the procedural generation library. It serves as a concrete implementation of the NoiseProperty strategy interface, acting as an essential adapter between a raw NoiseFunction and higher-level generation logic.

Its primary architectural role is to apply a standard set of transformations to a noise source, making it predictable and usable by other systems. Specifically, it performs two critical operations:

1.  **Seed Offsetting:** It introduces a deterministic offset to the world seed before passing it to the underlying NoiseFunction. This allows a single base noise algorithm (e.g., Perlin noise) to produce multiple, distinct, yet correlated noise maps. This pattern is crucial for generating related world features, such as layering temperature and humidity maps that are different but follow similar continental shapes.
2.  **Range Normalization:** It transforms the output of the wrapped NoiseFunction, which is typically in the range of -1.0 to 1.0, into a standardized range of 0.0 to 1.0. This normalization is required by most procedural systems that consume these values as percentages, weights, or blend factors.

This class embodies the Decorator pattern, adding functionality to a NoiseFunction object without altering its core interface.

### Lifecycle & Ownership
-   **Creation:** Instantiated directly via its constructor, typically during the static configuration of a world generator, biome, or other procedural feature. It is a simple value object, not managed by a service locator or dependency injection container.
-   **Scope:** The lifetime of a SingleNoiseProperty instance is bound to its owning configuration object. It is created once during initialization and persists for the duration of all generation tasks that use that configuration.
-   **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection when its parent configuration is discarded, for example, after a world has been fully generated. No explicit cleanup is required.

## Internal State & Concurrency
-   **State:** **Immutable**. The internal fields, seedOffset and function, are declared final and are assigned exclusively at construction time. An instance of this class cannot be modified after it is created.
-   **Thread Safety:** **Inherently thread-safe**. Due to its immutable nature, a SingleNoiseProperty instance can be safely shared and used across multiple generator threads without any external locking or synchronization. The overall thread safety is contingent on the thread safety of the wrapped NoiseFunction, which are themselves designed to be stateless and thread-safe.

## API Surface
The public API is minimal, focusing on the execution of the noise query.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(N) | Computes the 2D noise value. Complexity is dependent on the wrapped NoiseFunction. |
| get(seed, x, y, z) | double | O(N) | Computes the 3D noise value. Complexity is dependent on the wrapped NoiseFunction. |
| getSeedOffset() | int | O(1) | Returns the integer offset applied to the seed for all noise calculations. |
| getFunction() | NoiseFunction | O(1) | Returns the underlying NoiseFunction instance being decorated. |

## Integration Patterns

### Standard Usage
A SingleNoiseProperty is used to prepare a raw noise function for use in a generator. It is configured once and reused for many calculations.

```java
// 1. Define the base noise function (e.g., Perlin)
NoiseFunction basePerlin = new PerlinNoise(0.01);

// 2. Create two distinct properties from the same base function using different seed offsets
NoiseProperty temperature = new SingleNoiseProperty(1000, basePerlin);
NoiseProperty humidity    = new SingleNoiseProperty(2000, basePerlin);

// 3. Use the properties to get normalized values during world generation
int worldSeed = 12345;
double tempValue = temperature.get(worldSeed, 150.5, 75.2);   // Returns a value between 0.0 and 1.0
double humValue  = humidity.get(worldSeed, 150.5, 75.2);    // Returns a different value between 0.0 and 1.0
```

### Anti-Patterns (Do NOT do this)
-   **Per-call Instantiation:** Do not create a new SingleNoiseProperty inside a tight loop for every block or coordinate being evaluated. This is highly inefficient and generates significant garbage collector pressure. These are configuration objects, intended to be created once and reused.
-   **Null Function Injection:** The constructor does not perform null checks. Passing a null NoiseFunction will result in a fatal NullPointerException at runtime when the get method is invoked. Always ensure a valid NoiseFunction is provided.

## Data Pipeline
This class acts as a transformation node within a larger data flow. It does not orchestrate a pipeline but is a critical step within one.

> Flow:
> (World Seed, Coordinates) -> **SingleNoiseProperty.get()** -> (Seed + seedOffset, Coordinates) -> Wrapped NoiseFunction.get() -> Raw Noise Value [-1, 1] -> Normalization (`value * 0.5 + 0.5`) -> Clamped Noise Value [0, 1] -> Consumer (e.g., Biome Selector)

