---
description: Architectural reference for ShapeDistortion
---

# ShapeDistortion

**Package:** com.hypixel.hytale.server.worldgen.cave.shape.distorted
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class ShapeDistortion {
```

## Architecture & Concepts
The ShapeDistortion class is a fundamental data structure within the procedural world generation framework, specifically for cave systems. It does not generate geometry itself; rather, it encapsulates a set of noise-based modifiers that are applied to a primitive cave shape. This class acts as a configuration object or a "distortion profile".

Its primary architectural role is to decouple the high-level cave pathing and shape logic from the fine-grained, organic details. A generator can define a simple tunnel, and then apply a ShapeDistortion instance to introduce realistic variations in width, floor height, and ceiling height.

This model allows world designers to define complex cave aesthetics by composing different NoiseProperty objects—often loaded from external asset files—without altering the core generation algorithms. The class provides a clean, immutable interface for the generation pipeline to query distortion factors at any given world coordinate.

### Lifecycle & Ownership
- **Creation:** Instances are typically created via the static factory method `of`. This occurs within higher-level world generation coordinators when they parse biome or feature-specific configuration. The `DEFAULT` static final instance is used as a flyweight for the common case of no distortion, avoiding unnecessary object allocation.
- **Scope:** Transient. A ShapeDistortion object's lifetime is typically scoped to the generation of a specific set of world chunks or a single cave system. It is passed down through the generation call stack and discarded afterward.
- **Destruction:** Managed exclusively by the Java Garbage Collector. As a pure data holder with no external resources, it requires no explicit cleanup.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields holding noise properties are declared final and are assigned only once during construction. This design is critical for predictable and deterministic world generation. An instance of ShapeDistortion cannot be modified after creation.

- **Thread Safety:** **Fully thread-safe**. Its immutability guarantees that a single instance can be safely read by multiple world generation worker threads concurrently without any need for locks or synchronization.

    **WARNING:** While the ShapeDistortion class itself is thread-safe, this safety relies on the assumption that the provided NoiseProperty implementations are also thread-safe. Passing a mutable or non-thread-safe NoiseProperty can break this contract and lead to severe concurrency issues.

## API Surface
The public API is minimal, focused on querying the distortion factors for the three spatial axes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getWidthFactor(seed, x, z) | double | O(N) | Returns the width distortion multiplier. Complexity depends on the underlying noise function. |
| getFloorFactor(seed, x, z) | double | O(N) | Returns the floor height distortion factor. Complexity depends on the underlying noise function. |
| getCeilingFactor(seed, x, z) | double | O(N) | Returns the ceiling height distortion factor. Complexity depends on the underlying noise function. |
| of(width, floor, ceiling) | static ShapeDistortion | O(1) | Safely constructs a ShapeDistortion instance, handling null inputs and optimizing for the default case. |

## Integration Patterns

### Standard Usage
A ShapeDistortion profile is constructed from pre-loaded noise properties and passed into a generation algorithm that requires it. The `of` factory method is the canonical way to create instances.

```java
// Assume widthNoise and floorNoise are loaded from worldgen asset files
NoiseProperty widthNoise = assetManager.get("caves/standard_width_noise");
NoiseProperty floorNoise = assetManager.get("caves/bumpy_floor_noise");

// Create a distortion profile. The ceiling will use the default (no distortion)
// because null is passed.
ShapeDistortion distortionProfile = ShapeDistortion.of(widthNoise, floorNoise, null);

// The profile is then consumed by a cave generator
caveGenerator.carveTunnel(chunkPosition, distortionProfile);
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Avoid using `new ShapeDistortion(n1, n2, n3)`. The public constructor does not provide the null-safety checks or the singleton optimization for the default case that the `of` factory method does. Always prefer the static factory.

- **Stateful Noise Sources:** Do not provide NoiseProperty implementations that contain mutable state. The world generator assumes that calls to `getWidthFactor` with the same inputs will always produce the same output. Violating this assumption will result in non-deterministic generation and visible chunk borders.

## Data Pipeline
ShapeDistortion serves as a configuration input to the main voxel carving pipeline. It does not process data itself but provides the parameters that guide the process.

> Flow:
> WorldGen Asset (JSON) -> Deserializer -> **NoiseProperty** instances -> **ShapeDistortion.of()** -> Cave Shape Algorithm -> Voxel Buffer Modification

