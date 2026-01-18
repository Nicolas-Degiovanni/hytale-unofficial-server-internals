---
description: Architectural reference for IntSequenceValidator
---

# IntSequenceValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators
**Type:** Utility

## Definition
```java
// Signature
public class IntSequenceValidator extends IntArrayValidator {
```

## Architecture & Concepts
The **IntSequenceValidator** is a specialized, immutable component within the server-side asset validation framework. Its primary function is to enforce complex integrity rules on integer arrays found in asset definitions, particularly for Non-Player Characters (NPCs).

This validator operates as a predicate, evaluating two conditions simultaneously:
1.  **Boundary Constraints:** Every integer within the sequence must adhere to a defined numerical range (e.g., greater than 0 and less than or equal to 100).
2.  **Sequential Relationship:** Each element in the sequence must have a specific mathematical relationship to the preceding element (e.g., strictly increasing, weakly decreasing).

By combining these checks, it prevents the loading of malformed NPC asset data that could lead to server instability or undefined game behavior. It is a low-level building block used by higher-level asset parsers and builders to guarantee data correctness at the point of ingestion, before the data is consumed by the core game logic.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively through a set of static factory methods, such as **between** or **between01Monotonic**. The constructor is private to enforce controlled and predictable instantiation. Common validators are pre-allocated as static singletons (e.g., **VALIDATOR_BETWEEN_01_MONOTONIC**) for performance and memory efficiency.
- **Scope:** Transient and stateless. An instance is typically created on-the-fly by an asset loader, used to perform one or more validation checks, and then becomes eligible for garbage collection. The pre-allocated static instances persist for the entire application lifetime.
- **Destruction:** Managed entirely by the Java garbage collector. No manual cleanup or resource release is required.

## Internal State & Concurrency
- **State:** **Immutable**. All internal fields defining the validation logic (e.g., **lower**, **upper**, **relationSequence**) are final and set only during construction. Once an **IntSequenceValidator** is created, its behavior is fixed and cannot be altered.
- **Thread Safety:** **Fully thread-safe**. Due to its immutable nature, a single instance can be safely shared and used concurrently across multiple threads without any need for external synchronization or locks. This is critical in a multi-threaded server environment where assets may be loaded and validated in parallel.

## API Surface
The public contract is composed of static factory methods for instantiation and instance methods for validation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| between(lower, upper) | static IntSequenceValidator | O(1) | Creates a validator for a sequence where all values must be between the bounds, inclusively. |
| betweenMonotonic(lower, upper) | static IntSequenceValidator | O(1) | Creates a validator for a strictly increasing sequence within the specified bounds. |
| betweenWeaklyMonotonic(l, u) | static IntSequenceValidator | O(1) | Creates a validator for a non-decreasing sequence within the specified bounds. |
| test(int[] values) | boolean | O(N) | Executes the validation logic against the provided array. Returns true if valid, false otherwise. N is the length of the array. |
| errorMessage(int[] value, String name) | String | O(N) | Generates a detailed, human-readable error message explaining why validation failed. |

## Integration Patterns

### Standard Usage
This component is intended to be used within a broader asset loading or configuration pipeline. The typical pattern is to select a factory, create the validator, and use it to guard against invalid data.

```java
// Example: In an NPC asset loader
int[] keyframeTimestamps = parsedAsset.getAnimationTimestamps();

// Define the validation rule: timestamps must be between 0 and 10000,
// and must be strictly increasing.
IntSequenceValidator validator = IntSequenceValidator.betweenMonotonic(0, 10000);

if (!validator.test(keyframeTimestamps)) {
    // Reject the asset and provide a clear error for the content creator
    String error = validator.errorMessage(keyframeTimestamps, "animation.timestamps");
    throw new AssetLoadException(error);
}

// Proceed with asset processing...
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to use reflection to create instances; always use the provided static factory methods.
- **Manual Iteration:** Do not re-implement the validation logic in a separate loop. The **test** method is optimized and encapsulates the complete rule set.
- **Ignoring Error Messages:** Do not discard the output of **errorMessage**. It is designed to provide precise feedback to content designers, significantly reducing debugging time for asset-related issues. A generic "Validation Failed" error is insufficient.

## Data Pipeline
The **IntSequenceValidator** acts as a gatekeeper in the data flow from raw asset files to the in-memory game state.

> Flow:
> Asset File (JSON/XML) -> Server Asset Parser -> Raw `int[]` -> **IntSequenceValidator.test()** -> Boolean Gate -> (On Fail: Reject & Log) -> (On Pass: Processed Game Asset)

