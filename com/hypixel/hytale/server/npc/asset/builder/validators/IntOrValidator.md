---
description: Architectural reference for IntOrValidator
---

# IntOrValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Transient Utility

## Definition
```java
// Signature
public class IntOrValidator extends IntValidator {
```

## Architecture & Concepts
The IntOrValidator is a specialized component within the server's asset validation framework. It is designed to enforce a compound validation rule on an integer field, specifically checking if the value satisfies one of two distinct relational conditions (e.g., `value > 0` OR `value == -1`).

Architecturally, this class implements the **Specification Pattern**. It encapsulates a specific piece of business logic—a compound conditional check—into a standalone, reusable, and testable object. This decouples the validation rule itself from the entity being validated, allowing for flexible and declarative construction of complex validation chains within the NPC asset building pipeline.

Its primary role is to provide a clear, immutable definition for a validation rule that can be applied by a higher-level service, such as an AssetBuilder, during the deserialization and verification of NPC configuration files.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively through public static factory methods, such as `greater0OrMinus1`. The constructor is private to enforce controlled instantiation and promote the use of pre-configured, common validators. It is typically instantiated on-demand by an asset builder or parser when a field requiring this specific validation logic is encountered.

- **Scope:** Extremely short-lived. An IntOrValidator instance is transient and its lifetime is typically confined to the scope of a single asset's validation process. It holds no persistent state beyond its initial configuration.

- **Destruction:** The object is lightweight and becomes eligible for garbage collection as soon as the validation operation completes and all references to it are dropped. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields defining the two relational checks are `private final` and are set once during construction. Once an IntOrValidator is created, its validation logic is fixed and cannot be altered.

- **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, a single IntOrValidator instance can be safely shared and executed by multiple threads concurrently without any risk of race conditions or need for external synchronization.

## API Surface
The public contract is minimal, focusing entirely on object creation and rule execution.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| greater0OrMinus1() | IntOrValidator | O(1) | Static factory method. Creates a validator that checks if a value is greater than 0 OR equal to -1. |
| test(int value) | boolean | O(1) | Executes the core validation logic. Returns true if the input value satisfies either of the two configured conditions. |
| errorMessage(int value, String name) | String | O(1) | Generates a formatted, human-readable error message describing why the validation failed. |

## Integration Patterns

### Standard Usage
The IntOrValidator is intended to be used by a higher-level system responsible for processing data structures, such as an asset builder. The builder retrieves a validator instance and applies it to a corresponding data field.

```java
// Hypothetical usage within an AssetBuilder
// The validator is created once and used to test a field.

IntOrValidator durationValidator = IntOrValidator.greater0OrMinus1();
int npcEffectDuration = npcData.getEffectDuration(); // e.g., from a config file

if (!durationValidator.test(npcEffectDuration)) {
    String error = durationValidator.errorMessage(npcEffectDuration, "Effect Duration");
    errorCollector.add(error);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to instantiate this class using reflection. The private constructor is a deliberate design choice. Always use the provided static factory methods.

- **Redundant Creation:** While instantiation is cheap, avoid creating new instances of the same validator within a tight loop. If validating a large list of items against the same rule, create the validator once and reuse the instance.

    ```java
    // BAD: New instance created on every iteration
    for (int duration : allDurations) {
        if (!IntOrValidator.greater0OrMinus1().test(duration)) {
            // ... handle error
        }
    }

    // GOOD: Instance is reused
    IntOrValidator validator = IntOrValidator.greater0OrMinus1();
    for (int duration : allDurations) {
        if (!validator.test(duration)) {
            // ... handle error
        }
    }
    ```

## Data Pipeline
The IntOrValidator acts as a gate within a larger data validation pipeline. It consumes a single integer and produces a boolean result, which determines the subsequent flow of the pipeline.

> Flow:
> NPC Asset File (JSON/HOCON) -> Parser -> Asset Builder -> **IntOrValidator**.test(value) -> Validation Result (Pass/Fail) -> Error Collector / Finalized Asset

