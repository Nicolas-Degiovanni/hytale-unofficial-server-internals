---
description: Architectural reference for StringsNotEmptyValidator
---

# StringsNotEmptyValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class StringsNotEmptyValidator extends Validator {
```

## Architecture & Concepts
The StringsNotEmptyValidator is a specialized component within the server-side NPC asset validation framework. Its primary function is to enforce a rule where at least one string from a given pair or set must be non-null and non-empty. This is a common requirement in asset definitions where a model can be identified by one of several possible attributes, such as a model name or a mesh reference.

This class embodies a dual-mode design:

1.  **Stateless Utility:** The static methods `test` and `errorMessage` provide direct, stateless access to the core validation logic. This is intended for ad-hoc, one-off checks where the overhead of object instantiation is unnecessary.
2.  **Configurable Validator Instance:** The static factory methods `withAttributes` create an immutable instance of the validator. This instance encapsulates the *names* of the attributes being validated, allowing it to be integrated into a larger, polymorphic validation system that operates on a collection of `Validator` objects.

This design separates the pure validation logic from the configuration, making the class both flexible for direct use and extensible for framework integration.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively via the static factory methods `withAttributes`. The constructor is private to prevent direct instantiation, enforcing a controlled creation pattern. It is typically instantiated by a higher-level asset builder or configuration loader that assembles a list of validation rules for a given asset type.
-   **Scope:** Transient. An instance of this class is short-lived, designed to exist only for the duration of a single asset validation pass. It holds no persistent state and is not intended to be cached or reused across different validation contexts.
-   **Destruction:** The object is eligible for garbage collection as soon as the validation process that created it completes and all references to it are dropped.

## Internal State & Concurrency
-   **State:** Immutable. The internal `attributes` array is a final field, initialized once at construction time and never modified thereafter. This immutability guarantees predictable behavior.
-   **Thread Safety:** This class is inherently thread-safe. Both the static methods and the instances created by the factory methods can be safely used across multiple threads without any external synchronization. The stateless nature of the static methods and the immutability of the instances eliminate the possibility of race conditions.

## API Surface
The public API is composed entirely of static methods, which either perform validation directly or act as factories for validator instances.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(string1, string2) | boolean | O(1) | Performs the core validation logic, returning true if at least one string is not empty. |
| errorMessage(...) | String | O(1) | Generates a standardized, human-readable error message for failed validations. |
| withAttributes(attribute1, attribute2) | StringsNotEmptyValidator | O(1) | Factory method to create a configured validator instance for two attributes. |
| withAttributes(attributes) | StringsNotEmptyValidator | O(1) | Factory method to create a configured validator instance for an array of attributes. |

## Integration Patterns

### Standard Usage
For direct, one-off validation, use the static `test` method.

```java
// Example: Ad-hoc check during data processing
String modelName = asset.getModelName();
String meshPath = asset.getMeshPath();

if (!StringsNotEmptyValidator.test(modelName, meshPath)) {
    String error = StringsNotEmptyValidator.errorMessage("modelName", "meshPath", "NPC 'goblin'");
    throw new AssetValidationException(error);
}
```

For integration into a validation framework, use the factory method to create an instance that can be added to a collection of validators.

```java
// Example: Building a list of rules for an asset type
List<Validator> rules = new ArrayList<>();
rules.add(StringsNotEmptyValidator.withAttributes("modelName", "meshPath"));

// The framework would later iterate through these rules
for (Validator rule : rules) {
    rule.validate(asset);
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to instantiate this class with `new`. The constructor is private for a reason. Always use the static `withAttributes` factory methods.
-   **Redundant Instantiation:** Do not create a new instance inside a loop for simple checks. If you only need the boolean result, the static `test` method is significantly more performant and avoids unnecessary object allocation.

## Data Pipeline
This validator acts as a gate within a larger data loading and validation pipeline. It does not transform data but rather asserts conditions on it.

> Flow:
> Asset File (JSON/HOCON) -> Deserializer -> Raw Asset Object -> **StringsNotEmptyValidator** -> Validation Result (Pass/Fail) -> Engine Asset Registry

