---
description: Architectural reference for DoubleRangeCoordinateHashSupplier
---

# DoubleRangeCoordinateHashSupplier

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Utility

## Definition
```java
// Signature
public class DoubleRangeCoordinateHashSupplier implements IDoubleCoordinateHashSupplier {
```

## Architecture & Concepts
The DoubleRangeCoordinateHashSupplier is a foundational component within the procedural generation framework. Its primary architectural role is to act as a deterministic *transformer*, converting spatial coordinates into a bounded, floating-point value suitable for world generation parameters like terrain height, temperature, or humidity.

It bridges the gap between a generic, uniform hash function (HashUtil) and a specific, context-dependent numerical range (IDoubleRange). This class embodies the **Strategy Pattern** by delegating two key responsibilities:
1.  **Hashing Strategy:** The core pseudo-random number generation is delegated to the static HashUtil utility.
2.  **Range Mapping Strategy:** The transformation of the generated random number into a specific range is delegated to the injected IDoubleRange instance.

This design makes the supplier highly composable. By injecting different IDoubleRange implementations, developers can reuse the same coordinate hashing logic to produce outputs for vastly different scales and distributions without modifying this class.

### Lifecycle & Ownership
-   **Creation:** Instantiated by higher-level procedural systems, such as a BiomeGenerator or FeaturePlacer, during their own initialization phase. It is always constructed with a concrete IDoubleRange implementation that defines its behavior.
-   **Scope:** The lifetime of a DoubleRangeCoordinateHashSupplier instance is typically scoped to its owning generator. It is a lightweight, transient object designed to be created, used for a generation task, and then discarded.
-   **Destruction:** The object is managed by the Java garbage collector. No explicit cleanup or resource release is necessary. It becomes eligible for collection as soon as its parent generator is dereferenced.

## Internal State & Concurrency
-   **State:** The internal state is defined by the final *range* field. As this field is marked `final` and set only during construction, the object is effectively **immutable**. Its behavior is fixed for its entire lifetime.
-   **Thread Safety:** This class is inherently **thread-safe**. It contains no mutable state and its operations are pure functions. It can be safely shared and accessed by multiple worker threads in a parallel world generation system without requiring any external synchronization or locks.

    **WARNING:** Thread safety is contingent on the injected IDoubleRange implementation also being thread-safe. Standard engine implementations of this interface are guaranteed to be safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y, hash) | double | O(1) | Calculates a deterministic hash for the coordinate and maps it to a double within the configured range. |

## Integration Patterns

### Standard Usage
This component is designed to be configured once and reused for many coordinate lookups. It is typically held as a field within a larger generator class.

```java
// 1. Define the target range for the output values.
IDoubleRange terrainHeightRange = new LinearDoubleRange(-20.0, 80.0);

// 2. Create the supplier, injecting the range strategy.
IDoubleCoordinateHashSupplier heightSupplier = new DoubleRangeCoordinateHashSupplier(terrainHeightRange);

// 3. Use the supplier within a generation loop.
for (int x = 0; x < 16; x++) {
    for (int y = 0; y < 16; y++) {
        double height = heightSupplier.get(worldSeed, chunkX * 16 + x, chunkY * 16 + y, TERRAIN_HASH);
        // ... use height value
    }
}
```

### Anti-Patterns (Do NOT do this)
-   **Re-instantiation in Loops:** Never create a new DoubleRangeCoordinateHashSupplier inside a tight loop. While lightweight, the object allocation overhead is unnecessary and impacts performance. Instantiate it once and reuse it.
-   **Null Dependency Injection:** The constructor does not perform null-safety checks on its arguments. Passing a null IDoubleRange will not fail at construction but will cause a NullPointerException when *get* is invoked. The calling system is responsible for providing a valid dependency.

## Data Pipeline
The class functions as a simple, single-stage processor in a larger data flow. It transforms a set of discrete integer inputs into a final, continuous double output.

> Flow:
> (seed, x, y, hash) -> HashUtil.random() -> **DoubleRangeCoordinateHashSupplier** -> IDoubleRange.getValue() -> Final double value

