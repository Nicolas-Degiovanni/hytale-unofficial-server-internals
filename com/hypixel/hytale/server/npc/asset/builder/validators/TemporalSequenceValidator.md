---
description: Architectural reference for TemporalSequenceValidator
---

# TemporalSequenceValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class TemporalSequenceValidator extends TemporalArrayValidator {
```

## Architecture & Concepts

The TemporalSequenceValidator is a specialized, stateless validation component used within the server's NPC asset building system. Its primary function is to enforce complex rules on ordered collections of time-based values, ensuring data integrity for features like NPC schedules, seasonal behaviors, or event timelines defined in asset files.

Architecturally, this validator operates on a crucial abstraction: it treats all `TemporalAmount` instances (such as `Period` or `Duration`) as offsets relative to a fixed, global server epoch defined by `WorldTimeResource.ZERO_YEAR`. This design choice normalizes disparate time definitions into a consistent, comparable format based on `LocalDateTime`.

The validator simultaneously enforces three distinct constraints on an input array:

1.  **Boundary Constraints:** Every value in the sequence must fall within a configured lower and upper time bound.
2.  **Sequential Constraints:** Each value in the sequence must have a specific relationship to the preceding value (e.g., strictly increasing for monotonic sequences).
3.  **Type Homogeneity:** All `TemporalAmount` objects within the array must be of the same concrete type. It strictly forbids mixing `Period` and `Duration` instances, which represent different levels of precision and could lead to ambiguous comparisons.

## Lifecycle & Ownership

-   **Creation:** Instances are created exclusively through the provided static factory methods, such as `betweenMonotonic` or `betweenWeaklyMonotonic`. These are typically invoked by higher-level asset parsers or builder classes when interpreting validation rules from an asset definition.
-   **Scope:** Transient and short-lived. A validator instance is created to perform a specific validation task on a single asset and is discarded immediately after. It holds no long-term state and is not shared across different asset builds.
-   **Destruction:** The object is eligible for garbage collection as soon as the `test` method returns and its result has been consumed. No manual cleanup is required.

## Internal State & Concurrency

-   **State:** The validator's state is immutable. The boundary values and relational operators are provided at creation time via the private constructor and are never modified thereafter.
-   **Thread Safety:** This class is inherently thread-safe. Its immutable nature guarantees that multiple threads can invoke the `test` method on a shared instance without risk of data corruption or race conditions.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| betweenMonotonic(lower, upper) | TemporalSequenceValidator | O(1) | Creates a validator for a strictly increasing sequence within the specified bounds. |
| betweenWeaklyMonotonic(lower, upper) | TemporalSequenceValidator | O(1) | Creates a validator for a non-decreasing sequence within the specified bounds. |
| test(TemporalAmount[] values) | boolean | O(N) | Executes the validation logic against the provided array. Returns true if all constraints are met. |
| errorMessage(name, value) | String | O(1) | Generates a detailed, human-readable error message explaining which constraints failed. |

## Integration Patterns

### Standard Usage

This validator is intended to be used as part of a larger data validation pipeline, typically during server startup or when live-reloading game assets.

```java
// In a hypothetical AssetBuilder class
TemporalAmount[] scheduleTimes = asset.getScheduleTimes(); // e.g., [P1D, P3D, P10D]

// Define a rule: times must be between 1 day and 30 days, and strictly increasing.
TemporalSequenceValidator validator = TemporalSequenceValidator.betweenMonotonic(
    Period.ofDays(1),
    Period.ofDays(30)
);

if (!validator.test(scheduleTimes)) {
    throw new AssetValidationException(
        validator.errorMessage("scheduleTimes", scheduleTimes)
    );
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** The constructor is private and inaccessible. Always use the static factory methods to ensure the validator's internal state is configured correctly.
-   **Mixed Temporal Types:** Do not pass an array containing both `Period` and `Duration` objects. The validator will correctly reject this, but it signals a critical data modeling error in the source asset file that must be fixed.
-   **Ignoring Global Epoch:** All validation is relative to `WorldTimeResource.ZERO_YEAR`. Any system interacting with this validator must operate under the same assumption. Modifying this global constant without a full system audit will lead to unpredictable validation results.

## Data Pipeline

The validator acts as a gate in the asset loading data pipeline. It consumes a deserialized data structure and produces a binary pass/fail signal.

> Flow:
> NPC Asset File (JSON/YAML) -> Asset Deserializer -> `TemporalAmount[]` -> **TemporalSequenceValidator.test()** -> Boolean Result -> Asset Build Engine

