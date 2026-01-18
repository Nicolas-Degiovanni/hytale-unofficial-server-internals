---
description: Architectural reference for ObjectiveExistsValidator
---

# ObjectiveExistsValidator

**Package:** com.hypixel.hytale.builtin.adventure.npcobjectives.npc.validators
**Type:** Utility / Factory

## Definition
```java
// Signature
public class ObjectiveExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The ObjectiveExistsValidator is a specialized component within the server's asset loading and validation framework. Its sole responsibility is to ensure data integrity for assets, particularly NPC configurations, that reference game objectives.

This class acts as a guard clause during the asset deserialization process. When the server loads an NPC's definition from a data file, this validator is invoked to check if any string fields designated as objective identifiers correspond to an actual, loaded ObjectiveAsset. This prevents the server from instantiating misconfigured NPCs that would later cause runtime exceptions or undefined behavior due to missing objective data.

It is a critical link in the chain of asset validation, ensuring that relationships between different asset types (NPCs and Objectives) are valid before they are committed to the game world's memory.

### Lifecycle & Ownership
-   **Creation:** Instances are created exclusively through the static factory methods *required()* or *withConfig(EnumSet)*. The *required()* method provides a shared, stateless singleton instance, promoting efficiency. These methods are typically called by higher-level asset definition builders when constructing a validation pipeline for a specific asset type.
-   **Scope:** The default singleton instance returned by *required()* is static and persists for the entire server session. Instances created via *withConfig* are transient, owned by the specific validation rule set they are part of, and are garbage collected once the asset loading process completes.
-   **Destruction:** The singleton instance is reclaimed upon JVM shutdown. Transient instances are subject to standard garbage collection.

## Internal State & Concurrency
-   **State:** This class is stateless. It holds no mutable instance fields. Its validation logic depends on the external, static state of the global ObjectiveAsset asset map.
-   **Thread Safety:** The class is inherently thread-safe. As it contains no mutable state, instances can be safely shared across multiple threads. The core *test* method performs a read-only operation against the static *ObjectiveAsset.getAssetMap()*.

    **Warning:** The overall thread safety of the validation process is contingent on the thread safety of the underlying asset map. It is assumed that the asset map is populated in a single-threaded context during server initialization and is treated as immutable thereafter, making concurrent reads safe.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String objective) | boolean | O(1) | Executes the validation check. Returns true if an ObjectiveAsset exists for the given string key. |
| errorMessage(String, String) | String | O(1) | Generates a formatted, human-readable error message for a failed validation. |
| required() | ObjectiveExistsValidator | O(1) | Static factory method to retrieve the shared, default singleton instance. |
| withConfig(EnumSet) | ObjectiveExistsValidator | O(1) | Static factory method to create a new validator with specific configuration flags. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct use in gameplay systems. It is designed to be integrated into the asset building and validation pipeline, typically when defining the schema for an NPC or another entity that references objectives.

```java
// Example within a hypothetical asset builder
AssetBuilder npcBuilder = new AssetBuilder("MyNPC");

// The builder uses the validator to check the 'onDeathObjective' field
npcBuilder.addField("onDeathObjective", String.class)
          .withValidator(ObjectiveExistsValidator.required());

// The asset system later uses this configured builder to parse a file
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** The constructor is private and should not be accessed. Always use the static *required()* or *withConfig()* factory methods.
-   **Runtime Game Logic:** Do not use this validator to check for objectives during the game loop. It is a load-time data integrity tool. For runtime checks, query the appropriate asset manager or registry directly. Using this class at runtime is inefficient and semantically incorrect.

## Data Pipeline
The validator operates as a single step within the broader asset loading pipeline. It receives a string identifier and produces a boolean result that dictates the success or failure of the asset's validation phase.

> Flow:
> NPC Asset File (JSON/HOCON) -> Server Asset Parser -> **ObjectiveExistsValidator.test(objectiveName)** -> Global ObjectiveAsset Map -> Validation Result (Pass/Fail)

