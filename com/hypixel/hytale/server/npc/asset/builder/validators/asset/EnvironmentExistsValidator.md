---
description: Architectural reference for EnvironmentExistsValidator
---

# EnvironmentExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility / Factory

## Definition
```java
// Signature
public class EnvironmentExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The EnvironmentExistsValidator is a specialized component within the server's asset validation framework. Its sole responsibility is to confirm that a given string identifier corresponds to a fully loaded and registered Environment asset.

This class operates as a rule or a predicate within a larger asset processing pipeline, most likely an AssetBuilder responsible for parsing and validating NPC definition files. By encapsulating this single piece of logic, it decouples the complex asset parsing system from the specifics of how Environment assets are managed and registered.

The core of its operation relies on querying the global asset registry via the static method `Environment.getAssetMap().getAsset()`. This design implies that the validator is invoked at a stage where all prerequisite assets, specifically Environment assets, have already been discovered, loaded, and indexed into memory. It acts as a gatekeeper, preventing the creation of invalid NPC assets that reference non-existent game world environments.

## Lifecycle & Ownership
-   **Creation:** The class is not designed to be managed by a dependency injection container. A default, stateless instance is provided via the static `DEFAULT_INSTANCE` field, which is created at class-loading time. Configured instances are created transiently by calling the static factory method `withConfig`.
-   **Scope:** The `DEFAULT_INSTANCE` is a static singleton and persists for the entire lifetime of the server application. Instances created via `withConfig` are short-lived and typically scoped to the duration of a single asset validation operation.
-   **Destruction:** The `DEFAULT_INSTANCE` is eligible for garbage collection only when its class loader is unloaded. Transient instances are garbage collected as soon as they are no longer referenced by the asset validation process.

## Internal State & Concurrency
-   **State:** The class is effectively stateless. The default instance holds no state. Instances created with configuration hold an immutable `EnumSet` of configuration flags. The validation logic itself does not modify any internal fields; it relies entirely on the external, shared state of the global `Environment.getAssetMap`.
-   **Thread Safety:** This class is thread-safe. The `test` method is a pure function with respect to the class instance. Its safety during concurrent execution is dependent on the thread safety of the underlying `Environment.getAssetMap()`, which, as a central registry, is expected to support concurrent read operations. This allows the validator to be safely used in parallel asset building systems without requiring external locks.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String envName) | boolean | O(1) | Checks if an Environment asset with the given name exists in the global asset map. |
| errorMessage(String, String) | String | O(1) | Generates a formatted, human-readable error message for a validation failure. |
| required() | EnvironmentExistsValidator | O(1) | Static factory method to retrieve the shared, default instance. |
| withConfig(EnumSet) | EnvironmentExistsValidator | O(1) | Static factory method to create a new validator with specific configuration flags. |

## Integration Patterns

### Standard Usage
This validator is not intended to be used imperatively. It is designed to be supplied to a higher-level asset building or validation framework that applies it to specific fields during asset deserialization.

```java
// Hypothetical usage within an AssetBuilder definition
AssetBuilder builder = new AssetBuilder();

// The validator is registered to check the "spawnEnvironment" field of an NPC asset
builder.forField("spawnEnvironment")
       .addValidator(EnvironmentExistsValidator.required());

// The builder framework later invokes test() internally
builder.build(npcAssetFile);
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructor is private. Do not attempt to create an instance using `new EnvironmentExistsValidator()`. Always use the static factory methods `required()` or `withConfig()`.
-   **Premature Validation:** Do not invoke the `test` method before the server's asset loading phase is complete. Calling this validator before all `Environment` assets are loaded into the `Environment.getAssetMap` will result in false negatives and incorrect validation failures.

## Data Pipeline
The validator functions as a single step in a larger data validation and transformation pipeline for server assets.

> Flow:
> NPC Asset File (JSON/HOCON) -> Server Asset Parser -> Field Value (`String`) -> **EnvironmentExistsValidator.test()** -> Validation Result (boolean) -> Asset Builder Logic -> Final In-Memory Asset Object or Build Error

