---
description: Architectural reference for TemporalArrayValidator
---

# TemporalArrayValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Base Class / Template

## Definition
```java
// Signature
public abstract class TemporalArrayValidator extends Validator {
```

## Architecture & Concepts
The TemporalArrayValidator is an abstract base class that establishes a formal contract for validating arrays of TemporalAmount objects. It serves as a foundational component within the server's NPC asset processing and validation pipeline. Its primary role is to ensure that time-based configurations for NPCs—such as ability cooldowns, patrol schedule durations, or behavior timers—are syntactically and logically correct before they are loaded into the game.

This class embodies the **Template Method Pattern**. It defines the high-level validation algorithm by requiring all subclasses to implement two key methods: a `test` method for the core validation logic and an `errorMessage` method for generating context-aware failure messages. This design decouples the generic validation framework from the specific rules applied to different time-based properties, promoting extensibility and maintainability.

Concrete implementations of this class are responsible for specific rules, such as "all durations must be positive" or "the array must contain exactly two elements representing a min/max range".

### Lifecycle & Ownership
- **Creation:** Concrete subclasses of TemporalArrayValidator are instantiated once during the initialization of the server's asset validation system. They are typically registered with a central `ValidatorRegistry` or a similar service locator.
- **Scope:** Instances are singletons within the context of the asset validation framework. They are stateless and designed to be reused for the validation of any number of NPC assets throughout the server's lifetime.
- **Destruction:** The validator instances are destroyed when the server shuts down and the application context is torn down.

## Internal State & Concurrency
- **State:** This class is inherently **stateless and immutable**. It contains no member fields and its behavior is determined exclusively by the arguments passed to its methods.
- **Thread Safety:** The class is **thread-safe by design**. Due to its stateless nature, a single instance of a concrete validator can be safely invoked by multiple concurrent asset-loading threads without requiring any synchronization mechanisms.

**WARNING:** Subclasses *must* remain stateless. Introducing mutable state into a concrete validator implementation will violate the design contract and lead to severe concurrency issues and non-deterministic behavior in a multi-threaded asset loading environment.

## API Surface
The public contract is defined entirely by its abstract methods, which must be implemented by subclasses.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(TemporalAmount[] var1) | boolean | *Implementation-Defined* | Executes the specific validation logic against the provided array. Returns true if valid, false otherwise. |
| errorMessage(String var1, TemporalAmount[] var2) | String | *Implementation-Defined* | Generates a descriptive error message for a failed validation. The first argument is the name of the asset field being validated. |

## Integration Patterns

### Standard Usage
Developers do not interact with TemporalArrayValidator directly. Instead, they create a concrete implementation defining a specific validation rule. The asset building system then discovers and applies this validator to the relevant NPC asset fields.

```java
// 1. A concrete validator is implemented
public class PositiveDurationArrayValidator extends TemporalArrayValidator {
    @Override
    public boolean test(TemporalAmount[] durations) {
        if (durations == null) return false;
        for (TemporalAmount duration : durations) {
            if (duration.isNegative() || duration.isZero()) {
                return false;
            }
        }
        return true;
    }

    @Override
    public String errorMessage(String fieldName, TemporalAmount[] value) {
        return String.format("Field '%s' contains an invalid non-positive duration.", fieldName);
    }
}

// 2. The system uses the validator internally (conceptual)
Validator validator = validatorRegistry.getValidatorForField("patrolPauses");
boolean isValid = validator.test(npcAsset.getPatrolPauses());
if (!isValid) {
    throw new AssetBuildException(validator.errorMessage("patrolPauses", npcAsset.getPatrolPauses()));
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** It is impossible to instantiate this class with `new TemporalArrayValidator()` because it is abstract. The anti-pattern is attempting to use the type without a concrete implementation.
- **Stateful Implementations:** Do not add member fields to subclasses that store per-validation state. Validators must be reusable and produce the same output for the same input, regardless of previous invocations.
- **Ignoring the `errorMessage` Contract:** The `errorMessage` method must provide clear, actionable feedback. A generic message like "Invalid" is an anti-pattern, as it hinders debugging of NPC asset files.

## Data Pipeline
This validator acts as a gate within the data pipeline that transforms raw NPC definition files into live, in-game entities. It ensures data integrity at a critical transition point.

> Flow:
> NPC Definition File (JSON/YAML) -> Asset Deserializer -> Raw Asset Object -> Asset Builder -> **Concrete TemporalArrayValidator** -> Validated Asset -> Game Registry

