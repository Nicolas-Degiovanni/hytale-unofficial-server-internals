---
description: Architectural reference for AnyPresentValidator
---

# AnyPresentValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class AnyPresentValidator extends Validator {
```

## Architecture & Concepts
The AnyPresentValidator is a concrete implementation of the Validator strategy pattern, designed for use within the server-side NPC asset building system. Its primary function is to enforce a "logical OR" constraint on a set of related asset attributes. It ensures that for a given group of properties, at least one must be defined for the asset to be considered valid.

This component acts as a validation rule object. It is configured with a set of attribute names and provides the logic to test for their presence against a collection of BuilderObjectHelper instances. The separation of the static test method from the instance-level errorMessage method is a key design choice: the logic is pure and reusable, while the instance carries the necessary context for generating a human-readable error message.

## Lifecycle & Ownership
- **Creation:** Instantiated on-demand via its static factory methods, `withAttributes`. This class is not managed by a dependency injection container and is created directly by the asset validation framework when an "any present" rule is defined for an asset.
- **Scope:** Transient and short-lived. An instance of AnyPresentValidator exists only for the duration of a single asset validation pass. It is created, used to perform a check, and then becomes eligible for garbage collection.
- **Destruction:** The object is lightweight and holds no external resources. It is reclaimed by the Java garbage collector when it falls out of scope after the validation process completes.

## Internal State & Concurrency
- **State:** Immutable. The internal state consists of a final array of strings, `attributes`, which is populated at construction time and never modified. The object itself represents a specific, unchanging validation rule.
- **Thread Safety:** This class is inherently thread-safe. Its immutable state and the lack of side effects in its methods guarantee that a single instance can be safely shared and executed across multiple threads without synchronization. This is critical in a system that may perform parallel asset loading and validation.

## API Surface
The public API is minimal, focusing on object creation and the execution of the validation rule.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(BuilderObjectHelper<?>[] objects) | static boolean | O(N) | Executes the core validation logic. Iterates through the provided helpers and short-circuits, returning true on the first object where isPresent() is true. |
| errorMessage() | String | O(M) | Generates a context-specific error message using the attribute names stored in the instance. M is the number of attributes. |
| withAttributes(String... attributes) | static AnyPresentValidator | O(1) | The primary factory method for creating a validator instance. |

## Integration Patterns

### Standard Usage
The intended pattern is to create a validator instance for a specific rule and associate it with a validation step in the asset builder. The static `test` method is called with the relevant data, and if it returns false, the instance's `errorMessage` method is used to report the failure.

```java
// In a hypothetical asset builder configuration
String[] requiredAttributes = {"primaryWeapon", "offhandWeapon"};
AnyPresentValidator weaponValidator = AnyPresentValidator.withAttributes(requiredAttributes);

// During the validation phase
BuilderObjectHelper<?>[] weaponSlots = getWeaponSlotsFromAsset();
if (!AnyPresentValidator.test(weaponSlots)) {
    throw new AssetValidationException(weaponValidator.errorMessage());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private to enforce the use of the `withAttributes` factory methods. Do not attempt to use reflection to create instances.
- **Error Message Mismatch:** Do not call the static `test` method and, upon failure, attempt to construct a new error message manually. The validator instance is designed to be the single source of truth for its corresponding error message. Using the instance ensures the error message always matches the rule that was violated.

## Data Pipeline
AnyPresentValidator acts as a gate within the broader NPC asset data pipeline. It does not transform data but rather asserts conditions on it.

> Flow:
> Asset Definition (JSON/HOCON) -> Asset Deserializer -> BuilderObjectHelper Population -> **AnyPresentValidator** -> Validation Result (Pass/Fail) -> Final Asset Construction

