---
description: Architectural reference for IDoubleCoordinateSupplier2d
---

# IDoubleCoordinateSupplier2d

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Interface

## Definition
```java
// Signature
public interface IDoubleCoordinateSupplier2d {
   double get(int var1, double var2, double var4);
}
```

## Architecture & Concepts
The IDoubleCoordinateSupplier2d interface defines a strict contract for any component that can supply a double-precision floating-point value based on a 2D coordinate and an integer seed. This is a foundational component within the procedural generation library, acting as a functional interface or a "callback" for higher-level generation algorithms.

Its primary role is to abstract the source of a value at a given point in 2D space. This allows complex systems like terrain generators, biome mappers, or texture placement algorithms to be decoupled from the specific noise function (e.g., Perlin, Simplex, Worley) or data source being used. By passing an implementation of this interface, the calling algorithm can retrieve values on-demand without needing to know the underlying calculation logic.

The parameters are conventionally interpreted as:
- **var1:** The world seed or a regional seed for the generation task.
- **var2:** The coordinate on the X-axis.
- **var4:** The coordinate on the Z-axis (in a Y-up coordinate system).

## Lifecycle & Ownership
As an interface, IDoubleCoordinateSupplier2d has no lifecycle itself. However, its implementations are subject to strict lifecycle expectations.

- **Creation:** Implementations are typically created and configured once during the initialization of a larger procedural generation system, such as a WorldGenerator or BiomeProvider. They are often composed, chained, or decorated to create complex noise behavior.
- **Scope:** The scope is tied to the parent generator. An implementation may be a long-lived singleton if it represents a stateless noise function, or it may be a transient object created for a specific chunk generation task.
- **Destruction:** Implementations are eligible for garbage collection when the owning generator is destroyed or the specific generation task completes.

## Internal State & Concurrency
The contract of IDoubleCoordinateSupplier2d makes critical assumptions about the state and thread safety of its implementations.

- **State:** Implementations are **strongly expected** to be stateless. The `get` method should be a pure function, meaning its output depends solely on its input parameters. Caching results is permissible but must be managed in a thread-safe manner.
- **Thread Safety:** Implementations **must be thread-safe**. The procedural generation engine is heavily multi-threaded and will invoke the `get` method concurrently from multiple worker threads during world generation. Any internal state, including caches, must be protected by appropriate synchronization mechanisms (e.g., locks, concurrent data structures) to prevent race conditions. Failure to ensure thread safety will lead to severe and difficult-to-diagnose generation artifacts.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(int seed, double x, double z) | double | Varies | Returns the procedural value at the given 2D coordinate for the specified seed. Complexity depends entirely on the implementation (e.g., O(1) for simple noise, O(log n) for fractal noise). |

## Integration Patterns

### Standard Usage
This interface is almost never used directly. Instead, an implementation is passed as a strategy object to a higher-level service that requires values over a 2D domain.

```java
// A generator receives a supplier as part of its configuration.
// It does not know or care about the specific noise algorithm.
public class TerrainGenerator {
    private final IDoubleCoordinateSupplier2d heightMapSupplier;

    public TerrainGenerator(IDoubleCoordinateSupplier2d heightMapSupplier) {
        this.heightMapSupplier = heightMapSupplier;
    }

    public double getHeightAt(int seed, double x, double z) {
        // The generator invokes the supplier to get the base height value.
        return this.heightMapSupplier.get(seed, x, z);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not create implementations that rely on mutable internal state that changes between calls to `get`. This breaks the functional contract and will produce non-deterministic and inconsistent results when called from multiple threads.
- **Assuming Call Order:** Never design an implementation that assumes `get` will be called for coordinates in a specific sequence (e.g., sequentially along the X-axis). The engine's work scheduler provides no such guarantee.

## Data Pipeline
The IDoubleCoordinateSupplier2d is a fundamental source node in a data generation pipeline. It originates a value that is then consumed and transformed by subsequent stages.

> Flow:
> World Generator Request -> **IDoubleCoordinateSupplier2d (e.g., PerlinNoiseSupplier)** -> Value Transformation (e.g., scaling, clamping) -> Biome Selector -> Final Block Placement

