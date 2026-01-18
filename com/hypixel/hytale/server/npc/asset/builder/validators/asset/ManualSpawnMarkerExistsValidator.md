---
description: Architectural reference for ManualSpawnMarkerExistsValidator
---

# ManualSpawnMarkerExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility

## Definition
```java
// Signature
public class ManualSpawnMarkerExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The ManualSpawnMarkerExistsValidator is a specialized implementation of the AssetValidator strategy. Its primary function is to enforce data integrity during the server's asset loading and configuration parsing phases. It acts as a gatekeeper, ensuring that string identifiers meant to reference a SpawnMarker asset are not only valid but also point to a specific *type* of SpawnMarker—one configured for manual triggering.

This component is critical within the NPC and world-building systems. When a designer or developer defines an entity or script that references a spawn point by name, this validator is invoked at load-time to confirm two conditions:
1.  A SpawnMarker asset with the given name actually exists in the central asset registry.
2.  The existing SpawnMarker has its `isManualTrigger` flag set to true.

By performing this check upfront, the system prevents latent runtime errors, such as an NPC failing to spawn because its designated spawn point is either missing or misconfigured. It is a key part of a fail-fast data validation pipeline.

### Lifecycle & Ownership
- **Creation:** The validator is instantiated exclusively through its static factory methods. The `required()` method provides a shared, stateless singleton instance (`DEFAULT_INSTANCE`), promoting efficiency. The `withConfig()` method creates a new, transient instance for specialized validation rules, though this is less common for this specific validator. The `DEFAULT_INSTANCE` is created once at class-loading time.
- **Scope:** The default singleton instance is application-scoped and persists for the entire server lifetime. Transient instances created via `withConfig()` are scoped to the validation process that creates them and are eligible for garbage collection once that process completes.
- **Destruction:** The singleton instance is destroyed when the Java ClassLoader unloads the class, typically during server shutdown. Transient instances are managed by the garbage collector.

## Internal State & Concurrency
- **State:** This class is effectively stateless. The default instance holds no mutable state. Instances created with custom configurations hold a final, immutable `EnumSet` of configuration flags inherited from the parent class. The core validation logic in the `test` method reads from a global, static asset map but does not modify any internal fields.
- **Thread Safety:** The class is thread-safe. Its methods are pure functions with respect to its own state. The safety of the `test` method is contingent on the thread-safety of the global asset registry (`SpawnMarker.getAssetMap()`). Standard engine design dictates that asset registries are safe for concurrent reads, making this validator safe for use in multi-threaded asset loading pipelines.

## API Surface
| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String marker) | boolean | O(1) | The core validation logic. Returns true if a manual SpawnMarker with the given name exists. |
| errorMessage(String marker, String attributeName) | String | O(1) | Generates a detailed, human-readable error message for validation failures. |
| getDomain() | String | O(1) | Returns the asset domain this validator operates on, which is "SpawnMarker". |
| required() | ManualSpawnMarkerExistsValidator | O(1) | Factory method to retrieve the shared, default singleton instance. |
| withConfig(EnumSet) | ManualSpawnMarkerExistsValidator | O(1) | Factory method to create a new instance with specific validation configurations. |

## Integration Patterns

### Standard Usage
This validator is not intended to be used directly in game logic. Instead, it is supplied to a higher-level configuration parser or asset builder, which then invokes it as part of a larger validation chain.

```java
// Hypothetical usage within an asset builder
AssetBuilder builder = new AssetBuilder();
String spawnMarkerNameFromConfig = "my_dungeon_entry_marker";

// The builder uses the validator to check the field
builder.addField("spawnOn", spawnMarkerNameFromConfig)
       .withValidator(ManualSpawnMarkerExistsValidator.required());

// The builder's process() method would internally call test()
builder.process();
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** The constructor is private. Do not attempt to create an instance using `new ManualSpawnMarkerExistsValidator()`. Always use the static `required()` or `withConfig()` factory methods.
- **Incorrect Validator Choice:** Do not use this validator if you need to check for the existence of *any* SpawnMarker, regardless of its trigger type. Using it in such a case will cause valid, non-manual spawn markers to be incorrectly rejected.
- **Runtime Validation:** This component is designed for load-time validation. Using it for frequent, per-frame checks in the main game loop would be inefficient, as it involves lookups in a global map.

## Data Pipeline
The validator sits in the middle of the asset ingestion pipeline, acting as a predicate to filter or halt the process based on data validity.

> Flow:
> Raw Config File (e.g., JSON, HOCON) -> Server Asset Parser -> String value for a spawn marker -> **ManualSpawnMarkerExistsValidator.test()** -> Global SpawnMarker Asset Registry -> Boolean Result (Valid/Invalid) -> Asset Construction or Error Reporting

