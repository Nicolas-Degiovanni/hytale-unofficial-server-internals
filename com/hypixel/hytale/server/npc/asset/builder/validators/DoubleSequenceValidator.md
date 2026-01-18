---
description: Architectural reference for DoubleSequenceValidator
---

# DoubleSequenceValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class DoubleSequenceValidator extends DoubleArrayValidator {
```

## Architecture & Concepts
The DoubleSequenceValidator is a specialized, immutable component within the server's asset validation framework. Its primary function is to enforce complex numerical constraints on sequences of floating-point numbers, typically found in NPC asset definitions for properties like animation keyframe timings or probability distributions.

This class acts as a rule engine for one-dimensional double arrays. It decouples the validation logic from the asset parsing and loading systems. Instead of embedding `if/else` checks directly within asset builders, developers use this class to define a validation rule and apply it to the data.

Architecturally, it employs a **Factory Pattern** with a private constructor. This design has two key benefits:
1.  It provides a set of pre-configured, static instances for common validation scenarios (e.g., a sequence of probabilities between 0.0 and 1.0), promoting reuse and reducing memory overhead.
2.  It exposes a clear, descriptive API through static methods like `betweenMonotonic` or `fromExclToIncl`, making the intent of the validation rule explicit and improving code readability.

## Lifecycle & Ownership
-   **Creation:** Instances are never created directly using `new`. They are exclusively obtained through the provided static factory methods, such as `between01()` or `between(double, double)`. The system maintains a pool of static, final instances for the most common validation rules.
-   **Scope:** These validator objects are stateless and immutable. Their lifetime is determined entirely by their consumer. They can be safely stored as static members in a larger system, created on-the-fly for a single validation check, or held as an instance field within an asset builder.
-   **Destruction:** Lifecycle is managed by the Java Garbage Collector. As these objects are lightweight and contain only a few primitive fields, their creation and destruction impose negligible performance impact.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields defining the validation rules (e.g., `lower`, `upper`, `relationSequence`) are `private final` and are set only once during construction. An instance represents a fixed, unchangeable validation contract.
-   **Thread Safety:** **Fully thread-safe**. Its immutable nature guarantees that a single DoubleSequenceValidator instance can be shared and executed concurrently across multiple threads without any risk of race conditions or need for external synchronization. This is a critical feature for parallelized asset loading on the server.

## API Surface
The public API consists of static factory methods for instantiation and instance methods for performing validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(double[] values) | boolean | O(N) | Executes the validation logic against the provided array. Returns true if all rules pass. |
| errorMessage(double[] value, String name) | String | O(N) | Generates a detailed, human-readable error message if validation fails. |
| between01() | DoubleSequenceValidator | O(1) | Factory method. Returns a shared instance that validates if values are between 0.0 and 1.0. |
| between(double lower, double upper) | DoubleSequenceValidator | O(1) | Factory method. Creates a validator for a custom inclusive range. |
| betweenMonotonic(double lower, double upper) | DoubleSequenceValidator | O(1) | Factory method. Creates a validator for a range that must also be strictly increasing. |
| weaklyMonotonic() | DoubleSequenceValidator | O(1) | Factory method. Returns a shared instance that validates if a sequence is non-decreasing. |

## Integration Patterns

### Standard Usage
The validator is typically used within an asset builder or loader immediately after deserializing data. The goal is to fail fast if the asset data violates its contract.

```java
// Example: Validating keyframe timings in an NPC animation asset
DoubleSequenceValidator timingValidator = DoubleSequenceValidator.between01WeaklyMonotonic();
double[] keyframeTimings = myNpcAsset.getAnimationTimings();

if (!timingValidator.test(keyframeTimings)) {
    // Halt asset loading and report a clear error
    throw new AssetParseException(timingValidator.errorMessage(keyframeTimings, "Keyframe Timings"));
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Attempting to call `new DoubleSequenceValidator()` will result in a compile-time error. This is by design. Always use the static factory methods.
-   **Redundant Creation in Loops:** Avoid creating the same custom validator repeatedly inside a loop. Instantiate it once and reuse it.

    ```java
    // BAD: Creates a new object on every iteration
    for (NpcAnimation anim : animations) {
        DoubleSequenceValidator validator = DoubleSequenceValidator.between(0, anim.getDuration());
        if (!validator.test(anim.getTimings())) {
            // ...
        }
    }

    // GOOD: Reuses the same validator instance when the rule is the same
    DoubleSequenceValidator validator = DoubleSequenceValidator.between01();
    for (NpcAnimation anim : animations) {
        if (!validator.test(anim.getNormalizedTimings())) {
            // ...
        }
    }
    ```
-   **Ignoring Error Messages:** The `errorMessage` method provides critical context for asset creators. Do not just return a generic "Validation Failed" error; always propagate the detailed message from the validator.

## Data Pipeline
The DoubleSequenceValidator acts as a gate or filter within the broader asset loading data pipeline. It consumes a raw data structure and either allows it to pass through to the next stage or terminates the pipeline with a descriptive error.

> Flow:
> NPC Asset File (e.g., JSON) -> Deserializer -> Raw `double[]` -> **DoubleSequenceValidator** -> (If Valid) -> Engine-Ready Game Object / (If Invalid) -> Asset Load Exception -> Server Log

