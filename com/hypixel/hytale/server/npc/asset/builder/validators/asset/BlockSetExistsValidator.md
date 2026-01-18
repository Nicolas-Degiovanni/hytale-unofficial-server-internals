---
description: Architectural reference for BlockSetExistsValidator
---

# BlockSetExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Stateless Utility

## Definition
```java
// Signature
public class BlockSetExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The BlockSetExistsValidator is a specialized component within the server's asset processing framework. It functions as a concrete implementation of the **Strategy Pattern**, providing a single, focused validation rule that can be composed into a larger validation pipeline.

Its primary architectural role is to enforce data integrity at the asset level. When NPC or other game assets are defined in external files (e.g., JSON, HOCON), they often contain string-based references to other core assets, such as a BlockSet. This validator acts as a guard, ensuring that these references point to a valid, loaded BlockSet before the asset is fully constructed and integrated into the game.

This decouples the asset parsing and building logic from the asset registry. The builder does not need to know how to query the BlockSet asset map; it simply delegates the responsibility of existence-checking to this validator. This design prevents runtime errors, such as a NullPointerException, that would occur if an entity tried to use a non-existent BlockSet.

## Lifecycle & Ownership
-   **Creation:** Direct instantiation is prohibited via a private constructor. The class is accessed exclusively through its static factory methods. The primary method, `required()`, returns a shared, static singleton instance (**DEFAULT_INSTANCE**). The `withConfig()` method creates a new, transient instance with specific validation configurations.
-   **Scope:** The default singleton instance is global and persists for the entire server lifetime, created upon class loading. Instances created via `withConfig()` are transient and their lifetime is tied to the scope of the asset building process that requested them.
-   **Destruction:** The singleton instance is never destroyed. Transient instances are eligible for garbage collection once the asset validation process completes and they are no longer referenced.

## Internal State & Concurrency
-   **State:** The BlockSetExistsValidator is effectively immutable. The default instance is stateless. Configured instances hold an immutable `EnumSet` of configuration flags passed during creation. The core validation logic does not modify any internal fields.
-   **Thread Safety:** This class is thread-safe. Its `test` method performs a read-only operation against the global `BlockSet.getAssetMap()`. It is assumed that the underlying asset map is a thread-safe or concurrently readable collection, which is a standard requirement for engine-level asset registries. Multiple threads can safely invoke validation logic on any instance of this class without external locking.

## API Surface
The public contract is focused on object acquisition and the core validation test.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| required() | BlockSetExistsValidator | O(1) | Returns the shared, default singleton instance. This is the preferred method for standard validation. |
| withConfig(config) | BlockSetExistsValidator | O(1) | Creates and returns a new validator instance with specific behavioral flags. |
| test(blockSet) | boolean | O(1) | **Core Logic.** Returns true if a BlockSet with the given string name exists in the global asset map, false otherwise. |

## Integration Patterns

### Standard Usage
This validator is intended to be used by higher-level asset builders or parsers. The builder retrieves a validator instance and applies it to a specific field from an asset definition file.

```java
// Within a hypothetical AssetBuilder class
// The 'data' object represents parsed JSON/HOCON
String blockSetName = data.getString("spawnSurfaceBlock");

// Retrieve the validator and test the value
AssetValidator validator = BlockSetExistsValidator.required();
if (!validator.test(blockSetName)) {
    // Log the error and abort asset creation
    String error = validator.errorMessage(blockSetName, "spawnSurfaceBlock");
    log.error("Asset validation failed: " + error);
    return null; // Abort
}
// Proceed with asset construction...
```

### Anti-Patterns (Do NOT do this)
-   **Attempted Instantiation:** Do not attempt to create an instance using reflection. The class is designed to be used via its static factory methods to manage instance lifecycle correctly.
-   **Stateful Misconception:** Do not treat the validator as stateful. It is a pure function whose result depends only on its input and the global state of the asset registry. Caching its result externally is unsafe, as the asset registry could change.

## Data Pipeline
The BlockSetExistsValidator sits in the middle of the asset loading pipeline, acting as a gate for data correctness.

> Flow:
> Asset File (JSON) -> Server Asset Parser -> **BlockSetExistsValidator** -> Validation Result (Pass/Fail) -> Final Asset Instantiation

