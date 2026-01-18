---
description: Architectural reference for BooleanImplicationValidator
---

# BooleanImplicationValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class BooleanImplicationValidator extends Validator {
```

## Architecture & Concepts
The BooleanImplicationValidator is a specialized component within the server-side NPC asset validation framework. Its primary function is to enforce conditional rules between sets of boolean attributes, representing a logical implication (IF P THEN Q).

This class acts as a concrete, stateless rule engine. It is designed to be instantiated by a higher-level asset builder or configuration loader that parses NPC definition files. By encapsulating this logic, the core asset processing pipeline remains clean, while complex inter-attribute dependencies are handled in a modular and reusable manner.

For example, this validator can enforce a rule such as: "If *any* of the attributes {isAggressive, isTerritorial} are true, then *all* of the attributes {hasCombatAI, hasTargetingSystem} must also be true."

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively through the static factory method `withAttributes`. This is a design choice to prevent misconfiguration and enforce a clear instantiation pattern. It is expected to be called by an asset loading system when it encounters a validation rule definition.
- **Scope:** An instance of BooleanImplicationValidator is a short-lived, transient object. Its lifecycle is typically confined to a single validation pass for one NPC asset. It holds the rule configuration but no runtime state.
- **Destruction:** The object is managed by the Java Garbage Collector. Once the asset validation process that created it is complete, the validator instance is no longer referenced and will be reclaimed. No manual cleanup is necessary.

## Internal State & Concurrency
- **State:** The BooleanImplicationValidator is **immutable**. All internal fields are final and are set only once via the private constructor during instantiation. It does not cache data or modify its state after creation.
- **Thread Safety:** This class is inherently **thread-safe**. Due to its immutable nature, a single configured instance can be safely used across multiple threads to validate different sets of inputs without locks or risk of race conditions.

## API Surface
The public contract is minimal, focusing on creation and execution of the validation rule.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| withAttributes(...) | BooleanImplicationValidator | O(1) | Static factory method. Constructs and returns a new validator configured with the specified implication rule. |
| test(antecedents, consequents) | boolean | O(N+M) | Executes the validation logic. Returns true if the implication holds, false otherwise. N and M are the sizes of the antecedent and consequent sets, respectively. |
| errorMessage() | String | O(N+M) | Generates a detailed, human-readable error message explaining the rule that was violated. Intended for logging and debugging. |

## Integration Patterns

### Standard Usage
The validator is intended to be created by a factory or builder and used immediately to test a set of boolean values extracted from an asset.

```java
// Context: Inside an asset validation engine

// 1. Define the rule's parameters (typically from a config file)
String[] antecedents = {"isHostile", "isAggressive"};
String[] consequents = {"hasCombatAI"};

// 2. Create the validator instance using the factory method
BooleanImplicationValidator validator = BooleanImplicationValidator.withAttributes(
    antecedents, true,  // IF any of these are true
    consequents, true,  // THEN this must be true
    true               // 'any' antecedent match
);

// 3. Prepare the actual boolean values from the asset being validated
boolean[] antecedentValues = {true, false}; // isHostile=true, isAggressive=false
boolean[] consequentValues = {true};       // hasCombatAI=true

// 4. Execute the test
boolean isValid = validator.test(antecedentValues, consequentValues);
if (!isValid) {
    // Log the specific error for asset debugging
    log.warn("Asset validation failed: " + validator.errorMessage());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to use reflection to create an instance. Always use the `withAttributes` static factory method to ensure proper initialization.
- **Reusing for Different Rules:** A validator instance is configured for a single, specific rule. Do not attempt to modify its internal state to reuse it for a different rule; create a new instance instead.
- **Misaligned Input Arrays:** The boolean arrays passed to the `test` method must correspond exactly in order and size to the string arrays used to create the validator. The system relies on this positional mapping. Mismatches will lead to incorrect validation results.

## Data Pipeline
The BooleanImplicationValidator functions as a single step within a larger asset processing and validation pipeline.

> Flow:
> NPC Asset File (JSON/YAML) -> Server Asset Parser -> **BooleanImplicationValidator** (Instantiated with rules from file) -> Validation Coordinator -> Asset Accepted or Rejected with Error Message

