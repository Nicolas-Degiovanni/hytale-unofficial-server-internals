---
description: Architectural reference for EntityStatType
---

# EntityStatType

**Package:** com.hypixel.hytale.server.core.modules.entitystats.asset
**Type:** Data Asset

## Definition
```java
// Signature
public class EntityStatType
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, EntityStatType>>,
   NetworkSerializable<com.hypixel.hytale.protocol.EntityStatType> {
```

## Architecture & Concepts

The EntityStatType class is a data-driven asset that serves as a blueprint for all entity statistics within the game engine, such as health, mana, or stamina. It is not a live instance of a stat on an entity; rather, it is the immutable definition from which live stats (e.g., EntityStatValue) are created and configured.

This class is a central component of the Hytale Asset System. Instances are not created programmatically via constructors but are deserialized from JSON definition files at server startup. The static field **CODEC** of type AssetBuilderCodec dictates the entire serialization contract, defining how JSON keys map to the object's fields. This codec-based approach allows for powerful features like data inheritance from parent JSON assets, validation, and post-processing.

Architecturally, EntityStatType acts as a configuration provider for the entity-component-system (ECS). When an entity's stat component is initialized, it retrieves the corresponding EntityStatType from the global AssetStore to determine the stat's initial value, minimum/maximum bounds, regeneration behavior, and any special effects that trigger at boundary conditions.

## Lifecycle & Ownership

The lifecycle of an EntityStatType instance is managed exclusively by the Hytale Asset System. Developers should never attempt to manage its lifecycle manually.

-   **Creation:** Instances are created by the AssetStore during the server's initial bootstrap phase. The static **CODEC** is invoked to parse and deserialize all `*.json` files corresponding to entity stat definitions. The constructor is public for reflection-based instantiation by the codec but should be considered internal.
-   **Scope:** An EntityStatType object is a flyweight. Once loaded, a single instance (e.g., for "hytale:health") is shared globally and persists for the entire server session. It is referenced by all game systems and entity components that need to interact with that specific stat's definition.
-   **Destruction:** Instances are garbage collected only upon server shutdown when the AssetRegistry and its associated AssetStores are cleared.

## Internal State & Concurrency

-   **State:** The object is effectively immutable after deserialization. All its definitional fields (min, max, initialValue, etc.) are set once by the asset loader and are not designed to be modified at runtime. The only mutable field is `cachedPacket`, a SoftReference used for performance optimization when serializing the object for network transmission. This cache is an internal detail and does not affect the logical immutability of the stat's definition.

-   **Thread Safety:** The class is thread-safe for read operations, which is its primary use case after the initial loading phase. Due to its immutable nature, any thread can safely access its properties without synchronization.

    **WARNING:** The static `getAssetStore` method contains a potential race condition if called concurrently by multiple threads before the store is initialized. However, the engine's design ensures that asset loading is a single-threaded, blocking operation during startup, mitigating this risk in practice. Systems operating at runtime should assume the AssetStore is already safely published.

## API Surface

The public API is designed for read-only access to the stat's configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetStore() | AssetStore | O(1) | **STATIC.** Retrieves the global asset store for all EntityStatType definitions. |
| getId() | String | O(1) | Returns the unique asset identifier, e.g., "hytale:health". |
| getInitialValue() | float | O(1) | Returns the default starting value for this stat. |
| getMin() | float | O(1) | Returns the absolute minimum value for this stat. |
| getMax() | float | O(1) | Returns the absolute maximum value for this stat. |
| isShared() | boolean | O(1) | Returns true if this stat's value should be shared with clients. |
| getRegenerating() | Regenerating[] | O(1) | Returns the array of regeneration rules, or null if none. |
| toPacket() | protocol.EntityStatType | O(1) | Converts the definition into a network-serializable packet. Uses an internal cache. |

## Integration Patterns

### Standard Usage

EntityStatType definitions should never be instantiated directly. They are always retrieved by their unique string key from the global AssetStore.

```java
// Correctly retrieve the definition for the "health" stat.
AssetStore<String, EntityStatType, ?> store = EntityStatType.getAssetStore();
EntityStatType healthDefinition = store.getAsset("hytale:health");

if (healthDefinition != null) {
    float startingHealth = healthDefinition.getInitialValue();
    // Use the definition to configure an entity's health component.
}
```

### Anti-Patterns (Do NOT do this)

-   **Direct Instantiation:** Never call `new EntityStatType()`. The object will be uninitialized and invalid. The asset system is the sole authority for creating these objects.
    ```java
    // ANTI-PATTERN: Results in a useless, empty object.
    EntityStatType invalidStat = new EntityStatType();
    ```

-   **State Modification:** Do not attempt to use reflection or other means to modify the fields of a loaded EntityStatType. These objects are shared globally, and mutation will lead to unpredictable behavior and severe bugs across the entire server.

## Data Pipeline

The primary data flow for EntityStatType is during the asset loading phase. At runtime, it serves as a source of configuration data.

> **Loading Flow:**
> JSON File on Disk -> Asset System -> **AssetBuilderCodec (in EntityStatType)** -> Deserialization & Validation -> In-Memory Instance in AssetStore

> **Runtime Usage Flow:**
> Game Logic (e.g., Entity Spawn) -> AssetStore.getAsset("stat_id") -> **EntityStatType Instance** -> Read Properties (min, max, etc.) -> Configure Live Component (e.g., EntityStatValue)

