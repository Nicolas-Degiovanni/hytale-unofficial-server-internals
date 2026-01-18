---
description: Architectural reference for DoubleFunctionXZ
---

# DoubleFunctionXZ

**Package:** com.hypixel.hytale.builtin.hytalegenerator.materialproviders.functions
**Type:** Functional Contract

## Definition
```java
// Signature
@FunctionalInterface
public interface DoubleFunctionXZ {
   double apply(double var1, double var3);
}
```

## Architecture & Concepts
DoubleFunctionXZ is a functional interface that defines a mathematical contract: a function that accepts a two-dimensional coordinate (X and Z) and returns a single floating-point value. This interface is a fundamental building block within the world generation system, acting as a standardized representation for any 2D scalar field.

Its primary role is to decouple the *source* of a 2D value (e.g., Perlin noise, simplex noise, a mathematical formula) from the *consumer* of that value (e.g., a terrain heightmap generator, a biome distribution system, a temperature map). By using this interface, different noise algorithms or functions can be swapped, chained, or composed without altering the systems that use them. This is a classic Strategy pattern implementation, where the algorithm is encapsulated in a lambda or method reference.

The naming convention explicitly signals that this function operates on the horizontal XZ plane of the game world, which is the standard for top-down procedural generation tasks.

### Lifecycle & Ownership
As a functional interface, DoubleFunctionXZ does not have a traditional object lifecycle. Its implementations are typically ephemeral.

-   **Creation:** Implementations are almost exclusively created as lambda expressions or method references at the point of use. They are instantiated by higher-level generator services that need to define a specific mathematical behavior, such as a BiomeProvider configuring its placement logic.
-   **Scope:** The scope of an implementation is transient and localized. It exists only as long as the object that holds a reference to it. It is frequently passed as a method parameter and has no persistent lifetime beyond the immediate call stack.
-   **Destruction:** Handled entirely by the Java Garbage Collector. Once the lambda instance is no longer referenced, it is eligible for cleanup.

## Internal State & Concurrency
-   **State:** Implementations of DoubleFunctionXZ are architecturally required to be **stateless**. The `apply` method must be a pure function: for a given X and Z input, it must always produce the same output. Any required state, such as a world seed or noise parameters, should be captured from the enclosing scope when the lambda is defined, not mutated during execution.
-   **Thread Safety:** This interface is designed to be unconditionally thread-safe. The stateless, pure-function contract ensures that a single instance can be safely invoked by multiple world-generation threads concurrently without locks or synchronization. This is critical for achieving high-performance, parallelized chunk generation.

## API Surface
The public contract consists of a single method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(double x, double z) | double | O(1)\* | Computes and returns a scalar value for the given X and Z world coordinates. Throws no exceptions. |

\* The complexity of the `apply` call itself is constant. However, the computational cost of the underlying implementation (e.g., a multi-octave noise function) can vary significantly.

## Integration Patterns

### Standard Usage
The interface is intended to be implemented with a lambda expression and passed to a system that consumes 2D data.

```java
// A system that requires a 2D function to define terrain height
TerrainGenerator generator = new TerrainGenerator();

// Define a simple sine wave function for rolling hills
DoubleFunctionXZ heightFunction = (x, z) -> {
    return Math.sin(x / 32.0) * 10.0 + Math.sin(z / 32.0) * 10.0;
};

// Pass the function to the generator
generator.setHeightmapFunction(heightFunction);
generator.generateChunk(0, 0);
```

### Anti-Patterns (Do NOT do this)
-   **Stateful Implementations:** Never create a lambda that modifies external state or its own captured state from within the `apply` method. This violates the pure function contract, breaks thread safety, and will lead to severe and difficult-to-debug world generation artifacts like chunk borders.

    ```java
    // DO NOT DO THIS
    final double[] counter = {0.0};
    DoubleFunctionXZ badFunction = (x, z) -> {
        counter[0]++; // Mutating external state is forbidden
        return x + z + counter[0];
    };
    ```
-   **Misusing the Semantic Contract:** Do not use this interface for functions that are not conceptually tied to the XZ world plane. The name provides a clear semantic meaning that should be respected to maintain code clarity.

## Data Pipeline
DoubleFunctionXZ is not a data pipeline itself, but rather a key component *within* a pipeline. It acts as a data source or transformer for spatial information.

> Flow:
> World Generator requests data for a region -> Calls a **DoubleFunctionXZ** implementation (e.g., NoiseProvider) -> The function returns a scalar value (e.g., 0.71) -> The value is interpreted by a consumer (e.g., BiomeSelector) -> A decision is made (e.g., place Forest biome) -> Voxel data is generated

