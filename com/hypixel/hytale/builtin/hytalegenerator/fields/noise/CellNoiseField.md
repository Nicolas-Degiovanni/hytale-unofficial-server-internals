---
description: Architectural reference for CellNoiseField
---

# CellNoiseField

**Package:** com.hypixel.hytale.builtin.hytalegenerator.fields.noise
**Type:** Transient

## Definition
```java
// Signature
public class CellNoiseField extends NoiseField {
```

## Architecture & Concepts
The CellNoiseField is a concrete implementation of the NoiseField abstraction, specifically designed to generate cellular noise patterns. These patterns are essential for creating natural-looking textures and structures like cracked earth, stone formations, or organic tissues within the world generator.

This class functions as a high-level configuration wrapper around the core **FastNoiseLite** library. It encapsulates the complex setup for cellular noise, including parameters for jitter, octaves, and return types, presenting a simplified and consistent interface to the rest of the world generation system.

A key architectural feature is its support for **Domain Warping**. When enabled, the input coordinates are deliberately distorted by another noise function before the primary cellular noise is calculated. This breaks up the grid-like artifacts inherent in some procedural noise, resulting in more organic and less repetitive world features. The CellNoiseField manages the entire domain warp pipeline internally when configured to do so.

## Lifecycle & Ownership
- **Creation:** CellNoiseField instances are created by higher-level world generation controllers, such as a BiomeGenerator or FeaturePlacer. Configuration is typically loaded from world generation asset files and passed into the constructor. It is not managed by a dependency injection context.
- **Scope:** The lifetime of a CellNoiseField is tied to the specific world generation task that requires it. It is not a global or session-scoped object. An instance may be created to generate a single region and then discarded.
- **Destruction:** The object is managed by the Java garbage collector. It becomes eligible for collection as soon as the world generator that created it completes its task and releases its reference.

## Internal State & Concurrency
- **State:** The internal state of a CellNoiseField is **immutable** after construction. All configuration parameters like seed, scale, and jitter are set in the constructor and cannot be changed during the object's lifetime. The primary state is held within the private FastNoiseLite instance.
- **Thread Safety:** This class is **conditionally thread-safe**. Its state is not modified by any of its public methods after construction. Multiple threads can safely call the valueAt methods concurrently, provided the underlying FastNoiseLite library is itself thread-safe for concurrent read operations. No internal locking is performed.

## API Surface
The public contract is focused exclusively on retrieving noise values for given coordinates.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| valueAt(x, y, z, w) | double | O(N) | Calculates the 3D noise value, ignoring the w-coordinate. N is the number of octaves. |
| valueAt(x, y, z) | double | O(N) | Calculates the 3D noise value at the specified world coordinates. N is the number of octaves. |
| valueAt(x, z) | double | O(N) | Calculates the 2D noise value at the specified world coordinates. N is the number of octaves. |
| valueAt(x) | double | O(N) | Calculates the 1D noise value at the specified world coordinate. N is the number of octaves. |

## Integration Patterns

### Standard Usage
A CellNoiseField should be instantiated once with the desired configuration and reused for all coordinate lookups within a given generation task.

```java
// In a world generator or feature placer
// Configuration is typically loaded from an external asset
int seed = 1337;
double scale = 128.0;
double jitter = 0.45;
int octaves = 3;
FastNoiseLite.CellularReturnType type = FastNoiseLite.CellularReturnType.CellValue;

// Create the field once
NoiseField cellularPattern = new CellNoiseField(seed, scale, scale, scale, jitter, octaves, type);

// Use it repeatedly to determine block placement
for (int x = 0; x < 16; x++) {
    for (int z = 0; z < 16; z++) {
        double noiseValue = cellularPattern.valueAt(worldX + x, worldZ + z);
        if (noiseValue > 0.7) {
            // Place a special block
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Per-Call Instantiation:** Do not create a new CellNoiseField for every coordinate lookup. The constructor is a comparatively expensive operation that configures the underlying noise library.
- **Ignoring Constructor Validation:** The constructors throw IllegalArgumentException for invalid parameters (e.g., octaves < 1). This must be handled. Do not wrap instantiation in a blind try-catch block.
- **State Modification:** Do not attempt to modify the object's state after creation via reflection. The immutability of its configuration is critical for predictable and reproducible world generation.

## Data Pipeline
The flow of data for a single noise lookup is linear and synchronous.

> Flow:
> World Generator Coordinates (x, y, z) -> **CellNoiseField.valueAt()** -> Internal Coordinate Scaling -> (Optional) Domain Warp -> FastNoiseLite.getNoise() -> Final Noise Value (double) -> World Generator Decision Logic

