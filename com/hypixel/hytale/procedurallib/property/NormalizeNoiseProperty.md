---
description: Architectural reference for NormalizeNoiseProperty
---

# NormalizeNoiseProperty

**Package:** com.hypixel.hytale.procedurallib.property
**Type:** Transient

## Definition
```java
// Signature
public class NormalizeNoiseProperty implements NoiseProperty {
```

## Architecture & Concepts
NormalizeNoiseProperty is a foundational component within the procedural generation framework, implementing the **Decorator** design pattern. It acts as a stateless transformation filter that wraps another, more primitive, NoiseProperty instance.

Its core architectural function is to remap the output of an arbitrary noise function to a standardized, predictable range, typically 0.0 to 1.0. Raw noise functions, such as Simplex or Perlin, often produce values in an unbounded or inconvenient range (e.g., -1.0 to 1.0). This class provides the essential mathematical translation required to make these values usable by downstream systems, such as biome selectors or terrain height mappers, which expect a consistent and normalized input.

By chaining these decorators, developers can construct complex and layered noise graphs where each stage operates on a known data range, ensuring stable and predictable world generation.

### Lifecycle & Ownership
- **Creation:** Instances are created dynamically by higher-level procedural graph builders or configuration parsers. They are not managed as singletons or long-lived services. A new instance is typically created for each node in a noise generation graph that requires remapping.
- **Scope:** The lifetime of a NormalizeNoiseProperty object is strictly bound to the procedural generation graph it is a part of. It is a lightweight, transient value object.
- **Destruction:** The object is eligible for garbage collection as soon as the root of its noise graph is no longer referenced. No explicit destruction or resource cleanup is necessary.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields, including the reference to the wrapped NoiseProperty, are declared final and are set exclusively at construction time. The object's state cannot be modified post-instantiation.
- **Thread Safety:** **Fully thread-safe and reentrant**. Its immutable nature and lack of side effects guarantee that a single instance can be safely invoked by multiple world-generation threads concurrently without any risk of data corruption or race conditions. This is a critical property for enabling parallel chunk generation.

## API Surface
The public contract is defined by the NoiseProperty interface. The primary methods are the overloaded `get` functions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y) | double | O(C) | Fetches a value from the wrapped noise source, normalizes it using the configured min and range, and clamps the result to the 0.0-1.0 range. C is the complexity of the wrapped source. |
| get(seed, x, y, z) | double | O(C) | 3D variant of the normalization operation. C is the complexity of the wrapped source. |

## Integration Patterns

### Standard Usage
This class should be used to wrap a base noise source whose output range is known. The `min` and `range` parameters are used to define the expected input range for the normalization formula.

```java
// Assume baseNoise produces values roughly between -0.8 and 0.8
NoiseProperty baseNoise = new SimplexNoiseProperty(...);

// Remap the [-0.8, 0.8] range to [0.0, 1.0]
// min = -0.8, range = 1.6
NoiseProperty normalizedNoise = new NormalizeNoiseProperty(baseNoise, -0.8, 1.6);

// The value will now be reliably between 0.0 and 1.0
double value = normalizedNoise.get(seed, x, y);
```

### Anti-Patterns (Do NOT do this)
- **Incorrect Range Configuration:** Providing a `min` and `range` that do not accurately reflect the output of the wrapped NoiseProperty is a severe logical error. This will lead to skewed or heavily clamped data, resulting in visual artifacts like plateaus, terraces, or unexpected feature distribution in the final generated world.
- **Redundant Normalization:** Chaining multiple NormalizeNoiseProperty instances is almost always incorrect and inefficient. The first normalization pass should produce a standard 0.0-1.0 range, making subsequent normalizations logically redundant.

## Data Pipeline
NormalizeNoiseProperty serves as a transformation stage in a larger data processing pipeline. It consumes a raw noise value and produces a normalized value for downstream consumers.

> Flow:
> Upstream NoiseProperty -> Raw Noise Value -> **NormalizeNoiseProperty** -> Normalized Value [0.0, 1.0] -> Downstream Consumer (e.g., Terrain Generator, Biome Selector)

