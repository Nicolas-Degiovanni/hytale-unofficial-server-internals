---
description: Architectural reference for BeaconSpawnExistsValidator
---

# BeaconSpawnExistsValidator

**Package:** com.hypixel.hytale.server.npc.asset.builder.validators.asset
**Type:** Utility

## Definition
```java
// Signature
public class BeaconSpawnExistsValidator extends AssetValidator {
```

## Architecture & Concepts
The BeaconSpawnExistsValidator is a concrete implementation of the *Strategy* pattern, specializing the generic AssetValidator interface. Its primary function is to enforce referential integrity within the server's asset system, specifically for NPC configurations.

This class acts as a runtime guard during the asset loading and building pipeline. When an NPC asset is defined with a reference to a beacon spawn point (by name), this validator is invoked to confirm that the referenced BeaconNPCSpawn asset actually exists in the server's central asset registry. This prevents the instantiation of invalid NPC assets with dangling pointers to non-existent spawn configurations, which would otherwise lead to runtime exceptions or unpredictable spawning behavior.

It is a critical component for ensuring data consistency and preventing server instability caused by misconfigured game data.

### Lifecycle & Ownership
-   **Creation:** Instantiation is strictly controlled via two static factory methods. The `required()` method provides a shared, stateless singleton instance for default validation. The `withConfig()` method constructs a new, transient instance for specialized validation scenarios. Direct instantiation is prohibited by a private constructor.
-   **Scope:** The default singleton instance, `DEFAULT_INSTANCE`, is static and persists for the entire lifetime of the server JVM. Instances created via `withConfig()` are ephemeral and their lifetime is scoped to the asset builder or process that requested them.
-   **Destruction:** The singleton instance is unloaded during JVM shutdown. Transient instances are eligible for garbage collection as soon as they are no longer referenced by the asset building pipeline.

## Internal State & Concurrency
-   **State:** The default instance returned by `required()` is stateless and immutable. Instances created via `withConfig()` contain a small, immutable `EnumSet` of configuration flags inherited from the parent AssetValidator. The class performs no caching; each validation check is a real-time lookup against the global `BeaconNPCSpawn` asset map.
-   **Thread Safety:** This component is fully thread-safe. Its immutability and reliance on read-only operations against the central asset registry ensure that a single instance can be safely shared and executed by multiple concurrent asset-loading threads without locks or synchronization.

    **Warning:** The thread safety of this validator is contingent upon the thread safety of the underlying `BeaconNPCSpawn.getAssetMap()`. It is assumed this map is populated during a single-threaded server bootstrap phase and becomes effectively immutable thereafter.

## API Surface
The public contract is minimal and focused on the validation operation.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| test(String beacon) | boolean | O(1) | Executes the validation logic. Returns true if the asset exists, false otherwise. |
| required() | BeaconSpawnExistsValidator | O(1) | Returns the shared, default singleton instance of the validator. |
| withConfig(EnumSet) | BeaconSpawnExistsValidator | O(1) | Creates and returns a new validator instance with specific configuration. |
| errorMessage(String, String) | String | O(1) | Generates a formatted, human-readable error message for a failed validation. |

## Integration Patterns

### Standard Usage
This validator is intended to be used within a higher-level asset building or parsing system. It is typically chained with other validators to fully vet an incoming asset definition.

```java
// Within an asset builder or configuration parser
AssetBuilder builder = new AssetBuilder();
String beaconNameToValidate = "village_center_spawn";

// Retrieve the shared validator and apply it to a field
builder.addField("spawnPoint", beaconNameToValidate)
       .addValidator(BeaconSpawnExistsValidator.required());

// The builder will later invoke the test() method internally
ValidationResult result = builder.validate();
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not attempt to instantiate this class with `new BeaconSpawnExistsValidator()`. The constructor is private to enforce the use of the `required()` and `withConfig()` factory methods. This pattern ensures instance control and promotes the use of the shared singleton.
-   **Premature Validation:** Do not invoke this validator before the server's main asset loading phase is complete. Calling `test()` before the `BeaconNPCSpawn` asset map has been fully populated will result in false negatives, incorrectly rejecting valid configurations.

## Data Pipeline
The validator sits in the middle of the data flow from raw configuration files to in-memory game objects, acting as a gatekeeper for data quality.

> Flow:
> NPC Definition File (JSON/HOCON) -> Server Asset Parser -> **BeaconSpawnExistsValidator.test()** -> Validation Gate (Pass/Fail) -> In-Memory NPC Asset Creation

