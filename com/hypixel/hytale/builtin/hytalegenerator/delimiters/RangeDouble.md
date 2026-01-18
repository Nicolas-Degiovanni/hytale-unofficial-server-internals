---
description: Architectural reference for RangeDouble
---

# RangeDouble

**Package:** com.hypixel.hytale.builtin.hytalegenerator.delimiters
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class RangeDouble {
```

## Architecture & Concepts
The RangeDouble class is a fundamental, immutable value object designed to represent a half-open mathematical interval of the form `[min, max)`. Its primary role within the engine is to provide a safe, reusable, and self-documenting mechanism for boundary checks, particularly within procedural content generation (PCG) systems like the Hytale Generator.

This class encapsulates the logic for determining if a given double-precision floating-point number falls within a specified range. By centralizing this logic, it eliminates the risk of common off-by-one errors and inconsistencies that arise from scattered, inline boundary checks (`value >= min && value < max`). Its immutability makes it an ideal candidate for use in configuration objects, constants, and multi-threaded generation contexts where predictable behavior is paramount.

## Lifecycle & Ownership
- **Creation:** Instances are created directly by the consumer via the public constructor: `new RangeDouble(min, max)`. There is no factory or manager responsible for its instantiation. Ownership belongs entirely to the creating object.
- **Scope:** Transient. The lifecycle of a RangeDouble instance is tied to the object that holds a reference to it. It is typically short-lived, often created as a local variable for a specific calculation or held as a final field within a larger configuration or rule object.
- **Destruction:** The object is eligible for garbage collection as soon as it is no longer referenced. It holds no native resources and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** **Immutable**. The internal state, consisting of `minInclusive` and `maxExclusive`, is established at construction time and cannot be modified thereafter. All fields are declared as private and final.
- **Thread Safety:** **Inherently thread-safe**. Due to its immutable design, a RangeDouble instance can be safely shared across multiple threads without any external synchronization or locking mechanisms. This is a critical feature for parallelized world generation tasks.

## API Surface
The public contract is minimal and focused exclusively on range evaluation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| contains(double value) | boolean | O(1) | Evaluates if the provided value is within the `[min, max)` interval. This is the primary operation. |
| min() | double | O(1) | Returns the inclusive lower bound of the range. |
| max() | double | O(1) | Returns the exclusive upper bound of the range. |

## Integration Patterns

### Standard Usage
RangeDouble is intended to be used as a predicate in conditional logic, typically within generation algorithms to map continuous values (like noise) to discrete outcomes (like block types or biome selection).

```java
// Example: Determining terrain layer based on elevation noise
// This range is defined once and reused for many checks.
RangeDouble grassLayerRange = new RangeDouble(0.5, 0.75);

double currentElevation = noise.get(x, z);

if (grassLayerRange.contains(currentElevation)) {
    // Place grass block
}
```

### Anti-Patterns (Do NOT do this)
- **Redundant Instantiation:** Avoid creating new RangeDouble instances inside performance-critical loops. This generates unnecessary garbage collector pressure. Instantiate the range once outside the loop and reuse it.

    ```java
    // BAD: Creates a new object every iteration
    for (int i = 0; i < 10000; i++) {
        if (new RangeDouble(0.0, 1.0).contains(values[i])) {
            // ...
        }
    }

    // GOOD: Single instance is reused
    RangeDouble checkRange = new RangeDouble(0.0, 1.0);
    for (int i = 0; i < 10000; i++) {
        if (checkRange.contains(values[i])) {
            // ...
        }
    }
    ```

- **Boundary Misinterpretation:** The upper bound is *exclusive*. Code that assumes an inclusive upper bound will produce incorrect results. For example, `new RangeDouble(0.0, 100.0).contains(100.0)` will always return false.

## Data Pipeline
As a low-level utility, RangeDouble does not manage a data pipeline itself. Instead, it acts as a predicate or filter *within* a larger pipeline, transforming a continuous input value into a discrete boolean decision.

> Flow:
> Generator Input (e.g., Noise Value) -> **RangeDouble.contains()** -> Boolean Result -> Conditional System Logic (e.g., Biome Selector)

