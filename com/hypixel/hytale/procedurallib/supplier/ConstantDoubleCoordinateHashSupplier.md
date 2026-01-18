---
description: Architectural reference for ConstantDoubleCoordinateHashSupplier
---

# ConstantDoubleCoordinateHashSupplier

**Package:** com.hypixel.hytale.procedurallib.supplier
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class ConstantDoubleCoordinateHashSupplier implements IDoubleCoordinateHashSupplier {
```

## Architecture & Concepts
The ConstantDoubleCoordinateHashSupplier is a foundational component within the procedural generation library. It serves as the simplest possible implementation of the IDoubleCoordinateHashSupplier interface, designed to return a fixed, non-varying double value regardless of input coordinates or seeds.

Architecturally, this class represents a **leaf node** or a **constant source** within a larger procedural generation graph or expression tree. It is used in scenarios where a static value is required to drive an algorithm, such as:
*   Providing a baseline or floor value for terrain height.
*   Acting as a fixed blend factor between two other noise suppliers.
*   Serving as a default or fallback supplier in a composite generator.
*   Debugging or testing procedural systems by injecting a known, predictable value.

Its primary design goal is to provide a deterministic, computationally inexpensive source of data that is completely independent of spatial context.

### Lifecycle & Ownership
- **Creation:** Instances are created directly via the public constructor `new ConstantDoubleCoordinateHashSupplier(value)`. For the common values of 0.0 and 1.0, the static final instances ZERO and ONE should be used to prevent unnecessary object allocation. These objects are typically instantiated during the configuration phase of a world generator or other procedural system.
- **Scope:** The scope is determined by its owner. When used as part of a generator's configuration, it lives as long as the generator itself. The static instances ZERO and ONE are globally scoped and persist for the entire application lifetime.
- **Destruction:** As a simple Java object with no external resources, instances are managed entirely by the Garbage Collector. Destruction is non-deterministic and occurs after the object is no longer referenced.

## Internal State & Concurrency
- **State:** **Immutable**. The core state is a single final double field named result, which is set at construction time and cannot be changed thereafter. This immutability guarantees that an instance will behave identically throughout its lifecycle.
- **Thread Safety:** **Fully thread-safe**. Due to its immutable design, an instance can be safely shared across multiple threads without any risk of data corruption or race conditions. No external locking or synchronization is required. This property is critical for high-performance, parallelized procedural generation tasks.

## API Surface
The public contract is minimal and focused on fulfilling the IDoubleCoordinateHashSupplier interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(seed, x, y, hash) | double | O(1) | Returns the constant double value stored during construction. All input parameters are ignored. |
| getResult() | double | O(1) | A direct accessor for the stored constant value. |

## Integration Patterns

### Standard Usage
This supplier is used to inject a fixed value into a system that expects a dynamic, coordinate-aware supplier.

```java
// Standard Usage: Create a supplier that always returns 0.75
IDoubleCoordinateHashSupplier fixedHeightSupplier = new ConstantDoubleCoordinateHashSupplier(0.75);

// Optimized Usage: Use the static instance for common values
IDoubleCoordinateHashSupplier floorSupplier = ConstantDoubleCoordinateHashSupplier.ZERO;

// In a procedural context, the value is always the same, regardless of input
double val1 = fixedHeightSupplier.get(12345, 100, 250, 987L); // Returns 0.75
double val2 = fixedHeightSupplier.get(12345, -50, 300, 654L); // Returns 0.75
```

### Anti-Patterns (Do NOT do this)
- **Redundant Instantiation:** Do not create new instances for common values like 0.0 or 1.0. This creates unnecessary object churn.
    ```java
    // AVOID
    IDoubleCoordinateHashSupplier badZero = new ConstantDoubleCoordinateHashSupplier(0.0);

    // PREFER
    IDoubleCoordinateHashSupplier goodZero = ConstantDoubleCoordinateHashSupplier.ZERO;
    ```
- **Misuse in Dynamic Systems:** Do not use this class in a context where a procedurally generated, spatially-varying, or random value is expected. It is, by design, static and predictable. Using it in place of a noise function will result in flat, uniform output.

## Data Pipeline
This class acts as a **source node** in a data pipeline. It does not process incoming data; it originates a constant stream of data.

> Flow:
> Procedural System Request (with seed, x, y) -> **ConstantDoubleCoordinateHashSupplier.get()** -> Returns pre-configured constant double value

