---
description: Architectural reference for ItemDropListExistsValidator
---

# ItemDropListExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility

## Definition
```java
// Signature
public class ItemDropListExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The ItemDropListExistsValidator is a specialized component within the server's asset validation framework. Its primary function is to enforce referential integrity between NPC assets and ItemDropList assets. During the NPC asset building process, this validator ensures that any string key referencing an ItemDropList corresponds to an actual, loaded asset in the system.

This class acts as a critical guard, preventing the creation of NPCs with invalid loot tables. By failing the asset build early, it avoids potential runtime exceptions or silent game logic failures that would occur if an NPC attempted to use a non-existent drop list.

Architecturally, it decouples the NPC definition from the concrete ItemDropList implementation by relying on a string-based key. The validator's responsibility is to bridge this indirection and confirm the link is valid. It delegates the actual existence check to the InventoryHelper utility, demonstrating a clear separation of concerns: the validator defines the validation *contract* and *context*, while the helper provides the mechanism for querying the underlying asset registry.

### Lifecycle & Ownership
-   **Creation:** The class is not intended for direct instantiation via its constructor, which is private. It is acquired through one of two static factory methods:
    1.  **ItemDropListExistsValidator.required()**: Returns a shared, static, default instance. This is the most common usage.
    2.  **ItemDropListExistsValidator.withConfig()**: Creates a new, transient instance with specific validation configurations.
    The entity responsible for configuring the NPC asset build process is the owner of this object.

-   **Scope:** The default instance, retrieved via required(), is a static singleton that persists for the entire lifetime of the server process. Instances created via withConfig() are ephemeral and their lifetime is typically scoped to a single asset build operation.

-   **Destruction:** The static default instance is reclaimed by the JVM when its classloader is unloaded. Transient instances are eligible for garbage collection as soon as the asset build process that created them is complete.

## Internal State & Concurrency
-   **State:** The default instance is stateless and immutable. Instances created with custom configuration hold a small, immutable EnumSet defining their behavior. The core validation logic does not modify any internal fields during execution.

-   **Thread Safety:** This class is inherently thread-safe. Its immutable nature and lack of internal mutable state make it safe to use across multiple threads, which is essential for a parallelized asset loading system. All validation logic is delegated to InventoryHelper, which is assumed to be a thread-safe system utility. No external locking is required by the caller.

## API Surface
The public API is focused on providing a validation predicate and context for error reporting.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String value) | boolean | O(1) | The core validation method. Returns true if an ItemDropList with the given key exists. |
| errorMessage(String value, String attribute) | String | O(1) | Generates a formatted, human-readable error message for a failed validation. |
| getDomain() | String | O(1) | Returns the asset domain this validator operates on, which is "ItemDropList". |
| required() | ItemDropListExistsValidator | O(1) | Static factory method to retrieve the shared, default validator instance. |
| withConfig(EnumSet config) | ItemDropListExistsValidator | O(1) | Static factory method to create a new validator with specific configuration flags. |

## Integration Patterns

### Standard Usage
This validator is intended to be used declaratively within an asset builder's validation chain. The builder invokes the validator for the relevant asset attribute.

```java
// Within an Asset Builder configuration
// The builder would retrieve the validator and apply it.

AssetValidator validator = ItemDropListExistsValidator.required();
boolean isValid = validator.test("some_droplist_key_from_asset_file");

if (!isValid) {
    throw new AssetBuildException(validator.errorMessage("some_droplist_key_from_asset_file", "lootTable"));
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to instantiate this class using reflection. The private constructor and static factory methods enforce a controlled lifecycle. Always use `ItemDropListExistsValidator.required()` or `ItemDropListExistsValidator.withConfig()`.

-   **Bypassing the Validator:** Developers should not call `InventoryHelper.itemDropListKeyExists` directly for validation purposes. Using the validator ensures consistent error messaging, configuration handling, and adherence to the broader asset validation framework.

## Data Pipeline
This component acts as a gate in a data validation pipeline, not a transformation stage. It receives a string key and outputs a boolean decision that controls the flow of the asset build process.

> Flow:
> NPC Asset File (JSON) -> Asset Parser -> NPC Asset Builder -> **ItemDropListExistsValidator.test(key)** -> [Build Halts on False | Build Continues on True]

