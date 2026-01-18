---
description: Architectural reference for Simplex
---

# Simplex

**Package:** com.hypixel.hytale.builtin.hytalegenerator.fields.noise
**Type:** Utility

## Definition
```java
// Signature
class Simplex {
```

## Architecture & Concepts
The Simplex class is a high-performance, stateless implementation of the Simplex noise algorithm. It serves as a foundational mathematical utility within the procedural content generation (PCG) pipeline. Its primary role is to provide deterministic, gradient-based noise values for any given coordinate in 2D, 3D, or 4D space.

Architecturally, this class is a pure computational engine. It does not manage game state, entities, or engine resources. Instead, it provides the raw data used by higher-level systems, such as the HytaleGenerator, to create natural-looking features like terrain elevation, biome distribution, and texture variations.

The class is intentionally package-private, indicating it is an internal implementation detail of the noise generation system. It is not designed for direct consumption by general game logic but rather to be wrapped or utilized by a public-facing noise service within the same package.

## Lifecycle & Ownership
- **Creation:** The Simplex class is never instantiated. Its "lifecycle" begins when the Java ClassLoader loads it into memory, which occurs upon the first static reference to the class. At this moment, a static initializer block executes exactly once to populate the internal permutation and gradient lookup tables.
- **Scope:** Application-level. The pre-computed static data persists for the entire lifetime of the application.
- **Destruction:** The class and its static data are unloaded from memory only when the application terminates and its ClassLoader is garbage collected. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** From a caller's perspective, the class is entirely stateless. However, it maintains a significant amount of internal static state in the form of pre-computed lookup tables (perm, permMod12, grad3, grad4). This state is **immutable**; it is written once during class loading and is strictly read-only thereafter.

- **Thread Safety:** This class is unconditionally **thread-safe**. All public methods are pure functions that depend only on their input arguments and the immutable static data. No synchronization or locking mechanisms are required. This design is critical for performance, as it allows world generation tasks to be parallelized across multiple threads without contention.

## API Surface
The public API consists of three static method overloads for generating noise in different dimensions.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| noise(double xin, double yin) | double | O(1) | Computes the 2D Simplex noise value for the given coordinates. |
| noise(double xin, double yin, double zin) | double | O(1) | Computes the 3D Simplex noise value for the given coordinates. |
| noise(double x, double y, double z, double w) | double | O(1) | Computes the 4D Simplex noise value for the given coordinates. |

## Integration Patterns

### Standard Usage
This class should only be invoked by other components within the `...fields.noise` package. A typical consumer would be a higher-level noise generator that combines or modifies the raw output.

```java
// Within a class in the same package, e.g., BiomeGenerator
package com.hypixel.hytale.builtin.hytalegenerator.fields.noise;

class BiomeGenerator {
    public Biome getBiomeAt(int x, int z) {
        // Scale coordinates for desired frequency
        double scaledX = x * 0.005;
        double scaledZ = z * 0.005;

        // Get raw noise value. The call is direct and static.
        double noiseValue = Simplex.noise(scaledX, scaledZ);

        // Map the noise value to a specific biome
        if (noiseValue > 0.5) {
            return Biome.FOREST;
        } else {
            return Biome.PLAINS;
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The class has no public constructor and consists of only static members. Attempting to instantiate it with `new Simplex()` will result in a compile-time error.
- **State Modification via Reflection:** The deterministic nature of Simplex noise relies on its internal permutation tables remaining constant. Using reflection to modify the `perm` or `grad3` arrays is a severe violation that will lead to unpredictable and non-reproducible world generation.
- **External Package Access:** This class is package-private for a reason. Bypassing access restrictions to call it from outside its intended package creates a brittle dependency on an internal implementation detail that may change without notice.

## Data Pipeline
The Simplex class acts as a pure function within a larger data processing pipeline. It transforms coordinate data into noise data.

> Flow:
> World Coordinate (int x, int z) -> Scaling & Transformation -> **Simplex.noise(double, double)** -> Raw Noise Value (-1.0 to 1.0) -> Mapping Logic -> Game Data (e.g., Block Type, Biome ID)

