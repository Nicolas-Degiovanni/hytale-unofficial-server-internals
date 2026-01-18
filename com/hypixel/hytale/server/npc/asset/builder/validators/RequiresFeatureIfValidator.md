---
description: Architectural reference for RequiresFeatureIfValidator
---

# RequiresFeatureIfValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Transient

## Definition
```java
// Signature
public class RequiresFeatureIfValidator extends RequiredFeatureValidator {
```

## Architecture & Concepts
The RequiresFeatureIfValidator is a specific, stateless validation rule component within the server-side NPC asset building framework. Its primary architectural function is to enforce conditional dependencies between an NPC's attributes and its provided features.

This class embodies the Strategy pattern for validation. Instead of a monolithic builder class containing complex, nested if-else blocks, validation logic is encapsulated into small, reusable, and composable validator objects. This particular validator implements the rule: "If a given boolean attribute has a specific value, then a designated set of features must be provided by the asset."

It operates during the asset compilation or loading phase, acting as a guard against logically inconsistent NPC configurations. For example, it can enforce that if an NPC has an attribute `canSwim` set to true, it must also be provided with the `SWIMMING_AI` feature. This prevents runtime errors and simplifies the logic in the game simulation systems, which can then assume that a swim-enabled NPC has all the necessary components.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively through the static factory method `withAttributes`. This is typically invoked by a higher-level asset parser or a builder system that translates a declarative asset definition (e.g., a JSON or HOCON file) into a collection of validation rules.
- **Scope:** Extremely short-lived. A validator instance exists only for the duration of the asset validation process. It is created, its logic is queried, and it is then immediately eligible for garbage collection. It holds no long-term state.
- **Destruction:** Managed automatically by the JVM garbage collector. There are no native resources or explicit cleanup steps required.

## Internal State & Concurrency
- **State:** Immutable. All internal fields (`attribute`, `value`, `description`) are final and set during construction. This design choice is critical for a validation component, as it guarantees that a rule's definition cannot be altered mid-process, ensuring predictable and repeatable validation outcomes.
- **Thread Safety:** Inherently thread-safe. Due to its immutable nature, a single instance can be safely used across multiple threads without any external locking or synchronization. The core logic in the `staticValidate` method is also fully re-entrant and thread-safe, as it operates solely on its method parameters.

## API Surface
The primary public contract is the static factory and validation methods. The instance methods appear to be part of an incomplete or alternative integration path.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| withAttributes(attribute, value, feature) | RequiresFeatureIfValidator | O(1) | Factory method. Constructs a validator instance configured with the conditional rule. |
| staticValidate(evaluatorHelper, requiredFeature, requiredValue, value) | boolean | O(N) | **Primary Entry Point.** Executes the validation logic. N is the number of providers in the helper. Returns true if the condition is met or the required feature is present. |
| validate(evaluatorHelper) | boolean | O(1) | **WARNING:** This inherited instance method is not implemented and always returns false. It should not be used for validation. |
| getErrorMessage(context) | String | O(1) | **WARNING:** This inherited instance method is not implemented and always returns null. |

## Integration Patterns

### Standard Usage
This class is not intended for direct use by gameplay programmers but is a component of the underlying asset system. Its logic is invoked by a validation runner that iterates through rules defined for an asset.

```java
// Hypothetical usage within an asset validation engine
FeatureEvaluatorHelper assetContext = getCurrentAssetContext();
EnumSet<Feature> requiredSwimming = EnumSet.of(Feature.SWIMMING_AI);
boolean isAquaticFlag = assetDefinition.getBoolean("isAquatic");

// The engine calls the static method to check the rule
boolean isValid = RequiresFeatureIfValidator.staticValidate(
    assetContext,
    requiredSwimming,
    true, // The rule triggers if the flag is true
    isAquaticFlag
);

if (!isValid) {
    throw new AssetValidationException("Aquatic NPCs must have the SWIMMING_AI feature.");
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to construct this class with `new`. The constructor is private; use the `withAttributes` factory method. This is enforced by the compiler.
- **Using Instance Methods:** Do not call `validator.validate()`. The implementation of this instance method is a stub that always returns false. The correct and complete validation logic is in the static `staticValidate` method. Relying on the instance method will lead to incorrect validation failures.

## Data Pipeline
This validator acts as a gate in the NPC asset data pipeline. It does not transform data but rather asserts its logical integrity before it is passed to subsequent systems.

> Flow:
> Raw NPC Definition (JSON/HOCON) -> Asset Parser -> Validation Engine -> **RequiresFeatureIfValidator.staticValidate()** -> Validation Result (Pass/Fail) -> Finalized NPC Asset

---

