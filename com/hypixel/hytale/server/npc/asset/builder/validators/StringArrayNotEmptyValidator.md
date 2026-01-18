---
description: Architectural reference for StringArrayNotEmptyValidator
---

# StringArrayNotEmptyValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Singleton

## Definition
```java
// Signature
public class StringArrayNotEmptyValidator extends StringArrayValidator {
```

## Architecture & Concepts
The StringArrayNotEmptyValidator is a stateless, reusable component within the server's asset validation framework. Its primary function is to enforce a strict data integrity rule: a string array must not be null, must not be empty, and must not contain any null or empty string elements.

This class embodies the **Strategy Pattern**, where a specific validation algorithm is encapsulated into a concrete object. It is designed to be used by higher-level asset builders and processors, particularly during the deserialization and validation of NPC configuration files. By providing a centralized and non-instantiable implementation, it ensures consistent validation logic across the entire NPC asset pipeline, preventing invalid data from propagating into the core game simulation where it could lead to runtime exceptions or undefined behavior.

## Lifecycle & Ownership
- **Creation:** The single instance is created eagerly via a static final initializer when the StringArrayNotEmptyValidator class is first loaded by the JVM. This guarantees its availability before any code attempts to use it.
- **Scope:** Application-scoped. The singleton instance persists for the entire lifetime of the server process.
- **Destruction:** The instance is managed by the JVM's class loader and is garbage collected only upon application shutdown. No manual cleanup is required or possible.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no instance fields and its methods operate exclusively on the arguments provided. Its behavior is deterministic and idempotent.
- **Thread Safety:** The component is inherently **thread-safe**. As a stateless singleton, it can be safely accessed and utilized by multiple concurrent asset-building threads without any risk of race conditions or data corruption. No external locking or synchronization is necessary when calling its methods.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | StringArrayNotEmptyValidator | O(1) | Returns the shared, global singleton instance of the validator. |
| test(String[] list) | boolean | O(N) | Executes the validation logic. Returns false if the array is null, empty, or contains any null or empty strings. |
| errorMessage(String name, String[] list) | String | O(1) | Generates a context-aware, human-readable error message for failed validations. |

## Integration Patterns

### Standard Usage
This validator should be retrieved via its static get method and used within a larger validation or data processing chain. It is not intended to be used in isolation.

```java
// Within an asset builder or processor
String[] textureAliases = assetData.getTextureAliases();

if (!StringArrayNotEmptyValidator.get().test(textureAliases)) {
    throw new AssetValidationException(
        StringArrayNotEmptyValidator.get().errorMessage("textureAliases", textureAliases)
    );
}

// Asset is now guaranteed to have a valid, non-empty list of texture aliases.
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private to enforce the singleton pattern. Attempting to create an instance via reflection will break the design contract and is strictly forbidden.
- **Redundant Null Checks:** Do not perform a null check on the input array before passing it to the test method. The validator is explicitly designed to handle null inputs gracefully.

```java
// INCORRECT: Redundant and unnecessary check
if (list != null && !StringArrayNotEmptyValidator.get().test(list)) {
    // ...
}

// CORRECT: The validator handles the null case internally
if (!StringArrayNotEmptyValidator.get().test(list)) {
    // ...
}
```

## Data Pipeline
The StringArrayNotEmptyValidator acts as a gate within a larger data ingestion and validation pipeline. It does not transform data but rather asserts its structural integrity before it is accepted by the system.

> Flow:
> NPC Asset File (e.g., JSON) -> Deserializer -> Raw Data Transfer Object -> **StringArrayNotEmptyValidator** -> Validated NPC Asset -> Server Game Registry

