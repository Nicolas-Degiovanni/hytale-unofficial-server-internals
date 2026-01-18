---
description: Architectural reference for SpawnConfig
---

# SpawnConfig

**Package:** com.hypixel.hytale.server.core.asset.type.gameplay
**Type:** Transient Data Object

## Definition
```java
// Signature
public class SpawnConfig {
```

## Architecture & Concepts
SpawnConfig is a passive, data-driven configuration object. It does not contain any logic; its sole purpose is to represent the particle effect settings associated with a spawning event, as defined in an external asset file (e.g., a JSON or HOCON file).

The most critical architectural feature of this class is the public static **CODEC** field. This `BuilderCodec` instance acts as the schema and deserialization engine for the class. The Hytale asset loading system uses this codec to automatically parse raw configuration data and hydrate a SpawnConfig instance. This pattern decouples the game logic from the specifics of the configuration file format, allowing designers to modify spawn effects without requiring code changes.

This class is a terminal node in the asset graph; it holds data but does not manage other systems. It is typically a component of a larger configuration object, such as an entity or item definition.

### Lifecycle & Ownership
- **Creation:** Instances are created exclusively by the Hytale `Codec` framework during the asset loading phase. The static CODEC field is queried by the AssetManager, which then uses it to construct and populate the object from a data source. Direct manual instantiation is an anti-pattern.
- **Scope:** The lifetime of a SpawnConfig instance is bound to its parent asset. It is loaded into memory when the parent asset is loaded and persists as long as that parent asset is registered and active.
- **Destruction:** The object is marked for garbage collection when its parent asset is unloaded from the AssetManager. There is no manual destruction or cleanup method.

## Internal State & Concurrency
- **State:** Mutable. SpawnConfig is a simple Plain Old Java Object (POJO). Its fields are populated by the codec system after the default constructor is called. The internal `WorldParticle` arrays are also mutable.
- **Thread Safety:** This class is **not thread-safe**. It is designed to be populated on a single asset-loading thread. Once loaded, it should be treated as an immutable, read-only object by all consumer systems (e.g., the entity spawning system). Any external modification after initialization will result in race conditions and undefined behavior.

## API Surface
The public API is minimal, consisting only of accessors for the configured data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getFirstSpawnParticles() | WorldParticle[] | O(1) | Returns the array of particles for the initial spawn event. May return null if not defined in the asset. |
| getSpawnParticles() | WorldParticle[] | O(1) | Returns the array of particles for subsequent spawn events. May return null if not defined in the asset. |

## Integration Patterns

### Standard Usage
A developer should never create or manage a SpawnConfig instance directly. Instead, it is retrieved from a higher-level configuration object that has been loaded by the asset system.

```java
// Example: Retrieving spawn config from a loaded monster asset
MonsterAsset monster = assetManager.get("hytale:monster_t1");
SpawnConfig spawnConfig = monster.getSpawnConfig();

if (spawnConfig != null && spawnConfig.getFirstSpawnParticles() != null) {
    particleEngine.playEffects(entity.getPosition(), spawnConfig.getFirstSpawnParticles());
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new SpawnConfig()`. The object will be empty and bypass the entire data-driven asset pipeline. This is a critical violation of the engine's design.
- **Post-Load Modification:** Do not modify the arrays returned by the getters. Configuration should be treated as immutable once loaded. Changes must be made in the source asset files and reloaded through the engine.

## Data Pipeline
SpawnConfig is a destination in the asset loading pipeline. It transforms structured text data into a strongly-typed Java object for consumption by game systems.

> Flow:
> Asset File on Disk (e.g., `mob.json`) -> AssetManager -> **SpawnConfig.CODEC** -> **SpawnConfig Instance** -> Entity Spawning System

