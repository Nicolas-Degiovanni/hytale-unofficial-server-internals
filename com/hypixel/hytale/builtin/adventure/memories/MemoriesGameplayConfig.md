---
description: Architectural reference for MemoriesGameplayConfig
---

# MemoriesGameplayConfig

**Package:** com.hypixel.hytale.builtin.adventure.memories
**Type:** Configuration Model

## Definition
```java
// Signature
public class MemoriesGameplayConfig {
```

## Architecture & Concepts
MemoriesGameplayConfig is a strongly-typed data model that represents a specific block of configuration for the "Memories" adventure feature. It is not a service or a manager; its sole purpose is to hold validated configuration data loaded from game assets.

The core architectural element is the static `CODEC` field. This `BuilderCodec` instance provides a declarative schema for serialization and deserialization, effectively decoupling the data object from the underlying file format (e.g., JSON). The codec defines the expected keys, data types, and, critically, the validation rules that must pass for an instance to be created.

This class is designed to be a modular, plug-in configuration. It is not a standalone entity but is instead retrieved from a parent `GameplayConfig` object. This pattern allows gameplay features to define their own configuration structures without modifying the core engine, which then aggregates them into a single, cohesive configuration tree.

The use of `appendInherited` within the codec definition indicates that this configuration supports a hierarchical override system. Values can be defined in a parent configuration asset and will be inherited by child configurations unless explicitly overridden. Furthermore, the `Item.VALIDATOR_CACHE.getValidator()` demonstrates a powerful cross-system validation pattern, ensuring at load-time that the configured item ID is a valid and known `Item` in the game.

## Lifecycle & Ownership
-   **Creation:** Instances are created exclusively by the engine's asset loading system via the defined `BuilderCodec`. The codec reads data from a source file, validates it, and populates a new `MemoriesGameplayConfig` object. Manual instantiation is an anti-pattern and will result in a non-functional object.
-   **Scope:** The lifecycle of a `MemoriesGameplayConfig` instance is strictly tied to its parent `GameplayConfig` object. It persists as long as the parent configuration is held in memory, typically for the duration of a world or server session.
-   **Destruction:** The object is marked for garbage collection when the parent `GameplayConfig` is unloaded or replaced, such as during a server shutdown or world change.

## Internal State & Concurrency
-   **State:** The object's state is established once during creation by the codec. While the class is technically mutable (lacking final fields), it should be treated as **effectively immutable** after initialization. There are no public setters, and its state is not intended to change during runtime.
-   **Thread Safety:** This class is **not thread-safe**. It is a plain data object with no internal synchronization. However, as it is intended for read-only access after its creation on a main or asset-loading thread, it can be safely read by multiple threads without contention.

**WARNING:** Do not modify the state of this object via reflection after it has been loaded. This will lead to unpredictable behavior and desynchronization with the source configuration asset.

## API Surface
The public API is minimal, focusing entirely on data retrieval.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| get(GameplayConfig config) | MemoriesGameplayConfig | O(1) | Static factory method to retrieve the config from a parent GameplayConfig. Returns null if not present. |
| getMemoriesAmountPerLevel() | int[] | O(1) | Returns the array defining the number of memories required per level. |
| getMemoriesRecordParticles() | String | O(1) | Returns the asset identifier for the particle effect to play when a memory is recorded. |
| getMemoriesCatchItemId() | String | O(1) | Returns the item identifier used to "catch" or interact with memories. |

## Integration Patterns

### Standard Usage
The only correct way to obtain an instance of this class is by retrieving it from an existing, loaded `GameplayConfig` object.

```java
// Assume 'gameplayConfig' is provided by the engine context
MemoriesGameplayConfig memoriesConfig = MemoriesGameplayConfig.get(gameplayConfig);

if (memoriesConfig != null) {
    String particleId = memoriesConfig.getMemoriesRecordParticles();
    // ... use the configuration data in a gameplay system
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Never call `new MemoriesGameplayConfig()`. This bypasses the codec, validation, and data-loading pipeline, resulting in an object with null fields that will cause `NullPointerException`s.
-   **State Mutation:** Do not attempt to modify the object's fields after it has been loaded. Configuration should be treated as a read-only source of truth at runtime. Changes should be made in the source asset files and reloaded by the server.

## Data Pipeline
The data for this object originates from a gameplay configuration file on disk and is processed by the engine's asset and codec systems before being made available to gameplay logic.

> Flow:
> Gameplay Config File (.json) -> Engine AssetManager -> `BuilderCodec` Deserialization & Validation -> **MemoriesGameplayConfig Instance** (attached to parent `GameplayConfig`) -> Gameplay System Access<ctrl63>

