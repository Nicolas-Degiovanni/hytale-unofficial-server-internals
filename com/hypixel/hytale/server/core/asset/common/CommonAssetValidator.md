---
description: Architectural reference for CommonAssetValidator
---

# CommonAssetValidator

**Package:** com.hypixel.hytale.server.core.asset.common
**Type:** Utility / Immutable

## Definition
```java
// Signature
public class CommonAssetValidator implements Validator<String> {
```

## Architecture & Concepts
The CommonAssetValidator is a foundational component of the Hytale Codec and Schema framework. It serves as a specialized, pluggable rule engine for validating asset path strings during data deserialization and configuration loading. Its primary architectural role is to enforce strict conventions and guarantee the integrity of asset references *before* they are passed to the lower-level AssetManager.

This class acts as a gatekeeper, ensuring that any asset path defined in configuration files (such as JSON or other schema-driven formats) adheres to three core principles:
1.  **Directory Scoping:** The asset resides within a permitted root directory (e.g., a character texture must be in the Characters or NPC folder).
2.  **Type Correctness:** The asset has the correct file extension for its intended use (e.g., models must be blockymodel files).
3.  **Existence:** The asset is registered and known to the CommonAssetRegistry, preventing broken links at runtime.

By integrating directly with the schema system via the `updateSchema` method, it allows these validation rules to be declared declaratively alongside data definitions. This makes the validation process transparent to developers and centralizes asset convention enforcement, preventing scattered validation logic throughout the codebase. The class provides a large set of pre-configured static instances (e.g., TEXTURE\_ITEM, MODEL\_CHARACTER) which represent the canonical validation rules for most engine asset types.

## Lifecycle & Ownership
-   **Creation:** The vast majority of validators are static singletons, created at class-loading time (e.g., `CommonAssetValidator.TEXTURE_ITEM`). They are not instantiated by user code. Custom instances may be created via `new` for bespoke validation scenarios, but this is uncommon.
-   **Scope:** The static instances are global and persist for the entire application lifetime. They are effectively immortal process-level constants. Any transient, custom-created instances are scoped to their owner, typically a Schema definition object, and are eligible for garbage collection once the schema is no longer in use.
-   **Destruction:** Static instances are unloaded when the JVM shuts down. There is no manual cleanup mechanism.

## Internal State & Concurrency
-   **State:** This class is **strictly immutable**. Its internal fields, `requiredRoots`, `requiredExtension`, and `isUIAsset`, are final and set only during construction. It holds no mutable state and caches no data.
-   **Thread Safety:** The CommonAssetValidator is **inherently thread-safe**. Its immutability guarantees that its internal state cannot be corrupted by concurrent access. The `accept` method is a pure function with respect to the instance, operating only on its immutable fields and input parameters. It can be safely shared and invoked from any thread without external locking.

    **Warning:** Thread safety assumes that its primary dependency, the CommonAssetRegistry, is also thread-safe for read operations (`hasCommonAsset`).

## API Surface
The public contract is defined by the `Validator` interface.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| accept(asset, results) | void | O(N) | Executes the validation logic against the provided asset string. N is the number of configured `requiredRoots`. Populates the ValidationResults object with failures. |
| updateSchema(context, target) | void | O(1) | Injects validation metadata into a target StringSchema. This enables tools and editors to understand the asset constraints. |

## Integration Patterns

### Standard Usage
The CommonAssetValidator is not intended to be called directly. It should be attached to a StringSchema as part of a larger data structure definition. The Hytale Codec framework will then invoke it automatically during validation passes.

```java
// Correctly defining a schema that uses pre-configured validators
// This code would exist in a class that defines configuration formats.

ObjectSchema entityConfigSchema = new ObjectSchema()
    .withField("texturePath",
        new StringSchema().withValidator(CommonAssetValidator.TEXTURE_CHARACTER)
    )
    .withField("modelPath",
        new StringSchema().withValidator(CommonAssetValidator.MODEL_CHARACTER)
    )
    .withField("footstepSounds",
        new ArraySchema(new StringSchema().withValidator(CommonAssetValidator.SOUNDS))
    );

// The framework later uses this schema to validate a loaded config file.
// You do not call validator.accept() yourself.
```

### Anti-Patterns (Do NOT do this)
-   **Redundant Instantiation:** Do not create new instances when a static one exists. This is wasteful and bypasses the engine's standardized constants.
    ```java
    // BAD: Creates a new object unnecessarily
    Validator<String> myValidator = new CommonAssetValidator("png", "Characters", "NPC", "Items", "VFX");

    // GOOD: Uses the canonical, shared instance
    Validator<String> myValidator = CommonAssetValidator.TEXTURE_CHARACTER;
    ```
-   **Manual Invocation:** Do not call the `accept` method directly to perform one-off validation. This decouples the validation from the data definition it is meant to protect. If you need to check an asset, use the appropriate high-level service or registry.

## Data Pipeline
The validator functions as a step within a larger data deserialization and validation pipeline. It does not transform data but rather asserts its correctness.

> Flow:
> Raw Config File (e.g., JSON) -> Hytale Codec Deserializer -> Schema Validation Step -> **CommonAssetValidator.accept(path)** -> Populated ValidationResults -> Application Logic (Accepts data or logs error)

