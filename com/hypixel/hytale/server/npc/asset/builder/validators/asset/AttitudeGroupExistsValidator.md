---
description: Architectural reference for AttitudeGroupExistsValidator
---

# AttitudeGroupExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility

## Definition
```java
// Signature
public class AttitudeGroupExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The AttitudeGroupExistsValidator is a specialized component within the server's NPC asset loading and validation pipeline. Its sole responsibility is to confirm that a string identifier for an AttitudeGroup corresponds to a valid, registered asset in the game's asset management system.

This class embodies the **Strategy Pattern**, where it represents a specific validation algorithm for the generic AssetValidator contract. It is used by higher-level systems, such as an NPC Asset Builder, to guarantee data integrity during asset deserialization. By checking for the existence of referenced assets *before* an NPC is fully constructed, the system can fail fast with clear error messages, preventing runtime exceptions and corrupted game state caused by missing or misspelled asset names in configuration files.

This validator acts as a crucial link between raw configuration data (e.g., from JSON files) and the live, in-memory asset registry.

### Lifecycle & Ownership
- **Creation:** Instances are exclusively created via two static factory methods. The `required()` method returns a shared, static singleton instance for default validation. The `withConfig(EnumSet)` method constructs a new, transient instance for specialized validation scenarios. These methods are typically invoked by the asset building framework, not by general game logic code.
- **Scope:** The default singleton instance, returned by `required()`, is static and persists for the entire lifetime of the server. Instances created via `withConfig` are short-lived and scoped to the specific validation operation they are part of.
- **Destruction:** The singleton instance is reclaimed by the JVM upon server shutdown. Transient instances are eligible for garbage collection as soon as the asset validation process completes.

## Internal State & Concurrency
- **State:** This class is effectively stateless. It holds no mutable fields. The core validation logic in the `test` method does not alter any internal state; it performs a read-only query against the global `AttitudeGroup` asset map. Instances created with a configuration hold an immutable `EnumSet`.
- **Thread Safety:** The class is inherently thread-safe. Its stateless nature ensures that multiple threads can call its methods concurrently without risk of data corruption or race conditions. This is critical for parallel asset loading systems, where multiple NPC configurations may be validated simultaneously on different worker threads. The safety of the `test` method relies on the assumption that the underlying `AttitudeGroup.getAssetMap()` is also thread-safe.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String attitudeGroup) | boolean | O(1) | Executes the validation check. Queries the central asset map for the given group name. |
| errorMessage(String, String) | String | O(1) | Generates a formatted, human-readable error message for a failed validation. |
| required() | AttitudeGroupExistsValidator | O(1) | Factory method to retrieve the shared, default singleton instance. |
| withConfig(EnumSet) | AttitudeGroupExistsValidator | O(1) | Factory method to create a new instance with specific validation configurations. |

## Integration Patterns

### Standard Usage
This validator is not intended for direct use in gameplay code. It is designed to be plugged into a larger asset definition and building framework.

```java
// Conceptual example within an asset builder
// The builder would retrieve the validator and apply it to an input field.

String npcAttitudeField = "hostile_but_undefined"; // From a JSON file
AssetValidator validator = AttitudeGroupExistsValidator.required();

if (!validator.test(npcAttitudeField)) {
    String error = validator.errorMessage(npcAttitudeField, "attitude");
    throw new AssetLoadException(error);
}
// ... continue building the NPC asset
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not attempt to instantiate this class with `new`. The constructors are private for a reason. Always use the static `required()` or `withConfig()` factory methods.
- **Unnecessary Allocation:** Do not call `withConfig()` with an empty configuration set if the default behavior is sufficient. Prefer the `required()` method to use the shared singleton and avoid unnecessary object creation.

## Data Pipeline
The validator functions as a gate in the data flow from raw configuration files to finalized, in-memory game assets.

> Flow:
> NPC Definition File (JSON) -> Deserializer -> **AttitudeGroupExistsValidator** -> NPC Asset Builder -> Game State

