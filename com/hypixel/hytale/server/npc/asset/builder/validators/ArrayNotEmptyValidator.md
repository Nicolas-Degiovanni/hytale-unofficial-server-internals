---
description: Architectural reference for ArrayNotEmptyValidator
---

# ArrayNotEmptyValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Singleton

## Definition
```java
// Signature
public class ArrayNotEmptyValidator extends ArrayValidator {
```

## Architecture & Concepts
The ArrayNotEmptyValidator is a stateless, reusable component that embodies a single validation rule within the server's NPC Asset Builder framework. It implements the Strategy pattern, where it represents a concrete algorithm for checking if an array-like data structure is empty.

Its primary role is to enforce data integrity during the deserialization and construction of NPC assets. By encapsulating this simple but common rule into a dedicated class, the system decouples the validation logic from the asset builders themselves. This allows builders to be composed with a set of validators, promoting modularity and preventing the duplication of validation code.

This validator operates on a BuilderObjectArrayHelper, an abstraction that shields the validation logic from the underlying representation of the array data.

## Lifecycle & Ownership
- **Creation:** The ArrayNotEmptyValidator is an eagerly initialized Singleton. A single, static instance named INSTANCE is created by the JVM when the class is first loaded. Its lifecycle is not managed by a dependency injection framework.
- **Scope:** Application-scoped. The single instance persists for the entire lifetime of the server process.
- **Destruction:** The instance is destroyed only when the JVM shuts down. No explicit cleanup is required or possible.

## Internal State & Concurrency
- **State:** This class is **stateless**. It contains no instance fields and its behavior is exclusively determined by the arguments passed to its methods. It is therefore immutable by design.
- **Thread Safety:** The ArrayNotEmptyValidator is inherently **thread-safe**. As a stateless singleton, it can be safely shared and invoked by multiple threads concurrently without any risk of race conditions or need for external synchronization.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get() | ArrayNotEmptyValidator | O(1) | Retrieves the globally unique, static instance of the validator. |
| test(helper) | boolean | O(1) | Executes the validation logic. Returns true if the helper represents a non-empty collection, false otherwise. |
| errorMessage(name) | String | O(1) | A static utility that generates a standardized error message for a given field name. |

## Integration Patterns

### Standard Usage
This validator is intended to be used by higher-level asset builders or a central validation service. The typical pattern is to retrieve the singleton instance and use it to test an array helper, throwing an exception if the validation fails.

```java
// Example from within an NPC asset construction process
BuilderObjectArrayHelper<Behavior, ?> behaviors = npcAsset.getBehaviors();
ArrayNotEmptyValidator validator = ArrayNotEmptyValidator.get();

if (!validator.test(behaviors)) {
    // Halt asset creation and report a clear error
    throw new AssetValidationException(
        ArrayNotEmptyValidator.errorMessage("behaviors")
    );
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to create an instance using reflection. The class is designed as a singleton for performance and to represent a single, unchanging rule. Always use the static `get()` method.
- **Redundant Null Checks:** The validator's contract is to operate on a BuilderObjectArrayHelper. The helper itself is responsible for handling null or absent underlying data. Do not perform a null check on the helper before passing it to the validator.

## Data Pipeline
The ArrayNotEmptyValidator acts as a gate within the broader NPC asset loading pipeline. It does not transform data but rather asserts a condition about it.

> Flow:
> Raw NPC Asset File (JSON) -> Deserializer -> BuilderObjectArrayHelper -> **ArrayNotEmptyValidator.test()** -> Validation Result (Boolean) -> Asset Builder Decision (Continue or Throw)

