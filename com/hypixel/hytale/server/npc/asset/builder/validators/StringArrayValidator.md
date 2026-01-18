---
description: Architectural reference for StringArrayValidator
---

# StringArrayValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public abstract class StringArrayValidator extends Validator {
```

## Architecture & Concepts
The StringArrayValidator is an abstract base class that establishes a formal contract for validating string array fields within the server's NPC asset definition pipeline. It serves as a foundational component for ensuring data integrity during asset loading, preventing malformed or invalid NPC configurations from being loaded into the game world.

This class embodies the **Strategy Pattern**. Each concrete implementation represents a specific validation rule or strategy (e.g., ensuring an array is not empty, checking for valid identifiers, etc.). The NPC asset builder invokes these validators polymorphically, decoupling the validation logic from the core asset construction process. This design allows for the flexible and declarative addition of new validation rules without modifying the asset builder itself.

Its primary role is to act as a gatekeeper for string array data deserialized from configuration files, ensuring that all data conforms to engine-defined constraints before it is used to construct a final, in-memory NPC asset.

### Lifecycle & Ownership
- **Creation:** Concrete implementations of StringArrayValidator are instantiated by the NPC asset building system, typically via a factory or dependency injection, when a specific validation rule is required for an asset field. These are transient, short-lived objects.
- **Scope:** The lifetime of a validator instance is extremely brief, typically scoped to the validation of a single field on a single NPC asset during the loading process.
- **Destruction:** The object is eligible for garbage collection immediately after its validation methods complete. It holds no persistent state and requires no explicit cleanup.

## Internal State & Concurrency
- **State:** This class is designed to be **stateless**. The validation logic within the `test` method should operate exclusively on the input parameters provided. Concrete implementations must not maintain mutable state between invocations. This ensures that validation outcomes are deterministic and free from side effects.
- **Thread Safety:** The stateless design contract makes any compliant implementation inherently **thread-safe**. A single instance could theoretically be used across multiple threads without risk of data corruption, although in practice, new instances are typically created for each validation task.

## API Surface
The public contract consists entirely of abstract methods that must be implemented by a concrete validator subclass.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String[] var1) | boolean | O(n) | The core validation logic. Returns true if the input array is valid according to the specific rule, false otherwise. Complexity is typically linear to the size of the array. |
| errorMessage(String var1, String[] var2) | String | O(1) | Generates a context-rich error message, including the name of the field being validated and the invalid value. Crucial for asset pipeline debugging. |
| errorMessage(String[] var1) | String | O(1) | Generates a simpler error message containing only the invalid value. |

## Integration Patterns

### Standard Usage
Developers should extend this class to create a specific, reusable validation rule. This new class is then registered with the asset building framework to be applied automatically during NPC loading.

```java
// 1. Define a concrete validation strategy
public class NonEmptyStringArrayValidator extends StringArrayValidator {
    @Override
    public boolean test(String[] values) {
        return values != null && values.length > 0;
    }

    @Override
    public String errorMessage(String fieldName, String[] values) {
        return String.format("Field '%s' must be a non-empty array.", fieldName);
    }

    @Override
    public String errorMessage(String[] values) {
        return "The array must not be empty.";
    }
}

// 2. The asset builder system would then use it internally (conceptual)
// Validator validator = new NonEmptyStringArrayValidator();
// if (!validator.test(npcData.getSomeArrayField())) {
//     throw new AssetValidationException(validator.errorMessage(...));
// }
```

### Anti-Patterns (Do NOT do this)
- **Implementing Stateful Logic:** Do not add fields to a subclass that store state between calls to `test`. This breaks the stateless contract and can lead to unpredictable, order-dependent validation failures that are difficult to debug.
- **Ignoring Error Messages:** Do not implement `errorMessage` to return null or a generic, unhelpful string. The quality of these messages is critical for content creators and developers to diagnose problems in their asset files.
- **Direct Invocation:** While possible, directly instantiating and calling validators is not the intended use case. They are designed to be managed and executed by the higher-level asset building system to ensure all assets are validated consistently.

## Data Pipeline
This component operates at a critical juncture in the server-side asset loading pipeline, transforming raw data into validated, game-ready objects.

> Flow:
> NPC Definition File (e.g., JSON) -> Jackson Deserializer -> Raw Data Transfer Object -> NPC Asset Builder -> **StringArrayValidator** -> Validated NPC Asset -> NPC Registry

