---
description: Architectural reference for AnyBooleanValidator
---

# AnyBooleanValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class AnyBooleanValidator extends Validator {
```

## Architecture & Concepts
The AnyBooleanValidator is a specific, stateless rule implementation within the server's NPC asset validation framework. Its sole responsibility is to enforce a logical OR condition across a set of boolean attributes, ensuring that at least one of them is true.

This class embodies the Strategy pattern, where it represents a single, concrete validation algorithm that can be composed by a higher-level validation engine. The design cleanly separates the pure validation logic (the static `test` method) from the contextual data required for user-facing error reporting (the `attributes` field). This separation allows the core logic to be tested in isolation while instances of the validator provide rich, context-aware error messages during the asset build process.

It is designed to be a lightweight, immutable value object, created and used during the asset loading and validation phase.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively through the static factory methods `withAttributes`. This is typically done by an asset parser or a builder system that translates a declarative asset definition (e.g., from a JSON file) into a set of executable validation rules.
- **Scope:** Transient. An AnyBooleanValidator instance exists only for the duration of a single asset validation cycle. It holds no long-term state and is not intended to persist beyond the initial load and verification of an NPC asset.
- **Destruction:** The object is eligible for garbage collection as soon as the validation process that created it completes. There is no manual cleanup required.

## Internal State & Concurrency
- **State:** Immutable. The internal `attributes` array is final and is assigned only once during construction via a private constructor. Once an AnyBooleanValidator is created, its state cannot be modified.
- **Thread Safety:** This class is inherently thread-safe. Its immutability guarantees that it can be safely shared, passed between, and used by multiple threads simultaneously without locks or other synchronization primitives. The static methods are pure functions and are also completely thread-safe.

## API Surface
The public contract is minimal and focused on instance creation and error message generation. The core logic is exposed via a static `test` method.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| withAttributes(String...) | static AnyBooleanValidator | O(1) | Factory method to construct a validator instance. This is the required entry point. |
| test(boolean[]) | static boolean | O(N) | The core validation logic. Returns true if any value in the input array is true. |
| errorMessage() | String | O(M) | Generates a formatted, human-readable error message based on the attributes provided at creation. |

## Integration Patterns

### Standard Usage
The validator is intended to be created by a builder or factory and used by a validation runner. The runner is responsible for extracting the boolean values from an asset and passing them to the static `test` method. If the test fails, the runner retrieves the contextual error message from the instance.

```java
// In a hypothetical AssetBuilder...

// 1. Define the validation rule
String[] requiredFlags = {"isAggressive", "canFly", "isBoss"};
AnyBooleanValidator rule = AnyBooleanValidator.withAttributes(requiredFlags);

// 2. Extract values from the asset being processed
boolean[] assetValues = {
    asset.isAggressive(), 
    asset.canFly(), 
    asset.isBoss()
};

// 3. Execute validation
if (!AnyBooleanValidator.test(assetValues)) {
    throw new AssetValidationException(rule.errorMessage());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private to enforce the use of the named static factory methods. Do not attempt to use reflection to create instances.
- **Stateful Wrappers:** Do not wrap this validator in a stateful class. Its value comes from its immutability and transient nature. Storing it long-term provides no benefit.
- **Ignoring Instance for Error Messages:** While the static `test` method is public, relying on it alone and writing custom error messages defeats the purpose of the class. The instance provides standardized, context-aware error strings.

## Data Pipeline
The AnyBooleanValidator acts as a gate within the NPC asset loading pipeline. It does not transform data but rather asserts conditions on it.

> Flow:
> NPC Asset File (JSON) -> Asset Deserializer -> Validation Rule Set (contains **AnyBooleanValidator** instance) -> Validation Engine -> Validated Asset or Exception

