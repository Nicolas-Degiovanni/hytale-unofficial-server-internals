---
description: Architectural reference for GeneralNoise
---

# GeneralNoise

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Utility

## Definition
```java
// Signature
public final class GeneralNoise {
```

## Architecture & Concepts
The GeneralNoise class is a foundational, stateless utility that provides the core mathematical primitives for procedural noise generation. It does not generate complete noise patterns itself but serves as a toolkit of pure functions used by higher-level algorithms like Perlin or Simplex noise generators.

Its primary role is to establish a deterministic and repeatable basis for creating pseudo-random values from spatial coordinates. This determinism is critical for procedural world generation, ensuring that a given world seed always produces the exact same environment.

The core architectural components provided by this class are:
*   **Coordinate Hashing:** Transforms integer grid coordinates (x, y, z) and a seed into a well-distributed, pseudo-random integer. This is achieved using large prime numbers to decorrelate the input coordinates, preventing visible grid artifacts in the final noise.
*   **Gradient Vector Selection:** Uses the hashed coordinate values to select a pre-defined gradient vector from a constant lookup table (GRAD_2D, GRAD_3D).
*   **Gradient Contribution Calculation:** Provides functions (gradCoord2D, gradCoord3D) to compute the dot product between a gradient vector and a distance vector. This result represents the "influence" of a single grid point's gradient on the final noise value at a specific location.
*   **Interpolation Functions:** Defines standard smoothing functions (Linear, Hermite, Quintic) used by consuming algorithms to blend the influences from multiple grid points, transforming the discrete grid contributions into a smooth, continuous noise field.

This class sits at the lowest level of the procedural generation library, acting as a dependency for any system that requires coherent noise.

## Lifecycle & Ownership
- **Creation:** The GeneralNoise class is never instantiated. It is a final class with a private constructor that throws an UnsupportedOperationException to enforce a purely static usage pattern.
- **Scope:** As a utility class, its static methods and constants are loaded by the JVM and persist for the entire application lifetime. They are globally accessible.
- **Destruction:** The class and its associated data are unloaded by the JVM upon application termination. No manual memory management or cleanup is necessary.

## Internal State & Concurrency
- **State:** GeneralNoise is **completely stateless and immutable**. Its only internal data consists of static final constants (e.g., X_PRIME, GRAD_2D) which are initialized at class-loading time and cannot be modified.
- **Thread Safety:** This class is inherently **thread-safe**. All methods are pure functions, meaning their output is determined exclusively by their input arguments. They do not access or modify any shared, mutable state. Consequently, all methods can be safely called from multiple threads concurrently without any need for external synchronization or locks.

## API Surface
The public API consists of mathematical primitives for hashing, gradient calculation, and interpolation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| hash2D(seed, x, y) | int | O(1) | Computes a deterministic hash from a seed and 2D integer coordinates. |
| hash3D(seed, x, y, z) | int | O(1) | Computes a deterministic hash from a seed and 3D integer coordinates. |
| gradCoord2D(seed, x, y, xd, yd) | double | O(1) | Calculates the 2D gradient contribution for a point relative to its grid cell. |
| gradCoord3D(seed, x, y, z, xd, yd, zd) | double | O(1) | Calculates the 3D gradient contribution for a point relative to its grid cell. |
| lerp(a, b, t) | double | O(1) | Performs linear interpolation between two values. |
| InterpolationMode | enum | O(1) | Provides access to standard interpolation functions (LINEAR, HERMITE, QUINTIC). |

## Integration Patterns

### Standard Usage
This class should only be used via static method calls from higher-level procedural generation algorithms. It provides the low-level building blocks for those systems.

```java
// Example: A hypothetical noise generator using GeneralNoise primitives
public double getNoise(double x, double y) {
    int seed = 12345;
    int x0 = GeneralNoise.fastFloor(x);
    int y0 = GeneralNoise.fastFloor(y);

    // Get gradient contributions from surrounding grid points
    double n0 = GeneralNoise.gradCoord2D(seed, x0, y0, x - x0, y - y0);
    double n1 = GeneralNoise.gradCoord2D(seed, x0 + 1, y0, x - (x0 + 1), y - y0);
    // ... and so on for other corners

    // Interpolate the results
    GeneralNoise.InterpolationFunction smoother = GeneralNoise.InterpolationMode.QUINTIC.getFunction();
    double t = smoother.interpolate(x - x0);
    return GeneralNoise.lerp(n0, n1, t);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never attempt to create an instance of this class. The code is designed to prevent this and will throw an exception.
  - **BAD:** `GeneralNoise noise = new GeneralNoise();`
- **State Modification:** Do not attempt to modify the class constants (e.g., X_PRIME, GRAD_3D) via reflection. The determinism of the entire world generation system relies on these values being constant. Modifying them will lead to unpredictable and non-repeatable world generation.

## Data Pipeline
GeneralNoise is not a pipeline itself, but a critical processing stage within the broader procedural generation pipeline. It is invoked by noise algorithms to transform coordinate inputs into gradient values.

> Flow:
> World Generator -> Noise Algorithm (e.g., PerlinNoise) -> **GeneralNoise.hash3D()** -> **GeneralNoise.gradCoord3D()** -> Noise Algorithm (Interpolation) -> Final Noise Value

