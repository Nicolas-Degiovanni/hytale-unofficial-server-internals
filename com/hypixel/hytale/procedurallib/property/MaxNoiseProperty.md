---
description: Architectural reference for MaxNoiseProperty
---

# MaxNoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient

## Definition
```java
// Signature
public class MaxNoiseProperty implements NoiseProperty {
```

## Architecture & Concepts
The MaxNoiseProperty is a composite implementation of the NoiseProperty interface. It does not generate noise itself; instead, it acts as an aggregator or a filter that combines the output of multiple other NoiseProperty instances. Its core function is to evaluate a set of input noise functions at a given coordinate and return the single highest value among them.

This component is fundamental to blending procedural features. For example, one NoiseProperty might define mountain ranges while another defines rolling hills. By using MaxNoiseProperty, the world generator can ensure that where these two features overlap, the taller feature (the mountain) is the one that gets expressed in the final heightmap, creating a natural-looking transition.

Architecturally, it embodies the **Composite Pattern**. It treats a collection of NoiseProperty objects (the children) as a single NoiseProperty (the composite). This allows complex noise behaviors to be built up from simpler, reusable components.

A key internal detail is the `MAX_EPSILON` constant, used for a performance optimization. If any evaluated noise value is already extremely close to the maximum possible value of 1.0, the loop short-circuits and returns 1.0 immediately, avoiding redundant calculations on the remaining noise properties.

## Lifecycle & Ownership
-   **Creation:** Instantiated by a higher-level procedural generation system, typically during the assembly of a world generation "recipe". It is constructed with a pre-defined array of `NoiseProperty` objects that it will manage.
-   **Scope:** The lifetime of a MaxNoiseProperty instance is tied to the specific procedural rule or generator configuration that created it. It is not a global or session-scoped object.
-   **Destruction:** Eligible for garbage collection as soon as the parent generator configuration is discarded. It holds no external resources that require explicit cleanup.

## Internal State & Concurrency
-   **State:** The internal state consists of the `noiseProperties` array. This array is provided at construction and is never modified thereafter. The class is effectively **immutable**.
-   **Thread Safety:** This class is inherently **thread-safe**. The `get` methods are pure functions that depend only on their inputs and the immutable internal state. A single MaxNoiseProperty instance can be safely shared and invoked by multiple world generation threads simultaneously without locks or synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(N) | Calculates the maximum noise value from all contained properties at a 2D coordinate. N is the number of properties. |
| get(seed, x, y, z) | double | O(N) | Calculates the maximum noise value from all contained properties at a 3D coordinate. N is the number of properties. |
| getNoiseProperties() | NoiseProperty[] | O(1) | Returns a reference to the internal array of noise properties. |

## Integration Patterns

### Standard Usage
The primary use case is to combine multiple noise layers, selecting the most prominent feature at any given point.

```java
// Assume noiseA and noiseB are existing NoiseProperty instances
NoiseProperty[] layers = new NoiseProperty[] { noiseA, noiseB };

// MaxNoiseProperty will return the higher value of noiseA or noiseB
NoiseProperty blendedNoise = new MaxNoiseProperty(layers);

// Use the blended result in world generation
double height = blendedNoise.get(worldSeed, 128.5, 70.2);
```

### Anti-Patterns (Do NOT do this)
-   **Empty or Null Array:** Constructing a MaxNoiseProperty with a null or empty `noiseProperties` array will result in a runtime exception (`NullPointerException` or `ArrayIndexOutOfBoundsException`) when `get` is called. The constructor must be provided with at least one valid NoiseProperty.
-   **External Modification of Array:** The `getNoiseProperties` method returns a direct reference to the internal array. Modifying this array externally after construction violates the immutability assumption of the class and can lead to unpredictable behavior, especially in a multithreaded context.

## Data Pipeline
The data flow is a fan-out/fan-in process. A single coordinate input is distributed to multiple child processors, and their results are aggregated into a single output.

> Flow:
> (Seed, Coordinates) -> **MaxNoiseProperty.get()** -> [ChildProperty1.get(), ChildProperty2.get(), ...] -> Math.max() Aggregation -> Final double value

