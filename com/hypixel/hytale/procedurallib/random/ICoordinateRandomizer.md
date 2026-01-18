---
description: Architectural reference for ICoordinateRandomizer
---

# ICoordinateRandomizer

**Package:** com.hypixel.hytale.procedurallib.random
**Type:** Contract/Interface

## Definition
```java
// Signature
public interface ICoordinateRandomizer {
```

## Architecture & Concepts
The ICoordinateRandomizer interface defines a strict contract for deterministic, coordinate-based random number generation. It is a foundational component of the procedural generation library, responsible for ensuring that world generation is reproducible.

This interface decouples procedural algorithms (e.g., biome placement, ore distribution, feature spawning) from the specific implementation of the random number generator. The core principle is that for a given seed and a specific set of world coordinates, the output must always be identical. This allows for consistent world regeneration and parallelized generation tasks without the need for complex state synchronization.

Implementations of this interface are expected to be pure functions where the output is derived solely from the input parameters, typically involving a combination of hashing and noise functions.

### Lifecycle & Ownership
As an interface, ICoordinateRandomizer has no lifecycle itself. The following applies to its concrete implementations.

- **Creation:** Implementations are instantiated by high-level procedural generation controllers, such as a WorldGenerator or a ProceduralTask. They are typically created at the beginning of a specific, scoped generation pass (e.g., generating a single world chunk).
- **Scope:** The lifetime of an implementation is transient and is strictly tied to the scope of the generation task it was created for. It does not persist between sessions or even between distinct generation regions.
- **Destruction:** The object is eligible for garbage collection as soon as the generation task completes and all references to it are dropped. There is no manual destruction or cleanup required.

## Internal State & Concurrency
- **State:** The interface is stateless. Implementations are expected to be effectively immutable after construction. Any internal state, such as a world seed, must be set during instantiation and never modified thereafter. The methods do not and must not alter the object's internal state upon invocation.
- **Thread Safety:** Implementations of this interface **must be** thread-safe. The procedural generation engine heavily relies on parallel execution, where multiple worker threads will invoke methods on a shared randomizer instance for different coordinates simultaneously. The functional, stateless nature of the contract ensures that implementations can be safely shared across threads without locks or other synchronization primitives.

## API Surface
The public API provides methods to generate random double-precision floating-point numbers for 2D and 3D coordinate systems.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| randomDoubleX(int, double, double) | double | O(1) | Generates a deterministic random value for a 2D X-coordinate. |
| randomDoubleY(int, double, double) | double | O(1) | Generates a deterministic random value for a 2D Y-coordinate. |
| randomDoubleX(int, double, double, double) | double | O(1) | Generates a deterministic random value for a 3D X-coordinate. |
| randomDoubleY(int, double, double, double) | double | O(1) | Generates a deterministic random value for a 3D Y-coordinate. |
| randomDoubleZ(int, double, double, double) | double | O(1) | Generates a deterministic random value for a 3D Z-coordinate. |

**WARNING:** The first integer parameter in all methods acts as a salt or a secondary seed. It is critical for introducing variation and must not be a constant value across different generation features.

## Integration Patterns

### Standard Usage
An implementation of ICoordinateRandomizer is typically provided to a generation algorithm by a controlling system. The algorithm then uses it to make deterministic decisions based on world position.

```java
// A generator receives a pre-configured randomizer for its task
public void generateFeatures(WorldChunk chunk, ICoordinateRandomizer randomizer) {
    for (int x = 0; x < 16; x++) {
        for (int z = 0; z < 16; z++) {
            // Use a feature-specific salt (e.g., hash of "hytale:tree")
            int treeSalt = 12345;
            double worldX = chunk.getStartX() + x;
            double worldZ = chunk.getStartZ() + z;

            // Get a deterministic value for this specific coordinate
            double value = randomizer.randomDoubleX(treeSalt, worldX, worldZ);

            if (value > 0.95) {
                // Place a tree here
            }
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** An implementation must not change its internal state upon method invocation. Using a standard sequential RNG like java.util.Random internally would violate the core contract of coordinate-based determinism and break world generation.
- **Ignoring Salt Parameter:** Passing the same constant integer as the first parameter for all calls will result in correlated or identical random values for different features at the same coordinate, leading to undesirable generation patterns.
- **Cross-Thread State:** Introducing any mutable, shared state within an implementation is a severe violation of the concurrency contract and will lead to race conditions and non-deterministic output in a multithreaded environment.

## Data Pipeline
This component acts as a functional data source within the procedural generation pipeline. It does not transform data in a flow; rather, it creates it on demand.

> Flow:
> World Seed + Feature Salt + Coordinates -> **ICoordinateRandomizer Implementation** -> Deterministic Double -> Procedural Algorithm (e.g., BiomeGenerator, FeaturePlacer)

