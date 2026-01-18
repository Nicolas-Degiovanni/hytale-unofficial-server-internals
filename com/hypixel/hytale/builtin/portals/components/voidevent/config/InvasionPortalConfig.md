---
description: Architectural reference for InvasionPortalConfig
---

# InvasionPortalConfig

**Package:** com.hypixel.hytale.builtin.portals.components.voidevent.config
**Type:** Configuration Object / Data Transfer Object (DTO)

## Definition
```java
// Signature
public class InvasionPortalConfig {
```

## Architecture & Concepts
The InvasionPortalConfig class is a pure data container that represents the configuration for a server-side "Invasion Portal" world event. It is not a service or a manager; its sole responsibility is to provide a strongly-typed, in-memory representation of settings defined in an external asset file, likely JSON.

The key architectural feature is the static **CODEC** field, which leverages Hytale's serialization framework. This `BuilderCodec` defines the contract for how raw data from an asset file is mapped into an instance of this class. This pattern decouples the game logic that consumes the configuration from the underlying data format, allowing the configuration structure to evolve without breaking game code.

This object acts as a bridge between static configuration assets and dynamic game systems. Methods like `getBlockType` demonstrate this by transforming a simple string identifier (`blockKey`) into a fully realized, engine-aware `BlockType` object by querying the global asset registry.

## Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale `Codec` framework during the asset loading phase. The `BuilderCodec` reads a corresponding configuration asset and uses the private constructor to populate a new instance. Direct instantiation is not supported.
- **Scope:** The object's lifetime is tied to the system that loads and holds it, typically a manager responsible for the void event. It is effectively a read-only snapshot of the configuration, loaded once when the relevant game systems initialize.
- **Destruction:** The object is marked for garbage collection when the owning system is unloaded or the server shuts down. There are no explicit destruction or cleanup methods.

## Internal State & Concurrency
- **State:** The state of an InvasionPortalConfig instance is **effectively immutable**. Although its fields are not declared final, the design pattern of creation-via-deserialization with no public setters ensures its state is fixed upon loading. It serves as a read-only data source for other systems.
- **Thread Safety:** This class is **conditionally thread-safe for reads**. Accessing its properties from multiple threads is safe. However, the `getBlockType` method depends on the thread safety of the global `BlockType.getAssetMap`. Assuming the core asset registries are designed for concurrent reads (a standard engine pattern), this object can be safely shared across threads. **WARNING:** Any external attempt to mutate this object's state after creation will break this guarantee and lead to undefined behavior.

## API Surface
The public API is composed entirely of accessors for retrieving configuration values.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getBlockKey() | String | O(1) | Returns the raw string identifier for the portal's block asset. |
| getBlockType() | BlockType | O(log N) | Resolves the block key against the global asset map to return a live BlockType instance. Throws if the asset is not found. |
| getOnSpawnParticles() | String | O(1) | Returns the asset key for the particle system to play when the portal is created. May be null. |
| getSpawnBeacons() | String[] | O(1) | Returns the raw array of spawn beacon asset keys associated with the portal. May be null. |
| getSpawnBeaconsList() | List<String> | O(1) | Returns a null-safe List view of the spawn beacon asset keys. Returns an empty list if none are defined. |

## Integration Patterns

### Standard Usage
This configuration object is meant to be loaded by a higher-level manager system and passed to logic controllers that execute the world event.

```java
// Example: A theoretical VoidEventManager retrieves the config
InvasionPortalConfig portalConfig = assetManager.load("my_void_event.json", InvasionPortalConfig.CODEC);

// A spawner system uses the config to create a portal in the world
BlockType portalBlock = portalConfig.getBlockType();
world.setBlock(position, portalBlock);

particleSystem.spawn(position, portalConfig.getOnSpawnParticles());
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new InvasionPortalConfig()`. The constructor is private and reserved for the Codec system. Configuration must always originate from an asset file to ensure consistency.
- **State Mutation:** Do not use reflection or other means to modify the fields of this object after it has been loaded. Downstream systems rely on this configuration being a stable, read-only source of truth for the duration of their lifecycle.
- **Frequent Lookups:** Avoid calling `getBlockType` repeatedly within a tight loop. While fast, it is a map lookup. If a system needs the `BlockType` multiple times, it should call the method once and cache the result locally.

## Data Pipeline
This class is a destination in the asset loading pipeline. It transforms raw file data into a usable in-memory object.

> Flow:
> JSON Asset File -> Server Asset Loader -> **BuilderCodec Deserializer** -> **InvasionPortalConfig Instance** -> Game Logic (e.g., Event Spawner)

