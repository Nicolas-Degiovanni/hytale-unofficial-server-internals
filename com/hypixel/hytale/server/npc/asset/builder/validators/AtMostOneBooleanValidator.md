---
description: Architectural reference for AtMostOneBooleanValidator
---

# AtMostOneBooleanValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class AtMostOneBooleanValidator extends Validator {
```

## Architecture & Concepts
The AtMostOneBooleanValidator is a specialized, stateless rule component within the server's NPC asset validation framework. Its primary architectural role is to enforce mutual exclusivity among a set of boolean attributes during the asset building process.

This class embodies the Strategy pattern, where a specific validation algorithm is encapsulated into a distinct object. It is designed to be composed within a larger validation engine that processes NPC definition files. Rather than embedding complex if-else chains directly into an asset parser, the system delegates this specific constraint check to AtMostOneBooleanValidator. This promotes modularity, reusability, and simplifies the asset builder's core logic.

The class provides both static methods for direct, context-free validation and instance-based methods for generating context-aware error messages.

### Lifecycle & Ownership
- **Creation:** Instances are created on-demand via the static factory methods, typically `withAttributes`. This is usually done by a higher-level configuration parser or an asset builder that dynamically constructs a set of validation rules based on an asset's schema.
- **Scope:** Transient. An instance of this validator is short-lived, existing only for the duration of a single asset's validation cycle. It holds no long-term state and is not registered as a persistent service.
- **Destruction:** The object is eligible for garbage collection as soon as the validation engine that created it completes its operation and discards the reference.

## Internal State & Concurrency
- **State:** The class is immutable. The internal `attributes` array is private, final, and assigned only once during construction. It serves as read-only configuration data for generating error messages.
- **Thread Safety:** AtMostOneBooleanValidator is inherently thread-safe. Its immutable nature and the absence of side effects in its methods guarantee that it can be safely used across multiple threads without synchronization. The static `test` method is a pure function, further ensuring safe concurrent execution.

## API Surface
The public API is designed for both direct logical checks and for integration into a rule-based validation system.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(boolean[] values) | static boolean | O(N) | The core validation logic. Iterates through the input array to count true values. N is the size of the array. |
| errorMessage(String[] attributes) | static String | O(N) | Generates a standardized error message from an array of attribute names. N is the number of attributes. |
| errorMessage() | String | O(N) | Generates a context-specific error message using the instance's configured attribute names. |
| withAttributes(String...) | static AtMostOneBooleanValidator | O(1) | Factory method to construct a validator instance for a given set of attribute names. |

## Integration Patterns

### Standard Usage
This validator is intended to be used by a higher-level system that orchestrates the validation of an asset. The consumer would first gather the boolean values to be tested and then invoke the validator.

```java
// Example within a hypothetical AssetBuilder

// 1. Define the rule for a set of mutually exclusive properties
String[] exclusiveFlags = {"isFlying", "isSwimming"};
AtMostOneBooleanValidator mutualExclusivityRule = AtMostOneBooleanValidator.withAttributes(exclusiveFlags);

// 2. Gather the values from the asset being processed
boolean[] valuesToTest = {asset.isFlying(), asset.isSwimming()};

// 3. Execute the validation logic
if (!AtMostOneBooleanValidator.test(valuesToTest)) {
    // 4. Use the instance to get a descriptive error
    throw new AssetValidationException(mutualExclusivityRule.errorMessage());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to use reflection to create an instance. Always use the static `withAttributes` factory methods to ensure proper initialization.
- **Empty Attribute Set:** While technically possible, creating a validator with an empty or single-element array of attributes is meaningless and indicates a configuration error in the calling code. The logic will always pass but provides no value.

## Data Pipeline
This class acts as a conditional gate within a larger data processing pipeline for NPC assets. It does not transform data but rather validates it, potentially halting the pipeline if a constraint is violated.

> Flow:
> NPC Asset File (JSON/HOCON) -> Asset Deserializer -> Raw Asset Object -> Validation Engine -> **AtMostOneBooleanValidator** -> Validated Asset or Error Report

