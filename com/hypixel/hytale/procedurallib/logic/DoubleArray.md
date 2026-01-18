---
description: Architectural reference for DoubleArray and its nested value types.
---

# DoubleArray

**Package:** com.hypixel.hytale.procedurallib.logic
**Type:** Utility / Namespace

## Definition
```java
// Signature
public class DoubleArray {
    // Nested classes Double2 and Double3
}
```

## Architecture & Concepts
The DoubleArray class serves as a static namespace for fundamental, high-performance value objects used throughout the procedural generation library. It is not intended to be instantiated. Its primary purpose is to group the related Double2 and Double3 classes, which represent immutable 2D and 3D points or vectors using double-precision floating-point numbers.

These nested classes are designed as Plain Old Java Objects (POJOs) with an emphasis on immutability and minimal overhead. By exposing fields directly as public and final, they avoid the method call overhead of getters, a common and acceptable performance optimization in performance-critical domains like procedural generation. They are core data structures used to pass coordinate information into noise functions, geometry generators, and other computational algorithms.

## Lifecycle & Ownership
- **Creation:** Instances of the nested classes Double2 and Double3 are created on-demand by various systems within the procedural library. They are transient, short-lived objects. The outer DoubleArray class is never instantiated.
- **Scope:** The lifecycle of a Double2 or Double3 instance is typically confined to the scope of a single method call or a specific generation task. They are passed by value (as object references) between methods and are eligible for garbage collection once they are no longer referenced.
- **Destruction:** Managed entirely by the Java Garbage Collector. There are no manual cleanup or resource release requirements.

## Internal State & Concurrency
- **State:** The outer DoubleArray class is stateless. The nested Double2 and Double3 classes are **strictly immutable**. Their state (x, y, z coordinates) is set once at construction time via their public final fields and cannot be changed thereafter.
- **Thread Safety:** **Guaranteed thread-safe.** Due to their immutable nature, instances of Double2 and Double3 can be safely shared and read across multiple threads without any need for locks, synchronization, or other concurrency control mechanisms. This makes them ideal for use in parallelized world generation tasks.

## API Surface
The API consists solely of the constructors for the nested classes.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Double2(x, y) | constructor | O(1) | Creates an immutable 2D vector. |
| Double3(x, y, z) | constructor | O(1) | Creates an immutable 3D vector. |

## Integration Patterns

### Standard Usage
These classes are used as simple data carriers for coordinate information. They are the standard input for many procedural functions.

```java
// Example: Providing a 2D coordinate to a noise function
import com.hypixel.hytale.procedurallib.logic.DoubleArray.Double2;
import com.hypixel.hytale.procedurallib.noise.NoiseGenerator;

NoiseGenerator noise = ...;
Double2 samplePoint = new Double2(15.5, -30.1);
double noiseValue = noise.get(samplePoint);
```

### Anti-Patterns (Do NOT do this)
- **Attempted Instantiation:** Do not attempt to create an instance of the outer DoubleArray class. It is a static utility container and has no constructor.
- **Subclassing:** Do not extend DoubleArray, Double2, or Double3. They are not designed for inheritance and should be treated as final.
- **Mutable Wrappers:** Avoid creating mutable wrapper classes around Double2 or Double3 to simulate modification. This defeats the core design principle of immutability and can lead to severe, difficult-to-diagnose bugs in multi-threaded contexts. If a new point is needed, create a new instance.

## Data Pipeline
DoubleArray's nested classes are not a pipeline component themselves, but rather the data packets that flow *through* a pipeline. They represent a discrete, immutable snapshot of a position in space.

> Flow:
> Generation Algorithm -> **new DoubleArray.Double3(x, y, z)** -> Noise Function -> Voxel Placement Logic

