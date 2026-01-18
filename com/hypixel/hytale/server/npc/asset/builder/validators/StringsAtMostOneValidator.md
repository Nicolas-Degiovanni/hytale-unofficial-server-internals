---
description: Architectural reference for StringsAtMostOneValidator
---

# StringsAtMostOneValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility / Data Object

## Definition
```java
// Signature
public class StringsAtMostOneValidator extends Validator {
```

## Architecture & Concepts
The StringsAtMostOneValidator is a specialized component within the server-side NPC asset validation framework. Its primary architectural role is to enforce data integrity by ensuring that a set of mutually exclusive string attributes are not defined simultaneously within a single asset definition.

This class embodies a specific validation strategy: "at most one of a given set of strings may be non-null and non-empty". It is designed to be a lightweight, reusable rule that can be composed with other validators to build a comprehensive validation pipeline for complex NPC assets. It prevents logical contradictions in asset files, such as an NPC being assigned both a specific *skin* and a *skin set*, which would be ambiguous.

Architecturally, it functions as a leaf component in the asset loading process, executed by a higher-level asset builder or validation engine after deserialization but before the asset is committed to the runtime registry.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively through the static factory methods `withAttributes`. The constructor is private to enforce this pattern. Typically, a central asset building service instantiates these validators during its own initialization, configuring the validation rules for a specific asset type.
- **Scope:** An instance of StringsAtMostOneValidator is transient and has a very narrow scope. It is intended to be used for the duration of a single asset validation cycle and then discarded.
- **Destruction:** The object is managed by the Java Garbage Collector and requires no explicit cleanup. It becomes eligible for collection as soon as the validation process that created it completes.

## Internal State & Concurrency
- **State:** The internal state consists of a final string array, `attributes`, which is populated at construction time and never modified. This makes instances of StringsAtMostOneValidator **immutable**.
- **Thread Safety:** The class is inherently thread-safe due to its immutability. The static utility methods, `test` and `errorMessage`, are pure functions that operate only on their inputs, making them safe for concurrent execution. A single instance can be safely shared across multiple threads, though this is an unlikely usage pattern given its transient nature.

## API Surface
The public contract is dominated by static factory and utility methods. The instance-level validation logic is likely inherited from the parent `Validator` class and is not part of this class's direct surface area.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(string1, string2) | static boolean | O(1) | Core validation logic. Returns true if one or both of the input strings are null or empty. |
| errorMessage(...) | static String | O(1) | Generates a standardized, human-readable error message for validation failures. |
| withAttributes(...) | static StringsAtMostOneValidator | O(1) | Factory method to create a new validator instance for a given set of attribute names. |

## Integration Patterns

### Standard Usage
The static `test` method is suitable for direct, ad-hoc validation of two string variables. This is the most common pattern for simple, localized checks.

```java
// How a developer should normally use this for a simple check
String npcSkin = asset.getSkin();
String npcSkinSet = asset.getSkinSet();

if (!StringsAtMostOneValidator.test(npcSkin, npcSkinSet)) {
    String error = StringsAtMostOneValidator.errorMessage(
        "skin", npcSkin,
        "skinSet", npcSkinSet,
        "NpcDefinition"
    );
    throw new AssetValidationException(error);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to use reflection to create instances. Always use the static `withAttributes` factory methods.
- **Manual Null/Empty Checks:** Do not replicate the logic of the `test` method. It correctly handles both null and empty strings, which is a common source of bugs. Rely on the provided utility.
    ```java
    // BAD: Incomplete and error-prone check
    if (npcSkin != null && npcSkinSet != null) {
        // This fails to check for empty strings
    }

    // GOOD: Correct and intention-revealing
    if (!StringsAtMostOneValidator.test(npcSkin, npcSkinSet)) {
        // ...
    }
    ```

## Data Pipeline
This validator acts as a gate in the data pipeline for NPC assets. It does not transform data but rather asserts its logical consistency.

> Flow:
> NPC Asset File (JSON) -> Deserializer -> Raw NPC Data Object -> **StringsAtMostOneValidator.test()** -> Validation Result (Pass/Fail) -> [On Fail: Error Log] -> [On Pass: Asset Instantiation]

