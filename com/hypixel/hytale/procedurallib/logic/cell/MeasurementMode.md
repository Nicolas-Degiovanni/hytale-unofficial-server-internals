---
description: Architectural reference for MeasurementMode
---

# MeasurementMode

**Package:** com.hypixel.hytale.procedurallib.logic.cell
**Type:** Type-Safe Enumeration

## Definition
```java
// Signature
public enum MeasurementMode {
   CENTRE_DISTANCE,
   BORDER_DISTANCE;
}
```

## Architecture & Concepts
This enumeration defines the fundamental strategies for distance calculation within the procedural generation library, specifically for cell-based algorithms. It serves as a critical configuration parameter that dictates how proximity is determined between points and cellular regions. The choice of mode fundamentally alters the output of distance fields and any subsequent generation steps that rely on them.

-   **CENTRE_DISTANCE**: Calculates distance from the geometric center of a cell. This is the standard approach for classic Voronoi diagrams and point-based feature placement where the cell's defining point is the sole feature of interest.

-   **BORDER_DISTANCE**: Calculates distance from the nearest edge or boundary of a cell. This mode is essential for algorithms that require knowledge of clearance, thickness, or proximity to a region's perimeter, such as generating riverbanks, roads with shoulders, or biome transition zones.

This enum is a low-level, high-impact primitive in the procedural toolkit, providing a clear and type-safe way to control algorithmic behavior.

### Lifecycle & Ownership
-   **Creation:** Instances are created and managed exclusively by the Java Virtual Machine (JVM) during class loading. Only two instances of MeasurementMode will ever exist: CENTRE_DISTANCE and BORDER_DISTANCE. User code cannot and must not create instances.
-   **Scope:** Application-level. These constants are available for the entire lifetime of the application once the MeasurementMode class is initialized.
-   **Destruction:** Instances are reclaimed by the JVM when the class loader is unloaded, which typically occurs during application shutdown.

## Internal State & Concurrency
-   **State:** **Immutable**. Enum constants are singletons by definition and their state cannot be modified. They represent fixed, compile-time values.
-   **Thread Safety:** **Fully thread-safe**. As immutable singletons, instances of MeasurementMode can be safely passed between and accessed by multiple threads without any need for external synchronization or locking mechanisms.

## API Surface
The public contract of an enum consists of its defined constants.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| CENTRE_DISTANCE | MeasurementMode | O(1) | A constant representing a strategy to measure distance from a cell's central point. |
| BORDER_DISTANCE | MeasurementMode | O(1) | A constant representing a strategy to measure distance from a cell's nearest boundary. |

## Integration Patterns

### Standard Usage
MeasurementMode is used to configure procedural algorithms or to control branching logic based on the desired distance metric. It should be passed directly as a method parameter or used in a switch statement.

```java
// Example: Configuring a procedural noise generator
CellularNoiseGenerator generator = new CellularNoiseGenerator(seed);
generator.setDistanceMetric(MeasurementMode.BORDER_DISTANCE);
float noiseValue = generator.getNoise(x, y);
```

### Anti-Patterns (Do NOT do this)
-   **Unsafe Comparison:** Avoid comparing enum values using their string representation or ordinal value. This is brittle and can lead to subtle bugs if the enum definition is reordered or renamed. Always use direct object comparison or the equals method.
    -   **BAD:** `if (mode.toString().equals("CENTRE_DISTANCE")) { ... }`
    -   **BAD:** `if (mode.ordinal() == 0) { ... }`
    -   **GOOD:** `if (mode == MeasurementMode.CENTRE_DISTANCE) { ... }`
-   **Null References:** Functions accepting a MeasurementMode should be defensive against null inputs. A null mode will cause a NullPointerException in switch statements and can lead to undefined behavior in algorithms.

## Data Pipeline
MeasurementMode does not process data itself; it acts as a **control signal** that modifies the behavior of data processing components within a generation pipeline. It is a static input that governs transformation logic.

> Flow:
> **MeasurementMode (Configuration)** -> Procedural Algorithm (e.g., CellularNoiseGenerator) -> Distance Field (Data) -> Biome Placement Logic

In this flow, the selected MeasurementMode directly influences how the Procedural Algorithm calculates the intermediate Distance Field, which in turn determines the final output of the pipeline.

