---
description: Architectural reference for RepulsionConfig
---

# RepulsionConfig

**Package:** com.hypixel.hytale.server.core.modules.entity.repulsion
**Type:** Managed Data Model

## Definition
```java
// Signature
public class RepulsionConfig
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, RepulsionConfig>>,
   NetworkSerializable<com.hypixel.hytale.protocol.RepulsionConfig> {
```

## Architecture & Concepts
RepulsionConfig is a data-driven configuration object that defines the physical properties of the entity repulsion system. It is not an active component or service; rather, it is a passive data structure that encapsulates parameters like radius, minimum force, and maximum force.

Its primary architectural role is to decouple game logic from configuration data. Instead of hardcoding repulsion values within entity or physics code, systems reference a RepulsionConfig instance by its unique string identifier. This allows designers and developers to define, manage, and tune entity repulsion behaviors in external JSON asset files without modifying game code.

The class is deeply integrated with the Hytale Asset System. The static CODEC field, an AssetBuilderCodec, dictates how JSON data is deserialized into a RepulsionConfig object. The use of `appendInherited` within the codec definition reveals a powerful feature of the asset system: RepulsionConfig supports hierarchical data inheritance. A specific configuration can be defined as a variant of a parent, overriding only the necessary fields. This pattern is essential for creating consistent and maintainable asset libraries (e.g., a `giant_brute` config inheriting from a base `monster` config).

Furthermore, its implementation of the NetworkSerializable interface designates it as a Data Transfer Object (DTO) that can be efficiently synchronized from the server to game clients, ensuring that client-side predictions and server-side authority use identical parameters.

### Lifecycle & Ownership
- **Creation:** Instances are not intended to be manually instantiated via the `new` keyword in standard game logic. The Hytale AssetStore is the sole owner and creator of managed instances. During server initialization, the AssetStore discovers all repulsion configuration JSON files, and uses the static CODEC to deserialize them into RepulsionConfig objects. These objects are then cached in a global, static registry.
- **Scope:** An instance loaded from an asset is a global singleton that persists for the entire server session. It is stored within the static ASSET_STORE and is accessible from any system that requires it.
- **Destruction:** All managed instances are destroyed when the AssetRegistry is shut down or reloaded, typically when the server stops or a hot-reload is triggered.

## Internal State & Concurrency
- **State:** An instance of RepulsionConfig is **mutable**. Its fields are not final. However, once loaded from the AssetStore, it must be treated as a strictly **immutable** object. Modifying a globally-managed configuration at runtime will introduce unpredictable side effects and state corruption across all systems that reference it.
- **Thread Safety:** The class itself is not thread-safe. Direct access to its fields is not protected. The static `getAssetStore` method contains a lazy initialization pattern (`if (ASSET_STORE == null)`). This check is not atomic and can create a race condition if called concurrently from multiple threads during the server's bootstrap phase.

**WARNING:** All interaction with the static AssetStore and its contained RepulsionConfig instances must be confined to the main game thread to prevent concurrency violations. Asset loading should be completed before multi-threaded systems begin to operate.

## API Surface
The public API is minimal, focusing on retrieval from the global store and network serialization.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Returns the global, singleton registry for all RepulsionConfig assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | A convenience method to retrieve the underlying map of ID-to-config mappings from the AssetStore. |
| getId() | String | O(1) | Returns the unique asset identifier for this configuration instance (e.g., "hytale:creeper"). |
| toPacket() | com.hypixel.hytale.protocol.RepulsionConfig | O(1) | Creates and populates a network packet DTO with this configuration's data for client synchronization. |

## Integration Patterns

### Standard Usage
The correct pattern is to retrieve a pre-loaded, immutable configuration from the static asset map using its known string identifier. This object is then passed to systems that perform calculations.

```java
// In a physics or AI system, retrieve the globally managed config
IndexedLookupTableAssetMap<String, RepulsionConfig> configs = RepulsionConfig.getAssetMap();
RepulsionConfig monsterRepulsion = configs.get("hytale:large_monster");

if (monsterRepulsion != null) {
    // Use the immutable config data to perform calculations
    physicsSystem.applyRepulsionForce(entity, monsterRepulsion);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Do not use `new RepulsionConfig()` to access a defined asset. This bypasses the asset management system and creates a local, unmanaged copy. Always retrieve configurations from the global `getAssetMap`.
- **Runtime Modification:** Never modify the fields of a RepulsionConfig instance retrieved from the AssetStore. This is a global object, and changing it will affect every entity that uses that configuration, leading to severe and hard-to-debug bugs.

    ```java
    // DO NOT DO THIS
    RepulsionConfig config = RepulsionConfig.getAssetMap().get("hytale:some_config");
    config.radius = 999.0f; // This corrupts global state
    ```

## Data Pipeline
The flow of RepulsionConfig data begins with its definition in a file and ends with its consumption by a game system.

> Flow:
> JSON Asset File (`large_monster.json`) -> Server Startup -> AssetRegistry Loader -> **RepulsionConfig.CODEC** -> Instantiation -> Caching in **RepulsionConfig.ASSET_STORE** -> Game System requests by ID -> Physics Engine Calculation

---

