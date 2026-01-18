---
description: Architectural reference for ConstantFloatCoordinateHashSupplier
---

# ConstantFloatCoordinateHashSupplier

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Value Object

## Definition
```java
// Signature
public class ConstantFloatCoordinateHashSupplier implements IFloatCoordinateHashSupplier {
```

## Architecture & Concepts
The ConstantFloatCoordinateHashSupplier is a foundational component within the procedural generation framework. It serves as a terminal or "leaf" node in a supplier graph, providing a predictable, unchanging float value regardless of input coordinates, seed, or hash.

Its primary architectural role is to act as a **baseline provider**. In complex procedural systems, where various suppliers are chained and composed to create noise, patterns, and features, this class provides a simple, non-variant input. It is essential for scenarios requiring a uniform value across a region, such as defining a perfectly flat water level, a consistent biome attribute, or a default value in a conditional generation algorithm.

The class includes static pre-allocated instances for common values (0.0 and 1.0), promoting reuse and reducing memory allocation overhead for frequent use cases.

### Lifecycle & Ownership
-   **Creation:** Typically instantiated by a higher-level configuration or factory responsible for assembling a procedural generation graph. Developers can create instances directly using the public constructor or use the provided static instances `ZERO` and `ONE`.
-   **Scope:** The lifetime of a ConstantFloatCoordinateHashSupplier instance is tied to the parent object that references it, such as a biome definition or a terrain generator configuration. It is a lightweight, transient object.
-   **Destruction:** The object is managed by the Java Garbage Collector and is reclaimed when no longer referenced. No explicit cleanup is necessary.

## Internal State & Concurrency
-   **State:** This object is **Immutable**. Its state consists of a single `final float result`, which is set at construction time and cannot be modified thereafter.
-   **Thread Safety:** Due to its immutable nature, this class is inherently **thread-safe**. A single instance can be safely shared and accessed concurrently by multiple threads without any need for external synchronization. This is a critical property for high-performance, parallelized world generation tasks.

## API Surface
The public contract is minimal, focusing on value provision.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ZERO | static ConstantFloatCoordinateHashSupplier | O(1) | A shared, pre-allocated instance that always returns 0.0F. |
| ONE | static ConstantFloatCoordinateHashSupplier | O(1) | A shared, pre-allocated instance that always returns 1.0F. |
| ConstantFloatCoordinateHashSupplier(float) | Constructor | O(1) | Creates a new supplier that will always return the provided float value. |
| get(int, double, double, long) | float | O(1) | Returns the configured constant float value, ignoring all input parameters. |
| getResult() | float | O(1) | Retrieves the constant float value this supplier was constructed with. |

## Integration Patterns

### Standard Usage
This class is used as a simple input or a fallback within a larger procedural algorithm. It is often passed to components that expect an IFloatCoordinateHashSupplier interface.

```java
// Example: Configuring a terrain layer with a constant base height.
TerrainLayerConfig config = new TerrainLayerConfig();

// Use the static instance for a base height of 0.
IFloatCoordinateHashSupplier baseHeight = ConstantFloatCoordinateHashSupplier.ZERO;

// Or create a custom constant value.
IFloatCoordinateHashSupplier waterLevel = new ConstantFloatCoordinateHashSupplier(-32.0F);

config.setBaseHeightSupplier(baseHeight);
config.setWaterLevelSupplier(waterLevel);
```

### Anti-Patterns (Do NOT do this)
-   **Redundant Instantiation:** Avoid creating new instances for common values like 0.0 or 1.0. Use the static final fields `ZERO` and `ONE` to prevent unnecessary object allocations.
    ```java
    // AVOID
    IFloatCoordinateHashSupplier supplier = new ConstantFloatCoordinateHashSupplier(0.0F);

    // PREFER
    IFloatCoordinateHashSupplier supplier = ConstantFloatCoordinateHashSupplier.ZERO;
    ```
-   **Misuse in Dynamic Contexts:** Do not use this class in a system that requires a variable or noise-based output. Its purpose is to be constant; using it where variation is expected is a design error.

## Data Pipeline
This class acts as a **source node** in a data processing pipeline. It does not consume or transform data; it originates a constant value that is then fed into downstream systems.

> Flow:
> **ConstantFloatCoordinateHashSupplier** -> Procedural Algorithm (e.g., TerrainBlender, BiomeSelector) -> Voxel Data Structure

