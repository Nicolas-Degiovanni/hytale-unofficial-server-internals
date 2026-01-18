---
description: Architectural reference for Calculator
---

# Calculator

**Package:** com.hypixel.hytale.builtin.hytalegenerator.framework.math
**Type:** Utility

## Definition
```java
// Signature
public class Calculator {
```

## Architecture & Concepts
The Calculator class is a stateless utility that provides a centralized, shared library of common and specialized mathematical functions. It serves as a foundational component within the world generation framework, encapsulating low-level mathematical logic required for complex procedural algorithms.

Its primary architectural role is to promote code reuse and ensure consistent mathematical behavior across disparate systems such as terrain shaping, biome placement, and feature distribution. By centralizing these operations, it prevents logic duplication and simplifies the implementation of higher-level generator components. The methods are designed to be pure functions, operating exclusively on their input arguments without side effects.

## Lifecycle & Ownership
As a static utility class, Calculator does not follow a traditional object lifecycle.

- **Creation:** The class is never instantiated. It is loaded into the JVM by the system ClassLoader when one of its static methods is first invoked by another component.
- **Scope:** The class and its static methods have an application-wide scope. They are available for the entire duration of the application's runtime once loaded.
- **Destruction:** The class is unloaded from the JVM when its ClassLoader is garbage collected, which typically occurs only at application shutdown. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** The Calculator class is **stateless and immutable**. It contains no instance or static fields, meaning each method call is independent and produces output based solely on its input arguments.
- **Thread Safety:** This class is **fully thread-safe**. Due to its stateless nature, there is no shared data to protect. All methods can be safely and efficiently invoked from multiple threads concurrently without any need for external synchronization or locks. This is critical for parallelized world generation tasks.

## API Surface
The public API consists entirely of static methods for mathematical computations.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| toIntFloored(double d) | int | O(1) | Floors a double-precision number and casts the result to an integer. |
| max(double... n) | double | O(N) | Returns the maximum value from a variable-length array of doubles. Throws IllegalArgumentException if the input array is empty. |
| min(int... n) | int | O(N) | Returns the minimum value from a variable-length array of integers. Throws IllegalArgumentException if the input array is empty. |
| limit(int value, int floor, int ceil) | int | O(1) | Constrains a value to a specified range [floor, ceil]. **Warning:** Throws IllegalArgumentException if floor is not strictly less than ceil. |
| clamp(double wallA, double value, double wallB) | double | O(1) | Constrains a value between two bounds, automatically determining the minimum and maximum from the provided walls. |
| distance(Vector3d a, Vector3d b) | double | O(1) | Calculates the Euclidean distance between two 3D vectors. |
| isDivisibleBy(int number, int divisor) | boolean | O(log4(N)) | **Warning:** This method has a highly specialized implementation. It checks if a number is a power of 4, not a generic divisibility check. Do not use for general modulo operations. |
| smoothMin(double range, double a, double b) | double | O(1) | Calculates a smooth minimum between two values. Essential for blending Signed Distance Fields or noise layers without sharp creases. |
| floor(int value, int gridSize) | int | O(1) | Snaps an integer value down to the nearest multiple of the given grid size. |

## Integration Patterns

### Standard Usage
Consumers should invoke methods directly on the class. Using static imports is highly recommended to improve the readability of mathematical expressions within generator algorithms.

```java
// Recommended usage with static imports for clarity
import static com.hypixel.hytale.builtin.hytalegenerator.framework.math.Calculator.*;

public class TerrainShaper {
    private static final int MIN_TERRAIN_HEIGHT = 0;
    private static final int MAX_TERRAIN_HEIGHT = 255;

    public int calculateVoxelHeight(double noiseValue) {
        // Noise values can range outside of our desired bounds
        double scaledNoise = noiseValue * MAX_TERRAIN_HEIGHT;
        int finalHeight = toIntFloored(scaledNoise);

        // Ensure the final height is within world limits
        return clamp(MIN_TERRAIN_HEIGHT, finalHeight, MAX_TERRAIN_HEIGHT);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not create an instance of Calculator using `new Calculator()`. This provides no benefit, wastes a small amount of memory, and violates the design intent of a static utility class. All methods are static and should be accessed accordingly.
- **Misinterpreting isDivisibleBy:** Do not use the `isDivisibleBy` method for general-purpose divisibility checks. Its internal logic specifically tests if a number is a power of 4. For standard checks, use the `perfectDiv` method or the native modulo operator (`%`).

## Data Pipeline
The Calculator class does not orchestrate data pipelines but rather acts as a functional processing node within them. It is frequently used to transform, normalize, or constrain numerical data as it flows through a generation pipeline.

> **Example Flow: Terrain Height Calculation**
>
> Perlin Noise Generator -> Raw double value (-1.0 to 1.0) -> **Calculator.clamp()** -> Clamped value -> **Calculator.toIntFloored()** -> Integer Height -> Voxel Buffer Writer

