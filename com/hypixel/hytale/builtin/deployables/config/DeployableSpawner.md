---
description: Architectural reference for DeployableSpawner
---

# DeployableSpawner

**Package:** com.hypixel.hytale.builtin.deployables.config
**Type:** Data Asset

## Definition
```java
// Signature
public class DeployableSpawner implements JsonAssetWithMap<String, DefaultAssetMap<String, DeployableSpawner>> {
```

## Architecture & Concepts
The DeployableSpawner class is not a service or manager, but a pure **data asset** that represents the configuration for spawning a "deployable" entity. It serves as a read-only blueprint, deserialized from JSON files, that dictates the properties and placement of in-world objects like traps or turrets.

Its primary role is to bridge design-time configuration with runtime game logic. Game designers define spawner behaviors in external JSON files, and the engine's Asset System uses the static **CODEC** field within this class to load and parse those files into strongly-typed DeployableSpawner instances.

The class implements the JsonAssetWithMap interface, signaling its integration with the global **AssetRegistry**. The static `getAssetMap` method provides a centralized, service-locator-style access point to the complete collection of all loaded spawner configurations, indexed by their unique string identifiers.

## Lifecycle & Ownership
- **Creation:** Instances are not created manually using the `new` keyword. They are instantiated and populated exclusively by the Hytale **AssetRegistry** during the engine's asset loading phase. The static `CODEC` field dictates the entire deserialization and object construction process from a JSON source.
- **Scope:** The lifecycle of a DeployableSpawner instance is directly coupled to the AssetRegistry. Once loaded, an instance persists in a static, shared map for the entire duration of the game session (client or server). It is effectively a global, immutable configuration object.
- **Destruction:** Instances are eligible for garbage collection only when the AssetRegistry is cleared or reloaded. This typically occurs during major state transitions, such as shutting down the game or connecting to a different server with a new asset manifest.

## Internal State & Concurrency
- **State:** Instances are effectively **immutable** post-initialization. All fields are populated once by the CODEC during asset loading and are not designed to be mutated at runtime. The class exclusively provides getters, enforcing a read-only contract. The static `ASSET_MAP` field is mutable for lazy initialization but the collection it holds should be treated as a read-only cache.

- **Thread Safety:** The object is **conditionally thread-safe**.
    - **Instance Reads:** Reading data from an already-loaded instance (e.g., calling `getConfig`) is safe from any thread due to its immutable nature.
    - **Static Access:** The `getAssetMap` method contains a potential race condition in its lazy initialization block. If multiple threads access this method for the first time simultaneously before `ASSET_MAP` is initialized, it may result in multiple lookups against the AssetRegistry. While the AssetRegistry is expected to be robust, this pattern lacks explicit synchronization.

    **Warning:** All systems should ensure that asset loading is complete on the main thread before attempting to access `getAssetMap` from worker threads.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetMap() | static DefaultAssetMap | O(1) | Retrieves the global, cached map of all DeployableSpawner assets. The first call has higher latency as it queries the AssetRegistry. |
| getConfig() | DeployableConfig | O(1) | Returns the detailed configuration object for the deployable entity. |
| getPositionOffsets() | Vector3d[] | O(1) | Returns the array of relative 3D positions where entities can be spawned. |
| getId() | String | O(1) | Returns the unique asset identifier for this spawner configuration. |

## Integration Patterns

### Standard Usage
The correct pattern is to retrieve the global asset map and then look up a specific spawner configuration by its known ID.

```java
// Retrieve the central map of all spawner configurations
DefaultAssetMap<String, DeployableSpawner> spawnerMap = DeployableSpawner.getAssetMap();

// Look up a specific spawner by its asset ID (e.g., "hytale:stone_trap_spawner")
DeployableSpawner spawnerConfig = spawnerMap.get("hytale:stone_trap_spawner");

// Use the configuration data if it exists
if (spawnerConfig != null) {
    DeployableConfig config = spawnerConfig.getConfig();
    Vector3d[] offsets = spawnerConfig.getPositionOffsets();

    // Game logic uses the config and offsets to spawn an entity in the world...
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never call `new DeployableSpawner()`. This bypasses the asset system entirely, resulting in an unmanaged, uninitialized object that will cause NullPointerExceptions and is invisible to the rest of the engine.
- **Modifying Returned Collections:** The array returned by `getPositionOffsets` is a direct reference to the internal state of a shared asset. Modifying its contents will have unpredictable side effects across the entire application. Treat all returned data as strictly read-only.
- **Early Access:** Do not call `getAssetMap` during early engine initialization phases before the AssetRegistry has completed its loading cycle. Doing so will likely return a null or empty map.

## Data Pipeline
The flow for this class is one of asset ingestion, not continuous data processing.

> Flow:
> JSON Asset File on Disk -> AssetRegistry Loader -> **DeployableSpawner.CODEC** (Deserialization) -> In-Memory **DeployableSpawner** Instance -> Game Logic (Consumer)

