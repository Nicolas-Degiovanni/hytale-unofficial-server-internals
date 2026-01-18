---
description: Architectural reference for EntityEffect
---

# EntityEffect

**Package:** com.hypixel.hytale.server.core.asset.type.entityeffect.config
**Type:** Asset / Data Transfer Object

## Definition
```java
// Signature
public class EntityEffect
   implements JsonAssetWithMap<String, IndexedLookupTableAssetMap<String, EntityEffect>>,
   NetworkSerializable<com.hypixel.hytale.protocol.EntityEffect> {
```

## Architecture & Concepts
The EntityEffect class is a data-driven configuration object that defines the properties and behaviors of a status effect, buff, or debuff within the game world. It serves as an immutable template, loaded from JSON asset files at server startup, rather than representing a live, active effect on a specific entity.

Its primary role is to centralize the definition of effects like poison, regeneration, or magical shields. Game logic systems, such as the combat or magic modules, query the central asset registry for an EntityEffect instance by its unique string identifier to apply its defined behaviors to game entities.

The deserialization from JSON is governed by the static **CODEC** field, an instance of AssetBuilderCodec. This complex codec defines the mapping between JSON keys and the fields of the EntityEffect class. It supports asset inheritance, allowing for the creation of effect variants that build upon a base definition, reducing data duplication.

Upon deserialization, the **afterDecode** hook is executed. This critical step transforms raw, human-readable data (like string-based stat names) into optimized, runtime-ready formats (like integer-indexed maps), preparing the object for high-performance access during gameplay.

Furthermore, by implementing NetworkSerializable, this class can be converted into a lightweight network packet for transmission to clients. This allows the client to display relevant UI information, such as status icons and names, without needing to possess the full server-side configuration.

## Lifecycle & Ownership
- **Creation:** EntityEffect instances are exclusively instantiated and populated by the Hytale **AssetStore** during the server's initial boot sequence. The static CODEC is used to parse corresponding JSON files (e.g., `poison.json`) into memory. Direct instantiation by developers is an anti-pattern and will result in a non-functional object.
- **Scope:** Once loaded, an EntityEffect object is globally accessible via the AssetRegistry and persists for the entire lifetime of the server session. It is considered static game data.
- **Destruction:** All loaded EntityEffect instances are garbage collected when the server process terminates and the AssetRegistry is cleared.

## Internal State & Concurrency
- **State:** The object's state is considered **effectively immutable** after the `afterDecode` lifecycle hook completes. While the fields are not declared as final, they are designed to be written to once during asset loading and treated as read-only thereafter.
- **State Caching:** The class performs significant post-processing to optimize its data for runtime use. For example, it resolves string identifiers for sounds and entity stats into more efficient integer indices. It also maintains a **SoftReference** to a cached network packet representation of itself to reduce the overhead of repeated serialization.
- **Thread Safety:** The object is **thread-safe for read operations** due to its immutable nature post-initialization. Game logic can safely access its properties from any thread.

    **Warning:** The lazy initialization of the static STORE field in `getAssetStore` and the packet cache in `toPacket` are not protected by synchronization locks. While the asset system is typically initialized on a single main thread at startup, making this safe in practice, concurrent calls to `toPacket` on a newly loaded effect could theoretically result in redundant packet creation.

## API Surface
The public API consists primarily of static accessors for the global registry and getters for the effect's configuration data.

| Symbol | Type | Complexity | Description |
| :--- | :--- | :--- | :--- |
| getAssetMap() | static IndexedLookupTableAssetMap | O(1) | Provides access to the global, read-only map of all loaded EntityEffect assets, keyed by their string ID. |
| toPacket() | com.hypixel.hytale.protocol.EntityEffect | O(1) amortized | Serializes the asset into a network packet for client transmission. Utilizes an internal cache for high performance. |
| getStatModifiers() | Int2ObjectMap | O(1) | Returns the map of entity statistics modifiers applied by this effect, using optimized integer keys. |
| getDuration() | float | O(1) | Returns the default duration of the effect in seconds. |
| getOverlapBehavior() | OverlapBehavior | O(1) | Defines how the game should handle reapplying this effect to an entity that already has it. |

## Integration Patterns

### Standard Usage
An EntityEffect should always be retrieved from the global asset map using its registered ID. It is then used as a data source for game logic.

```java
// Retrieve the master configuration for the "poison" effect
IndexedLookupTableAssetMap<String, EntityEffect> effectMap = EntityEffect.getAssetMap();
EntityEffect poisonConfig = effectMap.get("hytale:poison");

if (poisonConfig != null) {
    // Use the configuration to apply the effect to a player
    player.getStatusEffectModule().applyEffect(poisonConfig);
}
```

### Anti-Patterns (Do NOT do this)
- **Direct Instantiation:** Never use `new EntityEffect()`. This bypasses the asset loading pipeline, resulting in an empty and invalid object that will cause NullPointerExceptions.
- **State Modification:** Do not attempt to modify the fields of an EntityEffect instance at runtime. These objects are shared globally and treated as immutable. Modifying one will have unpredictable side effects across the entire server.
- **Storing Live State:** Do not use this class to store instance-specific data, such as the remaining duration of an effect on a particular entity. This class is a template; live state belongs in a separate component attached to the entity.

## Data Pipeline
The data flow for an EntityEffect begins as a static file and ends as a runtime object used by game systems.

> Flow:
> JSON Asset File -> AssetStore (File Read) -> **EntityEffect.CODEC** (Deserialization) -> **EntityEffect Instance** (In-Memory) -> `afterDecode` Hook (Data Optimization) -> AssetRegistry (Global Storage) -> Game Logic (Runtime Access) -> `toPacket()` (Client Synchronization)

