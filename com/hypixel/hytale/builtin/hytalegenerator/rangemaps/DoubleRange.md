---
description: Architectural reference for DoubleRange
---

# DoubleRange

**Package:** com.hypixel.hytale.builtin.hytalegenerator.rangemaps
**Type:** Utility / Value Object

## Definition
```java
// Signature
public class DoubleRange {
```

## Architecture & Concepts
The DoubleRange class is a foundational, immutable value object used to represent a mathematical range of double-precision floating-point numbers. It serves as a core component within the world generation system, providing a robust and error-resistant mechanism for defining and checking numerical constraints.

Its primary architectural role is to encapsulate the logic of range validation, including boundary inclusivity. By centralizing this logic, it prevents the proliferation of manual, error-prone range checks (e.g., `value >= min && value < max`) throughout the generator codebase. This pattern is critical for defining constraints on environmental parameters such as biome temperature, elevation, humidity, or the distribution density of ores and other resources.

As a pure value object, it carries no dependencies and has no side effects, making it a highly predictable and reusable building block for complex procedural generation algorithms.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand by systems requiring range-based logic. Creation occurs either through the primary constructor for custom inclusivity or, more commonly, via the static factory methods DoubleRange.inclusive and DoubleRange.exclusive. This class is not managed by a service locator or dependency injection framework.
- **Scope:** The lifetime of a DoubleRange instance is typically transient and confined to the scope of the algorithm in which it is used. It is frequently created as a local variable within a method and becomes eligible for garbage collection once the method completes.
- **Destruction:** Cleanup is handled automatically by the Java Garbage Collector. No manual memory management is required.

## Internal State & Concurrency
- **State:** DoubleRange is **immutable**. Its internal state, consisting of the minimum and maximum bounds and their inclusivity flags, is set exclusively at construction time. There are no methods to modify the state of an instance after it has been created.
- **Thread Safety:** The immutable nature of this class makes it inherently **thread-safe**. A single instance can be safely shared and read by multiple threads concurrently without the need for external synchronization or locks. This is a critical feature for parallelized world generation tasks.

## API Surface
The public contract is minimal, focusing on creation and validation. Simple getters are omitted for brevity.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| includes(double v) | boolean | O(1) | The primary operational method. Returns true if the provided value falls within the defined range, respecting the inclusivity rules. |
| inclusive(double min, double max) | static DoubleRange | O(1) | Factory method for creating a range where both the minimum and maximum values are inclusive. Throws IllegalArgumentException if min > max. |
| exclusive(double min, double max) | static DoubleRange | O(1) | Factory method for creating a range where both the minimum and maximum values are exclusive. Throws IllegalArgumentException if min > max. |

## Integration Patterns

### Standard Usage
The intended use is to define a constraint and then test values against it. The static factory methods are the preferred means of instantiation for common use cases.

```java
// Define a valid humidity range for a swamp biome.
// The range is [0.75, 1.0].
DoubleRange swampHumidity = DoubleRange.inclusive(0.75, 1.0);

// Check if a procedurally generated humidity value is suitable.
double generatedHumidity = noise.getHumidityValue(x, z);
if (swampHumidity.includes(generatedHumidity)) {
    // Place swamp biome block
}
```

### Anti-Patterns (Do NOT do this)
- **Manual Range Checking:** Avoid re-implementing the range check manually. This defeats the purpose of the class and can lead to subtle bugs related to floating-point comparisons and inclusivity rules.
  ```java
  // BAD: Error-prone and ignores inclusivity settings
  if (value >= myRange.getMin() && value <= myRange.getMax()) {
      // ...
  }

  // GOOD: Correct and respects object's contract
  if (myRange.includes(value)) {
      // ...
  }
  ```
- **Ignoring Construction Exceptions:** The constructor and factory methods will throw an IllegalArgumentException if the minimum value is greater than the maximum. This is a programming error and should not be caught and ignored; the input data or logic should be corrected.

## Data Pipeline
DoubleRange is not a pipeline stage itself, but rather a predicate function used *within* a stage to filter or direct data flow. It typically acts as a gate in a procedural generation pipeline.

> Flow:
> Noise Function → Raw Parameter (e.g., elevation) → **DoubleRange.includes()** → Boolean Decision → Biome Selection Logic → Final Block Placement

