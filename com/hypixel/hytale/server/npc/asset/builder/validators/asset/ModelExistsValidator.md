---
description: Architectural reference for ModelExistsValidator
---

# ModelExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility

## Definition
```java
// Signature
public class ModelExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The ModelExistsValidator is a specific implementation of the **Strategy** pattern, inheriting from the abstract AssetValidator. Its primary function is to enforce referential integrity within the server's asset system. It acts as a gatekeeper, ensuring that any reference to a model asset by name corresponds to a model that has been successfully loaded into memory.

This validator is a critical component of the NPC and entity asset building pipeline. Before an entity's definition is considered valid and usable by the game server, it passes through a series of validators. This particular validator prevents the server from attempting to spawn entities with missing visual representations, which would otherwise result in runtime exceptions, invisible characters, or client-side crashes. It decouples the entity definition logic from the low-level asset management system by providing a simple, testable contract.

## Lifecycle & Ownership
- **Creation:** Instantiation is controlled exclusively through two static factory methods:
    1. **required()**: Returns a shared, static, default-configured singleton instance. This is the most common and efficient way to use the validator.
    2. **withConfig(EnumSet)**: Returns a new, transient instance with specific validation configurations. This is used for specialized cases where the default validation behavior needs to be altered.
- **Scope:** The default singleton instance returned by required() is application-scoped and persists for the entire lifetime of the server. Transient instances created by withConfig() are scoped to their creator, typically a short-lived asset builder or validation process.
- **Destruction:** The singleton instance is reclaimed by the JVM when its class loader is unloaded. Transient instances are eligible for garbage collection as soon as they are no longer referenced.

## Internal State & Concurrency
- **State:** The default singleton instance is **stateless**. Instances created via the withConfig factory hold a small, immutable EnumSet of configuration flags. The core validation logic does not modify any internal fields and relies entirely on the external state of the global ModelAsset registry.

- **Thread Safety:** This class is conditionally thread-safe. The validator itself contains no mutable state or locks. However, its correctness depends entirely on the state of the static `ModelAsset.getAssetMap()`. It is safe to use from multiple threads **only after** the server's initial asset loading phase is complete and the asset map is effectively immutable or thread-safe for reads.

    **Warning:** Invoking this validator from a separate thread while the main thread is still loading or modifying model assets will lead to non-deterministic behavior and race conditions. Validation should occur in a single-threaded context or after all assets are guaranteed to be loaded.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String model) | boolean | O(1) | The core validation method. Performs a lookup in the global model asset map. |
| errorMessage(String model, String attributeName) | String | O(1) | Generates a formatted, human-readable error message for failed validations. |
| required() | ModelExistsValidator | O(1) | Factory method to retrieve the shared, default singleton instance. |
| withConfig(EnumSet) | ModelExistsValidator | O(1) | Factory method to create a new validator with custom configuration. |

## Integration Patterns

### Standard Usage
This validator is intended to be used as part of a larger validation chain, typically within an asset builder or a configuration loader. The builder would retrieve the validator and apply it to the relevant data field.

```java
// Hypothetical usage within an NpcDefinitionBuilder
public class NpcDefinitionBuilder {
    private List<AssetValidator> validators;
    private NpcData data;

    public NpcDefinitionBuilder() {
        this.validators.add(ModelExistsValidator.required());
        // ... add other validators
    }

    public ValidationResult build() {
        // ...
        boolean modelIsValid = ModelExistsValidator.required().test(data.getModelName());
        if (!modelIsValid) {
            return ValidationResult.failure(
                ModelExistsValidator.required().errorMessage(data.getModelName(), "model")
            );
        }
        // ...
        return ValidationResult.success();
    }
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to instantiate this class using reflection. Always use the static `required()` or `withConfig()` factory methods.
- **Validation During Asset Loading:** Do not use this validator in a process that runs concurrently with the main asset loading sequence. A model may not yet be in the `ModelAsset` map when the validator checks, causing a false negative and failing a perfectly valid entity definition.

## Data Pipeline
The validator serves as a simple predicate in a larger data processing flow, transforming a string identifier into a boolean validation result.

> Flow:
> Raw Asset File (e.g., JSON) -> Deserializer -> Entity Data Object -> **ModelExistsValidator.test(modelName)** -> Validation Report -> Finalized Server Asset

