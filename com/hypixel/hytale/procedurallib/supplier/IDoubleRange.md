---
description: Architectural reference for IDoubleRange
---

# IDoubleRange

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface IDoubleRange {
   // ... methods
}
```

## Architecture & Concepts
The IDoubleRange interface is a foundational contract within Hytale's procedural generation library. It represents an abstract source of a double-precision floating-point value. Its primary role is to decouple procedural algorithms, such as terrain or feature generators, from the specific method used to generate a numeric value.

This abstraction is a form of the **Strategy Pattern**. Instead of a generator hard-coding a call to a random number generator or a Perlin noise function, it operates against the IDoubleRange interface. This allows the behavior of the generator to be dynamically configured by supplying different implementations of IDoubleRange. For example, the same terrain algorithm could produce flat plains, rolling hills, or jagged mountains simply by being provided with a Constant, a Random, or a Noise-based implementation of this interface, respectively.

The various overloads of the getValue method provide context-sensitivity, allowing the supplied value to depend on inputs like a random seed, a coordinate in 2D or 3D space, or another value supplier. This is critical for creating coherent and deterministic procedural worlds.

## Lifecycle & Ownership
As an interface, IDoubleRange itself has no lifecycle. The following pertains to its concrete implementations.

- **Creation:** Implementations are typically instantiated by configuration factories during the setup phase of a world generation process. They are often defined in data files (e.g., JSON assets describing a biome) and created on-demand by the procedural generation orchestrator.
- **Scope:** The lifetime of an IDoubleRange implementation is tied to the scope of the generator that uses it. It may be short-lived, existing only for the generation of a single world chunk, or it may persist for the entire world generation session if it represents a global parameter.
- **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for collection once the owning procedural generator and any related tasks are complete and no longer hold a reference.

## Internal State & Concurrency
- **State:** The interface is stateless. Implementations, however, can be either stateful or stateless.
    - A *stateless* implementation might return a constant value or derive a value purely from its input coordinates (e.g., a simplex noise function).
    - A *stateful* implementation might wrap a java.util.Random instance, whose internal state changes after each call.

- **Thread Safety:** **Not guaranteed.** The contract does not enforce thread safety. Procedural generation is a highly parallelized process. It is the responsibility of the *implementation* to be thread-safe if it is to be used across multiple worker threads.
    - **WARNING:** Sharing a stateful, non-thread-safe implementation (such as one wrapping a standard Random object) across parallel generation tasks will lead to race conditions, severe visual artifacts, and non-deterministic world output. Implementations intended for concurrent use must employ thread-local state, synchronization, or use atomic operations.

## API Surface
The public contract consists of overloaded methods for retrieving a value based on different contexts.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getValue(double) | double | Implementation-Dependent | Returns a value, potentially using the input double as a seed or parameter. |
| getValue(DoubleSupplier) | double | Implementation-Dependent | Returns a value, potentially sourcing an input from the provided supplier. |
| getValue(Random) | double | Implementation-Dependent | Returns a value using the provided Random instance. Critical for deterministic generation. |
| getValue(int, double, double, IDoubleCoordinateSupplier2d) | double | Implementation-Dependent | Returns a value for a 2D context, typically for terrain heightmaps or biome maps. |
| getValue(int, double, double, double, IDoubleCoordinateSupplier3d) | double | Implementation-Dependent | Returns a value for a 3D context, typically for 3D noise, ore distribution, or cave generation. |

## Integration Patterns

### Standard Usage
An algorithm, such as a terrain generator, receives an IDoubleRange implementation from a factory or configuration object. It then calls the appropriate getValue method within its generation loop, passing the current world coordinates to get a height or density value.

```java
// A procedural generator receives an IDoubleRange for determining feature density.
// This instance was likely configured in a JSON file and loaded by a factory.
IDoubleRange featureDensity = biomeConfig.getFeatureDensityRange();

for (int x = 0; x < CHUNK_SIZE; x++) {
    for (int z = 0; z < CHUNK_SIZE; z++) {
        // The generator does not know or care if the value comes from noise, a constant, or randomness.
        // It passes a supplier for the world coordinates.
        double density = featureDensity.getValue(seed, x, z, coordinateSupplier);
        if (density > DENSITY_THRESHOLD) {
            // Place a feature
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Implementation Casting:** Do not attempt to cast an IDoubleRange to a specific concrete class. This violates the abstraction and makes the generation algorithm rigid and non-configurable.
- **Ignoring Context:** Calling a context-sensitive implementation (like a noise function) with static values inside a loop. This defeats the purpose of using a coordinate-based supplier and will result in uniform, non-varied output.
- **Assuming Thread Safety:** Passing a single, stateful IDoubleRange instance to multiple parallel chunk generation threads without ensuring the implementation is thread-safe. This is a common and critical source of bugs in procedural generation.

## Data Pipeline
IDoubleRange is a data *source*, not a processing stage. It sits at the beginning of many procedural calculations.

> Flow:
> World Generation Configuration -> Factory creates **IDoubleRange Implementation** -> Procedural Algorithm (e.g., TerrainGenerator) -> Algorithm calls getValue(context) -> Returned double is used to generate Voxel Data or place Entities

