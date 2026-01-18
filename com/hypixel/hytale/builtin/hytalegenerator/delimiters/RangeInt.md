---
description: Architectural reference for RangeInt
---

# RangeInt

**Package:** com.hypixel.hytale.builtin.hytalegenerator.delimiters
**Type:** Value Object / Utility

## Definition
```java
// Signature
public class RangeInt {
```

## Architecture & Concepts
The RangeInt class is a fundamental, immutable data structure that represents a half-open integer interval, mathematically denoted as `[min, max)`. This means the range includes the minimum value but excludes the maximum value.

Its primary role within the engine, particularly in procedural generation systems, is to provide a safe, reusable, and unambiguous way to define and check numerical boundaries. By encapsulating the range-checking logic (`min <= value < max`), it eliminates a common source of off-by-one errors that can arise from manual comparisons.

The design choice of immutability is critical. Once a RangeInt is constructed, its boundaries cannot be altered. This makes it inherently thread-safe and allows instances to be freely shared across concurrent systems, such as parallel world generation tasks, without requiring synchronization or defensive copies.

### Lifecycle & Ownership
- **Creation:** Instantiated directly via its public constructor (`new RangeInt(...)`). It is not managed by a service locator or dependency injection framework. Typically, it is created by configuration loaders or procedural algorithms that require boundary definitions.
- **Scope:** Transient. The lifetime of a RangeInt object is bound to the scope of the object that created it. It is a lightweight object intended for short-lived or member-variable scope.
- **Destruction:** Managed exclusively by the Java Garbage Collector. When all references to an instance are lost, it becomes eligible for collection. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, consisting of `minInclusive` and `maxExclusive`, is established at construction time via `final` fields. This state can never be modified post-instantiation.
- **Thread Safety:** **Guaranteed thread-safe**. Due to its immutable nature, a RangeInt instance can be read by any number of threads concurrently without risk of data corruption or race conditions. No external locking is required when sharing instances.

## API Surface
The public contract is minimal, focusing on construction, boundary retrieval, and containment checks.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| RangeInt(min, max) | constructor | O(1) | Constructs a new immutable range. |
| contains(value) | boolean | O(1) | Performs the primary boundary check. Returns true if the value is within `[min, max)`. |
| getMinInclusive() | int | O(1) | Returns the lower bound of the range. |
| getMaxExclusive() | int | O(1) | Returns the upper bound of the range. |

## Integration Patterns

### Standard Usage
The class is intended for direct instantiation and use in conditional logic, often for validating parameters or guiding procedural algorithms.

```java
// Define a valid altitude range for a specific biome
RangeInt mountainAltitude = new RangeInt(128, 256);

// Check if a generated point is within the valid range
int currentY = generator.getHeightAt(x, z);
if (mountainAltitude.contains(currentY)) {
    // Place mountain-specific blocks
}
```

### Anti-Patterns (Do NOT do this)
- **Repetitive Instantiation:** Avoid creating new RangeInt instances within tight loops if the range is constant. Its immutability allows a single instance to be created once and reused safely.

  ```java
  // BAD: A new object is allocated on every iteration
  for (int y = 0; y < 256; y++) {
      if (new RangeInt(64, 128).contains(y)) {
          // ...
      }
  }

  // GOOD: The object is created once and reused
  RangeInt waterLevel = new RangeInt(64, 128);
  for (int y = 0; y < 256; y++) {
      if (waterLevel.contains(y)) {
          // ...
      }
  }
  ```
- **Subclassing for Mutability:** Do not extend this class to add mutable behavior. The guarantee of immutability is a core design feature; violating it would break its thread-safety contract.

## Data Pipeline
RangeInt does not process data itself. Instead, it acts as a *predicate* or *constraint* within a larger data pipeline, typically filtering or directing a flow based on its boundaries.

> **Flow:**
> Generator Configuration -> **RangeInt** (as a constraint) -> Voxel Placement Algorithm -> World Data

