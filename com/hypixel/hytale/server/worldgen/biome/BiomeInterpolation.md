---
description: Architectural reference for BiomeInterpolation
---

# BiomeInterpolation

**Package:** com.hypixel.hytale.server.worldgen.biome
**Type:** Immutable Value Object

## Definition
```java
// Signature
public class BiomeInterpolation {
```

## Architecture & Concepts
The BiomeInterpolation class is a fundamental configuration object within the server-side world generation pipeline. Its primary role is to define the rules for smoothing and blending the boundaries between adjacent biomes. This prevents the harsh, linear transitions that would otherwise occur, creating a more natural and organic world appearance.

Architecturally, this class acts as a parameter object, encapsulating the distance-based rules for biome influence. It operates on two key principles:

1.  A **global radius** that defines the default smoothing distance for all biome transitions.
2.  An optional, biome-specific **override map** that allows for finer control, enabling certain biomes to have a larger or smaller zone of influence than the default.

A critical design choice is the storage of squared radii values (e.g., biomeRadii2). This is a deliberate performance optimization. During world generation, distance checks are computationally expensive, often involving square root calculations. By working with squared distances and squared radii, the engine can perform these comparisons using only multiplication and addition, significantly reducing computational overhead in the performance-critical world generation loop.

The class also implements a Flyweight pattern through its static DEFAULT instance and the create factory method. This ensures that the most common configuration (a radius of 5 with no overrides) does not result in redundant object allocations, conserving memory.

## Lifecycle & Ownership
-   **Creation:** Instances are exclusively created via the static factory method BiomeInterpolation.create. This method is typically invoked by higher-level world generation configuration loaders or biome layout managers when parsing worldgen rules.
-   **Scope:** Transient and stateless. An instance of BiomeInterpolation exists only as long as the world generation configuration that references it is in memory. It holds no external resources or state.
-   **Destruction:** Instances are managed by the Java Garbage Collector and are eligible for cleanup once they are no longer referenced by any active world generation process.

## Internal State & Concurrency
-   **State:** Strictly immutable. The internal fields *radius* and *biomeRadii2* are final and are assigned only once during construction. This design guarantees that a configuration object cannot be altered after its creation.
-   **Thread Safety:** This class is inherently thread-safe due to its immutability. A single BiomeInterpolation instance can be safely shared and read by multiple concurrent world generation threads without any need for locks or other synchronization primitives. This is essential for a scalable, multi-threaded world generator.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| create(int, Int2IntMap) | BiomeInterpolation | O(1) | **Factory Method.** Constructs a new instance or returns the shared DEFAULT instance. |
| getRadius() | int | O(1) | Returns the default smoothing radius. |
| getBiomeRadius2(int) | int | O(1) | Returns the biome-specific squared radius, or the default squared radius if no override exists. |

## Integration Patterns

### Standard Usage
The intended use is to define a biome blending strategy during world generation setup. The factory method handles the creation and potential reuse of the object.

```java
// Example: Create a configuration with a default radius of 10
// and a larger radius for a specific "Mountain" biome (ID 4).
Int2IntMap customRadii = new Int2IntOpenHashMap();
int mountainBiomeId = 4;
int mountainRadius = 20;
customRadii.put(mountainBiomeId, mountainRadius * mountainRadius);

// The factory method is the only supported way to create an instance.
BiomeInterpolation customInterpolation = BiomeInterpolation.create(10, customRadii);

// This object would then be passed to the biome placement system.
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructor is protected and should never be invoked directly. Always use the static BiomeInterpolation.create factory method to ensure caching and proper initialization logic is applied.
-   **Post-Creation Map Mutation:** Do not modify the Int2IntMap that was passed into the create method after the BiomeInterpolation object has been created. The class does not create a defensive copy for performance reasons, and mutating the external map will lead to unpredictable behavior in the world generator.

## Data Pipeline
BiomeInterpolation does not process data itself; it is a configuration parameter *for* a data pipeline. It provides the rules that govern how the final biome map is generated.

> Flow:
> World Generation Ruleset -> **BiomeInterpolation** (Configuration) -> Biome Placement & Blending Algorithm -> Final Voxel Biome Data

