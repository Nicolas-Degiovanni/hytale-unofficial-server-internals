---
description: Architectural reference for ConstantCoordinateDoubleSupplier
---

# ConstantCoordinateDoubleSupplier

**Package:** com.hypixel.hytale.server.worldgen.util.function
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class ConstantCoordinateDoubleSupplier implements ICoordinateDoubleSupplier {
```

## Architecture & Concepts
The ConstantCoordinateDoubleSupplier is a foundational component within the server-side world generation framework. It serves as a concrete implementation of the ICoordinateDoubleSupplier interface, designed to return a fixed, unchanging double value regardless of the input coordinates or seed.

Its primary architectural role is to act as a **terminating node** or **static input** in complex, procedural generation graphs. Many world generation algorithms are designed to accept functional inputs (suppliers) that can produce dynamic, noise-based, or otherwise variable values depending on world position. This class provides a mechanism to inject a simple, constant value into these systems without requiring special-cased logic in the consuming algorithm.

It effectively implements a null-object pattern for coordinate-based functions, ensuring that systems requiring a functional dependency can be satisfied with a non-functional, static value. This simplifies the design of higher-level generators, which can treat all numerical inputs uniformly as suppliers.

### Lifecycle & Ownership
- **Creation:** Instances are created directly via the public constructor, typically during the configuration phase of a larger world generation system. For the common values of 0.0 and 1.0, the static final instances DEFAULT_ZERO and DEFAULT_ONE should be used to prevent unnecessary object allocation.
- **Scope:** The lifetime of an instance is bound to the object that holds a reference to it, such as a biome configuration or a feature generator profile. As a lightweight value object, it is considered transient. The static instances persist for the entire application lifetime.
- **Destruction:** Instances are eligible for garbage collection as soon as they are no longer referenced by any active world generation configuration.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state consists of a single `final double` field, which is initialized at construction and can never be modified. This guarantees that an instance will behave identically throughout its entire lifecycle.
- **Thread Safety:** **Unconditionally thread-safe**. Due to its immutable nature, a ConstantCoordinateDoubleSupplier instance can be safely shared and accessed concurrently by multiple world generation threads without any external synchronization or locking. This is a critical property for enabling parallelized chunk generation.

## API Surface
The public contract is minimal, focused entirely on fulfilling the ICoordinateDoubleSupplier interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| ConstantCoordinateDoubleSupplier(double) | constructor | O(1) | Creates a new supplier that will always return the given value. |
| getValue() | double | O(1) | Returns the constant value stored by this instance. |
| apply(seed, x, y) | double | O(1) | Ignores all input parameters and returns the configured constant value. |
| apply(seed, x, y, z) | double | O(1) | Ignores all input parameters and returns the configured constant value. |

## Integration Patterns

### Standard Usage
This class is used to supply a fixed value to a system that expects a dynamic, coordinate-aware supplier. This is common when setting base values for terrain, temperature, or other environmental parameters.

```java
// Example: Configuring a biome's base terrain height to a fixed value.
BiomeConfig biome = new BiomeConfig();

// The setBaseHeight method expects an ICoordinateDoubleSupplier, but for this
// flat biome, we need a constant height of 80.
biome.setBaseHeight(new ConstantCoordinateDoubleSupplier(80.0));

// For zero-value properties, always use the static singleton to improve performance.
biome.setErosionFactor(ConstantCoordinateDoubleSupplier.DEFAULT_ZERO);
```

### Anti-Patterns (Do NOT do this)
- **Redundant Instantiation:** Avoid creating new instances for common values like 0.0 or 1.0. This creates unnecessary object churn and bypasses the provided static instances.
  ```java
  // BAD: Creates a new object for no reason.
  feature.setDensity(new ConstantCoordinateDoubleSupplier(0.0));

  // GOOD: Reuses the existing, static instance.
  feature.setDensity(ConstantCoordinateDoubleSupplier.DEFAULT_ZERO);
  ```
- **Conditional Logic:** Do not write code that checks `instanceof ConstantCoordinateDoubleSupplier` to perform special logic. The purpose of this class is to allow consuming systems to be ignorant of the supplier's implementation. If you need conditional logic, it should be implemented in a different, higher-level supplier.

## Data Pipeline
The ConstantCoordinateDoubleSupplier acts as a data **source** within the world generation pipeline. It does not process or transform incoming data; it simply injects a pre-configured value into the system upon request.

> Flow:
> WorldGen Executor (for block at x,y,z) -> Requests value from `ICoordinateDoubleSupplier` -> **ConstantCoordinateDoubleSupplier**.apply(seed,x,y,z) -> Returns pre-defined double -> Executor uses value

---

