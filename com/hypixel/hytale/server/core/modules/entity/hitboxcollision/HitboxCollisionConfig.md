---
description: Architectural reference for HitboxCollisionConfig
---

# HitboxCollisionConfig

**Package:** com.hypixel.hytale.server.core.modules.entity.hitboxcollision
**Type:** Asset Model

## Definition
```java
// Signature
public class HitboxCollisionConfig
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, HitboxCollisionConfig>>,
   NetworkSerializable<com.hypixel.hytale.protocol.HitboxCollisionConfig> {
```

## Architecture & Concepts
The HitboxCollisionConfig class is a data-driven asset model that defines the physical properties of an entity's hitbox. It is not a service or manager, but rather the deserialized representation of a configuration file, typically JSON, loaded from disk.

This class is a cornerstone of Hytale's **Asset System**. Its primary role is to provide a strongly-typed, in-memory representation of collision rules that can be accessed globally by any game system. The static CODEC field declaratively defines the mapping between raw data fields in an asset file and the object's properties, including support for property inheritance from parent assets.

Crucially, this class implements NetworkSerializable. This signifies its dual role in the architecture:
1.  **Server-Side:** Used by the physics and entity management systems to perform authoritative collision detection and response.
2.  **Client-Side:** A network-optimized version of this object is sent to clients, enabling predictive movement and accurate rendering of collision effects without requiring the client to have direct access to server-side asset files.

The static AssetStore provides a global, singleton-like cache for all loaded HitboxCollisionConfig instances, ensuring that configuration data is loaded only once and is efficiently accessible throughout the application lifecycle.

## Lifecycle & Ownership
-   **Creation:** Instances are not created directly using the new keyword. They are instantiated and populated exclusively by the Hytale AssetStore during the engine's bootstrap or asset-loading phase. The static CODEC is used to decode the source data (e.g., a JSON file) into a fully-formed Java object.
-   **Scope:** An instance of HitboxCollisionConfig, once loaded into the AssetStore, persists for the entire application session. It is effectively a global, immutable singleton for that specific configuration ID.
-   **Destruction:** Objects are dereferenced and eligible for garbage collection only when the AssetRegistry is shut down, which typically occurs on server or client exit.

## Internal State & Concurrency
-   **State:** The object is **effectively immutable** post-initialization. Its fields are populated once by the AssetStore during deserialization and are not designed to be modified at runtime. All public methods are accessors (getters) or converters.
-   **Thread Safety:** The object is **thread-safe for reads**. Due to its immutable nature, any thread can safely access its properties without synchronization.

    **Warning:** The lazy initialization pattern in the static getAssetStore method is not inherently thread-safe. It assumes that the first call will happen from a single-threaded context during application startup. Concurrent first-time access from multiple threads could lead to a race condition, potentially creating multiple AssetStore instances.

## API Surface

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | static AssetStore | O(1) | Retrieves the global, cached store for all HitboxCollisionConfig assets. |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Retrieves the direct map of all loaded assets, keyed by their string ID. |
| getId() | String | O(1) | Returns the unique asset identifier for this configuration. |
| getCollisionType() | CollisionType | O(1) | Returns the primary collision behavior, e.g., Hard or Soft. |
| getSoftOffsetRatio() | float | O(1) | Returns the offset ratio used for resolving Soft collisions. |
| toPacket() | com.hypixel.hytale.protocol.HitboxCollisionConfig | O(1) | Serializes the object into a network-optimized protocol buffer for transmission to a client. |

## Integration Patterns

### Standard Usage
Game logic, such as a physics system, should retrieve a specific configuration from the global asset map using its known identifier.

```java
// Retrieve the map of all hitbox configurations
IndexedLookupTableAssetMap<String, HitboxCollisionConfig> configs = HitboxCollisionConfig.getAssetMap();

// Get the specific configuration for a player entity
HitboxCollisionConfig playerHitbox = configs.get("hytale:player");

if (playerHitbox != null) {
    CollisionType type = playerHitbox.getCollisionType();
    // ... apply physics logic based on the collision type
}
```

### Anti-Patterns (Do NOT do this)
-   **Direct Instantiation:** Do not use `new HitboxCollisionConfig()`. This creates a detached, uninitialized object that is not registered with the global AssetStore. Other game systems will not be able to find it, and it will lack essential data populated from the asset files.
-   **State Modification:** Do not attempt to modify the state of a HitboxCollisionConfig object after it has been loaded, for example, through reflection. The system is designed around the assumption that these configurations are immutable at runtime.

## Data Pipeline
The data for a HitboxCollisionConfig object follows a clear path from disk to the network.

> **Asset Loading Flow:**
> JSON File on Disk -> AssetStore Loader -> **HitboxCollisionConfig.CODEC** -> Deserialized HitboxCollisionConfig Instance -> Global AssetStore Cache

> **Network Serialization Flow:**
> Server Game Logic -> Retrieves **HitboxCollisionConfig** from Cache -> toPacket() -> NetworkSerializable Packet -> Network Layer -> Client


