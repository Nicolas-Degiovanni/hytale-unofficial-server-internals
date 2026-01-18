---
description: Architectural reference for DoubleThreshold.Single
---

# DoubleThreshold.Single

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Transient

## Definition
```java
// Signature
public static class Single implements IDoubleThreshold {
```

## Architecture & Concepts
The DoubleThreshold.Single class is a fundamental predicate component within the procedural generation library. It represents a single, continuous numerical range defined by a minimum and maximum value. Its primary function is to evaluate whether a given floating-point number falls within this specified interval.

This class is a core building block for creating conditional logic in world generation. It allows algorithms to make decisions based on continuous data inputs, such as noise values, temperature, humidity, or elevation. For example, a biome placement rule might use a DoubleThreshold.Single to determine if the local temperature value is suitable for a desert.

The class includes a mechanism to dynamically shrink the valid range towards its center using a *factor*. This is critical for creating layered or transitional effects, where a core region has different properties than its outer boundary.

## Lifecycle & Ownership
- **Creation:** Instances are created directly via their constructor, typically by higher-level procedural generation logic or deserialized from asset definitions (e.g., a biome configuration file). They are not managed by a central service.
- **Scope:** These are short-lived value objects. Their lifetime is scoped to the execution of a specific generation task that requires a conditional check.
- **Destruction:** Instances are eligible for garbage collection as soon as the reference is dropped, usually upon completion of the generation algorithm that created them.

## Internal State & Concurrency
- **State:** The state of a DoubleThreshold.Single instance is **immutable**. The *min* and *max* bounds are final fields set during construction. The *halfRange* field is a derived value, cached at construction for performance.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable state, a single instance can be safely shared and evaluated by multiple worker threads in a parallel generation system without requiring any synchronization or locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Single(min, max) | constructor | O(1) | Constructs a new threshold for the inclusive range [min, max]. |
| eval(d) | boolean | O(1) | Returns true if the value *d* is within the range [min, max]. |
| eval(d, factor) | boolean | O(1) | Returns true if *d* is within a range that has been symmetrically scaled inwards by the given *factor*. A factor of 0.0 uses the original [min, max] range. A factor of 1.0 shrinks the range to a single point at its center. |

## Integration Patterns

### Standard Usage
The class is used as a predicate in conditional logic. The factor-based evaluation is common for creating layered noise or falloff effects.

```java
// Example: Define a threshold for "temperate" zones
IDoubleThreshold temperateZone = new DoubleThreshold.Single(0.4, 0.7);

double currentTemperature = noise.get(x, y); // Value is between 0.0 and 1.0

// Check if the temperature is in the core temperate range
if (temperateZone.eval(currentTemperature, 0.5)) {
    // Place lush forest
} else if (temperateZone.eval(currentTemperature)) {
    // Place sparse woodland (transition zone)
}
```

### Anti-Patterns (Do NOT do this)
- **Invalid Range:** Constructing an instance where *min* is greater than *max* is a logical error. This will create a threshold that always evaluates to false, which may mask configuration issues.
- **Redundant Checks:** Avoid checking a value against multiple overlapping Single instances. If multiple ranges are required, compose them using DoubleThreshold.Multiple for clarity and correctness.

## Data Pipeline
This component acts as a gate or a decision point within a larger data flow, not as a data transformer.

> Flow:
> Generator Input (e.g., Noise Value) -> **DoubleThreshold.Single.eval()** -> Boolean Result -> Conditional Logic Branch (e.g., Place Biome A vs. Place Biome B)

---
---
description: Architectural reference for DoubleThreshold.Multiple
---

# DoubleThreshold.Multiple

**Package:** com.hypixel.hytale.procedurallib.condition
**Type:** Transient

## Definition
```java
// Signature
public static class Multiple implements IDoubleThreshold {
```

## Architecture & Concepts
The DoubleThreshold.Multiple class is a compositor for DoubleThreshold.Single objects. It represents a set of disjoint numerical ranges and evaluates to true if a given value falls within *any* of the contained ranges. Architecturally, it functions as a logical OR operator for one or more IDoubleThreshold conditions.

This component is essential for defining complex, multi-modal conditions in procedural generation. For example, a rule to place a specific ore might state that it can appear at low elevations (e.g., 0.0 to 0.2) *or* at very high elevations (e.g., 0.9 to 1.0), but not in between. DoubleThreshold.Multiple provides a clean and efficient way to express this compound rule.

It delegates all evaluation logic to its constituent Single instances, iterating through them until one returns true.

## Lifecycle & Ownership
- **Creation:** Instantiated directly via its constructor, taking an array of DoubleThreshold.Single objects. It is typically created by procedural algorithms or deserialized from complex asset definitions.
- **Scope:** Like its Single counterpart, this is a short-lived value object. Its lifetime is tied to the generation task that requires the compound condition.
- **Destruction:** Instances are marked for garbage collection once the generation algorithm completes and no longer holds a reference.

## Internal State & Concurrency
- **State:** The state is **immutable**. The internal array of Single thresholds is final and set at construction. Since the contained Single objects are also immutable, the entire object graph is constant.
- **Thread Safety:** This class is **thread-safe**. Its immutable nature and the thread-safety of its children allow it to be safely evaluated by multiple threads simultaneously without locks. This is critical for performance in parallelized world generation.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| Multiple(singles) | constructor | O(1) | Constructs a new composite threshold from an array of Single thresholds. |
| eval(d) | boolean | O(N) | Returns true if the value *d* falls within *any* of the contained Single ranges. N is the number of ranges. |
| eval(d, factor) | boolean | O(N) | Returns true if *d* falls within *any* of the contained Single ranges after they have been scaled by the given *factor*. |

## Integration Patterns

### Standard Usage
Use this class to combine multiple valid ranges into a single conditional check.

```java
// Example: A resource can spawn at low or high altitudes, but not mid-range.
DoubleThreshold.Single lowAltitude = new DoubleThreshold.Single(0.0, 0.2);
DoubleThreshold.Single highAltitude = new DoubleThreshold.Single(0.85, 1.0);

IDoubleThreshold spawnCondition = new DoubleThreshold.Multiple(
    new DoubleThreshold.Single[]{ lowAltitude, highAltitude }
);

double currentAltitude = noise.get(x, y);

if (spawnCondition.eval(currentAltitude)) {
    // Spawn the resource
}
```

### Anti-Patterns (Do NOT do this)
- **Single-Item Array:** Do not create a DoubleThreshold.Multiple with only one Single in its array. This adds unnecessary overhead and complexity. Use the DoubleThreshold.Single instance directly.
- **Empty Array:** Constructing with an empty or null array will likely lead to a NullPointerException or an object that always evaluates to false. The calling code should ensure the input array is valid and non-empty.
- **Overlapping Ranges:** While functionally correct, defining a Multiple with overlapping Single ranges (e.g., [0, 0.5] and [0.4, 0.8]) is inefficient and indicates a poorly defined condition. The ranges should be merged into a single range ([0, 0.8]) for clarity and performance.

## Data Pipeline
This component acts as a compound decision point, funneling a numerical input into a single boolean result based on multiple criteria.

> Flow:
> Generator Input (e.g., Elevation Value) -> **DoubleThreshold.Multiple.eval()** -> Boolean Result -> Conditional Logic Branch

