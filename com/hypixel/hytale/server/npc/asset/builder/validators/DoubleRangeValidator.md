---
description: Architectural reference for DoubleRangeValidator
---

# DoubleRangeValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class DoubleRangeValidator extends DoubleValidator {
```

## Architecture & Concepts
The DoubleRangeValidator is a specialized, immutable component within the server's asset validation framework. Its primary function is to enforce numerical constraints on floating-point values, typically those deserialized from NPC configuration files.

Architecturally, this class serves as a concrete implementation of a validation rule. It extends the base DoubleValidator, narrowing its focus to range-based comparisons. The design strongly favors immutability and clarity through the use of the **Factory Method** pattern. By providing static methods like `between` and `fromExclToExcl`, it offers an expressive, domain-specific language for defining validation boundaries, abstracting away the underlying relational operators.

This component is critical for ensuring data integrity during asset loading, preventing corrupt or nonsensical values (e.g., negative health, probabilities outside [0, 1]) from propagating into the game simulation.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively via the public static factory methods (`between`, `between01`, etc.). The constructor is private to enforce this pattern. A special-case singleton, `VALIDATOR_BETWEEN_01`, is pre-allocated for the extremely common 0-to-1 validation, optimizing for performance and memory usage.
-   **Scope:** Instances are typically transient and short-lived. They are created on-demand by an asset builder or loader, used to perform one or more validation checks, and are then eligible for garbage collection. They do not hold references to any engine services or heavy resources.
-   **Destruction:** Cleanup is handled automatically by the Java Virtual Machine's garbage collector. No manual destruction or resource release is necessary.

## Internal State & Concurrency
-   **State:** **Immutable**. All internal fields defining the range (`lower`, `upper`) and the relational operators are `private final` and are set only once during construction. An instance of DoubleRangeValidator cannot be modified after it is created.

-   **Thread Safety:** **Unconditionally thread-safe**. Its immutable nature guarantees that instances can be safely shared, published, and used across any number of threads without external synchronization. The `test` method is a pure function, producing output based solely on its inputs with no side effects.

## API Surface
The public API is composed entirely of factory methods for instantiation and the core `test` method for execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| between01() | DoubleRangeValidator | O(1) | Returns a shared, static instance for validating a value is within the inclusive range [0.0, 1.0]. |
| between(lower, upper) | DoubleRangeValidator | O(1) | Creates a validator for the inclusive range [lower, upper]. |
| fromExclToIncl(lower, upper) | DoubleRangeValidator | O(1) | Creates a validator for the range (lower, upper]. |
| fromExclToExcl(lower, upper) | DoubleRangeValidator | O(1) | Creates a validator for the exclusive range (lower, upper). |
| test(value) | boolean | O(1) | Executes the validation logic, returning true if the value satisfies both range constraints. |
| errorMessage(value, name) | String | O(1) | Generates a detailed, human-readable error message for a failed validation. |

## Integration Patterns

### Standard Usage
This validator is intended to be used within asset loading and building pipelines to guard against invalid data.

```java
// Example within an NPC asset builder
DoubleRangeValidator speedValidator = DoubleRangeValidator.fromExclToExcl(0.0, 50.0);
double rawSpeed = npcDefinition.getMovementSpeed();

if (!speedValidator.test(rawSpeed)) {
    // Halt asset creation and report a clear error
    throw new AssetValidationException(
        speedValidator.errorMessage(rawSpeed, "MovementSpeed")
    );
}

// If validation passes, proceed with the value
npc.setMovementSpeed(rawSpeed);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to use reflection to call the private constructor. The static factory methods are the sole entry point for creating instances.
-   **Redundant Creation:** For the common [0, 1] range, always use the `between01()` factory. Calling `DoubleRangeValidator.between(0.0, 1.0)` creates a new, unnecessary object, defeating the purpose of the pre-allocated singleton.

## Data Pipeline
The DoubleRangeValidator acts as a gate within the broader asset ingestion pipeline. It does not transform data but rather asserts its correctness.

> Flow:
> NPC Asset File (e.g., JSON) -> Deserializer -> Raw Data Object -> Asset Builder -> **DoubleRangeValidator.test()** -> [Success] Validated Game Object OR [Failure] AssetValidationException

