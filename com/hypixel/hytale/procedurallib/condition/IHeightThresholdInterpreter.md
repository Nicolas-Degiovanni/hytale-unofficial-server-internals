---
description: Architectural reference for IHeightThresholdInterpreter
---

# IHeightThresholdInterpreter

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Contract / Interface

## Definition
```java
// Signature
public interface IHeightThresholdInterpreter {
   int getLowestNonOne();
   int getHighestNonZero();
   float getThreshold(int var1, double var2, double var4, int var6);
   float getThreshold(int var1, double var2, double var4, int var6, double var7);
   double getContext(int var1, double var2, double var4);
   int getLength();
   default boolean isSpawnable(int height) { //... }
   static float lerp(float from, float to, float t) { //... }
}
```

## Architecture & Concepts
The IHeightThresholdInterpreter interface defines a stateless contract for interpreting world generation data at a specific vertical position. It is a core component of the procedural generation engine, acting as a Strategy Pattern for height-based conditional logic.

This interface decouples the high-level generation algorithms (e.g., biome placement, feature spawning) from the specific rules that govern *where* those elements can appear along the world's Y-axis. An implementation of this interface encapsulates a specific set of rules, such as "a gradual threshold change near coastlines" or "a sharp cutoff at a specific mountain altitude".

By passing different implementations of IHeightThresholdInterpreter to a generator, the engine can produce vastly different vertical terrain features and biome distributions without altering the generator's core logic. This makes the system highly data-driven and extensible.

### Lifecycle & Ownership
As an interface, IHeightThresholdInterpreter itself has no lifecycle. The lifecycle described here applies to its concrete implementations.

- **Creation:** Implementations are typically instantiated and configured by a higher-level system, such as a BiomeDataManager or WorldProfileLoader, often based on deserialized world generation asset files (e.g., JSON). They are not intended for direct instantiation by game logic.
- **Scope:** The lifetime of an implementation instance is tied to the procedural generation task it supports. For world generation, an instance may be created for a specific biome definition and persist as long as that biome is being used for chunk generation.
- **Destruction:** Instances are subject to standard Java garbage collection. They are typically released after the parent generator or biome definition is unloaded.

## Internal State & Concurrency
- **State:** The contract is designed to be **stateless**. All methods accept the necessary world context (coordinates, noise values) as parameters. Implementations should be immutable or effectively immutable after their initial configuration. Storing per-request state within an implementation is a severe anti-pattern.
- **Thread Safety:** Implementations **must be thread-safe**. The procedural generation engine is heavily parallelized, and a single IHeightThresholdInterpreter instance will be called concurrently from multiple worker threads generating different world chunks. The stateless design facilitates this requirement.

## API Surface
The public contract is designed for high-performance access during world generation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getLowestNonOne() | int | O(1) | Returns the minimum world height (Y-level) at which this interpreter is active. |
| getHighestNonZero() | int | O(1) | Returns the maximum world height (Y-level) at which this interpreter is active. |
| getThreshold(...) | float | O(1) | **Primary Method.** Calculates the conditional threshold at a given world position. The parameters likely correspond to X, Y, Z coordinates and pre-computed noise values. |
| getContext(...) | double | O(1) | Retrieves a context-specific value, potentially a secondary noise value or biome weight, at a given world position. |
| isSpawnable(height) | boolean | O(1) | A default convenience method to check if a given height falls within the min/max active range. |
| lerp(from, to, t) | float | O(1) | A static utility function for linear interpolation, commonly used in threshold calculations. |

## Integration Patterns

### Standard Usage
An implementation of this interface is typically retrieved from a data-driven configuration, such as a biome definition. A generator then uses it to make a decision for a block at a specific coordinate.

```java
// A generator receives an interpreter, likely from a biome's configuration
IHeightThresholdInterpreter interpreter = biome.getHeightInterpreter();
int worldX = 1024;
int worldY = 78;
int worldZ = -512;

// Check if the height is even valid for this interpreter
if (interpreter.isSpawnable(worldY)) {
    // Get the threshold for this specific location
    float threshold = interpreter.getThreshold(worldX, worldY, worldZ, someSeed, someNoiseValue);

    // Compare the threshold against another value to make a generation decision
    if (anotherNoiseValue > threshold) {
        // Place a feature, block, or entity
        world.setBlock(worldX, worldY, worldZ, Block.OAK_LOG);
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Stateful Implementations:** Do not create implementations that store state related to a specific generation request. This will fail catastrophically in the multi-threaded generator. All required data must be passed in as method arguments.
- **Ignoring Spawnable Range:** Do not call getThreshold for a height outside the range defined by getLowestNonOne and getHighestNonZero. While it may not fail, the results are undefined and computationally wasteful. Always use isSpawnable as a guard clause.

## Data Pipeline
This component acts as a function-like transformer within the larger procedural generation pipeline. It does not move data but rather produces a critical decision-making value from incoming spatial data.

> Flow:
> World Generator requests Chunk -> For each (X,Y,Z) in Chunk -> Noise Sampler provides noise values -> **IHeightThresholdInterpreter** calculates threshold -> Generator Logic compares noise to threshold -> Decision (e.g., Place Block, Skip) is made.

