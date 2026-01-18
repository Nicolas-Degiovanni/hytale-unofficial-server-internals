---
description: Architectural reference for RandomCoordinateDoubleSupplier
---

# RandomCoordinateDoubleSupplier

**Package:** com.hypixel.hytale.server.worldgen.util.function
**Type:** Utility

## Definition
```java
// Signature
public class RandomCoordinateDoubleSupplier implements ICoordinateDoubleSupplier {
```

## Architecture & Concepts
The RandomCoordinateDoubleSupplier is a foundational component within the server-side procedural world generation engine. Its primary role is to provide a deterministic, pseudo-random floating-point value for any given coordinate in the world. It acts as a stateless function object that maps an input coordinate and a world seed to a predictable output value within a pre-configured range.

This class is a concrete implementation of the ICoordinateDoubleSupplier strategy interface. This allows higher-level world generation systems, such as biome placement or ore distribution logic, to remain agnostic to the specific method of value generation. They can operate on any ICoordinateDoubleSupplier, whether it produces Perlin noise, simplex noise, or simple random values like this class.

The determinism is critical for world generation. By leveraging HashUtil, the class ensures that generating the same world with the same seed will produce an identical output every time. This is essential for multiplayer consistency and for players to be able to share world seeds.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand by higher-level world generation coordinators, often during the parsing of worldgen definition files (e.g., JSON biome descriptions). The required IDoubleRange dependency is typically defined in these same data files and injected during construction.
- **Scope:** Transient and short-lived. An instance of RandomCoordinateDoubleSupplier typically exists only for the duration of a specific, isolated generation task, such as populating a single chunk with features. It is not a long-lived service.
- **Destruction:** The object holds no native resources or external connections. It is a simple Java object, and its memory is reclaimed by the Garbage Collector once it falls out of scope.

## Internal State & Concurrency
- **State:** Effectively immutable. The internal state consists of a single `final` field, `range`, which is set at construction time and cannot be changed. The class performs calculations but does not mutate its own state.
- **Thread Safety:** This class is inherently thread-safe. Its immutability and the fact that its methods are pure functions (their output depends solely on their inputs) mean it can be safely used across multiple world generation threads without any need for locks, synchronization, or other concurrency controls. This is a vital property for enabling parallel chunk generation.

## API Surface
The public contract is defined by the ICoordinateDoubleSupplier interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| apply(seed, x, y) | double | O(1) | Returns a deterministic double for a 2D coordinate based on the seed. |
| apply(seed, x, y, z) | double | O(1) | Returns a deterministic double for a 3D coordinate based on the seed. |
| getRange() | IDoubleRange | O(1) | Retrieves the configured range that constrains the output values. |

## Integration Patterns

### Standard Usage
This component is used as a source of deterministic randomness to make decisions during world generation, such as whether to place a tree or select a specific block type.

```java
// A world generator receives a pre-configured supplier for a task.
// This supplier might be configured to return values between 0.0 and 1.0.
ICoordinateDoubleSupplier placementChance = new RandomCoordinateDoubleSupplier(new DoubleRange(0.0, 1.0));

int worldSeed = 12345;
int blockX = 150;
int blockZ = -450;

// Get a value unique to this coordinate.
double value = placementChance.apply(worldSeed, blockX, blockZ);

// Use the value to make a procedural decision.
if (value > 0.99) {
    // Place a rare feature at (blockX, blockZ).
}
```

### Anti-Patterns (Do NOT do this)
- **Instantiation in a Loop:** Never create a new RandomCoordinateDoubleSupplier inside a tight loop (e.g., for every block in a chunk). This creates excessive object churn and is highly inefficient. Instantiate it once for the entire generation task.
- **Ignoring the World Seed:** Passing a static value for the `seed` parameter will result in a repetitive, non-random pattern across the world. The global world seed must be correctly propagated to this function.

## Data Pipeline
This class acts as a transformation node in the worldgen data flow. It converts spatial information into a numerical value used for subsequent logical decisions.

> Flow:
> World Seed + Block Coordinate -> **HashUtil.random** -> Unbounded Hash Integer -> **RandomCoordinateDoubleSupplier** -> Bounded Double -> World Generation Logic (e.g., Feature Placer, Block Selector)

