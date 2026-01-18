---
description: Architectural reference for Range
---

# Range

**Package:** com.hypixel.hytale.math
**Type:** Transient

## Definition
```java
// Signature
public class Range {
```

## Architecture & Concepts
The Range class is a fundamental value object used throughout the engine to represent a numerical interval between a minimum and maximum value. It is a core component of the mathematics package, designed for simplicity and performance in scenarios requiring boundary checks, random value generation within a scope, or configuration of parameters that accept a range of inputs.

Unlike service or manager classes, Range does not orchestrate behavior. Instead, it serves as a data carrier, providing a clear and explicit contract for any system that operates on numerical intervals. Common use cases include defining weapon damage, entity health scaling, procedural generation parameters, and time-based event triggers. Its primary architectural role is to reduce ambiguity by encapsulating the concept of an interval, replacing the use of primitive arrays or multiple disconnected variables.

## Lifecycle & Ownership
- **Creation:** Instances are created on-demand by any system requiring a numerical range. It is typically instantiated directly via its constructor, often during the deserialization of configuration files or when defining game logic constants.
- **Scope:** The lifetime of a Range object is strictly tied to its owner. It is a short-lived object whose scope is confined to the method or containing class instance that created it. It does not persist across major engine state changes unless it is part of a configuration object that does.
- **Destruction:** Instances are managed by the Java Garbage Collector. They are eligible for collection as soon as they are no longer referenced by any active part of the system.

## Internal State & Concurrency
- **State:** The class holds a mutable state consisting of two float primitives: min and max. Although the fields are not declared as final, the public API provides no methods to mutate the state after construction. Therefore, instances are **effectively immutable** under standard usage.
- **Thread Safety:** This class is **not thread-safe**. It provides no internal synchronization. Because it is designed as a lightweight value object, it should not be shared and mutated across different threads. The standard and expected pattern is to create a new instance for each required range rather than sharing a single mutable instance.

## API Surface
The public contract is minimal, focusing exclusively on construction and data retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Range() | constructor | O(1) | Creates a new Range with default float values (0.0f). |
| Range(float, float) | constructor | O(1) | Creates a new Range with the specified min and max values. |
| getMin() | float | O(1) | Returns the lower bound of the range. |
| getMax() | float | O(1) | Returns the upper bound of the range. |

## Integration Patterns

### Standard Usage
The class is intended for direct instantiation and use as a data parameter or field within other components.

```java
// Define a damage range for a weapon configuration
Range swordDamage = new Range(10.5f, 15.0f);

// Use the range in game logic to calculate a random damage value
float actualDamage = calculateRandomIn(swordDamage);
```

### Anti-Patterns (Do NOT do this)
- **Invalid State:** The constructor does not validate that min is less than or equal to max. Creating a range such as `new Range(100f, 50f)` is possible and will lead to undefined behavior in systems that consume it. Consuming code must be resilient to this possibility.
- **State Assumption:** Do not assume the internal state is immutable. While the current API prevents modification, a future change or subclass could introduce mutability. Treat it as a simple data record for the duration of a single operation.
- **Floating Point Precision:** Be aware of standard floating-point inaccuracies when performing comparisons with min and max values.

## Data Pipeline
As a value object, Range does not process data. Instead, it is the data that flows through a pipeline. It typically originates from a static or deserialized source and is consumed by game logic.

> Flow:
> Game Configuration (JSON/YAML) -> Deserializer -> **Range** instance -> Game System (e.g., CombatService) -> Logic Execution

