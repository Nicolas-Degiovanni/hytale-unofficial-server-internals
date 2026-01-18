---
description: Architectural reference for RequiresFeatureIfEnumValidator
---

# RequiresFeatureIfEnumValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Transient

## Definition
```java
// Signature
public class RequiresFeatureIfEnumValidator<E extends Enum<E> & Supplier<String>> extends RequiredFeatureValidator {
```

## Architecture & Concepts
The RequiresFeatureIfEnumValidator is a specialized rule object within the NPC asset validation framework. Its primary function is to enforce conditional dependencies between an asset's attribute (represented by an enum) and the presence of a required Feature.

This validator codifies a common design pattern in game asset creation: "If an NPC has property X, it must also have capability Y". For example, if an NPC's `locomotionType` is set to `FLYING`, this validator ensures that the asset definition also includes a `WINGS` Feature.

It operates as a declarative rule. An instance of this class holds the parameters of the rule—the attribute name, the trigger enum value, and the required feature—but the validation logic itself is executed by an external entity using the static `staticValidate` method. The `FeatureEvaluatorHelper` acts as the context provider, supplying the set of all features currently available on the asset being validated.

**WARNING:** The instance methods `validate` and `getErrorMessage` are non-functional stubs. All logical validation is performed through the static `staticValidate` method. Instances of this class should be treated as data containers for the rule's parameters.

## Lifecycle & Ownership
- **Creation:** Instantiated exclusively through the static factory method `withAttributes`. This typically occurs during the initialization of an asset builder, where a collection of validation rules is assembled.
- **Scope:** Extremely short-lived. An instance exists only for the duration of a single asset build and validation cycle. It is created, its parameters are read by a validation runner, and it is then discarded.
- **Destruction:** Eligible for garbage collection immediately after the asset validation process completes. There is no persistent state and no cleanup logic.

## Internal State & Concurrency
- **State:** The object's state (the target attribute, trigger value, and required feature) is set upon creation and is never modified. The class is **Immutable**.
- **Thread Safety:** As an immutable data container, this class is inherently **thread-safe**. The primary logic in the `staticValidate` method is also thread-safe, as it operates solely on its arguments without modifying any shared static state. It can be safely used in parallel asset build systems.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| withAttributes(attribute, value, feature) | RequiresFeatureIfEnumValidator | O(1) | Static factory method. Creates a new validator instance with the specified rule parameters. |
| staticValidate(evaluatorHelper, requiredFeature, requiredValue, value) | boolean | O(N) | Executes the validation logic. Returns true if the condition is met or not applicable, false otherwise. N is the number of providers in the helper. |
| validate(evaluatorHelper) | boolean | O(1) | **Non-functional.** This instance method is a stub and always returns false. Do not use. |
| getErrorMessage(context) | String | O(1) | **Non-functional.** This instance method is a stub and always returns null. Do not use. |

## Integration Patterns

### Standard Usage
The correct pattern is to use the static `staticValidate` method within a broader validation loop, passing parameters from a rule instance and the current asset context.

```java
// Assume 'rules' is a list of validator objects and 'assetContext' is a FeatureEvaluatorHelper

// In a validation engine:
for (ValidatorRule rule : rules) {
    if (rule instanceof RequiresFeatureIfEnumValidator) {
        // Cast and extract parameters from the rule object
        RequiresFeatureIfEnumValidator r = (RequiresFeatureIfEnumValidator) rule;
        Enum<?> currentValue = assetContext.getEnumValue(r.attribute); // Hypothetical context method

        // Execute validation using the static method
        boolean isValid = RequiresFeatureIfEnumValidator.staticValidate(
            assetContext,
            r.getRequiredFeatures(), // Hypothetical getter
            r.value,
            currentValue
        );

        if (!isValid) {
            // Handle validation failure
        }
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Calling Instance Methods:** Never call the `validate` method on an instance of this class. It does not contain the correct logic and will produce incorrect validation results.

    ```java
    // INCORRECT: This will always return false.
    RequiresFeatureIfEnumValidator rule = RequiresFeatureIfEnumValidator.withAttributes(...);
    boolean result = rule.validate(evaluatorHelper); // This is wrong!
    ```

- **Direct Instantiation:** The constructor is private. Do not attempt to use reflection to create instances. Always use the `withAttributes` factory method.

## Data Pipeline
This component acts as a conditional gate in a validation pipeline. It does not transform data but rather produces a boolean decision that affects the control flow of the asset build process.

> Flow:
> Asset Builder -> Collects Validation Rules (including **RequiresFeatureIfEnumValidator**) -> Validation Engine -> **RequiresFeatureIfEnumValidator.staticValidate()** -> Boolean Result (Pass/Fail) -> Build Halts or Continues

