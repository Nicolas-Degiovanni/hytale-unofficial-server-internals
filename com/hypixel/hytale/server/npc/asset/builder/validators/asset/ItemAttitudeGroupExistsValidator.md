---
description: Architectural reference for ItemAttitudeGroupExistsValidator
---

# ItemAttitudeGroupExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility / Singleton Factory

## Definition
```java
// Signature
public class ItemAttitudeGroupExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The ItemAttitudeGroupExistsValidator is a specialized component within the server's NPC asset loading and validation pipeline. Its sole responsibility is to confirm that a given string identifier corresponds to a legally registered ItemAttitudeGroup asset.

This class embodies the **Strategy Pattern**. It is a concrete implementation of the abstract AssetValidator, designed to be plugged into a higher-level asset builder or parser. The asset building framework maintains a collection of such validators, iterating through them to ensure the integrity and correctness of NPC configuration data before it is loaded into the game world.

The validator's logic is decoupled from the asset parsing itself. It performs its check by querying the central asset registry for ItemAttitudeGroup, effectively acting as a guard clause against configuration errors that reference non-existent game data. This prevents runtime errors and ensures data consistency across the server.

### Lifecycle & Ownership
-   **Creation:** Instances are not created directly via a public constructor. They are acquired exclusively through one of two static factory methods:
    1.  **ItemAttitudeGroupExistsValidator.required()**: Returns a shared, static, default-configured singleton instance. This is the most common and efficient way to use the validator.
    2.  **ItemAttitudeGroupExistsValidator.withConfig(config)**: Creates a new, transient instance with specific validation behaviors (e.g., null handling) defined by the provided Config enum. This is used for specialized validation scenarios.
-   **Scope:** The default singleton instance returned by `required()` is static and persists for the entire server session. Instances created by `withConfig()` are transient and are typically scoped to the lifetime of the specific asset builder that requested them.
-   **Destruction:** The singleton instance is unloaded when the class is garbage collected during server shutdown. Transient instances are garbage collected once the asset loading process completes and they are no longer referenced.

## Internal State & Concurrency
-   **State:** This class is effectively stateless. The default singleton instance holds no mutable state. Instances created with custom configurations hold a final, immutable EnumSet of configuration flags. The validation logic itself depends on the external, shared state of the `ItemAttitudeGroup.getAssetMap()`.

-   **Thread Safety:** The class is inherently thread-safe due to its immutability. The `test` method's safety is contingent on the thread safety of the underlying `ItemAttitudeGroup.getAssetMap()`. Assuming the asset map is populated once at server startup and is subsequently treated as read-only, this validator can be safely used in parallel asset loading systems without explicit locking.

## API Surface
The public API is primarily for consumption by the asset building framework, not for direct use in game logic.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(attitudeGroup) | boolean | O(1) | The core validation logic. Returns true if an ItemAttitudeGroup with the given name exists in the asset registry. |
| errorMessage(name, attr) | String | O(1) | Generates a detailed, human-readable error message for failed validations. |
| getDomain() | String | O(1) | Returns "ItemAttitudeGroup", identifying the data domain this validator operates on. |
| required() | ItemAttitudeGroupExistsValidator | O(1) | Factory method to retrieve the shared, default singleton instance. |
| withConfig(config) | ItemAttitudeGroupExistsValidator | O(1) | Factory method to create a new instance with custom validation rules. |

## Integration Patterns

### Standard Usage
This validator is not intended to be invoked directly. It is registered declaratively as part of a larger asset definition schema, which is then processed by an asset builder.

```java
// Hypothetical usage within an asset building system
// A developer would configure a schema, not call the validator directly.

AssetSchema npcSchema = new AssetSchema();

// The framework uses the validator to check the 'defaultAttitude' field
npcSchema.addField("defaultAttitude", String.class)
         .withValidator(ItemAttitudeGroupExistsValidator.required());

// The AssetBuilder would later use this schema to validate raw data
AssetBuilder builder = new AssetBuilder(npcSchema);
builder.build(npcJsonData); // Throws ValidationException on failure
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to call `new ItemAttitudeGroupExistsValidator()`. The constructor is private and usage is enforced through the static factory methods.
-   **Manual Invocation:** Avoid calling the `test` method directly in game logic. The validator's purpose is for data validation at load-time, not for runtime checks. Rely on the asset system to have already sanitized the data.

## Data Pipeline
The validator functions as a single, critical step in the server's asset ingestion pipeline.

> Flow:
> Raw NPC Definition (JSON/HOCON) -> Asset Parser -> **ItemAttitudeGroupExistsValidator** -> Central Asset Registry -> Validated NPC Object in Memory

