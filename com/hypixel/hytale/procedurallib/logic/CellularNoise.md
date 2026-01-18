---
description: Architectural reference for CellularNoise
---

# CellularNoise

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Utility

## Definition
```java
// Signature
public final class CellularNoise {
```

## Architecture & Concepts
The CellularNoise class is a static utility class that serves as a foundational data provider for the procedural generation engine. It does not perform any calculations itself; instead, it holds pre-computed, constant lookup tables (LUTs) for generating 2D and 3D cellular noise, also known as Worley noise.

This class embodies a critical performance optimization. By providing a fixed set of 256 pseudo-random feature points for both 2D and 3D space, it eliminates the need for expensive random point generation during runtime. This approach guarantees deterministic and highly performant noise calculations across the entire engine, which is essential for consistent world generation.

Architecturally, CellularNoise is a dependency-free data source. It sits at the lowest level of the procedural logic stack, providing immutable data to higher-level systems like biome generators, terrain shapers, and texture synthesizers. These systems use the feature points from this class as the basis for calculating distance fields, which are then used to create cellular patterns.

## Lifecycle & Ownership
- **Creation:** This class is never instantiated. Its private constructor explicitly throws an UnsupportedOperationException to prevent instantiation. The class is loaded into the JVM by the ClassLoader when it is first referenced, at which point its static fields are initialized once.
- **Scope:** The class and its static data arrays persist for the entire lifetime of the application after being loaded. Its scope is global and application-wide.
- **Destruction:** The data is only eligible for garbage collection when the application's ClassLoader is unloaded, which typically only occurs on JVM shutdown. There is no manual destruction mechanism.

## Internal State & Concurrency
- **State:** The state of CellularNoise is **deeply immutable**. The `CELL_2D` and `CELL_3D` fields are `public static final` arrays. The array references cannot be changed, and the `DoubleArray.Double2` and `DoubleArray.Double3` objects they contain are themselves immutable value types. This design ensures that the lookup data can never be corrupted at runtime.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its completely static and immutable nature, its data can be safely read by any number of threads concurrently without requiring any locks or synchronization primitives. This is crucial for parallelizing world generation tasks.

## API Surface
The public contract of this class consists solely of two static fields. There are no methods.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CELL_2D | DoubleArray.Double2[] | O(1) | Provides a pre-computed, immutable lookup table of 256 feature points for 2D cellular noise generation. |
| CELL_3D | DoubleArray.Double3[] | O(1) | Provides a pre-computed, immutable lookup table of 256 feature points for 3D cellular noise generation. |

## Integration Patterns

### Standard Usage
A procedural generation algorithm accesses the static arrays directly to retrieve feature points for a given grid cell. A hashing function is typically used to map a cell's coordinates to an index in the array.

```java
// Example: A noise function retrieves a feature point for a 2D grid cell.
// The hash function must be deterministic.
int cellHash = Hashing.hash(cellX, cellY);
int index = cellHash & 255; // Mask to wrap within the array bounds

// Retrieve the pre-computed, random offset for this cell
DoubleArray.Double2 featurePoint = CellularNoise.CELL_2D[index];

// Calculate the final point's world position
double pointX = (double)cellX + featurePoint.x;
double pointY = (double)cellY + featurePoint.y;

// ... proceed with distance calculations using this point.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never attempt to create an instance with `new CellularNoise()`. This will fail with an UnsupportedOperationException and is a fundamental misuse of a static utility class.
- **Attempted Modification:** Do not attempt to modify the contents of the `CELL_2D` or `CELL_3D` arrays. While the Java language allows array elements to be modified even if the array reference is final, doing so would violate the architectural contract of this class. Modifying this shared, global data would destroy generation determinism and introduce severe, difficult-to-trace bugs.

## Data Pipeline
CellularNoise acts as a data source, not a processing stage. It injects pre-computed data into the beginning of a noise generation pipeline.

> Flow:
> **CellularNoise** (Static LUT) -> Procedural Noise Function -> Distance Field Calculation -> World Data or Texture Map

