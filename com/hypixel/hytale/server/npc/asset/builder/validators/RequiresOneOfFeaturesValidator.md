---
description: Architectural reference for RequiresOneOfFeaturesValidator
---

# RequiresOneOfFeaturesValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Transient

## Definition
```java
// Signature
public class RequiresOneOfFeaturesValidator extends RequiredFeatureValidator {
```

## Architecture & Concepts
The RequiresOneOfFeaturesValidator is a specific implementation of a validation rule within the NPC asset building framework. It embodies the **Specification Pattern**, where a complex business rule is encapsulated into a standalone, composable object.

Its primary role is to enforce a constraint during NPC asset creation: an asset must be configured with *at least one* feature from a predefined set. For example, a "Mythical Creature" NPC might require either a Horns feature, a Wings feature, or a GlowingAura feature to be considered valid.

This class operates as a stateless predicate. It is provided with the current state of the asset being built via the FeatureEvaluatorHelper and returns a boolean result indicating compliance. This design decouples the validation logic from the core asset builder, allowing for flexible and maintainable rule sets that can be easily combined or modified without altering the main build process.

### Lifecycle & Ownership
- **Creation:** Instantiated exclusively through the static factory method `withFeatures`. This is typically invoked by higher-level configuration code or a specific NPC asset builder that defines the validation requirements for a particular NPC type.
- **Scope:** This object is short-lived and transient. Its lifecycle is typically confined to a single asset build operation. It is created, its `validate` method is called, and it is then eligible for garbage collection.
- **Destruction:** Managed by the Java garbage collector. No explicit cleanup is required.

## Internal State & Concurrency
- **State:** The internal state consists of an EnumSet of required features and a pre-calculated description array. This state is set once during construction and is never modified. The object is therefore **Immutable**.
- **Thread Safety:** As an immutable object, RequiresOneOfFeaturesValidator is inherently **Thread-Safe**. A single instance can be safely shared and used across multiple threads without synchronization, although the typical use case is a new instance per validation cycle.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| withFeatures(EnumSet) | RequiresOneOfFeaturesValidator | O(N) | Static factory method to create a new validator instance. N is the number of features in the set. |
| validate(FeatureEvaluatorHelper) | boolean | O(P) | Executes the validation logic. Iterates through all providers (P) in the helper to check for a match. |
| getErrorMessage(String) | String | O(1) | Generates a human-readable error message if validation fails. |

## Integration Patterns

### Standard Usage
This validator is intended to be created and added to a collection of rules that are executed by a master asset builder or validation coordinator.

```java
// In a hypothetical NPC builder or configuration class
List<RequiredFeatureValidator> validators = new ArrayList<>();

// Define a rule: The NPC must have either a HELMET or a HAT feature.
EnumSet<Feature> headwearOptions = EnumSet.of(Feature.HELMET, Feature.HAT);
validators.add(RequiresOneOfFeaturesValidator.withFeatures(headwearOptions));

// The builder would later iterate through these validators
for (RequiredFeatureValidator validator : validators) {
    if (!validator.validate(featureHelper)) {
        throw new ValidationException(validator.getErrorMessage("NPC.HeadSlot"));
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to use reflection to create instances; always use the `withFeatures` static factory method.
- **Empty Feature Set:** Passing an empty EnumSet to `withFeatures` will create a validator that always returns `false`, as no feature can ever be provided from an empty set. This is logically incorrect and will always cause the build to fail.
- **Misunderstanding the Rule:** Do not use this class when you need to ensure *all* features from a set are present. This validator confirms the existence of *at least one*, not all.

## Data Pipeline
This component acts as a gate in a control flow rather than a step in a data transformation pipeline. It consumes state and produces a binary decision.

> Flow:
> NPC Asset Configuration -> Asset Builder -> FeatureEvaluatorHelper (State Snapshot) -> **RequiresOneOfFeaturesValidator.validate()** -> Boolean (Pass/Fail) -> Build Continues or Throws Exception

